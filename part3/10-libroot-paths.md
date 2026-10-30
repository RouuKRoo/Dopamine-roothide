# Глава III.10. Packages/libroot — публичный API путей

`libroot` — самая маленькая и самая важная из всех публичных библиотек Dopamine для автора стороннего твика. Её API — 3 функции, но без правильного их использования твик на RootHide-установке работает плохо или не работает совсем. В этой главе — вся правда об этих трёх функциях и о макросах, которые их обёртывают.

## 1. Назначение

Проблема, которую решает `libroot`: в rootless Dopamine был `/var/jb`, в RootHide — случайный путь вроде `/var/<rand1>/<rand2>/.jbroot`. Хардкод любого из вариантов делает твик непортируемым. Нужно обобщить.

Решение: `libroot.dylib` экспортирует три функции, которые возвращают **правильное** значение для текущего устройства. Твик линкуется с `-lroot` и получает корректный путь независимо от типа jailbreak.

## 2. Весь исходный код — `Packages/libroot/src/paths.c`

```c
#include <libjailbreak/jbclient_xpc.h>

const char *libroot_get_root_prefix(void)
{
    return "";
}

const char *libroot_get_jbroot_prefix(void)
{
    return jbclient_get_jbroot();
}

const char *libroot_get_boot_uuid(void)
{
    return jbclient_get_boot_uuid();
}
```

Это **весь** код библиотеки. Три функции, суммарно 15 строк включая `#include`.

Разбор:

- **`libroot_get_root_prefix()`** возвращает `""` — пустую строку. Это префикс для «реальной» rootfs; на jailbroken-устройстве rootfs не переадресуется, поэтому пустая строка — корректное значение.
- **`libroot_get_jbroot_prefix()`** возвращает результат `jbclient_get_jbroot()` — обращение к jbserver в launchd за актуальным путём jbroot. Это и есть ключевая точка «правильно получить путь».
- **`libroot_get_boot_uuid()`** возвращает boot UUID текущего cold-boot сеанса; полезно для идентификации «переживания» каких-то операций.

## 3. Контракт — что гарантирует библиотека

Из `Packages/libroot/control`:

```
Package: libroot-dopamine
Name: libroot (Dopamine)
Author: opa334
Maintainer: opa334
Architecture: iphoneos-arm64
Version: 1.0.1
Priority: optional
Section: Development
Description: libroot for Dopamine
```

Два факта:

- **Semver**: версия `1.0.1` — API стабильный, минорные изменения — обратно-совместимые.
- **Architecture**: `iphoneos-arm64` — но реально .dylib собирается с `-arch arm64 -arch arm64e` (см. `Packages/libroot/Makefile`), покрывая оба slice. Пакет Debian помечен arm64, что на jailbreak для Dopamine нормально.

## 4. Сборка — `Packages/libroot/Makefile`

```make
TARGET = libroot.dylib
CC = clang

CFLAGS = -I../../BaseBin/.include -Isrc \
         -isysroot $(shell xcrun --sdk iphoneos --show-sdk-path) \
         -arch arm64 -arch arm64e -miphoneos-version-min=15.0 \
         -fobjc-arc -O2
LDFLAGS = -dynamiclib -rpath /var/jb

all: $(TARGET) sign

sign: $(TARGET)
    @ldid -Cadhoc -S $^

$(TARGET): $(wildcard src/*.c ../../BaseBin/libjailbreak/src/jbclient_*.c)
    $(CC) $(CFLAGS) $(LDFLAGS) -o $@ $^

...
```

Два важных момента:

- **`-rpath /var/jb`** — rpath для поиска libraries в rootless-пространстве.
- **Статически линкуется с `jbclient_*.c`** — чтобы libroot.dylib была самодостаточной и не тянула весь `libjailbreak.dylib` в клиентский процесс.

Для автора твика это означает: `libroot.dylib` — лёгкая, 10–20 KB, не несёт за собой `libjailbreak` целиком.

## 5. Как подключить к твику

Minimum viable Makefile:

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = MyTweak
MyTweak_FILES = Tweak.xm
MyTweak_CFLAGS = -fobjc-arc
MyTweak_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

Ключевая строчка — `MyTweak_LIBRARIES = root`. Theos сам добавит `-lroot` при линковке. Соответствующий .deb должен указывать libroot как зависимость в `control`:

```
Depends: libroot-dopamine, mobilesubstrate | ellekit | substitute
```

## 6. Использование в коде

Простой пример:

```c
#include <libroot.h>
#include <stdio.h>

void dump_env(void)
{
    const char *rootPrefix = libroot_get_root_prefix();  // ""
    const char *jbPrefix = libroot_get_jbroot_prefix();   // "/var/<rand>/<rand>/.jbroot" или "/var/jb"
    const char *bootUUID = libroot_get_boot_uuid();       // UUID

    printf("rootPrefix: %s\n", rootPrefix);
    printf("jbPrefix:   %s\n", jbPrefix);
    printf("bootUUID:   %s\n", bootUUID);
}
```

Для получения конкретного пути — три паттерна.

### 6.1. Прямое строковое сложение (C-style)

```c
char dataPath[PATH_MAX];
snprintf(dataPath, sizeof(dataPath), "%s/usr/share/mytweak/data.json",
         libroot_get_jbroot_prefix());
FILE *f = fopen(dataPath, "r");
```

Работает, но многословно.

### 6.2. Макрос `JBROOT_PATH(...)` из libjailbreak

Из `BaseBin/libjailbreak/src/jbroot.h`:

```c
#define JBROOT_PATH(path) __jbroot_convert_path((path), __BUFFER_FOR_CHAR_P(path))
```

Использование:

```c
#include <libjailbreak/jbroot.h>

FILE *f = fopen(JBROOT_PATH("/usr/share/mytweak/data.json"), "r");
```

Разбор:

- `JBROOT_PATH(...)` требует определить `get_jbroot()` в вашем проекте, либо автоматически используется из `systemhook.dylib`.
- Для `const char *` — через `__jbroot_convert_path` с буфером на стеке (через `alloca(PATH_MAX)`).
- Для `NSString *` — через `__jbroot_convert_path` ObjC overload.

Это **header-only** — не требует линковки с libjailbreak, только включение заголовка.

### 6.3. Макрос `JBROOT_PATH_NSSTRING(...)` для Objective-C

Для чистого ObjC-кода:

```objc
NSString *path = JBROOT_PATH_NSSTRING(@"/usr/share/mytweak/data.json");
NSData *data = [NSData dataWithContentsOfFile:path];
```

Макрос раскрывается в inline-функцию, которая вызывает `__jbroot_convert_path` с NSString и возвращает NSString.

### 6.4. Пример для `CFBundle` ресурсов

Если ваш .deb кладёт ресурсы в `<jbroot>/Library/Application Support/MyTweak/`, доступ:

```objc
NSString *bundleDir = JBROOT_PATH_NSSTRING(@"/Library/Application Support/MyTweak/");
NSURL *bundleURL = [NSURL fileURLWithPath:bundleDir isDirectory:YES];
NSBundle *bundle = [NSBundle bundleWithURL:bundleURL];
UIImage *image = [UIImage imageNamed:@"icon" inBundle:bundle compatibleWithTraitCollection:nil];
```

Это полностью совместимо и с rootless, и с RootHide — макрос раскрывается в правильный путь на любой установке.

## 7. Что НЕ делать

Три антипаттерна, которые встречаются в старых твиках:

### 7.1. Хардкод `/var/jb/*`

```c
// ПЛОХО
FILE *f = fopen("/var/jb/usr/share/mytweak/data.json", "r");
```

На RootHide этот путь существует только как symlink в `/basebin/`, но ваши ресурсы живут в randomized-пути, не в `/var/jb/`. Код сломается на RootHide.

### 7.2. Чтение `$JBROOT` из environment

```c
// ПЛОХО (устарело)
const char *jbroot = getenv("JBROOT");
```

Такой переменной нет в современных сборках Dopamine. Единственный правильный путь — через libroot.

### 7.3. Разбор `/basebin` симлинка

```c
// ПЛОХО и переносимо лишь частично
char realBasebin[PATH_MAX];
realpath("/basebin", realBasebin);
// ... отрезать "/basebin" и работать с остатком
```

Работает на некоторых установках, но не является официальным API. Используйте `libroot_get_jbroot_prefix()`.

## 8. Отличие от `basebin-link`

Из `Packages/Makefile`:

```make
all:
    @$(MAKE) -C libkrw-provider package
    @$(MAKE) -C libroot package
    @$(MAKE) -C basebin-link package
```

Рядом с `libroot` собирается пакет `basebin-link`. Это НЕ библиотека — это пакет, кладущий символическую ссылку `/basebin` на реальный путь в `<jbroot>/basebin`. Для автора твика это невидимая инфраструктура, но полезно знать: `/basebin/libroot.dylib` — путь, по которому твик найдёт либу (через rpath `/var/jb` → `/basebin` symlink).

## 9. Таблица «ваша задача → что звать»

| Задача | API |
|---|---|
| Получить префикс jbroot | `libroot_get_jbroot_prefix()` |
| Получить boot UUID | `libroot_get_boot_uuid()` |
| Построить путь к ресурсу | `JBROOT_PATH("/usr/share/...")` |
| Построить путь (Objective-C) | `JBROOT_PATH_NSSTRING(@"/...")` |
| Получить префикс rootfs | `libroot_get_root_prefix()` (почти всегда `""`) |

## 10. Что подтверждено

- **Подтверждено исходниками**:
  - Все три функции и их реализации — `Packages/libroot/src/paths.c` целиком.
  - Макросы `JBROOT_PATH`, `JBROOT_PATH_NSSTRING` — `BaseBin/libjailbreak/src/jbroot.h`.
  - Сборочные флаги — `Packages/libroot/Makefile`.
  - Пакет `basebin-link` — отдельный, `Packages/basebin-link/`.
- **Подтверждено Makefile**: `-rpath /var/jb` в LDFLAGS; `-arch arm64 -arch arm64e`.
- **Требует проверки на устройстве**: что `libroot.dylib` установлен в `/usr/lib/libroot.dylib` (под префиксом jbroot).
- **Не подтверждено**: гарантии обратной совместимости для `libroot_get_boot_uuid()` между major-версиями проекта.

## 11. Выводы

Главный тезис: **`libroot` — это способ сделать твик portable между rootless и RootHide, и это обязательная практика на iOS 26 с Dopamine**. Три функции, два header-only макроса, одна зависимость в `control` — минимально необходимый набор.

Если твик собирается с `MyTweak_LIBRARIES = root` и использует `JBROOT_PATH(...)` вместо хардкодных путей, он автоматически корректно работает:
- на rootless-установке (`/var/jb/...`),
- на RootHide-установке (`/var/<rand>/<rand>/.jbroot/...`),
- даже если пользователь переключается между этими режимами.

## 12. Что дальше

- Как это всё собирается и разворачивается на устройстве — глава III.11.
- Сводная таблица — глава III.12.
- Для рецепта «корректная работа с путями» — Cookbook 20 (fakeroot paths).
- Для деталей раскрытия макросов — главу II.8 про файловую систему.
