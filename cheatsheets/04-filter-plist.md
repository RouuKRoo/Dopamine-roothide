# Шпаргалка 4. Filter plist — все варианты

Filter plist — файл `<TweakName>.plist` рядом с `<TweakName>.dylib`, который TweakLoader читает, чтобы решить, в какие процессы загружать ваш твик.

## Четыре условия (keys)

| Ключ | Проверка |
|---|---|
| `Bundles` | Bundle ID процесса/расширения |
| `Classes` | Наличие Objective-C класса в адресном пространстве |
| `Executables` | Имя исполнимого файла (basename, без пути) |
| `CoreFoundationVersion` | Диапазон версий CF (min, max) |

Все заданные условия должны выполниться одновременно (AND), если не задан `Mode: Any`.

## Минимальный пример

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

Это грузит твик только в приложение `com.example.testbed`.

## Варианты

### 1. Одно приложение + его расширения

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
        <string>com.example.testbed.ShareExtension</string>
        <string>com.example.testbed.NotificationService</string>
        <string>com.example.testbed.Intents</string>
    </array>
</dict>
```

### 2. SpringBoard

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.apple.springboard</string>
    </array>
</dict>
```

### 3. Settings.app

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.apple.Preferences</string>
    </array>
</dict>
```

### 4. Демон без bundle id

```xml
<key>Filter</key>
<dict>
    <key>Executables</key>
    <array>
        <string>debugserver</string>
    </array>
</dict>
```

### 5. Все UIKit приложения

```xml
<key>Filter</key>
<dict>
    <key>Classes</key>
    <array>
        <string>UIApplication</string>
    </array>
</dict>
```

**ВНИМАНИЕ**: это широкий фильтр. Твик загрузится во все приложения с UIKit. Используйте осторожно (глава IV.3).

### 6. Cutoff по версии iOS

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
    <key>CoreFoundationVersion</key>
    <array>
        <real>1800.0</real>
        <real>9999.0</real>
    </array>
</dict>
```

Пара (min, max). Твик загружается, только если `CFCoreFoundationVersionNumber` в диапазоне.

### 7. Комбинированный

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.apple.springboard</string>
    </array>
    <key>Classes</key>
    <array>
        <string>SBIconView</string>
    </array>
    <key>CoreFoundationVersion</key>
    <array>
        <real>1800.0</real>
        <real>9999.0</real>
    </array>
</dict>
```

Загрузка только если:
- процесс — SpringBoard **и**
- класс `SBIconView` присутствует **и**
- CF в диапазоне.

### 8. Mode: Any (OR вместо AND)

```xml
<key>Filter</key>
<dict>
    <key>Mode</key>
    <string>Any</string>
    <key>Bundles</key>
    <array>
        <string>com.apple.springboard</string>
    </array>
    <key>Executables</key>
    <array>
        <string>debugserver</string>
    </array>
</dict>
```

Твик загружается если либо bundle = `com.apple.springboard`, либо executable = `debugserver`.

## Таблица CFCoreFoundationVersion (ориентировочно)

Точные значения требуют проверки на устройстве.

| iOS | CFVersion (приблизительно) |
|---|---|
| 15.0 | 1800 |
| 15.1–15.7 | 1830–1860 |
| 16.0 | ~1900 |
| 16.1–16.6 | 1910–1950 |
| 17.0 | ~2000 |
| 17.1–17.6 | 2010–2050 |
| 18.0 | ~2100 |
| 18.1–18.7 | 2110–2150 |
| 26.0 / 26.0.1 | **требует проверки** |

Чтобы снять с устройства:

```c
#include <CoreFoundation/CoreFoundation.h>
printf("%.1f\n", CFCoreFoundationVersionNumber);
```

Собрать + запустить на устройстве; выведется число.

## Как узнать bundle id

### На устройстве

```bash
ssh mobile@<ip>
# Для системных:
plutil -p /Applications/MobileSafari.app/Info.plist | grep BundleIdentifier
# Для пользовательских:
plutil -p /var/containers/Bundle/Application/*/Example.app/Info.plist | grep BundleIdentifier
```

### Через ideviceinstaller на хосте

```bash
ideviceinstaller --list-apps
```

### Через log stream

```bash
log stream --device <udid> --predicate 'eventMessage CONTAINS "some keyword"'
# Строки процесса показывают его identifier
```

## Как узнать executable name

```bash
# Для системных демонов:
plutil -p /System/Library/LaunchDaemons/com.apple.cfprefsd.xpc.daemon.plist \
    | grep -A2 ProgramArguments
```

Или `ps -A` на устройстве.

## Таблица частых целей

| Что хотите хукнуть | Bundles / Executables |
|---|---|
| Своё приложение | `com.example.testbed` |
| SpringBoard | `com.apple.springboard` |
| Settings | `com.apple.Preferences` |
| Safari | `com.apple.mobilesafari` |
| App Store | `com.apple.AppStore` |
| Messages | `com.apple.MobileSMS` |
| Phone | `com.apple.mobilephone` |
| Maps | `com.apple.Maps` |
| Camera | `com.apple.camera` |
| Photos | `com.apple.mobileslideshow` |
| Music | `com.apple.Music` |
| Mail | `com.apple.mobilemail` |
| Calendar | `com.apple.mobilecal` |
| backboardd | `Executables: backboardd` |
| cfprefsd | `Executables: cfprefsd` |
| lsd | `Executables: lsd` |
| apsd | `Executables: apsd` |
| nehelper | `Executables: nehelper` |
| debugserver | `Executables: debugserver` |

## Что НЕ использовать

| Паттерн | Почему плохо |
|---|---|
| `Classes = NSObject` | **Всё** на iOS — subclass NSObject. Twic загрузится во все процессы. |
| `Classes = UIApplication` **без** Bundles | Все UIKit приложения. Один баг — 20 крэшей. |
| Без filter plist вовсе | Твик загрузится во все процессы системы. Критично. |
| `Bundles = com.apple.*` без указания конкретного | Невалидный синтаксис. |

## Проверка собранного .deb

```bash
dpkg-deb -c MyTweak.deb | grep .plist
# ./var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.plist

dpkg-deb -x MyTweak.deb /tmp/x
plutil -p /tmp/x/var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.plist
```

Содержимое должно быть читаемо и корректно.

## См. также

- Глава IV.3 — выбор процесса и фильтры.
- Шпаргалка 2 — Makefile (TWEAK_NAME должен совпадать с filter plist).
- Для диагностики «твик не загружается» — глава VII.1.
