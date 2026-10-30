# Глава IV.2. Минимальный Theos-проект: Makefile, control, filter plist

После установки среды (глава IV.1) следующий шаг — понять, из чего состоит минимальный Theos-проект, и зачем нужен каждый файл. Это глава-разбор-по-косточкам, после которой `nic.pl`-шаблон перестанет быть «магией», а станет понятной структурой, которую легко модифицировать под свои нужды.

## 1. Дерево файлов пустого проекта

После `nic.pl` → выбор шаблона `tweak` + ввод параметров вы получаете примерно такое дерево:

```
HelloTweak/
├── Makefile
├── HelloTweak.plist          — filter plist
├── Tweak.x                   — исходник хука (или Tweak.xm для ObjC++)
└── control                   — метаданные Debian-пакета
```

Четыре файла. Разберём каждый.

## 2. Makefile — связывает всё вместе

Содержимое после `nic.pl`:

```make
TARGET := iphone:clang:latest:15.0
INSTALL_TARGET_PROCESSES = SpringBoard

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = HelloTweak

HelloTweak_FILES = Tweak.x
HelloTweak_CFLAGS = -fobjc-arc

include $(THEOS_MAKE_PATH)/tweak.mk
```

Для RootHide доработаем:

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = SpringBoard
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = HelloTweak

HelloTweak_FILES = Tweak.x
HelloTweak_CFLAGS = -fobjc-arc
HelloTweak_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

Разбор каждой строки:

### `TARGET := iphone:clang:latest:15.0`

- `iphone` — целевая платформа.
- `clang` — компилятор.
- `latest` — какой SDK использовать (из установленных Theos берёт самый новый).
- `15.0` — `-miphoneos-version-min` (минимальная поддерживаемая версия iOS). Для нашей цели (iOS 26) — 15.0 оптимально: совместимо и с iOS 15+, и c iOS 26.

Если вам нужно использовать API только из iOS 17+ (например, SwiftData, новые виджеты), поставьте `17.0`. Твик при этом не будет работать на iOS 15/16.

### `ARCHS = arm64 arm64e`

Сборка под оба slice. На A13 запускается arm64, но при запуске на A14+ (если переносите твик) — arm64e. Собирая оба, вы покрываете всё.

Если явно знаете, что twik — только для A13 — можно оставить `ARCHS = arm64`. .deb получится меньше.

### `INSTALL_TARGET_PROCESSES = SpringBoard`

Указывает, какой(ие) процесс(ы) Theos должен killall'ом перезапустить после `make install`. По умолчанию — SpringBoard, т. е. делается respring.

Если ваш твик нацелен на отдельное приложение, пишете:

```make
INSTALL_TARGET_PROCESSES = Safari
```

Theos вызовет `killall -9 Safari` на устройстве — что перезапустит приложение и загрузит твик.

Если целевых процессов нет (например, твик — только для своего приложения, которое вы вручную запускаете), оставьте `SpringBoard`.

### `THEOS_PACKAGE_SCHEME = roothide`

Главный флаг RootHide-совместимости. Критическая строка. См. главу IV.1.

### `include $(THEOS)/makefiles/common.mk`

Подключение общих правил Theos. Должно идти **до** определения `TWEAK_NAME`.

### `TWEAK_NAME = HelloTweak`

Имя продукта. Используется:

- для имени результирующего `.dylib` (`HelloTweak.dylib`).
- для именования переменных файлов (ниже — `HelloTweak_FILES`).
- для имени .deb (вместе с версией из `control`).

### `HelloTweak_FILES = Tweak.x`

Список исходных файлов. Varargs: `Tweak.x Something.m SomethingElse.cpp`.

Расширения:

- `.x` — Logos ObjC (препроцессор превращает в .m).
- `.xm` — Logos ObjC++ (превращает в .mm).
- `.c` — чистый C.
- `.m` — чистый ObjC.
- `.mm` — ObjC++.
- `.swift` — Swift (сложнее с настройкой, см. главу IV.7 для подробностей).

### `HelloTweak_CFLAGS = -fobjc-arc`

Флаги компилятора для именно этого таргета. `-fobjc-arc` — включить Automatic Reference Counting (почти всегда вам нужно).

Другие полезные:

- `-Wall -Wno-deprecated-declarations` — больше предупреждений, кроме deprecation (iOS имеет много deprecated API, что замусоривает лог).
- `-O2` — оптимизация (но осторожно с хуками, иногда ломает inline).

### `HelloTweak_LIBRARIES = root`

Линковка с `-lroot` — тот самый `libroot.dylib` из главы III.10. **Обязательно** для RootHide-совместимости.

Другие частые:

- `substrate` — классический substrate (Legacy; на ellekit не нужен, он совместим).
- `jailbreak` — `libjailbreak` для продвинутых jbclient_* функций.
- `objc` — для ObjC-рантайма (обычно подтягивается автоматически).

### `HelloTweak_FRAMEWORKS` и `HelloTweak_PRIVATE_FRAMEWORKS`

Эти переменные указывают нужные frameworks:

```make
HelloTweak_FRAMEWORKS = UIKit Foundation
HelloTweak_PRIVATE_FRAMEWORKS = SpringBoard SpringBoardFoundation
```

- Публичные — из `$SDK/System/Library/Frameworks/`.
- Приватные — из `$SDK/System/Library/PrivateFrameworks/`.

На `nic.pl`-шаблоне эти строки отсутствуют — Theos автоматически линкует с минимумом. Добавляйте явно, если ваш хук использует private классы (например, SpringBoard).

### `include $(THEOS_MAKE_PATH)/tweak.mk`

Финальная строка — подключение правил сборки твика. Должна быть последней.

## 3. control — Debian-метаданные пакета

Шаблон:

```
Package: com.example.hellotweak
Name: HelloTweak
Version: 0.0.1
Architecture: iphoneos-arm
Description: A hello tweak
Maintainer: Author Name
Author: Author Name
Section: Tweaks
Depends: ellekit | mobilesubstrate
```

Разбор полей:

### `Package: com.example.hellotweak`

Уникальный идентификатор пакета в reverse-DNS. Обязателен к уникальности в системе — два пакета с одним Package поссорятся при установке.

### `Name: HelloTweak`

Человекочитаемое имя, показываемое в Sileo/Zebra.

### `Version: 0.0.1`

Semver. Инкрементируйте при каждом релизе: `0.0.1 → 0.0.2 → 0.1.0 → 1.0.0`.

### `Architecture: iphoneos-arm`

Для rootless+roothide на arm64/arm64e — `iphoneos-arm`. Theos сам подставит правильное значение в зависимости от `THEOS_PACKAGE_SCHEME`:

- `rootless` → `iphoneos-arm64`
- `roothide` → `iphoneos-arm64`
- classic → `iphoneos-arm`

Можете оставить `iphoneos-arm` — Theos разберётся.

### `Description`

Короткое описание. В Sileo/Zebra показывается первой строкой.

Если нужно длинное описание — на второй и далее строках начинайте с ` ` (пробел):

```
Description: A hello tweak
 This tweak demonstrates the simplest possible hook.
 It replaces viewDidLoad logging in SpringBoard.
```

### `Depends: ellekit | mobilesubstrate`

Зависимости. Через `|` — альтернативы: любая из перечисленных подходит.

Полезные зависимости:

- `ellekit | mobilesubstrate | substitute` — хуковый менеджер.
- `firmware (>= 15.0)` — минимальная iOS.
- `firmware (>= 15.0) & firmware (<= 26.9)` — диапазон.
- `libroot-dopamine` — libroot. Если используете `MyTweak_LIBRARIES = root`.

Пример готового:

```
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

### Дополнительные поля

- `Section: Tweaks` — категория в репозитории. Для твиков — `Tweaks`.
- `Depiction: https://...` — URL с описанием (опционально).
- `Icon: file:///...` — иконка (опционально).
- `Priority: optional` — приоритет (почти всегда `optional`).

## 4. Filter plist — куда загружать твик

Файл `HelloTweak.plist` (имя — такое же, как `TWEAK_NAME`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Filter</key>
    <dict>
        <key>Bundles</key>
        <array>
            <string>com.apple.springboard</string>
        </array>
    </dict>
</dict>
</plist>
```

Разбор:

- **`Filter`** — верхний dict, обязательный.
- **`Bundles`** (массив строк) — список bundle id, в которые загружать твик. Пример выше ловит SpringBoard.
- Также можно указать:

### `Classes`

```xml
<key>Classes</key>
<array>
    <string>UIApplication</string>
</array>
```

Твик загружается в процесс только если в его адресном пространстве есть указанный ObjC-класс. Полезно для «загрузить в любое приложение с UIKit».

### `Executables`

```xml
<key>Executables</key>
<array>
    <string>SpringBoard</string>
</array>
```

Фильтр по имени исполняемого файла (не bundle id). Полезно для демонов, у которых нет bundle id.

### `CoreFoundationVersion`

```xml
<key>CoreFoundationVersion</key>
<array>
    <real>1800.0</real>
    <real>9999.0</real>
</array>
```

Диапазон (min, max) версий CoreFoundation. Твик загружается только в процессы с версией CF в диапазоне. Используется для cutoff по версии iOS.

Таблица примерных значений (ориентир):

| iOS | CFVersion |
|---|---|
| 15.0 | 1800 |
| 16.0 | 1900 |
| 17.0 | 1950 (точно требует проверки) |
| 26.0 | **требует проверки на устройстве** |

Для iOS 26 точное значение надо снимать с устройства через `log stream` или просто через `plutil -p /System/Library/CoreServices/SystemVersion.plist`.

### Все варианты вместе

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
        <string>com.example.testbed.ShareExt</string>
    </array>
    <key>Classes</key>
    <array>
        <string>UIApplication</string>
    </array>
    <key>CoreFoundationVersion</key>
    <array>
        <real>1800.0</real>
        <real>9999.0</real>
    </array>
</dict>
```

Все три условия должны выполниться одновременно (AND).

### Дополнительный ключ `Mode`

```xml
<key>Mode</key>
<string>Any</string>
```

Меняет AND на OR для `Bundles`/`Classes`/`Executables`. Нечастый случай.

## 5. Tweak.x — заготовка исходника

Минимум:

```objc
#import <Foundation/Foundation.h>

%ctor {
    NSLog(@"[HelloTweak] loaded in %@", [NSProcessInfo processInfo].processName);
}
```

Разбор:

- `#import <Foundation/Foundation.h>` — минимальный include.
- `%ctor` — Logos-макрос, который превращается в `__attribute__((constructor))` функцию.
- `NSLog(@"[HelloTweak] loaded in %@", ...)` — показывает, что твик успешно загрузился.

Этого хватит, чтобы увидеть в `log stream`:

```
2026-10-30 10:15:32.123 SpringBoard[123:456] [HelloTweak] loaded in SpringBoard
```

На устройстве. Если видите эту строку — твик загружается корректно, filter plist подходит, подпись принята.

## 6. Собираем и смотрим результат

```bash
cd HelloTweak
make clean package THEOS_PACKAGE_SCHEME=roothide
```

В каталоге `./packages/` или в корне проекта появится:

```
com.example.hellotweak_0.0.1_iphoneos-arm.deb
```

Содержимое — проверить:

```bash
dpkg-deb -c com.example.hellotweak_0.0.1_iphoneos-arm.deb
```

Для `roothide` схемы увидим что-то вроде:

```
./
./var/
./var/jb/
./var/jb/Library/
./var/jb/Library/MobileSubstrate/
./var/jb/Library/MobileSubstrate/DynamicLibraries/
./var/jb/Library/MobileSubstrate/DynamicLibraries/HelloTweak.dylib
./var/jb/Library/MobileSubstrate/DynamicLibraries/HelloTweak.plist
```

Пути внутри `.deb` указывают на `/var/jb/`, но при распаковке на RootHide-устройстве dpkg их развернёт в правильный randomized-jbroot (это делает `dopamine-basebin-link` + патчи dpkg в составе Dopamine).

## 7. Что в собранной `.dylib`

Инспекция собранного бинарника:

```bash
# Проверить что это fat binary (arm64 + arm64e)
file .theos/obj/debug/HelloTweak.dylib

# Посмотреть зависимости
otool -L .theos/obj/debug/HelloTweak.dylib

# Проверить install_name
otool -l .theos/obj/debug/HelloTweak.dylib | grep -A2 LC_ID_DYLIB
```

Правильный `otool -L` должен показать:

```
.theos/obj/debug/HelloTweak.dylib (architecture arm64):
    /var/jb/Library/MobileSubstrate/DynamicLibraries/HelloTweak.dylib (compatibility ...)
    @rpath/libroot.dylib (...)
    /System/Library/Frameworks/Foundation.framework/Foundation (...)
    /usr/lib/libobjc.A.dylib (...)
    /usr/lib/libSystem.B.dylib (...)
```

Если `@rpath/libroot.dylib` отсутствует — вы забыли `MyTweak_LIBRARIES = root` в Makefile.

## 8. Что подтверждено

- **Подтверждено документацией Theos** (roothide/Theos README): `THEOS_PACKAGE_SCHEME=roothide` — главный флаг для RootHide-совместимости.
- **Подтверждено практикой проекта**: libroot ставится как зависимость `libroot-dopamine` (видно в `DOBootstrapper.m:21`).
- **Требует проверки на устройстве**: конкретное значение `CoreFoundationVersion` для iOS 26.0 и 26.0.1 — читать с устройства.
- **Не подтверждено**: что любая версия `nic.pl` шаблона `tweak` работает идентично — Theos обновляется, шаблоны могут слегка меняться.

## 9. Выводы

Главный тезис: минимальный Theos-проект — это **четыре файла**: Makefile, control, filter plist, Tweak.x. Если каждый из них написан правильно, вы получаете `.deb`, который устанавливается на устройство и загружает ваш `%ctor`.

Критические строки, которые нельзя забывать:

- `THEOS_PACKAGE_SCHEME = roothide` в Makefile.
- `HelloTweak_LIBRARIES = root` в Makefile.
- `Depends: ..., libroot-dopamine` в control.
- Корректный `Bundles`/`Executables` в filter plist.

## 10. Что дальше

- Как правильно выбрать bundle id целевого процесса и писать filter plist — глава IV.3.
- Первый настоящий ObjC hook с `%hook`/`%orig` — глава IV.4.
- Детальная шпаргалка синтаксиса Logos — `cheatsheets/01-logos-syntax.md`.
- Детальная шпаргалка Makefile — `cheatsheets/02-theos-makefile.md`.
