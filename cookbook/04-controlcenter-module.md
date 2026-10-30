# Рецепт 4. Собственный модуль Пункта управления

**Цель**: собрать app extension (`.appex`) с `NSExtensionPointIdentifier = com.apple.control-center.module`, установить в jbroot, увидеть иконку в Control Center.

Это **не** твик — это app extension. Твик нужен только если хотите вмешаться в поведение самого Control Center (хуки SpringBoard). Для добавления **своего** модуля — достаточно NSExtension.

## 1. Структура

```
MyCCModule/
├── Makefile
├── MyCCModule.m
├── MyCCModule.h
├── Info.plist
└── layout/
    └── Library/
        └── ControlCenter/
            └── Bundles/
                └── MyCCModule.bundle/
                    ├── Info.plist
                    └── icon.png
```

## 2. Theos Makefile для bundle-extension

Создание `.appex` через Theos не так прямолинейно, как твик. Проще собрать как **bundle** в правильный путь:

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

BUNDLE_NAME = MyCCModule

MyCCModule_FILES = MyCCModule.m
MyCCModule_INSTALL_PATH = /Library/ControlCenter/Bundles
MyCCModule_FRAMEWORKS = UIKit Foundation
MyCCModule_PRIVATE_FRAMEWORKS = ControlCenterUIKit
MyCCModule_CFLAGS = -fobjc-arc

include $(THEOS_MAKE_PATH)/bundle.mk
```

## 3. Info.plist (внутри bundle)

Theos автоматически создаёт base `Info.plist`, но для CC модуля нужны специальные ключи:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleDevelopmentRegion</key>
    <string>en</string>
    <key>CFBundleExecutable</key>
    <string>MyCCModule</string>
    <key>CFBundleIdentifier</key>
    <string>com.example.ccmodule</string>
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    <key>CFBundleName</key>
    <string>MyCCModule</string>
    <key>CFBundlePackageType</key>
    <string>BNDL</string>
    <key>CFBundleShortVersionString</key>
    <string>1.0</string>
    <key>CFBundleVersion</key>
    <string>1</string>
    <key>NSPrincipalClass</key>
    <string>MyCCModule</string>
    <key>CCSModuleType</key>
    <integer>0</integer>
    <key>CCSModuleIconName</key>
    <string>icon</string>
    <key>CCSModuleDisplayName</key>
    <string>My Module</string>
</dict>
</plist>
```

## 4. MyCCModule.h

```objc
#import <UIKit/UIKit.h>

@interface MyCCModule : NSObject
@end
```

## 5. MyCCModule.m

```objc
#import "MyCCModule.h"
#import <os/log.h>

@implementation MyCCModule

- (instancetype)init {
    self = [super init];
    if (self) {
        os_log(OS_LOG_DEFAULT, "MyCCModule initialized");
    }
    return self;
}

// Основной метод, который Control Center вызывает для получения UIView модуля:
- (UIView *)view {
    static UIView *view;
    if (!view) {
        view = [[UIView alloc] initWithFrame:CGRectZero];
        view.backgroundColor = [UIColor systemBlueColor];
        view.layer.cornerRadius = 12;
        UILabel *label = [[UILabel alloc] initWithFrame:CGRectZero];
        label.text = @"Hi!";
        label.textColor = [UIColor whiteColor];
        label.textAlignment = NSTextAlignmentCenter;
        label.translatesAutoresizingMaskIntoConstraints = NO;
        [view addSubview:label];
        [NSLayoutConstraint activateConstraints:@[
            [label.centerXAnchor constraintEqualToAnchor:view.centerXAnchor],
            [label.centerYAnchor constraintEqualToAnchor:view.centerYAnchor],
        ]];
    }
    return view;
}

- (BOOL)isEnabled { return YES; }

@end
```

## 6. control

```
Package: com.example.ccmodule
Name: My Control Center Module
Version: 0.0.1
Architecture: iphoneos-arm
Description: Custom Control Center module
Section: Tweaks
Depends: firmware (>= 15.0)
Depiction: https://example.com/depictions/com.example.ccmodule
```

## 7. Сборка и установка

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 SpringBoard"
```

После respring'а Control Center должен показать ваш модуль.

## 8. Проверка на устройстве

Если модуль не появился:

```bash
ssh mobile@<ip>
ls /var/jb/Library/ControlCenter/Bundles/MyCCModule.bundle/
# Info.plist MyCCModule icon.png

# Проверить NSPrincipalClass:
plutil -p /var/jb/Library/ControlCenter/Bundles/MyCCModule.bundle/Info.plist | grep Principal

# Проверить в Settings → Control Center → "Customize Controls":
# Модуль должен быть в списке "More Controls" — пользователь добавляет через +.
```

## 9. Ограничения

- **CCSModuleType = 0** — простой toggle module. Другие типы (slider, button) — см. community docs для Control Center bundles.
- **Private API**: `ControlCenterUIKit` — private framework; имена классов и методов могут меняться между минорными iOS.
- **На iOS 26**: конкретный protocol module expected to conform to — надо проверить class-dump на устройстве. Общий принцип тот же.

## 10. Этика

Control Center модуль — публично предусмотренная Apple extension-point. Это **нормальное расширение**, не jailbreak-tweak в агрессивном смысле.

## 11. См. также

- Глава II.4 — Control Center.
- Глава II.12 — расширения приложений.
- Шпаргалка 2 — Makefile с `BUNDLE_NAME`.
