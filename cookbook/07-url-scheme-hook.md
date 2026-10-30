# Рецепт 7. Перехват openURL:options:completionHandler:

**Цель**: ловить и логировать все попытки собственного тестового приложения открыть внешний URL (deep link, другое приложение по его URL scheme).

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = URLSchemeTrace
URLSchemeTrace_FILES = Tweak.x
URLSchemeTrace_CFLAGS = -fobjc-arc
URLSchemeTrace_FRAMEWORKS = UIKit Foundation
URLSchemeTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## URLSchemeTrace.plist

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
Package: com.example.urlschemetrace
Name: URLSchemeTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace openURL calls in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## Tweak.x

```objc
#import <UIKit/UIKit.h>
#import <os/log.h>

static os_log_t gLog;
static NSSet<NSString *> *gBlockedSchemes;

%ctor {
    gLog = os_log_create("com.example.urlschemetrace", "url");
    gBlockedSchemes = [NSSet setWithArray:@[@"itms-services", @"spotify", @"tg"]];
    os_log(gLog, "URLSchemeTrace loaded");
}

%hook UIApplication

- (void)openURL:(NSURL *)url
         options:(NSDictionary<UIApplicationOpenExternalURLOptionsKey, id> *)options
completionHandler:(void (^)(BOOL))completion
{
    os_log(gLog, "openURL: %{public}@ options=%{public}@",
           url.absoluteString, options);

    // Опционально: заблокировать определённые схемы
    if ([gBlockedSchemes containsObject:url.scheme.lowercaseString]) {
        os_log(gLog, "BLOCKED scheme: %{public}@", url.scheme);
        if (completion) completion(NO);
        return;
    }

    // Иначе — пропустить, обернув completion:
    void (^wrapped)(BOOL) = ^(BOOL success) {
        os_log(gLog, "openURL completed: %{public}@ success=%d",
               url.absoluteString, success);
        if (completion) completion(success);
    };

    %orig(url, options, wrapped);
}

// Legacy метод для старых iOS и приложений:
- (BOOL)openURL:(NSURL *)url {
    os_log(gLog, "openURL (legacy): %{public}@", url.absoluteString);
    if ([gBlockedSchemes containsObject:url.scheme.lowercaseString]) {
        os_log(gLog, "BLOCKED (legacy): %{public}@", url.scheme);
        return NO;
    }
    return %orig(url);
}

// Проверка «можно ли открыть»:
- (BOOL)canOpenURL:(NSURL *)url {
    BOOL result = %orig(url);
    os_log(gLog, "canOpenURL: %{public}@ → %d", url.absoluteString, result);
    return result;
}

%end
```

Разбор:

- `openURL:options:completionHandler:` — главный метод iOS 10+.
- `openURL:` legacy — для совместимости.
- `canOpenURL:` — проверка; тоже стоит логировать (приложение часто сначала проверяет).
- `gBlockedSchemes` — для демонстрации, как блокировать (например, itms-services для App Store deep links).
- `wrapped` completion — чтобы увидеть результат (success/failure).

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

log stream --predicate 'subsystem == "com.example.urlschemetrace"' --device <udid>
```

Триггер: в TestbedApp любая кнопка, которая `[UIApplication.sharedApplication openURL:url options:@{} completionHandler:...]`.

Пример лога:

```
canOpenURL: https://example.com → 1
openURL: https://example.com options={...}
openURL completed: https://example.com success=1
```

## Расширение: подменить URL перед открытием

```objc
- (void)openURL:(NSURL *)url
         options:(NSDictionary<UIApplicationOpenExternalURLOptionsKey, id> *)options
completionHandler:(void (^)(BOOL))completion
{
    NSURL *replacement = url;

    // Подмена HTTP → HTTPS (для демо):
    if ([url.scheme isEqualToString:@"http"]) {
        NSURLComponents *comps = [NSURLComponents componentsWithURL:url
                                                 resolvingAgainstBaseURL:NO];
        comps.scheme = @"https";
        replacement = comps.URL;
        os_log(gLog, "upgraded http → https: %{public}@", replacement);
    }

    %orig(replacement, options, completion);
}
```

## Расширение: Universal Links

Universal Links обрабатываются `-application:continueUserActivity:restorationHandler:` (в AppDelegate), не через `openURL:`. Для их перехвата:

```objc
%hook UIApplicationDelegate
- (BOOL)application:(UIApplication *)app
   continueUserActivity:(NSUserActivity *)userActivity
     restorationHandler:(void (^)(NSArray *))restorationHandler
{
    os_log(gLog, "continueUserActivity: type=%{public}@ url=%{public}@",
           userActivity.activityType, userActivity.webpageURL.absoluteString);
    return %orig(app, userActivity, restorationHandler);
}
%end
```

## Этика

- Для собственного приложения — для диагностики.
- Для чужого приложения — pointless (openURL из чужого app заберёт у вас данные).
- Для системы — хук на SpringBoard-level `openURL` — только в своих legitimate сценариях.

## См. также

- Глава II.12 — установка приложений и URL schemes.
- Рецепт 2 — URLSession-запросы.
- Атлас UIKit — глава II.6.
