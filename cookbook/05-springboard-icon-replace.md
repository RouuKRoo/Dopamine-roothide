# Рецепт 5. Замена иконки системного приложения в SpringBoard

**Цель**: заменить иконку конкретного приложения в home screen на свою PNG.

**Внимание**: конкретные селекторы `SBIconView` / `_UIImageViewRaw` **на iOS 26** могут отличаться от iOS 15–18. Этот рецепт — **шаблон**; verifier на устройстве через class-dump обязателен.

## 1. Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = SpringBoard
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = IconReplace
IconReplace_FILES = Tweak.x
IconReplace_CFLAGS = -fobjc-arc
IconReplace_FRAMEWORKS = UIKit Foundation
IconReplace_PRIVATE_FRAMEWORKS = SpringBoardFoundation SpringBoardUI
IconReplace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## 2. Filter plist

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.apple.springboard</string>
    </array>
</dict>
```

## 3. Tweak.x — шаблон

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <libroot.h>
#import <os/log.h>
#import <objc/runtime.h>

static os_log_t gLog;
static NSString *gTargetBundleId = @"com.apple.mobilesafari"; // bundle id для замены
static UIImage *gReplacementImage = nil;

static UIImage *load_replacement_image(void) {
    NSString *path = @(JBROOT_PATH("/Library/Application Support/IconReplace/replacement.png"));
    return [UIImage imageWithContentsOfFile:path];
}

@interface SBIcon : NSObject
- (NSString *)applicationBundleID;
@end

@interface SBIconView : UIView
- (id)icon;
- (void)setIcon:(id)icon;
@end

%hook SBIconView

- (void)setIcon:(id)icon {
    %orig(icon);
    @try {
        NSString *bid = [icon applicationBundleID];
        if ([bid isEqualToString:gTargetBundleId]) {
            if (!gReplacementImage) {
                gReplacementImage = load_replacement_image();
            }
            if (gReplacementImage) {
                // Поиск imageView внутри SBIconView (может быть subview)
                for (UIView *sub in self.subviews) {
                    if ([sub isKindOfClass:[UIImageView class]]) {
                        [(UIImageView *)sub setImage:gReplacementImage];
                        os_log(gLog, "icon replaced for %{public}@", bid);
                        break;
                    }
                }
            }
        }
    } @catch (NSException *e) {
        os_log_error(gLog, "exception in setIcon: %{public}@", e);
    }
}

%end

%ctor {
    gLog = os_log_create("com.example.iconreplace", "icon");
    os_log(gLog, "IconReplace loaded, target=%{public}@", gTargetBundleId);
}
```

Разбор:

- `@interface SBIcon` и `@interface SBIconView` — минимальная декларация private классов (нам нужны только эти 2 селектора).
- `setIcon:` хукаем, чтобы перехватывать момент установки иконки.
- `[icon applicationBundleID]` — получаем bundle id приложения, которому принадлежит иконка.
- Фильтр по `gTargetBundleId` — заменяем только у нужного приложения.
- Загрузка PNG через `JBROOT_PATH`.
- `@try/@catch` — защита от падения, если на iOS 26 поведение другое.
- Поиск `UIImageView` в subviews — грубый approach; на iOS 26 структура может отличаться, обязательна верификация через class-dump.

## 4. Ресурсы

Положить PNG:

```
layout/
└── Library/
    └── Application Support/
        └── IconReplace/
            └── replacement.png
```

Theos автоматически включит в .deb.

## 5. control

```
Package: com.example.iconreplace
Name: IconReplace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Replace one app icon in SpringBoard
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## 6. Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 SpringBoard"
```

На home screen иконка Safari должна смениться на вашу PNG.

## 7. Если не работает

Поиск правильного класса на iOS 26:

```objc
// Добавить в %ctor для диагностики:
unsigned int count = 0;
Class *classes = objc_copyClassList(&count);
for (unsigned int i = 0; i < count; i++) {
    const char *name = class_getName(classes[i]);
    if (strstr(name, "IconView") || strstr(name, "IconImageView")) {
        os_log(gLog, "candidate: %s", name);
    }
}
free(classes);
```

Запустите, посмотрите log → найдёте реальный класс на вашей iOS 26.

Другой подход — хукнуть `UIImageView` setImage: внутри SBIconView:

```objc
// Опасный но часто работающий фолбэк:
%hook UIImageView
- (void)setImage:(UIImage *)image {
    UIView *parent = self.superview;
    while (parent && ![parent isKindOfClass:NSClassFromString(@"SBIconView")]) {
        parent = parent.superview;
    }
    if (parent) {
        id icon = [parent valueForKey:@"icon"];
        NSString *bid = [icon applicationBundleID];
        if ([bid isEqualToString:gTargetBundleId]) {
            if (!gReplacementImage) gReplacementImage = load_replacement_image();
            if (gReplacementImage) image = gReplacementImage;
        }
    }
    %orig(image);
}
%end
```

Это шире, но может быть необходимо для стабильности на iOS 26.

## 8. Респринг и rollback

**Перед первой установкой** — приготовьте план отката (глава IV.12):

```bash
# Если SpringBoard упал в loop:
ssh mobile@<ip>
touch /var/jb/basebin/.safe_mode
killall -9 SpringBoard
# Теперь твики выключены — можно удалить:
sudo dpkg -r com.example.iconreplace
rm /var/jb/basebin/.safe_mode
killall -9 SpringBoard
```

## 9. Этика

Это **косметический** твик для вашего устройства. Нормально.

## 10. См. также

- Глава II.1 — SpringBoard.
- Глава IV.12 — SpringBoard-твик с safe mode.
- Case 7 — crash-safe skeleton.
