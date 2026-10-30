# Рецепт 13. Переопределение User-Agent URLSession

**Цель**: твик, который заменяет User-Agent header всех HTTP-запросов из `URLSession` в своём тестовом приложении.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = UAOverride
UAOverride_FILES = Tweak.x
UAOverride_CFLAGS = -fobjc-arc
UAOverride_FRAMEWORKS = Foundation
UAOverride_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## UAOverride.plist

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
Package: com.example.uaoverride
Name: UAOverride
Version: 0.0.1
Architecture: iphoneos-arm
Description: Override User-Agent for URLSession in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## Tweak.x

```objc
#import <Foundation/Foundation.h>
#import <os/log.h>

static os_log_t gLog;
static NSString * const kCustomUA = @"MyCustomUA/1.0 (TestbedApp)";

%ctor {
    gLog = os_log_create("com.example.uaoverride", "ua");
    os_log(gLog, "UAOverride loaded, UA = %{public}@", kCustomUA);
}

%hook NSURLRequest

// Подменяем значение, которое возвращается при чтении User-Agent header
- (NSString *)valueForHTTPHeaderField:(NSString *)field {
    if ([field caseInsensitiveCompare:@"User-Agent"] == NSOrderedSame) {
        os_log(gLog, "valueForHTTPHeaderField User-Agent → %{public}@", kCustomUA);
        return kCustomUA;
    }
    return %orig(field);
}

- (NSDictionary<NSString *, NSString *> *)allHTTPHeaderFields {
    NSDictionary *orig = %orig;
    if (!orig) return @{@"User-Agent": kCustomUA};
    NSMutableDictionary *patched = [orig mutableCopy];
    patched[@"User-Agent"] = kCustomUA;
    return patched;
}

%end

%hook NSMutableURLRequest

- (void)setValue:(NSString *)value forHTTPHeaderField:(NSString *)field {
    if ([field caseInsensitiveCompare:@"User-Agent"] == NSOrderedSame) {
        os_log(gLog, "setValue:forHTTPHeaderField User-Agent: %{public}@ → %{public}@",
               value, kCustomUA);
        %orig(kCustomUA, field);
        return;
    }
    %orig(value, field);
}

- (void)addValue:(NSString *)value forHTTPHeaderField:(NSString *)field {
    if ([field caseInsensitiveCompare:@"User-Agent"] == NSOrderedSame) {
        %orig(kCustomUA, field);
        return;
    }
    %orig(value, field);
}

%end

%hook NSURLSessionConfiguration

- (NSDictionary *)HTTPAdditionalHeaders {
    NSDictionary *orig = %orig;
    NSMutableDictionary *patched = orig ? [orig mutableCopy] : [NSMutableDictionary dictionary];
    patched[@"User-Agent"] = kCustomUA;
    return patched;
}

- (void)setHTTPAdditionalHeaders:(NSDictionary *)headers {
    NSMutableDictionary *patched = headers ? [headers mutableCopy] : [NSMutableDictionary dictionary];
    patched[@"User-Agent"] = kCustomUA;
    os_log(gLog, "setHTTPAdditionalHeaders patched User-Agent");
    %orig(patched);
}

%end
```

Разбор:

- Четыре места, где может задаваться User-Agent:
  1. `[request valueForHTTPHeaderField:]` — чтение.
  2. `[mutableRequest setValue:forHTTPHeaderField:]` — установка.
  3. `[config HTTPAdditionalHeaders]` — session-level.
  4. `[config setHTTPAdditionalHeaders:]` — установка session-level.
- Нужно хукнуть **все четыре** — потому что приложение может использовать любой вариант.
- `caseInsensitiveCompare:` — HTTP headers case-insensitive по RFC 7230.
- Для session-config — заменяем только User-Agent, остальные headers оставляем.

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"
```

Запустить приложение, сделать запрос. На receiving end (сервер или прокси) — User-Agent будет `MyCustomUA/1.0 (TestbedApp)`.

Проверка через mitmproxy / Charles Proxy / own server:

```
GET /api/v1/thing HTTP/1.1
Host: api.example.com
User-Agent: MyCustomUA/1.0 (TestbedApp)
```

## Расширение: UA из настроек

```objc
static NSString *gCurrentUA;

static void reloadUA(void) {
    CFPropertyListRef v = CFPreferencesCopyAppValue(CFSTR("userAgent"),
                                                     CFSTR("com.example.uaoverride"));
    if (v && CFGetTypeID(v) == CFStringGetTypeID()) {
        gCurrentUA = (__bridge_transfer NSString *)v;
    } else {
        if (v) CFRelease(v);
        gCurrentUA = @"MyCustomUA/1.0";
    }
}

// Везде вместо kCustomUA — gCurrentUA.
```

Плюс Darwin notification для изменений (глава IV.8).

## Этика

Для своего приложения — valid test automation. Для чужого — обход fingerprinting (может быть легитимно для privacy, но некоторые сайты блокируют нестандартные UA).

## См. также

- Рецепт 2 — URLSession-запросы.
- Атлас Network — глава II.9.
- Глава IV.8 — настройки через CFPreferences.
