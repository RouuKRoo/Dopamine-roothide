# Шпаргалка 2. Переменные Theos Makefile

Все переменные, которые встречаются в Theos твик-проекте, с описанием и примерами.

## Глобальные (ставить до include common.mk)

| Переменная | Пример | Назначение |
|---|---|---|
| `TARGET` | `iphone:clang:latest:15.0` | Платформа:компилятор:SDK:min_version |
| `ARCHS` | `arm64 arm64e` | Целевые архитектуры (пробел — fat) |
| `SDKVERSION` | `16.5` | Явная версия SDK (обычно `latest` достаточно) |
| `DEPLOYMENT_TARGET` | `15.0` | `-miphoneos-version-min` |
| `INSTALL_TARGET_PROCESSES` | `SpringBoard` | Процессы для `killall -9` после `make install` |
| `THEOS_PACKAGE_SCHEME` | `roothide` | **Обязательно** для RootHide |
| `FINALPACKAGE` | `1` | Production сборка (O2 + strip) |
| `DEBUG` | `1` | Debug сборка (O0 + symbols) |
| `ARC` | `1` | Включить ARC глобально (устарело; используйте `_CFLAGS`) |
| `GO_EASY_ON_ME` | `1` | Пропускать warning'и |

## Project-level

```make
TWEAK_NAME = MyTweak
TOOL_NAME = my-cli-tool
LIBRARY_NAME = libmystuff
BUNDLE_NAME = MyPrefs
APPLICATION_NAME = MyApp
```

Один проект имеет **один** из этих типов (TWEAK/TOOL/LIBRARY/BUNDLE/APPLICATION).

Множественные типы — через `SUBPROJECTS`:

```make
SUBPROJECTS += prefs
SUBPROJECTS += helper
include $(THEOS_MAKE_PATH)/aggregate.mk
```

## Per-target переменные (с префиксом имени)

| Переменная | Пример | Назначение |
|---|---|---|
| `<NAME>_FILES` | `MyTweak_FILES = Tweak.x helper.m` | Исходники |
| `<NAME>_CFLAGS` | `-fobjc-arc -Wall` | C/ObjC флаги |
| `<NAME>_CCFLAGS` | `-std=c++17` | C++ флаги |
| `<NAME>_LDFLAGS` | `-Wl,-rpath,/var/jb` | Linker флаги |
| `<NAME>_LIBRARIES` | `root ellekit` | Линковка `-l<lib>` |
| `<NAME>_FRAMEWORKS` | `UIKit Foundation` | Публичные frameworks |
| `<NAME>_PRIVATE_FRAMEWORKS` | `SpringBoardFoundation` | Private frameworks |
| `<NAME>_WEAK_FRAMEWORKS` | `MetricKit` | Weak-linked frameworks |
| `<NAME>_WEAK_LIBRARIES` | `libnew` | Weak libraries |
| `<NAME>_INSTALL_PATH` | `/Library/PreferenceBundles` | Куда класть в .deb (для bundle/tool) |
| `<NAME>_RESOURCE_DIRS` | `Resources` | Папка ресурсов для bundle |
| `<NAME>_ENTRY_POINT` | `main` | Для tool/application entry (устарело) |
| `<NAME>_CODESIGN_FLAGS` | `-Sent.xml` | Флаги для ldid |
| `<NAME>_FILTER` | `custom.plist` | Нестандартное имя filter plist |

## Пример для RootHide-твика

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = MyTweak

MyTweak_FILES = Tweak.x helper.m
MyTweak_CFLAGS = -fobjc-arc -Wall -Wno-deprecated-declarations
MyTweak_LDFLAGS = -Wl,-rpath,/var/jb
MyTweak_LIBRARIES = root
MyTweak_FRAMEWORKS = UIKit Foundation
MyTweak_PRIVATE_FRAMEWORKS = SpringBoardFoundation

include $(THEOS_MAKE_PATH)/tweak.mk
SUBPROJECTS += prefs
include $(THEOS_MAKE_PATH)/aggregate.mk
```

## Переменные окружения / CLI

Передаются через `make VAR=value` или `export VAR=value`:

| Переменная | Назначение |
|---|---|
| `THEOS` | Путь к Theos (обычно `~/theos`) |
| `THEOS_DEVICE_IP` | IP устройства для `make install` |
| `THEOS_DEVICE_PORT` | SSH порт (default 22) |
| `THEOS_DEVICE_USER` | SSH user (default root или mobile) |
| `THEOS_PLATFORM_SDK_ROOT` | Переопределить путь SDK |
| `THEOS_MAKE_PATH` | `$(THEOS)/makefiles` |
| `SYSROOT` | Альтернативный sysroot |

## Цели (`make <target>`)

| Target | Что делает |
|---|---|
| `make` или `make all` | Собрать |
| `make clean` | Очистить `.theos/obj/` |
| `make package` | Собрать + упаковать в .deb |
| `make install` | Собрать + установить через SSH |
| `make package install` | То же, что `install` явно |
| `make do` | `package install` (shortcut) |
| `make distclean` | Очистить всё, включая `packages/` |

## Advanced: условная сборка

```make
ifeq ($(FINALPACKAGE),1)
    MyTweak_CFLAGS += -DNDEBUG
else
    MyTweak_CFLAGS += -DDEBUG=1 -g
endif

ifeq ($(THEOS_PACKAGE_SCHEME),roothide)
    MyTweak_CFLAGS += -DBUILD_ROOTHIDE=1
endif
```

## Отладка Makefile

```bash
# Увидеть все расширенные переменные:
make -n            # dry run
make V=1           # verbose
make SHELL='sh -x' # trace shell commands

# Проверить обработку конкретной цели:
make -d <target> 2>&1 | less
```

## См. также

- Theos docs в `$THEOS/vendor/`.
- Шпаргалка 3 — control-файл.
- Шпаргалка 4 — filter plist.
- Глава IV.1, IV.2.
