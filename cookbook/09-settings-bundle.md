# Рецепт 9. PreferenceBundle с libroot

**Цель**: добавить страницу настроек в Settings.app → MyTweak с переключателем, slider'ом и text field. Чтение/запись через `CFPreferences` с правильными путями через `libroot` (автоматически через cfprefsd redirect).

Это компактная версия того, что подробно разбирается в главе IV.11. Здесь — только настройки, без твика-клиента.

## Структура

```
MyTweakPrefs/
├── Makefile
├── MyTweakPrefsListController.h
├── MyTweakPrefsListController.m
├── Resources/
│   ├── Info.plist
│   └── Root.plist
├── entry.plist
└── control
```

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

BUNDLE_NAME = MyTweakPrefs

MyTweakPrefs_FILES = MyTweakPrefsListController.m
MyTweakPrefs_INSTALL_PATH = /Library/PreferenceBundles
MyTweakPrefs_FRAMEWORKS = UIKit Preferences
MyTweakPrefs_LIBRARIES = root
MyTweakPrefs_CFLAGS = -fobjc-arc

include $(THEOS_MAKE_PATH)/bundle.mk

internal-stage::
	$(ECHO_NOTHING)mkdir -p $(THEOS_STAGING_DIR)/Library/PreferenceLoader/Preferences$(ECHO_END)
	$(ECHO_NOTHING)cp entry.plist $(THEOS_STAGING_DIR)/Library/PreferenceLoader/Preferences/MyTweak.plist$(ECHO_END)
```

## MyTweakPrefsListController.h

```objc
#import <Preferences/PSListController.h>

@interface MyTweakPrefsListController : PSListController
@end
```

## MyTweakPrefsListController.m

```objc
#import "MyTweakPrefsListController.h"
#import <notify.h>

#define PREFS_DOMAIN @"com.example.mytweak"
#define NOTIFY_CHANGE "com.example.mytweak.preferences.changed"

@implementation MyTweakPrefsListController

- (NSArray *)specifiers {
    if (!_specifiers) {
        _specifiers = [self loadSpecifiersFromPlistName:@"Root" target:self];
    }
    return _specifiers;
}

- (void)setPreferenceValue:(id)value specifier:(PSSpecifier *)specifier {
    NSString *key = [specifier propertyForKey:@"key"];
    NSString *defaults = [specifier propertyForKey:@"defaults"] ?: PREFS_DOMAIN;

    CFPreferencesSetAppValue((__bridge CFStringRef)key,
                             (__bridge CFPropertyListRef)value,
                             (__bridge CFStringRef)defaults);
    CFPreferencesAppSynchronize((__bridge CFStringRef)defaults);

    notify_post(NOTIFY_CHANGE);
}

- (id)readPreferenceValue:(PSSpecifier *)specifier {
    NSString *key = [specifier propertyForKey:@"key"];
    NSString *defaults = [specifier propertyForKey:@"defaults"] ?: PREFS_DOMAIN;
    id defaultValue = [specifier propertyForKey:@"default"];

    CFPropertyListRef v = CFPreferencesCopyAppValue((__bridge CFStringRef)key,
                                                    (__bridge CFStringRef)defaults);
    if (!v) return defaultValue;
    return (__bridge_transfer id)v;
}

@end
```

## Resources/Info.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleDevelopmentRegion</key>
    <string>en</string>
    <key>CFBundleExecutable</key>
    <string>MyTweakPrefs</string>
    <key>CFBundleIdentifier</key>
    <string>com.example.mytweakprefs</string>
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    <key>CFBundleName</key>
    <string>MyTweakPrefs</string>
    <key>CFBundlePackageType</key>
    <string>BNDL</string>
    <key>NSPrincipalClass</key>
    <string>MyTweakPrefsListController</string>
</dict>
</plist>
```

## Resources/Root.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>items</key>
    <array>
        <dict>
            <key>cell</key>
            <string>PSGroupCell</string>
            <key>label</key>
            <string>MyTweak</string>
            <key>footerText</key>
            <string>Example settings page using CFPreferences and Darwin notifications.</string>
        </dict>

        <dict>
            <key>cell</key>
            <string>PSSwitchCell</string>
            <key>label</key>
            <string>Enabled</string>
            <key>key</key>
            <string>enabled</string>
            <key>defaults</key>
            <string>com.example.mytweak</string>
            <key>default</key>
            <true/>
        </dict>

        <dict>
            <key>cell</key>
            <string>PSSliderCell</string>
            <key>label</key>
            <string>Intensity</string>
            <key>key</key>
            <string>intensity</string>
            <key>defaults</key>
            <string>com.example.mytweak</string>
            <key>default</key>
            <real>0.5</real>
            <key>min</key>
            <real>0.0</real>
            <key>max</key>
            <real>1.0</real>
            <key>showValue</key>
            <true/>
        </dict>

        <dict>
            <key>cell</key>
            <string>PSEditTextCell</string>
            <key>label</key>
            <string>Message</string>
            <key>key</key>
            <string>customMessage</string>
            <key>defaults</key>
            <string>com.example.mytweak</string>
            <key>default</key>
            <string>Hello</string>
            <key>placeholder</key>
            <string>Enter message</string>
        </dict>

        <dict>
            <key>cell</key>
            <string>PSSegmentCell</string>
            <key>label</key>
            <string>Mode</string>
            <key>key</key>
            <string>mode</string>
            <key>defaults</key>
            <string>com.example.mytweak</string>
            <key>default</key>
            <integer>0</integer>
            <key>validTitles</key>
            <array>
                <string>Normal</string>
                <string>Aggressive</string>
                <string>Debug</string>
            </array>
            <key>validValues</key>
            <array>
                <integer>0</integer>
                <integer>1</integer>
                <integer>2</integer>
            </array>
        </dict>

        <dict>
            <key>cell</key>
            <string>PSGroupCell</string>
            <key>footerText</key>
            <string>Changes apply immediately via Darwin notifications.</string>
        </dict>
    </array>
    <key>title</key>
    <string>MyTweak</string>
</dict>
</plist>
```

## entry.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>entry</key>
    <dict>
        <key>bundle</key>
        <string>MyTweakPrefs</string>
        <key>cell</key>
        <string>PSLinkCell</string>
        <key>detail</key>
        <string>MyTweakPrefsListController</string>
        <key>isController</key>
        <true/>
        <key>label</key>
        <string>MyTweak</string>
    </dict>
</dict>
</plist>
```

## control

```
Package: com.example.mytweakprefs
Name: MyTweak Prefs
Version: 0.0.1
Architecture: iphoneos-arm
Description: Settings page for MyTweak
Section: Tweaks
Depends: firmware (>= 15.0), preferenceloader, libroot-dopamine
```

## Клиент-твик (подписка на Darwin notification)

В Tweak.x вашего основного твика:

```objc
#import <notify.h>

static BOOL gEnabled = YES;
static float gIntensity = 0.5;
static NSString *gMessage = @"Hello";
static int gMode = 0;

static void reloadPrefs(void) {
    Boolean exists = false;
    Boolean b = CFPreferencesGetAppBooleanValue(CFSTR("enabled"), CFSTR("com.example.mytweak"), &exists);
    gEnabled = exists ? b : YES;

    CFPropertyListRef v = CFPreferencesCopyAppValue(CFSTR("intensity"), CFSTR("com.example.mytweak"));
    if (v) {
        gIntensity = [(__bridge_transfer NSNumber *)v floatValue];
    } else {
        gIntensity = 0.5;
    }

    v = CFPreferencesCopyAppValue(CFSTR("customMessage"), CFSTR("com.example.mytweak"));
    if (v && CFGetTypeID(v) == CFStringGetTypeID()) {
        gMessage = (__bridge_transfer NSString *)v;
    } else {
        if (v) CFRelease(v);
        gMessage = @"Hello";
    }

    v = CFPreferencesCopyAppValue(CFSTR("mode"), CFSTR("com.example.mytweak"));
    if (v) {
        gMode = [(__bridge_transfer NSNumber *)v intValue];
    } else {
        gMode = 0;
    }
}

%ctor {
    reloadPrefs();
    int token = 0;
    notify_register_dispatch("com.example.mytweak.preferences.changed",
                             &token, dispatch_get_main_queue(),
                             ^(int t) { reloadPrefs(); });
}
```

## Сборка

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 Preferences"
```

Settings.app → scroll down → MyTweak → страница настроек.

## Проверка

```bash
# Settings.plist создан:
ssh mobile@<ip>
plutil -p /var/jb/var/mobile/Library/Preferences/com.example.mytweak.plist
```

## См. также

- Глава IV.8 — настройки твика и ресурсы.
- Глава IV.11 — полный учебный проект с prefs + твиком.
- Глава II.7 — Settings.app и preferences.
