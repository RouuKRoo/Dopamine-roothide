# Рецепт 1. Логирование файловых операций в своём приложении

**Цель**: твик, логирующий вызовы `open()`, `stat()`, `access()` в собственном тестовом приложении.

Это компактная версия учебного проекта №3 (глава IV.13). Полный разбор рекурсии и защиты — там; здесь — готовый код.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = FSTrace
FSTrace_FILES = Tweak.x
FSTrace_CFLAGS = -fobjc-arc
FSTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## control

```
Package: com.example.fstrace
Name: FSTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace file-system calls in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## FSTrace.plist

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

## Tweak.x

```objc
#import <Foundation/Foundation.h>
#import <sys/stat.h>
#import <fcntl.h>
#import <unistd.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.fstrace", "fs");
    os_log(gLog, "FSTrace loaded in %{public}@",
           [NSProcessInfo processInfo].processName);
}

%hookf(int, access, const char *path, int mode) {
    int ret = %orig(path, mode);
    os_log(gLog, "access(%{public}s, 0x%x) = %d", path, mode, ret);
    return ret;
}

%hookf(int, open, const char *path, int flags, ...) {
    mode_t mode = 0;
    if (flags & O_CREAT) {
        va_list ap; va_start(ap, flags);
        mode = va_arg(ap, int); va_end(ap);
    }
    int ret = %orig(path, flags, mode);
    os_log(gLog, "open(%{public}s, 0x%x, 0%o) = %d", path, flags, mode, ret);
    return ret;
}

%hookf(int, stat, const char *path, struct stat *buf) {
    int ret = %orig(path, buf);
    os_log(gLog, "stat(%{public}s) = %d", path, ret);
    return ret;
}
```

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

# Запустить TestbedApp на устройстве, в log stream:
log stream --predicate 'subsystem == "com.example.fstrace"' --device <udid>
```

Ожидаемо:

```
FSTrace loaded in TestbedApp
stat(/var/mobile/.../Preferences, 0x16d...) = 0
access(/private/var/.../file, 0x4) = 0
open(/var/mobile/..., 0x1, 0) = 7
```

## Ключевые решения

- `os_log` вместо `NSLog` — предотвращает рекурсию (NSLog сам делает `access`).
- `%hookf open` vararg — защищаемся от undefined behavior при отсутствии O_CREAT.
- `%{public}s` — чтобы пути не маскировались в системном логе.

## См. также

- Глава IV.13 — расширенная версия с pthread recursion guard.
- Глава IV.5 — перехват C-функций.
- Шпаргалка 1 — Logos synтаксис.
