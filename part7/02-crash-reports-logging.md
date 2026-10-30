# Глава VII.2. Crash reports и полезное логирование

Крэш — это информация. `.ips` файл даёт вам stack trace, список загруженных dylib, точный offset в вашем коде. В этой главе — как читать `.ips`, как символизировать back trace к строкам, и как писать логи так, чтобы они помогали, а не замусоривали систему.

## 1. Где лежат crash reports

Два места:

- `/var/mobile/Library/Logs/CrashReporter/` — пользовательские приложения.
- `/var/mobile/Library/Logs/CrashReporter/Analytics/` — агрегированные (на старых iOS).
- Для rootless/RootHide + crash reporter самого Dopamine — также `$(jbroot)/private/var/mobile/Library/Logs/CrashReporter/` (глава III.3 `libroothider/crashreporter.m`).

Через SSH:

```bash
ssh mobile@<ip>
ls -t /var/mobile/Library/Logs/CrashReporter/ | head -20
# TargetApp-2026-10-30-100012.ips
# SpringBoard-2026-10-30-095530.ips
# ...
```

Самые свежие — первые.

## 2. Формат `.ips` — JSON, несмотря на вид

На iOS 15+ все `.ips` — JSON (хоть выглядят как многострочный текст, потому что JSON-поля перенесены на новые строки).

### 2.1. Как увидеть структуру

```bash
cat /var/mobile/Library/Logs/CrashReporter/TargetApp-....ips | head -2
# Первая строка — meta JSON (одна строка)
# Со второй строки — ещё один JSON (длинный, многострочный)
```

Расщепить правильно:

```bash
head -1 TargetApp-....ips | python3 -m json.tool
tail -n +2 TargetApp-....ips | python3 -m json.tool
```

Первый JSON — метаданные (`bug_type`, `timestamp`, `incident_id`). Второй — тело отчёта.

### 2.2. Ключевые поля тела

| Поле | Что значит |
|---|---|
| `procName` | Имя упавшего процесса (`SpringBoard`, `TargetApp`) |
| `pid` | PID на момент краха |
| `cpuType` | `ARM-64` для A13 |
| `osVersion.train` / `.build` | «iOS 26.0.1» / `23B5064e` |
| `faultingThread` | Индекс треда, вызвавшего краш |
| `threads` | Массив тредов |
| `threads[i].frames` | Stack trace треда i |
| `usedImages` | Список всех загруженных dylib + их address range |
| `termination.indicator` | Тип завершения («Segmentation fault: 11», «Namespace: SIGNAL») |
| `exception` | Детали исключения (NSException, если было) |
| `lastExceptionBacktrace` | Отдельный back trace для ObjC exception |

## 3. Чтение back trace

### 3.1. Простой случай

Пример:

```json
{
  "faultingThread": 0,
  "threads": [{
    "triggered": true,
    "name": "main",
    "frames": [
      { "imageIndex": 5, "imageOffset": 2048, "symbol": "-[UIView setHidden:]", "symbolLocation": 32 },
      { "imageIndex": 12, "imageOffset": 1024 }
    ]
  }],
  "usedImages": [
    { "name": "UIKit", "base": 4299948032, "path": "/System/Library/..." },
    ...
    { "name": "MyTweak.dylib", "base": 7000000000, "path": "/var/jb/..." }
  ]
}
```

Разбор:

- Trigger thread — index 0 (`main`).
- Frame 0: `imageIndex: 5` → `usedImages[5]` → UIKit; `imageOffset: 2048` + `base` = полный адрес.
- Frame 1: `imageIndex: 12` → MyTweak.dylib; `imageOffset: 1024`.

Фрейм 1 — ваш код. Нужно узнать, какая функция/строка соответствует `imageOffset: 1024`.

### 3.2. Символизация

На хосте, где лежит ваша dylib с debug symbols:

```bash
atos -o .theos/obj/MyTweak.dylib -arch arm64 -l 0 0x400
# или с загрузочным адресом:
# atos -o <dylib> -arch arm64 -l <base> <addr>
```

- `-o`: путь к бинарнику.
- `-arch`: архитектура.
- `-l 0`: load address = 0 (мы используем offset).
- `0x400`: imageOffset в hex (1024).

Выдаст:

```
-[MyHookClass hookedMethod:] (in MyTweak.dylib) (Tweak.x:42)
```

Готово — крах в вашем файле `Tweak.x`, строка 42.

### 3.3. Если нет debug symbols

При `FINALPACKAGE=1` Theos делает `strip` на dylib — символов нет.

Решения:

- **Пересобрать без FINALPACKAGE** (debug build), запустить тот же сценарий, получить новый крах, атос работает.
- **Сохраняйте debug build** параллельно с release. Один для устройства, другой для символизации.
- **dSYM**: Theos не создаёт dSYM по умолчанию, но можно добавить `TWEAK_NAME_DEBUG_INFORMATION_FORMAT = dwarf-with-dsym` в Makefile.

## 4. Типы крешей

### 4.1. SIGSEGV / Segmentation fault

Доступ к невалидной памяти. Обычно — nil pointer dereference, use-after-free, запись в read-only.

```json
"termination": { "indicator": "Segmentation fault: 11", ...}
```

Fix: проверка на nil, retain/release, не писать в const.

### 4.2. SIGABRT / abort

Вызов `abort()` — обычно `NSInternalInconsistencyException` или `__assert_fail`.

```json
"termination": { "indicator": "SIGABRT (6)", ...},
"exception": { "type": "EXC_CRASH", ...},
"lastExceptionBacktrace": [...]
```

Fix: читать `lastExceptionBacktrace` и смотреть, какой `NSException` был брошен.

### 4.3. SIGKILL — Jetsam

iOS убила процесс за превышение памяти:

```json
"termination": { "indicator": "Namespace: SIGNAL", "code": 9, "byPid": 1, ...},
"jetsamData": {...}
```

Fix: уменьшить использование памяти в твике, не держать большие caches.

### 4.4. Watchdog kill

iOS убила процесс за блокирование main thread:

```json
"termination": { "indicator": "Namespace: WATCHDOG", ...}
```

Fix: не делать I/O/сеть на main thread.

### 4.5. EXC_BAD_ACCESS

Низкоуровневое — ARM read невалидного адреса:

```json
"exception": { "type": "EXC_BAD_ACCESS", "codes": "KERN_INVALID_ADDRESS", ...}
```

Fix: как SIGSEGV.

## 5. ObjC exception backtrace

Если крах — через `NSException`, он отдельно сохраняется:

```json
"lastExceptionBacktrace": [
  { "imageIndex": 1, "imageOffset": 100, "symbol": "__exceptionPreprocess" },
  { "imageIndex": 2, "imageOffset": 200, "symbol": "objc_exception_throw" },
  { "imageIndex": 15, "imageOffset": 500, "symbol": "-[MyClass checkThing]" },
  ...
]
```

Читается тем же `atos`, но trigger — обычно 3-й или 4-й фрейм снизу (первые — внутренности exception runtime).

Если видите в exception description:

```json
"exception": {
  "type": "EXC_CRASH",
  "signal": "SIGABRT",
  "subtype": "Namespace: ...",
  "description": "*** -[__NSArrayI objectAtIndex:]: index 5 beyond bounds [0 .. 2]"
}
```

Это ваше сообщение — array index out of bounds.

## 6. Полезное логирование — правила

### 6.1. Используйте `os_log` вместо `NSLog`

```objc
// Нет:
NSLog(@"hook called with %@", obj);

// Да:
static os_log_t gLog;
%ctor { gLog = os_log_create("com.example.mytweak", "events"); }

os_log(gLog, "hook called with %{public}@", obj);
```

Почему `os_log`:

- Thread-safe.
- Не блокирует (кладёт в кольцевой буфер).
- Не делает file I/O (не задевает fileExistsAtPath).
- Поддерживает levels (debug/info/notice/error/fault).
- Фильтруется на хосте через `log stream`.

### 6.2. Levels

```objc
os_log_debug(gLog, "...");     // debug — не отображается по умолчанию
os_log_info(gLog, "...");       // info — не отображается по умолчанию
os_log(gLog, "...");            // default
os_log_error(gLog, "...");      // error
os_log_fault(gLog, "...");      // fault — высший уровень
```

На устройстве по умолчанию видны только default/error/fault. Чтобы увидеть debug/info:

```bash
# На устройстве:
sudo log config --mode "level:debug" --subsystem "com.example.mytweak"
# Теперь все levels видны
```

### 6.3. `{public}` vs `{private}`

```objc
os_log(gLog, "path = %{public}@", path);    // показывается
os_log(gLog, "path = %{private}@", path);   // <private> в логе
os_log(gLog, "path = %@", path);            // то же что {private} — скрыто
```

На устройстве:

- `{public}` — всегда видно.
- `{private}` — `<private>` по умолчанию, видно только через `sudo log config --private-data-enabled`.

Для твика в собственном приложении — `{public}` ОК. Для твика в чужом приложении — осторожно с чувствительными данными.

### 6.4. Форматы

| Format | Для |
|---|---|
| `%{public}@` | NSString / NSObject с description |
| `%{public}s` | C-string |
| `%d` | int |
| `%lld` / `%ld` | long long / long |
| `%p` | pointer |
| `%.3f` | float with precision |
| `%zu` | size_t |
| `%x` / `%llx` | hex |

### 6.5. Подсистемы и категории

Структура:

```objc
os_log_create("com.example.mytweak", "lifecycle");    // ctor, dealloc
os_log_create("com.example.mytweak", "io");            // file ops
os_log_create("com.example.mytweak", "net");           // network
```

Разделение облегчает фильтрацию:

```bash
log stream --predicate 'subsystem == "com.example.mytweak" AND category == "io"'
```

## 7. Собственный файловый логгер

Если `os_log` не достаточно (например, нужен persistent log в контейнере приложения), используйте POSIX-based logger — пример из главы IV.13 (FSTrace).

Ключевые моменты:

- `open` + `write` + `close` — только POSIX, без Foundation.
- `pthread_mutex_t` для synchronization.
- `O_APPEND` для безопасной конкуренции.
- Ротация — `fstat` + reopen.

Полный пример:

```c
#include <sys/stat.h>
#include <sys/syscall.h>

typedef struct {
    int fd;
    pthread_mutex_t lock;
    size_t maxSize;
} FileLogger;

static FileLogger gFL = { -1, PTHREAD_MUTEX_INITIALIZER, 10 * 1024 * 1024 };

static void fl_init(FileLogger *fl, const char *path) {
    fl->fd = open(path, O_WRONLY | O_CREAT | O_APPEND, 0644);
}

static void fl_log(FileLogger *fl, const char *fmt, ...) {
    if (fl->fd < 0) return;

    char buf[1024];
    va_list ap;
    va_start(ap, fmt);
    int len = vsnprintf(buf, sizeof(buf), fmt, ap);
    va_end(ap);
    if (len <= 0) return;

    pthread_mutex_lock(&fl->lock);
    (void)write(fl->fd, buf, (len < (int)sizeof(buf)) ? len : (int)sizeof(buf) - 1);
    // Ротация
    struct stat st;
    if (fstat(fl->fd, &st) == 0 && st.st_size > (off_t)fl->maxSize) {
        close(fl->fd);
        // переоткрыть (стирая)
        fl->fd = open("/path/to/log", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    }
    pthread_mutex_unlock(&fl->lock);
}
```

## 8. Как собрать `.ips` с устройства

### 8.1. Через Settings (без SSH)

Settings → Privacy & Security → Analytics & Improvements → Analytics Data → фильтр по имени процесса.

На устройстве можно переслать `.ips` файлом через AirDrop или скопировать в Files.app.

### 8.2. Через SSH

```bash
scp mobile@<ip>:/var/mobile/Library/Logs/CrashReporter/TargetApp-*.ips ./crash-dumps/
```

### 8.3. Через libimobiledevice (без jailbreak даже)

```bash
idevicecrashreport -k ./crash-dumps/
```

Копирует все crash reports с устройства.

## 9. Пример: диагностика конкретного крэша

Сценарий: твик `MyTweak` установлен, при запуске `TargetApp` — немедленный крэш.

### 9.1. Шаги

1. Получить `.ips`:

```bash
scp mobile@<ip>:/var/mobile/Library/Logs/CrashReporter/TargetApp-*.ips /tmp/
```

2. Прочитать tail:

```bash
tail -n +2 /tmp/TargetApp-*.ips | python3 -m json.tool | less
```

3. Найти `faultingThread`:

```
"faultingThread": 0,
```

4. Прочитать его `frames`:

```json
"threads": [{
  "triggered": true,
  "frames": [
    {"imageIndex": 2, "imageOffset": 1024, "symbol": "objc_msgSend"},
    {"imageIndex": 42, "imageOffset": 2000}
  ]
}]
```

5. Проверить `usedImages[42]`:

```
"usedImages": [..., {"name": "MyTweak.dylib", "base": ..., "path": ".../MyTweak.dylib"}]
```

6. Атос:

```bash
atos -o .theos/obj/MyTweak.dylib -arch arm64 -l 0 0x7d0
# -[MyHookClass setupHooks] (Tweak.x:67)
```

7. Открыть `Tweak.x:67`, увидеть:

```objc
[someObject performSelector:@selector(doStuff)];
```

`someObject` — nil. Понятная причина.

8. Фикс:

```objc
if (someObject) {
    [someObject performSelector:@selector(doStuff)];
}
```

Весь цикл — 10 минут от crash до фикса.

## 10. Таблица «симптом → раздел `.ips` → действие»

| Симптом | Где искать в `.ips` | Что делать |
|---|---|---|
| Процесс падает при запуске | `faultingThread.frames`, самый ранний ваш фрейм | atos → проверить код `%ctor` |
| ObjC exception | `lastExceptionBacktrace`, `exception.description` | понять exception, фиксить |
| Jetsam kill | `jetsamData` + «SIGKILL» | уменьшить память |
| Watchdog | `Namespace: WATCHDOG` | не блокировать main thread |
| Падает только иногда | много `.ips`, искать паттерн | thread safety |

## 11. Полезное логирование — чеклист

```
[ ] os_log вместо NSLog
[ ] Subsystem = reverse-DNS вашего пакета
[ ] Category = раздел ответственности (lifecycle, io, net, ui)
[ ] Levels (debug/info/default/error/fault) соответствуют важности
[ ] %{public} только для не-чувствительных данных
[ ] В hot-path хуке — минимум логов, или os_log_debug (включается опционально)
[ ] Нет NSLog или printf — не задевает file I/O
[ ] Нет синхронного ожидания — не блокирует main thread
```

## 12. Что подтверждено

- **Подтверждено Apple docs**: `os_log*` family API, `atos` CLI, format `%{public}`.
- **Подтверждено Apple**: `.ips` формат — JSON с iOS 15+.
- **Подтверждено Dopamine**: `libroothider/crashreporter.m` — специальный crash reporter для jailbreak компонентов (глава III.3).
- **Подтверждено практикой**: `idevicecrashreport` из libimobiledevice — стандартный инструмент сбора .ips.
- **Требует проверки на устройстве**: точный путь crash reports в randomized jbroot — через `$(jbroot)/private/var/mobile/Library/Logs/CrashReporter/`.

## 13. Выводы

Главный тезис: **крах — это информация, если вы умеете её читать**. `.ips` + `atos` + ваша debug dylib = точный номер строки, где падает.

Три якорных правила:

1. **Сохраняйте debug build dylib** (не только FINALPACKAGE) — для символизации.
2. **os_log с subsystem/category** — для фильтруемого лога.
3. **Prefer %{public}** в своём приложении, `{private}` — в чужом.

## 14. Что дальше

- Матрица 26.0 vs 26.0.1 — глава VII.3.
- Шпаргалка команд инспекции — `cheatsheets/09-analysis-commands.md`.
- Для cookbook-рецепта «logger с ротацией» — Cookbook.
- Для crash recovery pattern — глава V.5.
