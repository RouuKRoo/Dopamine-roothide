# Кейс 13. Твик без Logos — только C/ObjC + ellekit API

Крайний случай — твик, собранный **без Theos вовсе**: один `.c`/`.m` файл, собственный Makefile с `clang`, подпись через `ldid`, упаковка через `dpkg-deb`. Полезно, когда:

- Нужна integration в кастомную build-system.
- Хочется минимизировать зависимости.
- Понять, что реально делает Theos под капотом.

## 1. Структура проекта

```
no-logos-tweak/
├── Makefile              — свой, не Theos
├── Tweak.m              — весь код
├── no-logos.plist       — filter
├── layout/
│   └── DEBIAN/
│       └── control
└── sign.sh              — подпись + упаковка
```

## 2. Tweak.m — полностью рабочий код

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import <unistd.h>
#import <os/log.h>
#import <ellekit.h>

static os_log_t gLog;

// Hook UIViewController viewDidLoad
static void (*orig_viewDidLoad)(UIViewController *self, SEL _cmd);
static void my_viewDidLoad(UIViewController *self, SEL _cmd) {
    os_log(gLog, "VC viewDidLoad: %{public}@", NSStringFromClass(self.class));
    orig_viewDidLoad(self, _cmd);
}

__attribute__((constructor))
static void initializer(void) {
    gLog = os_log_create("com.example.nologos", "tweak");
    os_log(gLog, "no-logos tweak loaded in %{public}@",
           [NSProcessInfo processInfo].processName);

    EKHookMessage([UIViewController class],
                  @selector(viewDidLoad),
                  (IMP)my_viewDidLoad,
                  (IMP *)&orig_viewDidLoad);
}
```

Это **весь** код твика. 24 строки.

## 3. Собственный Makefile (без Theos)

```make
# Переменные
TWEAK_NAME     := NoLogos
PACKAGE_ID     := com.example.nologos
VERSION        := 0.0.1
ARCH           := iphoneos-arm
CC             := xcrun --sdk iphoneos clang
LDID           := ldid
DPKG_DEB       := dpkg-deb

# Пути
SDK_PATH       := $(shell xcrun --sdk iphoneos --show-sdk-path)
# Для RootHide Dopamine jbroot (ad hoc; обычно /var/jb на хосте при staging):
JBROOT         := /var/jb

# Флаги компиляции
CFLAGS := \
    -arch arm64 \
    -arch arm64e \
    -miphoneos-version-min=15.0 \
    -isysroot $(SDK_PATH) \
    -fobjc-arc \
    -Wall \
    -Os \
    -I$(HOME)/ellekit/include     # путь к ellekit headers

# Флаги линковки
LDFLAGS := \
    -dynamiclib \
    -install_name "$(JBROOT)/Library/MobileSubstrate/DynamicLibraries/$(TWEAK_NAME).dylib" \
    -framework UIKit \
    -framework Foundation \
    -lobjc \
    -L$(HOME)/ellekit/lib \
    -lellekit \
    -Wl,-rpath,$(JBROOT)

# Пути вывода
BUILD_DIR     := build
STAGING_DIR   := $(BUILD_DIR)/stage
TWEAKS_DIR    := $(STAGING_DIR)$(JBROOT)/Library/MobileSubstrate/DynamicLibraries

.PHONY: all clean package

all: $(BUILD_DIR)/$(TWEAK_NAME).dylib

$(BUILD_DIR)/$(TWEAK_NAME).dylib: Tweak.m | $(BUILD_DIR)
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $<
	$(LDID) -Sadhoc $@

$(BUILD_DIR):
	mkdir -p $(BUILD_DIR)

package: $(BUILD_DIR)/$(TWEAK_NAME).dylib
	rm -rf $(STAGING_DIR)
	mkdir -p $(TWEAKS_DIR)
	mkdir -p $(STAGING_DIR)/DEBIAN
	cp $(BUILD_DIR)/$(TWEAK_NAME).dylib $(TWEAKS_DIR)/
	cp no-logos.plist $(TWEAKS_DIR)/$(TWEAK_NAME).plist
	cp layout/DEBIAN/control $(STAGING_DIR)/DEBIAN/control
	$(DPKG_DEB) --root-owner-group -b $(STAGING_DIR) \
	           $(BUILD_DIR)/$(PACKAGE_ID)_$(VERSION)_$(ARCH).deb

clean:
	rm -rf $(BUILD_DIR)
```

Разбор:

- Компилятор: `xcrun --sdk iphoneos clang` — iOS compiler.
- `-arch arm64 -arch arm64e` — fat binary.
- `-fobjc-arc` — ARC.
- `-install_name` — внутренне имя dylib.
- `-Wl,-rpath,/var/jb` — rpath для `@rpath/libellekit.dylib`.
- `-lellekit` — линковка с ellekit.
- Подпись через `ldid -Sadhoc`.
- Упаковка в .deb через `dpkg-deb --root-owner-group -b`.

**Внимание про ellekit headers/lib**: нужны либо из установленной ellekit на хосте, либо собранные самостоятельно. На macOS с Theos — обычно в `$THEOS/vendor/include/` и `$THEOS/vendor/lib/`.

## 4. no-logos.plist

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

## 5. layout/DEBIAN/control

```
Package: com.example.nologos
Name: NoLogos
Version: 0.0.1
Architecture: iphoneos-arm
Description: Tweak built without Theos/Logos
Maintainer: You
Author: You
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## 6. Сборка

```bash
cd no-logos-tweak
make clean
make package
# → build/com.example.nologos_0.0.1_iphoneos-arm.deb
```

Проверка:

```bash
dpkg-deb -c build/*.deb
# ./var/jb/Library/MobileSubstrate/DynamicLibraries/NoLogos.dylib
# ./var/jb/Library/MobileSubstrate/DynamicLibraries/NoLogos.plist

otool -L build/NoLogos.dylib
# должны быть зависимости: Foundation, UIKit, libobjc, libellekit, libSystem
```

## 7. Установка и проверка

```bash
scp build/com.example.nologos_0.0.1_iphoneos-arm.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/com.example.nologos_*.deb && killall -9 TestbedApp"
```

Запустить TestbedApp на устройстве; через `log stream`:

```
no-logos tweak loaded in TestbedApp
VC viewDidLoad: ViewController
```

Готово.

## 8. Что это демонстрирует

- Theos — удобство, но не магия. Можно без него.
- Logos — препроцессор; `%hook` раскрывается в `EKHookMessage(...)` вызов.
- `.deb` — просто архив с конкретной структурой.
- Подпись — `ldid -Sadhoc`.
- Для RootHide-совместимости — путь `/var/jb/Library/MobileSubstrate/DynamicLibraries/`.

## 9. Когда это полезно

**Используйте Theos** в 99% случаев. Удобство > самостоятельность.

**Этот подход** имеет смысл только когда:

- Строите вашу собственную CI/CD без Theos.
- Пишете multi-platform build-script (например, собираете ту же dylib для разных jailbreak).
- Учебно — чтобы понять, как это работает.
- Нужна интеграция с существующей build-системой (CMake, Bazel), которая не знает про Theos.

## 10. Что подтверждено

- **Подтверждено**: `xcrun --sdk iphoneos clang` + `ldid` + `dpkg-deb` — рабочий путь.
- **Подтверждено**: `.deb` структура — `./var/jb/Library/MobileSubstrate/DynamicLibraries/` для rootless-схемы.
- **Требует проверки**: пути к ellekit headers/lib — зависят от того, как и где установлен ellekit на хосте.
- **Не подтверждено**: что это даёт полную RootHide-совместимость — Theos под `THEOS_PACKAGE_SCHEME=roothide` делает дополнительную магию с symlink'ами и путями при install; этот Makefile на RootHide может потребовать postinst-скрипта для переназначения путей в randomized jbroot. Для production — используйте Theos.

## 11. Выводы

Главный тезис: **твик — это dylib + filter plist + control-метаданные + подпись**. Theos автоматизирует всё это; тот же результат достижим вручную.

Для учебы — полезно. Для production — Theos.

## 12. См. также

- Кейс 12 — ellekit direct (но с Theos).
- Глава IV.2 — минимальный Theos проект.
- Глава IV.10 — сборка и инспекция.
