# Рецепт 11. Логирование всех `dlsym` в своём процессе

**Цель**: увидеть, какие символы запрашивает ваше тестовое приложение (помогает понять, что оно подгружает динамически).

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = DlsymTrace
DlsymTrace_FILES = Tweak.x
DlsymTrace_CFLAGS = -fobjc-arc
DlsymTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## DlsymTrace.plist

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
</dict>
```

## control

```
Package: com.example.dlsymtrace
Name: DlsymTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace dlsym calls in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## Tweak.x

```objc
#import <Foundation/Foundation.h>
#import <dlfcn.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.dlsymtrace", "dl");
    os_log(gLog, "DlsymTrace loaded");
}

%hookf(void *, dlsym, void *handle, const char *name) {
    void *result = %orig(handle, name);
    os_log(gLog, "dlsym(handle=%p, name=%{public}s) = %p",
           handle, name ?: "<null>", result);
    return result;
}

%hookf(void *, dlopen, const char *path, int mode) {
    void *result = %orig(path, mode);
    os_log(gLog, "dlopen(path=%{public}s, mode=0x%x) = %p",
           path ?: "<null>", mode, result);
    return result;
}

%hookf(int, dlclose, void *handle) {
    int result = %orig(handle);
    os_log(gLog, "dlclose(%p) = %d", handle, result);
    return result;
}
```

Разбор:

- `dlsym`/`dlopen`/`dlclose` — все три главные функции `<dlfcn.h>`.
- `handle` — `RTLD_DEFAULT`, `RTLD_SELF`, `RTLD_NEXT` или opaque указатель от `dlopen`.
- `name` — символ (без underscore prefix).
- `mode` — `RTLD_NOW` / `RTLD_LAZY` / `RTLD_LOCAL` / `RTLD_GLOBAL`.

## Нюанс: `__attribute__((musttail))`

Если ваш хук идёт на `dlsym` и приложение использует `__builtin_return_address(0)` в вызываемой функции (для определения caller), standard подход сломает trace stack:

```objc
%hookf(void *, dlsym, void *handle, const char *name) {
    void *result = %orig(handle, name);
    os_log(gLog, "dlsym(%p, %s) = %p", handle, name, result);
    return result;
}
```

Это норма для диагностики. Для «прозрачного» хука (не меняющего return address) — см. технику `musttail` в главе III.5 (там это используется в systemhook).

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

log stream --predicate 'subsystem == "com.example.dlsymtrace"' --device <udid>
```

При старте приложения увидите **много** вызовов — Foundation, UIKit, CoreFoundation все подгружают символы динамически. Это нормально.

## Фильтр: только интересное

Логов будет тысячи. Отфильтруйте:

```objc
%hookf(void *, dlsym, void *handle, const char *name) {
    void *result = %orig(handle, name);
    // Логируем только символы, содержащие "interesting":
    if (name && strcasestr(name, "interesting")) {
        os_log(gLog, "dlsym: %{public}s = %p", name, result);
    }
    return result;
}
```

Или по handle:

```objc
if (handle == RTLD_DEFAULT || handle == RTLD_NEXT) {
    os_log(gLog, "dlsym (default/next): %{public}s", name);
}
```

## См. также

- Deep-dive 02 — dyld 4+ и chained fixups.
- Глава IV.5 — перехват C-функций.
- Рецепт 12 — трейс `+load`/`-init`.
