# Глава IV.5. Перехват C-функции

Objective-C swizzle через `%hook` покрывает большинство UIKit/Foundation задач, но примерно четверть реальной работы в твике — это **C-функции**: `open`, `stat`, `access`, `fopen`, `dlopen`, `dlsym`, `fcntl`, `setuid`, `uname`, функции CoreFoundation, Security framework. Для них нужен другой инструмент — fishhook или ellekit API. В этой главе — как это работает и когда что применять.

## 1. Почему `%hook` не работает для C

Objective-C: вызов `[obj method]` компилируется в `objc_msgSend(obj, @selector(method))`. Эта функция читает **таблицу методов класса** в рантайме. Поменяв запись в таблице (`method_setImplementation`), мы меняем поведение всех будущих вызовов.

C: вызов `open(...)` компилируется в прямой branch к адресу функции **или** к записи в GOT/stubs. Таблицы методов нет. Поменять поведение можно только:

1. **Через GOT/stubs** (indirect calls): fishhook подменяет указатель в GOT, после чего все вызовы через GOT идут в вашу функцию.
2. **Inline patch** (direct calls): записать `BR x16` в начало функции, перепрыгивающий на вашу реализацию. Это то, что делает ellekit/substrate под капотом для некоторых случаев.

Для автора твика это означает: не каждый C-вызов можно перехватить, и инструмент надо выбирать.

## 2. Инструменты и их отличия

| Инструмент | Способ | Что перехватывает | Что НЕ перехватывает |
|---|---|---|---|
| **fishhook** | Подмена указателя в GOT (indirect) | Любой вызов функции **из другой dylib** в код вашего процесса | Вызовы внутри одной dylib (direct) |
| **ellekit** (`EKHookFunction`) | Inline hook с trampoline (direct) | Любую функцию, кроме inlined | inlined, static, с особыми PAC-signatures |
| **substrate** `MSHookFunction` | Inline hook (legacy) | Как ellekit, но с меньшей надёжностью на arm64e | — |
| **litehook** | Inline hook (внутренний для Dopamine) | Используется внутри Dopamine, не публично | — |
| **Logos** `%hookf` | Обёртка над одним из вышеперечисленных | Задаётся runtime | — |

**Для твиков на RootHide Dopamine рекомендован ellekit** через `%hookf`. Это даёт синтаксис Logos и backend ellekit.

## 3. `%hookf` — Logos для C-функций

Шаблон:

```objc
%hookf(ReturnType, functionName, Arg1Type arg1, Arg2Type arg2) {
    // тело хука
    return %orig(arg1, arg2);
}
```

Пример — перехват `access(2)`:

```objc
#import <Foundation/Foundation.h>
#import <unistd.h>
#import <os/log.h>

static os_log_t gLog;

%ctor { gLog = os_log_create("com.example.fshook", "access"); }

%hookf(int, access, const char *path, int mode) {
    int ret = %orig(path, mode);
    os_log(gLog, "access(%{public}s, 0x%x) = %d", path, mode, ret);
    return ret;
}
```

Разбор:

- `%hookf` — макрос Logos для C-функций.
- `int, access, const char *path, int mode` — тип возврата, имя функции, типизированные параметры.
- `%orig(path, mode)` — вызов оригинала.
- `os_log` — логирование результата.

За сценой Logos:

1. Декларирует функцию `_logos_orig$access` с той же сигнатурой.
2. При инициализации (`%init`) вызывает ellekit API, чтобы установить ваш хук + получить оригинал в `_logos_orig$access`.
3. Все вызовы `access(...)` из вашего процесса (и косвенно — из других dylibs, если они вызывают через GOT) идут в ваш хук.

## 4. Полный проект-пример: file-system tracer

Задача: твик, логирующий `access`, `open`, `stat` в своём приложении.

### 4.1. Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = testbed
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = FSTracer

FSTracer_FILES = Tweak.x
FSTracer_CFLAGS = -fobjc-arc
FSTracer_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

### 4.2. control

```
Package: com.example.fstracer
Name: FSTracer
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace file-system calls in my app
Maintainer: Me
Author: Me
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

### 4.3. FSTracer.plist

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

### 4.4. Tweak.x

```objc
#import <Foundation/Foundation.h>
#import <os/log.h>
#import <sys/stat.h>
#import <fcntl.h>
#import <unistd.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.fstracer", "fs");
    os_log(gLog, "FSTracer loaded in %{public}@",
           [NSProcessInfo processInfo].processName);
}

%hookf(int, access, const char *path, int mode) {
    int ret = %orig(path, mode);
    os_log(gLog, "access(%{public}s, 0x%x) = %d (errno=%d)",
           path, mode, ret, errno);
    return ret;
}

%hookf(int, open, const char *path, int flags, ...) {
    mode_t mode = 0;
    if (flags & O_CREAT) {
        va_list ap;
        va_start(ap, flags);
        mode = va_arg(ap, int);
        va_end(ap);
    }
    int ret = %orig(path, flags, mode);
    os_log(gLog, "open(%{public}s, 0x%x, 0%o) = %d", path, flags, mode, ret);
    return ret;
}

%hookf(int, stat, const char *path, struct stat *buf) {
    int ret = %orig(path, buf);
    os_log(gLog, "stat(%{public}s, %p) = %d (size=%lld)",
           path, buf, ret, (ret == 0) ? (long long)buf->st_size : -1LL);
    return ret;
}
```

Разбор ключевых решений.

**Vararg `open`**. Функция `open(2)` имеет две формы:

- `int open(const char *path, int flags)` — 2 аргумента.
- `int open(const char *path, int flags, mode_t mode)` — 3 аргумента, если `O_CREAT` указан.

Для безопасности мы объявляем vararg (`...`) и разбираем `mode` вручную только при `O_CREAT`. Иначе `va_arg` для несуществующего аргумента — undefined behavior.

**`errno` в логе `access`**. Если `access` вернул -1, errno указывает причину (ENOENT, EACCES, …). Полезно для диагностики.

**`stat` size**. Логируем размер файла при успешном stat. Это демонстрирует доступ к out-полю `struct stat`.

### 4.5. Сборка и установка

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/com.example.fstracer_*.deb"
```

На устройстве запускаем testbed, в `log stream` видим:

```
FSTracer loaded in testbed
stat(/var/mobile/Containers/Data/Application/<UUID>/Library/Preferences/com.example.testbed.plist, 0x16d...) = 0 (size=432)
access(/private/var/mobile/.../somefile, 0x4) = 0 (errno=0)
open(/var/mobile/..., 0x1, 0) = 7
```

**Это работающий tracer.** Та же схема расширяется на `fopen`, `fstat`, `lstat`, `unlink`, `rename`, `chmod`, `fcntl` и десятки других POSIX функций.

## 5. fishhook напрямую (без Logos)

Если по какой-то причине Logos не подходит (например, пишете чисто-C dylib без Theos), используйте fishhook напрямую.

Пример эквивалентного кода:

```c
#include <stdio.h>
#include <unistd.h>
#include "fishhook.h"

static int (*orig_access)(const char *, int);

static int my_access(const char *path, int mode) {
    int ret = orig_access(path, mode);
    fprintf(stderr, "access(%s, 0x%x) = %d\n", path, mode, ret);
    return ret;
}

__attribute__((constructor))
static void init(void) {
    struct rebinding r = { "access", (void *)my_access, (void **)&orig_access };
    rebind_symbols(&r, 1);
}
```

Разбор:

- `struct rebinding` — одна запись «что перехватить, чем, куда сохранить оригинал».
- `rebind_symbols` — API fishhook.
- `orig_access` будет указывать на реальную `access` после вызова.

Для многих символов — массив:

```c
struct rebinding rebindings[] = {
    { "access", (void *)my_access, (void **)&orig_access },
    { "open",   (void *)my_open,   (void **)&orig_open   },
};
rebind_symbols(rebindings, sizeof(rebindings)/sizeof(rebindings[0]));
```

Fishhook работает **только через GOT/binds**, т. е. перехватывает только межмодульные вызовы. Если `.dylib` вашего приложения внутри сама себя зовёт `access(...)` — fishhook это не поймает. Для таких случаев нужен inline hook (ellekit `EKHookFunction`).

## 6. ellekit API напрямую

Для inline hook без Logos:

```c
#include <stdio.h>
#include <ellekit.h>

static int (*orig_access)(const char *, int);
static int my_access(const char *path, int mode) {
    int ret = orig_access(path, mode);
    fprintf(stderr, "access(%s) = %d\n", path, ret);
    return ret;
}

__attribute__((constructor))
static void init(void) {
    EKHookFunction((void *)&access, (void *)&my_access, (void **)&orig_access);
}
```

`EKHookFunction` создаёт trampoline, записывает `BR x16` в начало настоящей `access`, восстанавливает прологовые инструкции в trampoline. Работает и для direct, и для indirect вызовов, но дороже в терминах настройки.

**Для 95% случаев Logos `%hookf` достаточно** — он использует правильный backend автоматически.

## 7. Таблица «какой инструмент для какого случая»

| Задача | Инструмент |
|---|---|
| Хукнуть `open/stat/access` в собственном приложении | `%hookf` |
| Хукнуть `objc_msgSend` | **НЕ делать** (ломает runtime, но технически — ellekit) |
| Хукнуть вызов внутри dylib самой на себя (static dispatch) | ellekit `EKHookFunction` |
| Хукнуть символ, экспортируемый вашей dylib другим процессам | `%hookf` |
| Логировать dlopen/dlsym (без рекурсии) | `%hookf` с `__attribute__((musttail))` трюком (см. systemhook/main.c) |
| Хукнуть syscall напрямую (минуя libSystem) | Невозможно — syscall идёт через `svc`-инструкцию, не через функцию |

## 8. Рекурсия — самая частая ошибка

Представьте такой хук:

```objc
%hookf(int, access, const char *path, int mode) {
    NSLog(@"access(%s)", path);   // ОПАСНО
    return %orig(path, mode);
}
```

Проблема: `NSLog` внутри делает `access(...)` при проверке лог-файла. Ваш хук перехватывает этот `access`, снова вызывает `NSLog`, снова `access`, и так до переполнения стека.

Решения:

**Решение 1**: использовать `os_log` вместо `NSLog`. `os_log` реализован через mach messaging, не делает file I/O.

**Решение 2**: thread-local recursion guard:

```objc
#import <pthread.h>

static pthread_key_t gReentrancyKey;
%ctor {
    pthread_key_create(&gReentrancyKey, NULL);
}

%hookf(int, access, const char *path, int mode) {
    if (pthread_getspecific(gReentrancyKey)) {
        return %orig(path, mode);  // в рекурсии — просто пропускаем лог
    }
    pthread_setspecific(gReentrancyKey, (void *)1);

    int ret = %orig(path, mode);
    NSLog(@"access(%s) = %d", path, ret);

    pthread_setspecific(gReentrancyKey, NULL);
    return ret;
}
```

Этот паттерн защищает от рекурсии, но замусоривает код. **Рекомендация: используйте `os_log`**, и 95% случаев рекурсии вас не коснутся. Расширенно про рекурсию — глава IV.6.

## 9. `__attribute__((musttail))` — тонкий трюк

Из главы III.5 помним `dyld_dlsym_hook` в systemhook:

```c
void *dyld_dlsym_hook(void *dyld, void *handle, const char *name)
{
    if (handle == gLibSandboxHandle && !strcmp(name, "sandbox_apply")) {
        return sandbox_apply_hook;
    }
    __attribute__((musttail)) return dyld_dlsym_orig(dyld, handle, name);
}
```

Что делает `musttail`:

- Компилятор обязан превратить вызов `dyld_dlsym_orig(...)` в tail-call — прямой `BR` вместо `BL`.
- Регистры (включая LR) не сохраняются; `__builtin_return_address(0)` в оригинале вернёт **адрес, из которого позвали dyld_dlsym_hook**, а не адрес внутри хука.

Для автора твика это важно, когда:

- Хукаете функцию, которая внутри вызывает `__builtin_return_address` или подобное.
- Хотите сделать хук **прозрачным** в терминах stack trace.

Если ваш хук просто логирует и вызывает оригинал — можно без `musttail`. Если хук в середине цепочки `dlopen`-подобных вещей — лучше с `musttail`.

## 10. Какие C-функции осмысленно хукать

| Категория | Примеры | Когда полезно |
|---|---|---|
| **File I/O** | `open`, `stat`, `access`, `fopen`, `unlink`, `rename` | Трейсер поведения приложения |
| **Dynamic loading** | `dlopen`, `dlsym` | Увидеть, какие dylibs ваше приложение подгружает |
| **Process control** | `fork`, `posix_spawn`, `execve` | На iOS редко применимо |
| **Networking** | `connect`, `send`, `recv` | **Избегайте** — очень легко уйти в системные пути (см. главу II.9) |
| **CoreFoundation** | `CFPreferencesCopyAppValue`, `CFBundleCreate` | Перехват настроек/бандлов |
| **Security** | `SecItemCopyMatching`, `SecTrustEvaluate` | Pinning bypass в своём app |
| **Mach** | `mach_msg`, `task_for_pid` | **Избегайте** — Dopamine сам этим занимается |

Правило: **чем выше уровень, тем лучше**. Хукать `URLSession` безопаснее, чем `connect`. Хукать `CFPreferencesCopyAppValue` безопаснее, чем `open` на plist-файл.

## 11. Что подтверждено

- **Подтверждено практикой проекта**: `litehook_hook_function` и `litehook_rebind_symbol` — внутренние API Dopamine (`BaseBin/launchdhook/src/jetsam_hook.c:24–27` для примера).
- **Подтверждено примером из systemhook**: `__attribute__((musttail))` для `dyld_dlsym_hook` — `BaseBin/systemhook/src/main.c:107`.
- **Подтверждено по проекту**: ellekit — рекомендован как backend для твиков в составе RootHide Dopamine (пакет `ellekit` в зависимостях твиков).
- **Требует проверки на устройстве**: что `%hookf` ваш backend именно ellekit, а не другой — зависит от установленного менеджера хуков.
- **Не подтверждено**: что все внутренние direct-call пути в iOS 26 стабильны и хукаются одинаково — Apple периодически меняет inlining.

## 12. Выводы

Главный тезис: **для C-функций есть свой инструмент — `%hookf` + ellekit под капотом**. Это не альтернатива `%hook` для ObjC, а параллельный путь для случаев, когда вам нужны `open`/`stat`/`SecItem*` или подобное.

Три правила на всю жизнь:

1. Хукайте максимально высокий уровень — ObjC если можно, иначе высокоуровневое C API.
2. Используйте `os_log` в логах из C-хуков — он не даёт рекурсии.
3. Не хукайте низкоуровневые syscall-функции системно — Dopamine сам этим занят на вашу пользу.

## 13. Что дальше

- `%orig`, рекурсия, разделяемое состояние — глава IV.6 (расширение раздела 8 этой главы).
- Потоки, ARC, retain cycles — глава IV.7.
- Шпаргалка по синтаксису Logos (с `%hookf`) — `cheatsheets/01-logos-syntax.md`.
- Рецепт полного fs-tracer с ротацией лога и отключением — Cookbook (планируется).
