# Рецепт 18. Фиксация sandbox-отказов в своём приложении

**Цель**: приложение пытается обратиться к файлу/ресурсу, но iOS отвергает по sandbox. Нужно знать — к какому и почему.

## Два подхода

- **Хост:** `log stream --predicate '...'` с фильтром по `sandbox denied` — работает без твика.
- **Твик:** хукнуть POSIX вызовы (`open`, `access`, …) и фиксировать те, что возвращают `EACCES`/`EPERM`.

## Подход A — через log stream (без твика)

На хосте с подключённым устройством:

```bash
log stream --device <udid> --predicate 'subsystem == "com.apple.sandbox.reporting"' --style compact
# или шире:
log stream --device <udid> --predicate 'eventMessage CONTAINS "deny" AND eventMessage CONTAINS "file-read"'
```

При любом sandbox denial увидите строки:

```
default kernel: Sandbox: TestbedApp(1234) deny(1) file-read-data /private/var/mobile/forbidden/path
```

Это **самый простой** путь. Твик не нужен.

## Подход B — твик, если нужен контекст

Иногда нужен детальный контекст: стек вызова, callback-функция, etc. Для этого — хук POSIX.

### Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = SandboxDenialTrace
SandboxDenialTrace_FILES = Tweak.x
SandboxDenialTrace_CFLAGS = -fobjc-arc
SandboxDenialTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

### SandboxDenialTrace.plist

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
</dict>
```

### control

```
Package: com.example.sandboxtrace
Name: SandboxDenialTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Log POSIX calls denied by sandbox in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

### Tweak.x

```objc
#import <Foundation/Foundation.h>
#import <sys/stat.h>
#import <fcntl.h>
#import <unistd.h>
#import <errno.h>
#import <string.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.sandboxtrace", "sandbox");
    os_log(gLog, "SandboxDenialTrace loaded");
}

static inline bool is_sandbox_denial(int ret) {
    if (ret == -1) {
        return errno == EACCES || errno == EPERM;
    }
    return false;
}

%hookf(int, open, const char *path, int flags, ...) {
    mode_t mode = 0;
    if (flags & O_CREAT) {
        va_list ap; va_start(ap, flags);
        mode = va_arg(ap, int); va_end(ap);
    }
    int ret = %orig(path, flags, mode);
    if (is_sandbox_denial(ret)) {
        os_log_error(gLog, "DENIED open(%{public}s, 0x%x) errno=%d (%{public}s)",
                     path, flags, errno, strerror(errno));
    }
    return ret;
}

%hookf(int, access, const char *path, int mode) {
    int ret = %orig(path, mode);
    if (is_sandbox_denial(ret)) {
        os_log_error(gLog, "DENIED access(%{public}s, 0x%x) errno=%d (%{public}s)",
                     path, mode, errno, strerror(errno));
    }
    return ret;
}

%hookf(int, stat, const char *path, struct stat *buf) {
    int ret = %orig(path, buf);
    if (is_sandbox_denial(ret)) {
        os_log_error(gLog, "DENIED stat(%{public}s) errno=%d",
                     path, errno);
    }
    return ret;
}

%hookf(int, chdir, const char *path) {
    int ret = %orig(path);
    if (is_sandbox_denial(ret)) {
        os_log_error(gLog, "DENIED chdir(%{public}s) errno=%d",
                     path, errno);
    }
    return ret;
}

%hookf(ssize_t, readlink, const char *path, char *buf, size_t bufsize) {
    ssize_t ret = %orig(path, buf, bufsize);
    if (ret == -1 && (errno == EACCES || errno == EPERM)) {
        os_log_error(gLog, "DENIED readlink(%{public}s) errno=%d",
                     path, errno);
    }
    return ret;
}

%hookf(int, unlink, const char *path) {
    int ret = %orig(path);
    if (is_sandbox_denial(ret)) {
        os_log_error(gLog, "DENIED unlink(%{public}s) errno=%d",
                     path, errno);
    }
    return ret;
}
```

Разбор:

- `is_sandbox_denial` — фильтр по `EACCES` (13) и `EPERM` (1).
- `os_log_error` — выделяется в лог как ошибка.
- 6 главных POSIX функций. Для полного покрытия можно добавить `fopen`, `rename`, `mkdir`, `chmod`, `chown`.
- `strerror(errno)` — human-readable название ошибки.

### Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

log stream --predicate 'subsystem == "com.example.sandboxtrace"' --device <udid>
```

Триггер — любое приложение-действие с запрещённым путём. В логе:

```
DENIED open(/private/var/mobile/forbidden/file, 0x0) errno=1 (Operation not permitted)
```

## Расширение: stack trace при denial

```objc
#import <execinfo.h>

static void log_backtrace(void) {
    void *frames[32];
    int n = backtrace(frames, 32);
    char **syms = backtrace_symbols(frames, n);
    if (!syms) return;
    for (int i = 0; i < n; i++) {
        os_log(gLog, "  [%d] %{public}s", i, syms[i]);
    }
    free(syms);
}

%hookf(int, open, const char *path, int flags, ...) {
    mode_t mode = 0;
    if (flags & O_CREAT) {
        va_list ap; va_start(ap, flags);
        mode = va_arg(ap, int); va_end(ap);
    }
    int ret = %orig(path, flags, mode);
    if (is_sandbox_denial(ret)) {
        os_log_error(gLog, "DENIED open(%{public}s, 0x%x) errno=%d",
                     path, flags, errno);
        log_backtrace();
    }
    return ret;
}
```

Выдаст call stack. В логах → `atos` для символизации.

## Когда использовать подход A vs B

| Задача | Подход |
|---|---|
| «Что в целом denies приложение?» | A (log stream) |
| «Где в коде именно эта попытка?» | B (твик + backtrace) |
| «Хочу агрегировать по типу denial'а» | B (твик + custom aggregation) |

## См. также

- Глава II.8 — файловая система и контейнеры.
- Шпаргалка 9 — инспекционные команды (log stream).
- Рецепт 1 — logging file ops (похожий паттерн, без фильтра на error).
