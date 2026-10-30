# Рецепт 10. Снятие ограничения скриншота в своём приложении

**Цель**: приложение использует `UITextField.secureTextEntry = YES` или другие приёмы, которые блокируют скриншоты системным security view. Для **собственного** приложения — отключить это блокирование для test-automation.

**Этика**: только для собственного приложения. Снятие блокировки скриншотов чужого app = обход мер защиты пользовательских данных.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = AllowScreenshot
AllowScreenshot_FILES = Tweak.x
AllowScreenshot_CFLAGS = -fobjc-arc
AllowScreenshot_FRAMEWORKS = UIKit Foundation
AllowScreenshot_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## AllowScreenshot.plist

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
Package: com.example.allowscreenshot
Name: AllowScreenshot
Version: 0.0.1
Architecture: iphoneos-arm
Description: Allow screenshots in TestbedApp (own-app only)
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## Tweak.x

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.allowscreenshot", "ss");
    os_log(gLog, "AllowScreenshot loaded");
}

// 1. Запрет secureTextEntry — глушим его для всех UITextField
%hook UITextField

- (void)setSecureTextEntry:(BOOL)secure {
    os_log(gLog, "setSecureTextEntry:%d (ignored)", (int)secure);
    %orig(NO);
}

- (BOOL)isSecureTextEntry {
    return NO;
}

%end

// 2. Для UITextView — то же самое, если применимо
%hook UITextView

- (void)setSecureTextEntry:(BOOL)secure {
    %orig(NO);
}

%end

// 3. Для UIWindow — некоторые приложения ставят overlay-view, когда app in background:
// хукаем -becomeKeyWindow / -resignKeyWindow для диагностики
%hook UIWindow

- (void)didMoveToWindow {
    %orig;
    os_log(gLog, "UIWindow didMoveToWindow: hidden=%d", (int)self.hidden);
}

%end

// 4. Отключение App Switcher snapshot blur:
// Приложения часто хукают -applicationWillResignActive чтобы показать
// placeholder вместо реального содержимого в app switcher.
// Если такое есть — хукаем:
%hook UIApplication

- (void)_setForcedDimmed:(BOOL)dimmed {
    os_log(gLog, "_setForcedDimmed:%d (ignored)", (int)dimmed);
    %orig(NO);
}

%end
```

Разбор:

- **`setSecureTextEntry`**: главный механизм. iOS автоматически блокирует отображение secure field в скриншотах (чёрный rectangle вместо текста).
- Хук **принудительно** передаёт `NO` в `%orig` — поле становится обычным.
- `isSecureTextEntry` возвращает `NO` — на случай, если приложение сверяется с getter.
- **UITextView**: аналогично (iOS 15+ поддерживает secureTextEntry и там).
- **UIWindow didMoveToWindow**: для диагностики (не всегда помогает подавить blur).
- **`_setForcedDimmed:`**: приватный метод; на iOS 26 может не существовать или называться иначе. Если не нужен — удалите.

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"
```

Запустить TestbedApp → ввести что-то в поле `secureTextEntry = YES` → сделать скриншот (Power + Volume Up) → текст виден на скриншоте.

## Расширение: подмена app switcher snapshot

Многие банковские приложения перед `-applicationDidEnterBackground` подменяют root view на placeholder:

```objc
%hook UIApplicationDelegate

- (void)applicationDidEnterBackground:(UIApplication *)application {
    os_log(gLog, "applicationDidEnterBackground called");
    // Не даём приложению подменить snapshot:
    // Правильный способ — зависит от конкретной реализации app
    // Этот рецепт — только для собственного app, где вы знаете, что заменяете.
    %orig(application);
}

%end
```

Для чужого приложения — нужен reverse engineering конкретного метода подмены snapshot. Этот рецепт этого не касается.

## Расширение: `UIScreen.capturedDidChangeNotification`

Приложения часто подписываются на эту нотификацию, чтобы знать, когда записывается экран:

```objc
%hook UIScreen

- (BOOL)isCaptured {
    BOOL real = %orig;
    os_log(gLog, "isCaptured real=%d (returning NO)", (int)real);
    return NO;  // приложение считает, что запись НЕ активна
}

%end
```

Это маскирует факт записи экрана от приложения. **Строго для собственного app** — обман чужого app = нарушение privacy.

## Этика (повтор)

Все эти приёмы — для **собственного** приложения в целях testing/automation.

- Банковское приложение, которое вы разрабатываете → OK для dev-build.
- Чужое банковское приложение → **нет**.

Apple ввела эти защиты, чтобы пользователь не мог случайно оказаться в ситуации «screenshot сохранил мой пароль в iCloud». Снимать защиту для чужих приложений — явный security risk.

## См. также

- Глава II.6 — UIKit и Scene в iOS 26.
- Глава V.3 — runtime-патчи поведения.
- Этика и границы — глава I.5.
