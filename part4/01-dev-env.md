# Глава IV.1. Воспроизводимая среда разработки (Theos + RootHide)

Эта глава — первая в практической части. Её задача — довести вашу машину до состояния «запустил `make`, получил `.deb`, скопировал на устройство, твик загрузился». Все следующие главы Части IV предполагают, что эта среда работает.

## 1. Что такое Theos

**Theos** — это набор Makefile-правил + препроцессор + инструменты упаковки для разработки iOS-твиков. По сути, Theos решает одну задачу: взять ваш `Tweak.x` + `Makefile` → получить `.deb` или `.tipa`, пригодный для установки на jailbroken-устройство.

Технические детали для понимания:

- Theos **не компилятор** — он вызывает Clang/Swift из iOS SDK.
- Theos **не оболочка** — он написан на Perl + Make, работает в shell.
- Theos **не менеджер пакетов** — он только создаёт .deb; установка на устройство делается через `dpkg` или `Sileo`/`Zebra`.

Два варианта Theos:

- **Классический Theos** (github.com/theos/theos) — изначальный, поддерживает rootless, но не знает про RootHide-случайный путь.
- **roothide/Theos** (github.com/roothide/Theos) — fork, который добавляет `THEOS_PACKAGE_SCHEME=roothide`, корректно упаковывающий .deb под randomized jbroot.

**Для нашей целевой среды (iOS 26 + Dopamine RootHide на A13) нужен roothide/Theos.** Классический Theos собирает тwики, которые ломаются на RootHide из-за хардкода путей внутри .deb (постскрипты, пути установки).

## 2. Платформы разработки

Theos работает:

- **macOS** (нативный, рекомендованный путь). Нужен Xcode (для iPhoneOS SDK) или отдельный `iPhoneOS<ver>.sdk`.
- **Linux** (поддерживается). Нужны отдельно скачанный iOS SDK, `clang`, `ldid`, `dpkg-deb`.
- **Windows через WSL** — работает по тем же правилам, что Linux.
- **На самом jailbroken устройстве** — технически возможно через `iDownload` + установку Theos на iOS, но для продакшена не рекомендуется.

В этой книге рассматриваем macOS и Linux. Разница в основном в том, где лежит SDK, остальное идентично.

## 3. Установка на macOS

### 3.1. Предварительные требования

- macOS 13.0 или новее (на Big Sur и старее могут быть проблемы с подписью).
- Xcode (из App Store или developer.apple.com). После установки — `xcode-select --install`.
- Homebrew (`https://brew.sh`). После установки — пакеты:

```bash
brew install ldid dpkg xz libarchive
```

- `ldid` — для ad-hoc подписи .dylib.
- `dpkg` — для сборки .deb.
- `xz` — для компрессии (чаще xz, иногда zstd).
- `libarchive` — для вспомогательных операций с архивами.

### 3.2. Клонирование roothide/Theos

```bash
git clone --recursive https://github.com/roothide/Theos.git ~/theos
export THEOS=~/theos
```

Строку `export THEOS=~/theos` добавьте в `~/.zshrc` (или `~/.bashrc`), чтобы переменная сохранялась между сессиями.

### 3.3. Проверка

```bash
cd /tmp
$THEOS/bin/nic.pl
```

Должен запуститься интерактивный мастер создания проекта. Если — работает — всё готово.

## 4. Установка на Linux

### 4.1. Пакеты

Для Ubuntu/Debian:

```bash
sudo apt update
sudo apt install build-essential perl git curl \
                 ldid dpkg-dev fakeroot
```

Для Arch:

```bash
sudo pacman -S base-devel perl git curl ldid dpkg fakeroot
```

### 4.2. SDK

Linux не имеет штатного SDK Apple. Скачайте:

- **iPhoneOS16.5.sdk** — минимально подходит для сборки под iOS 15.0+ (включая 26.0/26.0.1).
- SDK можно взять из публично доступных источников (теневой градус легитимности, но повсеместно используется).

Положите SDK в `$THEOS/sdks/`:

```bash
mkdir -p $THEOS/sdks
# ... скопировать iPhoneOS16.5.sdk сюда ...
```

Theos автоматически найдёт SDK в этом каталоге.

### 4.3. Клонирование roothide/Theos

Аналогично macOS:

```bash
git clone --recursive https://github.com/roothide/Theos.git ~/theos
export THEOS=~/theos
```

### 4.4. Проверка

```bash
cd /tmp
$THEOS/bin/nic.pl
```

## 5. Проверочная сборка

Самый быстрый тест: создать пустой tweak и собрать.

```bash
cd /tmp
$THEOS/bin/nic.pl
# Выбрать Template: tweak (option с номером, обычно 15 или около)
# Project Name: TestTweak
# Package Name: com.example.testtweak
# Author/Maintainer: your name
# [iphone/tv/mac]: iphone (по умолчанию)
# MobileSubstrate Bundle filter: com.apple.springboard
# [...]
```

После генерации:

```bash
cd TestTweak
make package THEOS_PACKAGE_SCHEME=roothide
```

Если команда проходит без ошибок, в текущем каталоге появится файл `com.example.testtweak_0.0.1_iphoneos-arm64.deb` (или с таким же названием, но меньшей версией в начале). Это первый .deb вашего авторства.

## 6. Что такое `THEOS_PACKAGE_SCHEME`

**Критически важная переменная.** Она управляет тем, как собирается и упаковывается ваш твик.

| Значение | Что делает | Для чего |
|---|---|---|
| не задано | Классический packaging (путь `/Library/MobileSubstrate/...`) | Старые jailbreak (до iOS 15) |
| `rootless` | Пути `/var/jb/Library/MobileSubstrate/...` | Rootless jailbreak (iOS 15–17), простой Dopamine |
| `roothide` | Пути учитывают randomized jbroot | **RootHide Dopamine (наш целевой)** |

Для книги мы **всегда** указываем `THEOS_PACKAGE_SCHEME=roothide`. Это можно задать:

- через переменную среды: `export THEOS_PACKAGE_SCHEME=roothide` в `.zshrc`;
- в командной строке: `make package THEOS_PACKAGE_SCHEME=roothide`;
- в самом `Makefile` проекта: `THEOS_PACKAGE_SCHEME = roothide` отдельной строкой.

Последний вариант — рекомендованный: твик помечен как «предназначен для RootHide», и забыть флаг невозможно.

## 7. Хуковые библиотеки — что устанавливать на устройство

Theos не включает в себя рантайм хуков. На устройстве должен быть установлен один из менеджеров:

| Менеджер | Статус | Пакет | Комментарий |
|---|---|---|---|
| **Ellekit** | Рекомендован | `ellekit` | PAC-aware, активно поддерживается, работает с rootless/RootHide |
| **MobileSubstrate** | Legacy | `mobilesubstrate` | Классический, не рекомендован для iOS 15+ |
| **Substitute** | Legacy | `substitute` | Промежуточный, редко встречается на iOS 26 |

На Dopamine RootHide из коробки устанавливается **Ellekit**. В `control` вашего твика указываете:

```
Depends: ellekit | mobilesubstrate | substitute
```

Это даёт Sileo/Zebra возможность выбрать установленный менеджер.

## 8. Как установить .deb на устройство

Три способа:

### 8.1. Через Sileo/Zebra

Самый простой. Поместите .deb в `/var/mobile/Documents/` (через AirDrop, iCloud или Files.app), откройте в Sileo/Zebra, установите.

### 8.2. Через SSH + dpkg

Нужен включённый SSH (пакет `openssh`) на устройстве. Пароль по умолчанию — `alpine`, рекомендуется сменить.

```bash
scp com.example.testtweak_0.0.1_iphoneos-arm64.deb mobile@<ip>:/tmp/
ssh mobile@<ip>
sudo dpkg -i /tmp/com.example.testtweak_0.0.1_iphoneos-arm64.deb
```

### 8.3. Через `make install` в Theos

Theos умеет это сделать за вас, если задан target `THEOS_DEVICE_IP`:

```bash
export THEOS_DEVICE_IP=<ip-телефона>
make package install
```

Theos сам сделает `scp` + `ssh dpkg -i`. Полезно для быстрой итерации.

Важно: после установки обычно нужен **respring** (перезапуск SpringBoard) или перезапуск целевого приложения — глава II.5 объясняет жизненный цикл.

## 9. Проверочный чеклист

Прежде чем идти в главу IV.2, убедитесь:

| Пункт | Как проверить |
|---|---|
| Theos установлен и `$THEOS` в среде | `echo $THEOS` показывает путь |
| `nic.pl` работает | команда запускается, предлагает шаблоны |
| SDK найден | Theos при сборке не пишет «iphoneos SDK not found» |
| `ldid` в PATH | `which ldid` показывает путь |
| `dpkg-deb` в PATH | `which dpkg-deb` показывает путь |
| На устройстве установлен Dopamine | SpringBoard запускается после Jailbreak нажатия |
| На устройстве установлен ellekit (или аналог) | в Sileo виден установленный пакет |
| SSH работает | `ssh mobile@<ip>` без пароля (с ключом) или с паролем |

Если хотя бы один пункт не выполнен — остановитесь и исправьте, иначе ошибки позже будут непонятны.

## 10. Что подтверждено и что требует проверки

- **Подтверждено документацией**: `THEOS_PACKAGE_SCHEME=roothide` — переменная из roothide/Theos; информация взята из README-варианта repo.
- **Подтверждено по RouuKRoo/Dopamine-roothide BUILD.md**: сборка на github actions — альтернативный путь получить .ipa без локальной среды. Это полезно, если вы не хотите ставить Theos сами, но для разработки твика Theos всё равно нужен.
- **Требует проверки на устройстве**: что ваш первый пустой tweak реально загружается в SpringBoard — проверяется respring и просмотром `log stream --process SpringBoard`.
- **Не подтверждено**: прямых указаний «именно Theos версия X.Y работает с iOS 26» в репозитории нет; используйте последнюю доступную версию roothide/Theos.

## 11. Выводы

Главный тезис: `THEOS_PACKAGE_SCHEME=roothide` — ключевой флаг. Без него на RootHide Dopamine ваш .deb не распакуется в правильные пути, и твик не загрузится.

Остальная среда — рутинная: клонируете roothide/Theos, ставите `ldid`+`dpkg-deb`, скачиваете SDK (или получаете из Xcode), и у вас есть всё, что нужно для сборки.

## 12. Что дальше

- Создание минимального проекта с полным разбором Makefile/control/filter plist — глава IV.2.
- Что писать в filter plist и как выбирать процесс — глава IV.3.
- Первый Objective-C хук в собственном приложении — глава IV.4.
- Подробности сборки `.deb` и инспекция результата — глава IV.10.
