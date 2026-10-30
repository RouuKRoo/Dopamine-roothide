# Рецепт 3. Перехват доступа к pasteboard

**Цель**: логировать все вызовы `UIPasteboard` в собственном тестовом приложении — чтение, запись, items.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = PBTrace
PBTrace_FILES = Tweak.x
PBTrace_CFLAGS = -fobjc-arc
PBTrace_FRAMEWORKS = UIKit Foundation
PBTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## control

```
Package: com.example.pbtrace
Name: PBTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace UIPasteboard access in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## PBTrace.plist

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
#import <UIKit/UIKit.h>
#import <os/log.h>

static os_log_t gLog;
static pthread_key_t gReentrancy;

%ctor {
    gLog = os_log_create("com.example.pbtrace", "pasteboard");
    pthread_key_create(&gReentrancy, NULL);
    os_log(gLog, "PBTrace loaded");
}

static inline bool enter(void) {
    if (pthread_getspecific(gReentrancy)) return false;
    pthread_setspecific(gReentrancy, (void *)1);
    return true;
}
static inline void leave(void) { pthread_setspecific(gReentrancy, NULL); }

%hook UIPasteboard

- (NSString *)string {
    NSString *result = %orig;
    if (enter()) {
        os_log(gLog, "[%{public}@] -string → %{public}@",
               self.name, result ?: @"nil");
        leave();
    }
    return result;
}

- (void)setString:(NSString *)string {
    if (enter()) {
        os_log(gLog, "[%{public}@] -setString: %{public}@",
               self.name, string ?: @"nil");
        leave();
    }
    %orig(string);
}

- (NSURL *)URL {
    NSURL *result = %orig;
    if (enter()) {
        os_log(gLog, "[%{public}@] -URL → %{public}@",
               self.name, result.absoluteString ?: @"nil");
        leave();
    }
    return result;
}

- (void)setURL:(NSURL *)url {
    if (enter()) {
        os_log(gLog, "[%{public}@] -setURL: %{public}@",
               self.name, url.absoluteString ?: @"nil");
        leave();
    }
    %orig(url);
}

- (UIImage *)image {
    UIImage *result = %orig;
    if (enter()) {
        os_log(gLog, "[%{public}@] -image → %{public}@",
               self.name, result ? @"UIImage" : @"nil");
        leave();
    }
    return result;
}

- (void)setItems:(NSArray<NSDictionary<NSString *, id> *> *)items {
    if (enter()) {
        os_log(gLog, "[%{public}@] -setItems: count=%lu",
               self.name, (unsigned long)items.count);
        leave();
    }
    %orig(items);
}

%end
```

Разбор:

- `pthread_key_t gReentrancy` — recursion guard: UIPasteboard внутри может вызывать `-string` на себе же.
- `self.name` — имя pasteboard'а (`UIPasteboardNameGeneral` или custom).
- Логируются и чтения (`string`, `URL`, `image`), и записи (`setString:`, `setURL:`, `setItems:`).

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

log stream --predicate 'subsystem == "com.example.pbtrace"' --device <udid>
```

Триггер:
- В TestbedApp любой action, использующий pasteboard (копировать, вставить).
- В логе — `-string`, `-setString:`.

## Этика

iOS 14+ показывает **banner** «AppX pasted from ...» при чтении pasteboard. Это защита пользователя. Этот рецепт — для диагностики **собственного** приложения, не для скрытой слежки.

Для чужого приложения — технически возможно, но:

- Нарушает ожидания пользователя.
- iOS 26 продолжает усиливать защиту (глава II.6).
- Этически — чтение чужого clipboard'а в фоне = tracking.

Для собственного приложения — валидный use case.

## См. также

- Глава IV.6 — рекурсия и thread-local guard.
- Шпаргалка 4 — filter plist.
