# Глава IV.11. Учебный проект №1: твик для собственного тестового приложения

Это первый полный учебный проект книги. Задача — провести вас через весь путь: от создания тестового приложения в Xcode до работающего твика с настройками и Darwin notification. По его завершении у вас на устройстве будет приложение, в котором видимые элементы UI меняются от переключателя в Settings.app.

## 1. Цель и ожидаемое поведение

**Тестовое приложение `com.example.testbed`:**

- Один экран с центральной `UILabel`, показывающей текст «Original Label».
- Кнопка «Reload», которая просто перерисовывает лейбл (чтобы увидеть эффект твика в рантайме).

**Твик `com.example.testbedtweak`:**

- Срабатывает только в этом приложении (filter plist).
- Подменяет текст `UILabel` на кастомную строку из настроек (по умолчанию «Hello from Dopamine»).
- Имеет PreferenceBundle в Settings → «TestbedTweak» с переключателем «Enabled» и text field для кастомного текста.
- Реагирует на изменение настроек через Darwin notification без respring.

**Статус проверки**: пример статически корректен, цельные Makefile/control/plist приведены. Работа на устройстве A13 c iOS 26.0/26.0.1 требует проверки читателем на его устройстве.

## 2. Дерево файлов (итоговое)

```
~/dev/testbed/                     — Xcode приложение (iOS App)
    TestbedApp/
        AppDelegate.swift
        SceneDelegate.swift
        ViewController.swift
        ...

~/dev/TestbedTweak/                — Theos проект
    Makefile
    TestbedTweak.plist
    Tweak.x
    control
    Resources/                     — ресурсы твика
    prefs/                          — PreferenceBundle
        Makefile
        Resources/
            Root.plist
            Info.plist
        TestbedPrefsListController.h
        TestbedPrefsListController.m
        entry.plist
```

## 3. Шаг 1: Создание тестового приложения

В Xcode:

1. File → New → Project → iOS → App.
2. Product Name: `TestbedApp`.
3. Interface: Storyboard (упрощённо) или SwiftUI (не рекомендуется для первого твика — хуже хукается).
4. Language: Swift.
5. Organization Identifier: `com.example` → Bundle Identifier станет `com.example.testbed`.
6. Deployment target: iOS 15.0.

`ViewController.swift`:

```swift
import UIKit

class ViewController: UIViewController {
    @IBOutlet weak var label: UILabel!
    @IBOutlet weak var reloadButton: UIButton!

    override func viewDidLoad() {
        super.viewDidLoad()
        label.text = "Original Label"
    }

    @IBAction func reloadTapped(_ sender: Any) {
        label.text = "Original Label"
    }
}
```

В Main.storyboard — расположите `UILabel` и `UIButton`, подключите outlets/action.

Соберите приложение (`Cmd + B`). Для jailbreak-разработки удобно подписать его как self-signed + установить через TrollStore / Sideloadly, чтобы оно пережило `make install` твика.

## 4. Шаг 2: Создание Theos проекта

```bash
cd ~/dev
$THEOS/bin/nic.pl
```

Ответы:

- Template: `[15] iphone/tweak`.
- Project Name: `TestbedTweak`.
- Package Name: `com.example.testbedtweak`.
- Author: `You`.
- MobileSubstrate Bundle filter: `com.example.testbed`.

## 5. Makefile твика

Открываем `Makefile`:

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = TestbedTweak

TestbedTweak_FILES = Tweak.x
TestbedTweak_CFLAGS = -fobjc-arc
TestbedTweak_FRAMEWORKS = UIKit Foundation
TestbedTweak_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
SUBPROJECTS += prefs
include $(THEOS_MAKE_PATH)/aggregate.mk
```

Три отличия от минимального шаблона:

- `INSTALL_TARGET_PROCESSES = TestbedApp` — Theos killall'ом перезапустит ваше приложение после `make install`.
- `TestbedTweak_LIBRARIES = root` — libroot (для `JBROOT_PATH`).
- `SUBPROJECTS += prefs` + `aggregate.mk` — говорит Theos собирать подпроект `prefs/` вместе с главным.

## 6. control

```
Package: com.example.testbedtweak
Name: TestbedTweak
Version: 0.0.1
Architecture: iphoneos-arm
Description: Tweak for TestbedApp — demonstrates hooks + preferences
Maintainer: You
Author: You
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine, preferenceloader
```

Добавлено `preferenceloader` — для PreferenceBundle.

## 7. TestbedTweak.plist (filter)

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

## 8. Tweak.x — главное тело

```objc
#import <UIKit/UIKit.h>
#import <os/log.h>
#import <notify.h>

static os_log_t gLog;
static dispatch_queue_t gPrefsQueue;
static BOOL gEnabled = YES;
static NSString *gCustomText = @"Hello from Dopamine";
#define PREFS_DOMAIN CFSTR("com.example.testbedtweak")

static void reloadPrefs(void) {
    Boolean enabledExists = false;
    Boolean enabledValue = CFPreferencesGetAppBooleanValue(CFSTR("enabled"), PREFS_DOMAIN, &enabledExists);
    gEnabled = enabledExists ? enabledValue : YES;

    CFStringRef customText = CFPreferencesCopyAppValue(CFSTR("customText"), PREFS_DOMAIN);
    if (customText && CFGetTypeID(customText) == CFStringGetTypeID()) {
        gCustomText = (__bridge_transfer NSString *)customText;
    } else {
        if (customText) CFRelease(customText);
        gCustomText = @"Hello from Dopamine";
    }
    os_log(gLog, "prefs reloaded: enabled=%d, custom=%{public}@", (int)gEnabled, gCustomText);
}

%ctor {
    gLog = os_log_create("com.example.testbedtweak", "tweak");
    gPrefsQueue = dispatch_queue_create("com.example.testbedtweak.prefs", DISPATCH_QUEUE_SERIAL);
    dispatch_sync(gPrefsQueue, ^{ reloadPrefs(); });

    int token = 0;
    notify_register_dispatch("com.example.testbedtweak.preferences.changed",
                             &token, gPrefsQueue, ^(int t) {
        reloadPrefs();
    });

    os_log(gLog, "TestbedTweak loaded in %{public}@",
           [NSProcessInfo processInfo].processName);
}

%hook UILabel

- (void)setText:(NSString *)text {
    if (!gEnabled) {
        %orig(text);
        return;
    }
    // Подменяем только "Original Label", остальные лейблы (внутри UIKit) оставляем в покое
    if ([text isEqualToString:@"Original Label"]) {
        NSString *replacement;
        @synchronized (gCustomText) {
            replacement = gCustomText;
        }
        os_log(gLog, "replacing '%{public}@' with '%{public}@'", text, replacement);
        %orig(replacement);
    } else {
        %orig(text);
    }
}

%end
```

Разбор ключевых решений.

**`reloadPrefs`**:
- Читает `enabled` (Boolean) и `customText` (String) из CFPreferences.
- Защищается от nil и неверного типа.
- Устанавливает глобалы, которые читают хуки.

**`%ctor`**:
- Создаёт `os_log_t`.
- Создаёт serial queue для синхронизации доступа к префам.
- Регистрирует Darwin notification handler через `notify_register_dispatch` — удобная обёртка над `notify_register_check` + подпиской на queue.
- `reloadPrefs` вызывается синхронно для начальной загрузки.

**`%hook UILabel setText:`**:
- Проверяет `gEnabled` — если выключен, зовёт оригинал без изменений.
- Проверяет **текст** — подменяет только «Original Label», чтобы не трогать UIKit internals (которые тоже setText делают для своих текстовок).
- Внутри `@synchronized(gCustomText)` — страхует от расходящегося `gCustomText` на других потоках.

**Почему хукаем `UILabel setText:`, а не `ViewController`**: в тестовом приложении `label.text = "Original Label"` — это именно вызов `-[UILabel setText:]`. Хук здесь работает и для `viewDidLoad`, и для `reloadTapped`.

## 9. PreferenceBundle — подпроект `prefs/`

Создаём каталог `prefs/`:

```bash
mkdir prefs
cd prefs
```

### 9.1. prefs/Makefile

```make
include $(THEOS)/makefiles/common.mk

BUNDLE_NAME = TestbedPrefs

TestbedPrefs_FILES = TestbedPrefsListController.m
TestbedPrefs_INSTALL_PATH = /Library/PreferenceBundles
TestbedPrefs_FRAMEWORKS = UIKit Preferences
TestbedPrefs_LIBRARIES = root
TestbedPrefs_CFLAGS = -fobjc-arc

include $(THEOS_MAKE_PATH)/bundle.mk

internal-stage::
	$(ECHO_NOTHING)mkdir -p $(THEOS_STAGING_DIR)/Library/PreferenceLoader/Preferences$(ECHO_END)
	$(ECHO_NOTHING)cp entry.plist $(THEOS_STAGING_DIR)/Library/PreferenceLoader/Preferences/TestbedTweak.plist$(ECHO_END)
```

### 9.2. prefs/TestbedPrefsListController.h

```objc
#import <Preferences/PSListController.h>

@interface TestbedPrefsListController : PSListController
@end
```

### 9.3. prefs/TestbedPrefsListController.m

```objc
#import "TestbedPrefsListController.h"
#import <notify.h>

#define PREFS_DOMAIN @"com.example.testbedtweak"
#define NOTIFY_CHANGE "com.example.testbedtweak.preferences.changed"

@implementation TestbedPrefsListController

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

    CFPropertyListRef value = CFPreferencesCopyAppValue((__bridge CFStringRef)key,
                                                        (__bridge CFStringRef)defaults);
    if (!value) return defaultValue;
    return (__bridge_transfer id)value;
}

@end
```

### 9.4. prefs/Resources/Root.plist

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
            <string>TestbedTweak</string>
            <key>footerText</key>
            <string>Replaces "Original Label" text in TestbedApp.</string>
        </dict>
        <dict>
            <key>cell</key>
            <string>PSSwitchCell</string>
            <key>label</key>
            <string>Enabled</string>
            <key>key</key>
            <string>enabled</string>
            <key>defaults</key>
            <string>com.example.testbedtweak</string>
            <key>default</key>
            <true/>
        </dict>
        <dict>
            <key>cell</key>
            <string>PSEditTextCell</string>
            <key>label</key>
            <string>Custom text</string>
            <key>key</key>
            <string>customText</string>
            <key>defaults</key>
            <string>com.example.testbedtweak</string>
            <key>default</key>
            <string>Hello from Dopamine</string>
            <key>placeholder</key>
            <string>Enter text</string>
        </dict>
        <dict>
            <key>cell</key>
            <string>PSGroupCell</string>
            <key>footerText</key>
            <string>Changes apply immediately without respring.</string>
        </dict>
    </array>
    <key>title</key>
    <string>TestbedTweak</string>
</dict>
</plist>
```

### 9.5. prefs/Resources/Info.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleDevelopmentRegion</key>
    <string>en</string>
    <key>CFBundleExecutable</key>
    <string>TestbedPrefs</string>
    <key>CFBundleIdentifier</key>
    <string>com.example.testbedprefs</string>
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    <key>CFBundleName</key>
    <string>TestbedPrefs</string>
    <key>CFBundlePackageType</key>
    <string>BNDL</string>
    <key>NSPrincipalClass</key>
    <string>TestbedPrefsListController</string>
</dict>
</plist>
```

### 9.6. prefs/entry.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>entry</key>
    <dict>
        <key>bundle</key>
        <string>TestbedPrefs</string>
        <key>cell</key>
        <string>PSLinkCell</string>
        <key>detail</key>
        <string>TestbedPrefsListController</string>
        <key>isController</key>
        <true/>
        <key>label</key>
        <string>TestbedTweak</string>
    </dict>
</dict>
</plist>
```

## 10. Сборка и установка

Из корня `TestbedTweak/`:

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide FINALPACKAGE=1
```

В `packages/` появится:

```
com.example.testbedtweak_0.0.1_iphoneos-arm.deb
```

Проверка (глава IV.10):

```bash
dpkg-deb -c packages/*.deb
```

Должны увидеть:

```
./var/jb/Library/MobileSubstrate/DynamicLibraries/TestbedTweak.dylib
./var/jb/Library/MobileSubstrate/DynamicLibraries/TestbedTweak.plist
./var/jb/Library/PreferenceBundles/TestbedPrefs.bundle/
./var/jb/Library/PreferenceBundles/TestbedPrefs.bundle/TestbedPrefs
./var/jb/Library/PreferenceBundles/TestbedPrefs.bundle/Root.plist
./var/jb/Library/PreferenceBundles/TestbedPrefs.bundle/Info.plist
./var/jb/Library/PreferenceLoader/Preferences/TestbedTweak.plist
```

Копируем на устройство и устанавливаем:

```bash
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip>
sudo dpkg -i /tmp/com.example.testbedtweak_0.0.1_iphoneos-arm.deb
```

## 11. Процедура проверки

### 11.1. Базовый тест

1. Запустите `TestbedApp` на устройстве.
2. Увидите на экране текст **«Hello from Dopamine»** (вместо «Original Label»). Это значит, хук сработал.

Если видите «Original Label» — твик не загрузился. Диагностика:

```bash
ssh mobile@<ip>
log stream --predicate 'subsystem == "com.example.testbedtweak"'
```

Если лога нет вообще — проверьте filter plist и `dpkg -l | grep testbedtweak`.

### 11.2. Тест настроек

1. Откройте Settings.app → прокрутите вниз → увидите **«TestbedTweak»**.
2. Нажмите → вы в прежнем списке с переключателем «Enabled» (включен) и полем «Custom text».
3. Введите в поле «My custom text» и нажмите «Done».
4. Переключитесь обратно в `TestbedApp` → нажмите кнопку «Reload» → лейбл теперь показывает «My custom text».

Если изменение не подхватилось — Darwin notification не прошёл; проверьте, что в твике вызывается `reloadPrefs()` после уведомления.

### 11.3. Тест выключения

1. В Settings.app → TestbedTweak → переключите «Enabled» в off.
2. В `TestbedApp` нажмите «Reload» → лейбл показывает «Original Label».

Если лейбл всё ещё подменён — проверьте код `%hook UILabel setText:`, в самом начале должна быть ветка `if (!gEnabled) { %orig(text); return; }`.

## 12. Типовые ошибки

| Ошибка | Причина | Решение |
|---|---|---|
| После установки `.deb` → твик не загружается | Filter plist не ловит `com.example.testbed` | Проверьте точный bundle id через `plutil -p .../Info.plist` |
| Лейбл не меняется, в логе ничего нет | `%ctor` не отрабатывает | ellekit не установлен? `dpkg -l | grep ellekit` |
| Лейбл меняется, но настройки не действуют | Darwin notification не доходит | Проверьте, что ваши `notify_post`/`notify_register_dispatch` используют одно и то же имя |
| Приложение падает при запуске | `UILabel.setText:` подменяется для внутренних UIKit лейблов | Проверка `[text isEqualToString:@"Original Label"]` должна быть первой |
| Settings.app → TestbedTweak не появляется | PreferenceLoader не установлен или entry.plist не на месте | `dpkg -l | grep preferenceloader`; `ls /var/jb/Library/PreferenceLoader/Preferences/` |

## 13. Отключение и откат

Полное удаление:

```bash
ssh mobile@<ip>
sudo dpkg -r com.example.testbedtweak
# или через Sileo/Zebra -> удалить
killall -9 TestbedApp Preferences
```

После `dpkg -r`:
- `.dylib`, `.plist`, `.bundle` удаляются из jbroot.
- Файл с настройками (`com.example.testbedtweak.plist` в cfprefsd) **остаётся**. Это ок — при переустановке он подхватится обратно.

Если нужно полностью очистить:

```bash
sudo rm /var/jb/var/mobile/Library/Preferences/com.example.testbedtweak.plist
# или через настоящий путь на RootHide (через libroot_get_jbroot_prefix)
```

Для простоты, если устройство ваше — просто удалите `.plist` через Filza/iFile.

## 14. Что подтверждено

- **Подтверждено Theos**: `SUBPROJECTS += prefs` + `aggregate.mk` — стандартный способ собрать твик + PreferenceBundle в одном `.deb`.
- **Подтверждено Apple docs**: `PSListController`, `PSSwitchCell`, `PSEditTextCell`, `CFPreferences*`, `notify_post/notify_register_dispatch`.
- **Подтверждено проектом**: cfprefsd автоматически перенаправляет не-apple preferences в rootless-пространство (глава III.9 про `BaseBin/rootlesshooks/cfprefsd.x`).
- **Требует проверки на устройстве**: что реальная сборка собирается без warning'ов и устанавливается на A13+iOS 26.0/26.0.1.
- **Не подтверждено**: что конкретные `PSCellTypes` (`PSEditTextCell`, `PSSwitchCell`) выглядят идентично на iOS 26 как на 18 — Apple периодически меняет визуальный стиль.

## 15. Выводы

Главный тезис: **целостный твик состоит из четырёх вещей**: твик-dylib + filter plist + PreferenceBundle + Darwin notification. Если это освоено на простом примере с `UILabel`, то для сложных задач остаётся только найти правильные `%hook` цели.

Этот проект — ваш «hello world» для твикописания. От него можно расширяться:

- Больше настроек (слайдеры, segmented controls).
- Больше хуков (разные классы).
- Собственные ресурсы (иконка твика в Settings.app).
- Логика «работает только в определённые часы».

## 16. Что дальше

- Проект №2: безопасный SpringBoard-твик — глава IV.12.
- Проект №3: системная подсистема с риском — глава IV.13.
- Патчи — часть V.
- Для шпаргалки по синтаксису Logos — `cheatsheets/01-logos-syntax.md`.
