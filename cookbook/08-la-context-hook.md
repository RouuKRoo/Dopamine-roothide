# Рецепт 8. Перехват LAContext (биометрическая проверка)

**Цель**: хукнуть `-[LAContext evaluatePolicy:localizedReason:reply:]` в собственном тестовом приложении для unit-тестирования UI, зависящего от биометрии.

**Этика**: строго для **собственного** приложения. Обход биометрии чужого приложения — незаконно во многих юрисдикциях и этически неприемлемо.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = LAStub
LAStub_FILES = Tweak.x
LAStub_CFLAGS = -fobjc-arc
LAStub_FRAMEWORKS = LocalAuthentication Foundation
LAStub_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## control

```
Package: com.example.lastub
Name: LAStub
Version: 0.0.1
Architecture: iphoneos-arm
Description: Stub LAContext.evaluatePolicy for test automation in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## LAStub.plist

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
</dict>
```

## Tweak.x

```objc
#import <LocalAuthentication/LocalAuthentication.h>
#import <Foundation/Foundation.h>
#import <os/log.h>

static os_log_t gLog;

// Режим подмены — можно управлять через CFPreferences
typedef NS_ENUM(NSInteger, LAMode) {
    LAModePassThrough,       // пропускать (реальное поведение)
    LAModeForceSuccess,      // всегда YES
    LAModeForceFailure,      // всегда NO
};

static LAMode gMode = LAModeForceSuccess;

%ctor {
    gLog = os_log_create("com.example.lastub", "la");
    os_log(gLog, "LAStub loaded, mode=%d", (int)gMode);
}

%hook LAContext

- (BOOL)canEvaluatePolicy:(LAPolicy)policy error:(NSError **)error {
    BOOL real = %orig(policy, error);
    os_log(gLog, "canEvaluatePolicy(%ld) real=%d", (long)policy, (int)real);

    if (gMode == LAModeForceSuccess) {
        if (error) *error = nil;
        return YES;
    }
    if (gMode == LAModeForceFailure) {
        if (error) {
            *error = [NSError errorWithDomain:LAErrorDomain
                                         code:LAErrorBiometryNotAvailable
                                     userInfo:@{NSLocalizedDescriptionKey: @"Stubbed failure"}];
        }
        return NO;
    }
    return real;
}

- (void)evaluatePolicy:(LAPolicy)policy
         localizedReason:(NSString *)localizedReason
                   reply:(void (^)(BOOL success, NSError *error))reply
{
    os_log(gLog, "evaluatePolicy(%ld, reason=%{public}@) mode=%d",
           (long)policy, localizedReason, (int)gMode);

    if (gMode == LAModeForceSuccess) {
        // Имитация задержки (как реальная биометрия)
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1 * NSEC_PER_SEC)),
                       dispatch_get_main_queue(), ^{
            if (reply) reply(YES, nil);
        });
        return;
    }

    if (gMode == LAModeForceFailure) {
        NSError *err = [NSError errorWithDomain:LAErrorDomain
                                           code:LAErrorAuthenticationFailed
                                       userInfo:@{NSLocalizedDescriptionKey: @"Stubbed failure"}];
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1 * NSEC_PER_SEC)),
                       dispatch_get_main_queue(), ^{
            if (reply) reply(NO, err);
        });
        return;
    }

    // PassThrough: обёртка для логирования результата
    void (^wrapped)(BOOL, NSError *) = ^(BOOL success, NSError *error) {
        os_log(gLog, "evaluatePolicy real result: success=%d error=%{public}@",
               (int)success, error.localizedDescription ?: @"nil");
        if (reply) reply(success, error);
    };

    %orig(policy, localizedReason, wrapped);
}

%end
```

Разбор:

- `gMode` — глобальный флаг: PassThrough / ForceSuccess / ForceFailure.
- `canEvaluatePolicy:` — проверяется первой; если возвращает NO, приложение даже не покажет prompt.
- `evaluatePolicy:...:reply:` — главный метод; `reply` — callback с результатом.
- Задержка 0.1s в имитации — чтобы приложение не ломалось от «мгновенного» ответа (UI-animation может ожидать небольшого времени).
- `LAErrorDomain` и коды — стандартные (`LAErrorBiometryNotAvailable`, `LAErrorAuthenticationFailed`, `LAErrorUserCancel`, ...).

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

log stream --predicate 'subsystem == "com.example.lastub"' --device <udid>
```

Триггер: в TestbedApp любой action, запрашивающий Face ID / Touch ID → prompt **не появляется**, но приложение считает, что аутентификация прошла.

## Расширение: три режима через CFPreferences

```objc
#define PREFS_DOMAIN CFSTR("com.example.lastub")

static void reload_mode(void) {
    Boolean exists = false;
    int64_t val = CFPreferencesGetAppIntegerValue(CFSTR("mode"), PREFS_DOMAIN, &exists);
    if (exists) gMode = (LAMode)val;
    os_log(gLog, "mode reloaded: %d", (int)gMode);
}

%ctor {
    gLog = os_log_create("com.example.lastub", "la");
    reload_mode();

    int token = 0;
    notify_register_dispatch("com.example.lastub.changed", &token,
                             dispatch_get_main_queue(), ^(int t) {
        reload_mode();
    });
}
```

Приложить PreferenceBundle с segmented control (0/1/2) → переключение в рантайме.

## Расширение: biometryType подмена

```objc
%hook LAContext

- (LABiometryType)biometryType {
    LABiometryType real = %orig;
    os_log(gLog, "biometryType real=%ld", (long)real);
    if (gMode == LAModeForceSuccess) {
        return LABiometryTypeFaceID;  // имитация FaceID независимо от устройства
    }
    return real;
}

%end
```

Это даёт возможность тестировать UI «на устройстве с Face ID» на устройстве с Touch ID.

## Этика (напомним)

Для **собственного** приложения — valid use case:

- Unit-testing UI, завимого от биометрии.
- Автоматические скриншоты для App Store (где биометрия всегда «успешна»).
- Fuzzing сценариев error paths.

Для **чужого** приложения — незаконно. Биометрия защищает banking apps, password managers, enterprise MFA. Обход — криминальный акт.

## См. также

- Глава II.2 — экран блокировки и аутентификация.
- Атлас LocalAuthentication (framework-atlas, если будет написан).
- Рецепт 6 — keychain (с которым LAContext часто работает).
