# Продвинуто 6. Форматы пакетов: `.deb` vs `.tipa` vs `.ipa`

Три формата, которые встречаются в jailbreak-экосистеме. Структура, назначение, когда что использовать.

## 1. Три формата — обзор

| Формат | Что внутри | Устанавливается через | Для |
|---|---|---|---|
| **`.deb`** | Debian package: metadata + files | `dpkg -i`, Sileo, Zebra | Tweaks, CLI утилиты, система |
| **`.tipa`** | TrollStore IPA — обёртка над `.ipa` | TrollStore | Apps, которые требуют permasigning |
| **`.ipa`** | iPhone App — zip с .app | App Store, Sideloadly, Xcode | Обычные приложения |

## 2. `.deb` — главный формат твиков

### 2.1. Структура

```bash
dpkg-deb -c my.deb
# ./DEBIAN/control            — metadata (plain text key-value)
# ./DEBIAN/preinst            — script (optional)
# ./DEBIAN/postinst           — script (optional)
# ./DEBIAN/prerm              — script (optional)
# ./DEBIAN/postrm             — script (optional)
# ./DEBIAN/md5sums            — checksums (optional)
# ./var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.dylib
# ./var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.plist
# ...
```

Внутренне `.deb` — архив `ar` с двумя `.tar.gz` (или `.tar.xz`, `.tar.zst`):

```bash
ar t my.deb
# debian-binary
# control.tar.gz           — DEBIAN/*
# data.tar.xz              — файлы системы (./var/jb/...)
```

### 2.2. Создание

Через Theos — автоматически (`make package`).

Вручную:

```bash
# Структура:
pkg/
├── DEBIAN/
│   └── control
└── var/jb/Library/MobileSubstrate/DynamicLibraries/
    ├── MyTweak.dylib
    └── MyTweak.plist

# Собрать:
dpkg-deb --root-owner-group -b pkg mytweak_1.0.0_iphoneos-arm.deb
```

### 2.3. Установка

```bash
dpkg -i mytweak_1.0.0_iphoneos-arm.deb      # команда
```

Или через GUI (Sileo, Zebra).

### 2.4. Depends resolution

Если в control:

```
Depends: ellekit | mobilesubstrate, libroot-dopamine (>= 1.0.1)
```

dpkg проверит:

1. `libroot-dopamine` версии ≥ 1.0.1 установлен? Иначе — error.
2. `ellekit` ИЛИ `mobilesubstrate` установлен? Иначе — error.

Для Dopamine на устройстве всё это обычно уже есть.

### 2.5. Удаление

```bash
dpkg -r com.example.mytweak           # remove, оставить настройки
dpkg --purge com.example.mytweak       # remove полностью
```

Запускает prerm → удаляет файлы → postrm (глава V.5 про скрипты).

## 3. `.tipa` — TrollStore pack

TrollStore — утилита для установки `.ipa` без удаления через 7 дней (standard sideload). Работает на устройствах, которые уязвимы к `CoreTrust` bug (`iOS 14.0–16.6.1, 17.0`).

`.tipa` — специальный `.ipa` с особыми правилами подписи, подходящими для TrollStore.

### 3.1. Структура

Фактически `.tipa` — то же, что `.ipa`:

```
Payload/
└── MyApp.app/
    ├── MyApp         — binary
    ├── Info.plist
    ├── embedded.mobileprovision  (может отсутствовать)
    └── _CodeSignature/
        └── CodeResources
```

Отличие — в подписи: TrollStore permasigns через конкретный trick, который .tipa предполагает.

### 3.2. Создание через Theos

```make
TARGET := iphone:clang:latest:14.0
ARCHS = arm64

include $(THEOS)/makefiles/common.mk

APPLICATION_NAME = MyApp
MyApp_FILES = main.m AppDelegate.m
MyApp_FRAMEWORKS = UIKit Foundation

include $(THEOS_MAKE_PATH)/application.mk
```

`make package` создаст `.ipa`. Для TrollStore — переименовать в `.tipa` или использовать специальный флаг.

### 3.3. Для jailbreak + Dopamine

TrollStore и Dopamine **сосуществуют**:

- Dopamine устанавливается через TrollStore (`.tipa`).
- Tweaks на устройство — через `.deb` (Sileo/Zebra).
- Standalone apps с расширенными правами — через TrollStore (`.tipa`).

Автор твика может опубликовать:

- Основной твик — `.deb`.
- Дополнительное приложение-companion — `.tipa` (чтобы оно работало даже без jailbreak).

## 4. `.ipa` — стандартный App Store формат

### 4.1. Структура — как `.tipa`

```
Payload/
└── MyApp.app/
    └── ...
iTunesArtwork             — иконка 512x512 (optional)
iTunesMetadata.plist     — App Store metadata (optional)
```

### 4.2. Создание

Xcode → Archive → Export as .ipa.

Или через `xcodebuild`:

```bash
xcodebuild -scheme MyApp -configuration Release archive \
    -archivePath MyApp.xcarchive
xcodebuild -exportArchive -archivePath MyApp.xcarchive \
    -exportPath export \
    -exportOptionsPlist ExportOptions.plist
```

### 4.3. Установка

- **App Store** — через TestFlight или release.
- **Sideloadly** — через Apple ID (requires reinstall каждые 7 дней на бесплатном аккаунте).
- **TrollStore** — permasigning.
- **MDM** — enterprise.

## 5. Таблица «что выбирать»

| Задача | Формат |
|---|---|
| Tweak для jailbroken device | **.deb** |
| CLI tool для jailbroken device | **.deb** |
| App для jailbroken device, но не tweak | **.deb** с `_INSTALL_PATH = /Applications` |
| App для sideload без jailbreak | **.ipa** |
| App для permasigning (TrollStore) | **.tipa** |
| Dopamine сам | **.tipa** (устанавливается через TrollStore) |

## 6. Распаковка на хосте

```bash
# .deb
dpkg-deb -x my.deb /tmp/x

# .ipa или .tipa
unzip my.ipa -d /tmp/x    # это zip-архив
ls /tmp/x/Payload/
```

## 7. Инспекция подписи

Для `.deb` — codesign каждого .dylib:

```bash
dpkg-deb -x my.deb /tmp/x
find /tmp/x -name "*.dylib" -exec codesign -dv {} \;
```

Для `.ipa`/`.tipa` — codesign .app:

```bash
unzip my.ipa -d /tmp/x
codesign -dvvv /tmp/x/Payload/MyApp.app
```

## 8. Multi-arch

- `.deb` — хранит fat Mach-O (arm64 + arm64e внутри).
- `.ipa` — то же.
- Theos автоматически делает fat при `ARCHS = arm64 arm64e`.

Проверка:

```bash
lipo -archs /tmp/x/Payload/MyApp.app/MyApp
# arm64 arm64e
```

## 9. Поле Architecture в control

| Значение | Для |
|---|---|
| `iphoneos-arm` | Rootless/RootHide на arm64/arm64e (условно, по factial content) |
| `iphoneos-arm64` | Explicit arm64 only |
| `iphoneos-arm64e` | Explicit arm64e only |

Для Theos с `THEOS_PACKAGE_SCHEME=roothide` — обычно `iphoneos-arm`.

## 10. Что подтверждено

- **Подтверждено Debian Policy**: формат `.deb`, control fields.
- **Подтверждено Apple docs**: формат `.ipa`.
- **Подтверждено практикой**: `.tipa` — TrollStore community convention.
- **Подтверждено**: `make update tipa` в корневом Makefile Dopamine использует `.tipa` формат.
- **Требует проверки**: подробности TrollStore permasigning механики — вне scope этой книги.

## 11. Выводы

Главный тезис: **твик = `.deb`, standalone-app = `.ipa`/`.tipa`**. Для 99% ваших задач — `.deb`.

## 12. См. также

- Шпаргалка 3 — control-файл.
- Глава IV.10 — сборка и инспекция.
- Advanced 7 — TrollStore vs Dopamine.
