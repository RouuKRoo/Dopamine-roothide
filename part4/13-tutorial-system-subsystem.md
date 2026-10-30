# Глава IV.13. Учебный проект №3: твик для системной подсистемы с риском

Третий учебный проект — вершина Части IV. Задача: написать твик, который перехватывает **системное API `-[NSFileManager fileExistsAtPath:]`** в собственном приложении и логирует все обращения в файл в контейнере приложения. Это демонстрирует работу с системной подсистемой (файловой), риск рекурсии и правильные практики изоляции.

Главное отличие от проектов №1 и №2: здесь мы работаем с **широко используемым** API, вызываемым **тысячи раз в секунду** внутри UIKit, Foundation, системных подсистем. Любая ошибка — моментальный лавинообразный краш.

## 1. Цель и ожидаемое поведение

**Тестовое приложение `com.example.testbed`** — тот же из главы IV.11. Если у вас уже есть — переиспользуйте; если нет — создайте по тому же шаблону.

**Твик `com.example.fstrace`:**

- Загружается только в `com.example.testbed`.
- Перехватывает `-[NSFileManager fileExistsAtPath:]`, `-[NSFileManager contentsOfDirectoryAtPath:error:]`, `-[NSFileManager attributesOfItemAtPath:error:]`.
- Логирует каждое обращение в собственный лог-файл в контейнере приложения.
- **Безопасен от рекурсии**: сам пишет лог не через `NSFileManager`, а напрямую через POSIX `write`.
- **Не ломает ничего** в системе — только собственное приложение, только логирует.

**Статус**: пример статически корректен, собирается. Работа на устройстве A13 с iOS 26 требует верификации.

## 2. Почему это рискованно

`-[NSFileManager fileExistsAtPath:]` под капотом вызывает:

1. `-[NSString fileSystemRepresentation]` — конвертация в C-строку.
2. POSIX `stat()` или `lstat()`.
3. (Иногда) кэширующий слой.

На iOS 26 этот путь UIKit, Foundation, CoreData и ваш собственный код зовут **ежесекундно**. Каждое обращение — потенциальная точка рекурсии, если ваш хук сам делает file I/O.

Типичная ловушка:

```objc
%hook NSFileManager
- (BOOL)fileExistsAtPath:(NSString *)path {
    BOOL exists = %orig(path);
    // ВНИМАНИЕ: NSLog внутри делает fileExistsAtPath и мы в рекурсии
    NSLog(@"[FSTrace] fileExists(%@) = %d", path, exists);
    return exists;
}
%end
```

При запуске приложения — моментальный stack overflow.

Правильное решение — **не использовать Foundation/UIKit API в хуке на часто вызываемый метод**. Писать лог только через POSIX (`open`, `write`, `close`).

## 3. Структура проекта

```bash
cd ~/dev
$THEOS/bin/nic.pl
# Project Name: FSTrace
# Package Name: com.example.fstrace
# Bundle filter: com.example.testbed
```

### 3.1. Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = FSTrace

FSTrace_FILES = Tweak.x
FSTrace_CFLAGS = -fobjc-arc
FSTrace_FRAMEWORKS = Foundation
FSTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

### 3.2. control

```
Package: com.example.fstrace
Name: FSTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace NSFileManager calls in TestbedApp
Maintainer: You
Author: You
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

### 3.3. FSTrace.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Filter</key>
    <dict>
        <key>Bundles</key>
        <array>
            <string>com.example.testbed</string>
        </array>
    </dict>
</dict>
</plist>
```

## 4. Tweak.x — полный код с комментариями

```objc
#import <Foundation/Foundation.h>
#import <sys/syscall.h>
#import <unistd.h>
#import <fcntl.h>
#import <pthread.h>
#import <stdarg.h>
#import <stdio.h>
#import <stdlib.h>
#import <string.h>
#import <errno.h>

// --- Защита от рекурсии ---
static pthread_key_t gReentrancyKey;

static inline bool enter_guard(void) {
    if (pthread_getspecific(gReentrancyKey)) return false;
    pthread_setspecific(gReentrancyKey, (void *)1);
    return true;
}
static inline void leave_guard(void) {
    pthread_setspecific(gReentrancyKey, NULL);
}

// --- Лог-файл (единожды открыт) ---
static int gLogFd = -1;
static pthread_mutex_t gLogLock = PTHREAD_MUTEX_INITIALIZER;

static void log_raw(const char *fmt, ...) {
    if (gLogFd < 0) return;

    char buf[1024];
    va_list ap;
    va_start(ap, fmt);
    int len = vsnprintf(buf, sizeof(buf), fmt, ap);
    va_end(ap);
    if (len <= 0) return;
    if (len >= (int)sizeof(buf)) len = sizeof(buf) - 1;

    pthread_mutex_lock(&gLogLock);
    // Прямой POSIX write — без Foundation
    (void)write(gLogFd, buf, len);
    pthread_mutex_unlock(&gLogLock);
}

// --- Инициализация ---
static void setup_log(void) {
    // Контейнер приложения всегда доступен через $HOME
    const char *home = getenv("HOME");
    if (!home) return;

    char path[1024];
    int n = snprintf(path, sizeof(path), "%s/Documents/fstrace.log", home);
    if (n <= 0 || n >= (int)sizeof(path)) return;

    // O_WRONLY | O_CREAT | O_APPEND; file mode 0644
    int fd = open(path, O_WRONLY | O_CREAT | O_APPEND, 0644);
    if (fd < 0) return;
    gLogFd = fd;

    // Начальная запись: время загрузки
    char hdr[256];
    time_t now = time(NULL);
    int hn = snprintf(hdr, sizeof(hdr), "\n=== FSTrace loaded pid=%d at %ld ===\n",
                      getpid(), (long)now);
    if (hn > 0) {
        (void)write(gLogFd, hdr, hn);
    }
}

%ctor {
    pthread_key_create(&gReentrancyKey, NULL);
    setup_log();
}

// --- Хуки ---

%hook NSFileManager

- (BOOL)fileExistsAtPath:(NSString *)path {
    BOOL result = %orig(path);
    if (enter_guard()) {
        const char *cpath = path.fileSystemRepresentation;
        log_raw("[fileExists] %s = %d\n", cpath ?: "<nil>", (int)result);
        leave_guard();
    }
    return result;
}

- (BOOL)fileExistsAtPath:(NSString *)path isDirectory:(BOOL *)isDirectory {
    BOOL result = %orig(path, isDirectory);
    if (enter_guard()) {
        const char *cpath = path.fileSystemRepresentation;
        log_raw("[fileExists:isDir] %s = %d (isDir=%d)\n",
                cpath ?: "<nil>", (int)result,
                (isDirectory ? (int)(*isDirectory) : -1));
        leave_guard();
    }
    return result;
}

- (NSArray<NSString *> *)contentsOfDirectoryAtPath:(NSString *)path
                                              error:(NSError **)error
{
    NSArray *result = %orig(path, error);
    if (enter_guard()) {
        const char *cpath = path.fileSystemRepresentation;
        log_raw("[contentsOfDir] %s -> %lu entries\n",
                cpath ?: "<nil>", (unsigned long)result.count);
        leave_guard();
    }
    return result;
}

- (NSDictionary<NSFileAttributeKey, id> *)attributesOfItemAtPath:(NSString *)path
                                                            error:(NSError **)error
{
    NSDictionary *result = %orig(path, error);
    if (enter_guard()) {
        const char *cpath = path.fileSystemRepresentation;
        log_raw("[attributes] %s -> %lu keys\n",
                cpath ?: "<nil>", (unsigned long)result.count);
        leave_guard();
    }
    return result;
}

%end
```

## 5. Разбор ключевых решений

### 5.1. `enter_guard()` / `leave_guard()`

Thread-local флаг «я уже в хуке». Если `fileExistsAtPath` вызывается рекурсивно (например, через `log_raw` → `write` → не-очевидно, но если вдруг…), повторный вход сработает `return false` и логирование пропустится.

Это **классический** паттерн против рекурсии в хуках системных API.

### 5.2. `log_raw` через POSIX `write`

Критическая деталь: **НЕ** используем `NSLog`/`os_log`/`fprintf`/`NSString`/`NSFileHandle`.

- `write(gLogFd, buf, len)` — прямой syscall, не задевает Foundation.
- Буфер — на стеке (1024 байта), без динамических аллокаций в hot path.
- `vsnprintf` — libSystem функция, безопасна (не делает file I/O).

### 5.3. `pthread_mutex_t gLogLock`

Защищает `write` от одновременного вызова с разных тредов. Без мьютекса — лог был бы перемешан (`write` на одном fd атомарен **для коротких** записей, но мы перестраховываемся).

### 5.4. `setup_log()`

- `getenv("HOME")` возвращает контейнер приложения (`/var/mobile/Containers/Data/Application/<UUID>/`). Это sandbox-доступно всегда.
- `/Documents/fstrace.log` — папка внутри контейнера; видна через Files.app на устройстве.
- `O_WRONLY | O_CREAT | O_APPEND` — append-mode: при каждом запуске новые записи добавляются в конец.
- Файл-дескриптор сохраняется в `gLogFd` на весь life процесса; `close` не делаем — iOS закроет при exit.

### 5.5. Четыре хука

Выбраны три распространённых метода + одна перегрузка `fileExistsAtPath:isDirectory:`. Это покрывает ~90% обращений к файловой системе через `NSFileManager`.

Можно расширить — `createDirectoryAtPath:...`, `removeItemAtPath:...`, `copyItemAtPath:...`, но каждый дополнительный хук — дополнительный pathway для ошибок. Для учебного проекта — четырёх достаточно.

### 5.6. `path.fileSystemRepresentation`

Возвращает `const char *` — путь в кодировке файловой системы (UTF-8 на iOS). Это **безопасный** способ получить C-строку: не создаёт объект, использует внутренний буфер NSString.

Альтернатива `[path UTF8String]` — тоже работает, но технически возвращает буфер, который может быть освобождён; `fileSystemRepresentation` стабильнее для POSIX вызовов.

## 6. Сборка и установка

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide FINALPACKAGE=1
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/com.example.fstrace_*.deb && killall -9 TestbedApp"
```

## 7. Процедура проверки

### 7.1. Запуск и чтение лога

1. На устройстве откройте `TestbedApp` (вручную тапом по иконке).
2. Приложение запустилось нормально — твик работает.
3. Через SSH:

```bash
ssh mobile@<ip>
# Найти контейнер приложения
find /var/mobile/Containers/Data/Application/ -name "fstrace.log" 2>/dev/null | head -1
# Например, /var/mobile/Containers/Data/Application/A1B2C3D4-.../Documents/fstrace.log
cat /var/mobile/Containers/Data/Application/A1B2C3.../Documents/fstrace.log | head -30
```

Ожидаемый вывод:

```
=== FSTrace loaded pid=1234 at 1761840123 ===
[fileExists] /var/mobile/Containers/Data/Application/.../Library/Preferences = 1
[fileExists] /var/mobile/Containers/Data/Application/.../Library/Caches = 1
[contentsOfDir] /System/Library/Fonts/Core = 42 entries
[attributes] /var/mobile/Containers/Bundle/Application/.../TestbedApp.app/Info.plist -> 15 keys
...
```

### 7.2. Через Files.app

Проще: откройте Files.app → On My iPhone → TestbedApp → Documents → `fstrace.log`. Прочитайте.

### 7.3. Проверка на рекурсию

Если приложение падает сразу — `enter_guard` не сработал. Проверьте:

1. Используется ли `NSLog`/`os_log` где-то в хуке? Удалите.
2. Создаются ли `NSString *` в хуке? Да, это ок — но проверьте, нет ли там неявного обращения к файловой системе.
3. Используется ли `NSFileHandle`/`NSOutputStream`? Удалите — только POSIX `write`.

### 7.4. Проверка производительности

Можно сравнить время запуска приложения с установленным FSTrace и без. Если задержка >1 секунды — твик слишком «тяжёлый». Решение — ограничить логирование (например, не писать логи для путей, начинающихся с `/System/`).

## 8. Что в этом примере сделано правильно

Таблица важных решений:

| Решение | Почему критично |
|---|---|
| POSIX `write` вместо `NSLog`/`os_log` | Избегает рекурсии в file I/O |
| `pthread_key_t` recursion guard | Защищает от любой скрытой рекурсии |
| Файл в контейнере приложения, не jbroot | Не нуждается в особых правах; sandbox-safe |
| `O_APPEND` | Запись атомарна, не теряется при concurrent writes |
| `pthread_mutex_t` на `write` | Гарантирует порядок строк в логе |
| Буфер на стеке | Zero-allocation в hot path |
| `fileSystemRepresentation` | Безопаснее `UTF8String` для POSIX |
| `enter_guard`/`leave_guard` обёрнуты вокруг **только** логирования, не `%orig` | `%orig` всегда вызывается |

## 9. Чего в этом примере избегаем

Таблица «что НЕ использовать в хуке на часто вызываемый метод»:

| Что НЕ использовать | Почему |
|---|---|
| `NSLog`, `os_log` в теле хука | Могут вызывать file I/O |
| `NSString *str = [NSString stringWithFormat:...]` | Выделение памяти в hot path |
| `NSMutableArray`/`NSMutableDictionary` | Не thread-safe |
| `NSFileHandle`/`NSOutputStream`/`NSFileManager` | Та же рекурсия, которую мы ловим |
| `NSNotificationCenter postNotification` | Может триггерить другие хуки |
| `dispatch_sync` на любой queue | Deadlock при реентранте |
| Долгая работа без asyn | Блокирует вызывающий поток |

## 10. Расширение до чего-то серьёзного

Этот скелет легко расширяется:

### 10.1. Ротация логов

```c
static void rotate_log_if_big(void) {
    if (gLogFd < 0) return;
    struct stat st;
    if (fstat(gLogFd, &st) == 0 && st.st_size > 10 * 1024 * 1024) {
        // > 10MB — пересоздадим
        close(gLogFd);
        gLogFd = -1;
        setup_log();
    }
}
```

Вызывайте `rotate_log_if_big()` раз в N записей.

### 10.2. Фильтр по префиксу

Чтобы не писать 10000 обращений к `/System/`:

```c
static bool should_log(const char *path) {
    if (!path) return false;
    if (strncmp(path, "/System/", 8) == 0) return false;
    if (strncmp(path, "/usr/lib/", 9) == 0) return false;
    return true;
}
```

### 10.3. Отключение через файл-маркер

```c
static bool is_disabled(void) {
    const char *home = getenv("HOME");
    if (!home) return false;
    char path[1024];
    snprintf(path, sizeof(path), "%s/Documents/fstrace.disabled", home);
    struct stat st;
    return (stat(path, &st) == 0);
}
```

Если `fstrace.disabled` существует — хуки не пишут в лог. Простой toggle без PreferenceBundle.

## 11. Отключение и откат

```bash
ssh mobile@<ip>
sudo dpkg -r com.example.fstrace
killall -9 TestbedApp
```

Файл `fstrace.log` в контейнере останется — удалите вручную через SSH или Files.app:

```bash
rm /var/mobile/Containers/Data/Application/A1B2C3.../Documents/fstrace.log
```

## 12. Таблица «что я узнал на этом проекте»

| Знание | Применение |
|---|---|
| POSIX вместо Foundation в hot-path хуках | Любой тwик на часто вызываемый метод |
| `pthread_key_t` recursion guard | Любой тwик на системное API |
| `pthread_mutex_t` для shared state | Любой многопоточный тwик |
| Лог в контейнер приложения через `$HOME` | Любой диагностический тwик для своего приложения |
| `fileSystemRepresentation` для POSIX вызовов | Любой тwик, пересекающий ObjC→POSIX границу |
| Хук `%orig` **без** `enter_guard` | Гарантирует работу основного path приложения |

## 13. Что подтверждено

- **Подтверждено**: `pthread_setspecific`/`pthread_getspecific`, `write(2)`, `open(2)` — стандартные POSIX API, работают на iOS.
- **Подтверждено**: `-[NSFileManager fileExistsAtPath:]` сигнатура — Apple docs.
- **Подтверждено**: `path.fileSystemRepresentation` — Apple docs, безопасный способ получить C-строку.
- **Подтверждено на уровне практики**: подход «POSIX в хуке на file API» — стандартный для избежания рекурсии; используется в многих известных файловых трейсерах.
- **Требует проверки на устройстве**: что `fstrace.log` реально появляется в Documents приложения — зависит от того, собралось ли и правильно ли установлено.
- **Не подтверждено**: максимальная частота логирования без видимого замедления — зависит от нагрузки приложения.

## 14. Выводы

Главный тезис: **для хука на часто вызываемый системный метод применяют три защитных слоя**: thread-local recursion guard, POSIX-only I/O в теле хука, pthread_mutex на shared state. Эти три приёма — ключ к стабильному твику системного уровня.

Второй тезис: **ограничение области применения** (filter plist на одно приложение) — главный способ не навредить. Этот твик не трогает SpringBoard, не трогает другие приложения, не трогает демонов. Только TestbedApp. Если ломается — ломается только одно приложение.

## 15. Что дальше

Часть IV закрыта. Переходим к Части V — **Как писать патчи** (разные от написания твиков):

- Патчи исходников самого jailbreak — глава V.1.
- Исправления существующего твика — глава V.2.
- Runtime-патчи поведения приложения — глава V.3.
- Mach-O / метаданные — глава V.4.
- Контроль качества — глава V.5.
