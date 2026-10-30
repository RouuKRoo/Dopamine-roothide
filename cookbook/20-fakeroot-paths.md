# Рецепт 20. Корректное обращение к файлам в rootless + RootHide

**Цель**: шаблон правильного обращения к jailbreak-путям в твике — через `libroot`, без хардкода.

Это финальный рецепт cookbook, суммирующий правила глав III.10 и IV.9 в один рабочий образец.

## Задача

Твик должен:

1. Прочитать свой config.json из jbroot.
2. Записать свой log-файл в jbroot.
3. Загрузить своё изображение из своего бандла.
4. Проверить, установлено ли вспомогательное приложение.

Всё через правильные пути — работает и на rootless, и на RootHide.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = PathsExample

PathsExample_FILES = Tweak.x
PathsExample_CFLAGS = -fobjc-arc
PathsExample_FRAMEWORKS = UIKit Foundation
PathsExample_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## control

```
Package: com.example.pathsexample
Name: PathsExample
Version: 0.0.1
Architecture: iphoneos-arm
Description: Reference tweak demonstrating correct path handling
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## PathsExample.plist

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
</dict>
```

## Tweak.x

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <libroot.h>
#import <libjailbreak/jbroot.h>
#import <os/log.h>

static os_log_t gLog;
static NSBundle *gTweakBundle;

%ctor {
    gLog = os_log_create("com.example.pathsexample", "paths");

    // 1. Префикс jbroot — работает везде
    const char *jbroot = libroot_get_jbroot_prefix();
    os_log(gLog, "jbroot = %{public}s", jbroot);

    // 2. Boot UUID — для diagnostics
    const char *bootUUID = libroot_get_boot_uuid();
    os_log(gLog, "boot UUID = %{public}s", bootUUID);

    // 3. Загрузка своего бандла — через JBROOT_PATH_NSSTRING
    NSString *bundlePath = JBROOT_PATH_NSSTRING(@"/Library/Application Support/PathsExample/");
    gTweakBundle = [NSBundle bundleWithPath:bundlePath];
    if (gTweakBundle) {
        os_log(gLog, "tweak bundle loaded from %{public}@", bundlePath);
    } else {
        os_log_error(gLog, "tweak bundle NOT found at %{public}@", bundlePath);
    }

    // 4. Чтение config.json
    NSString *configPath = JBROOT_PATH_NSSTRING(@"/Library/Application Support/PathsExample/config.json");
    NSData *data = [NSData dataWithContentsOfFile:configPath];
    if (data) {
        NSError *err;
        NSDictionary *cfg = [NSJSONSerialization JSONObjectWithData:data options:0 error:&err];
        if (cfg) {
            os_log(gLog, "config loaded: %{public}@", cfg);
        } else {
            os_log_error(gLog, "config.json parse error: %{public}@", err);
        }
    } else {
        os_log(gLog, "no config.json (using defaults)");
    }

    // 5. Запись log-файла в jbroot (не в контейнер приложения, т. к. он может
    //    отличаться; для диагностики же лучше контейнер — но здесь — для примера)
    //
    //    Для реальной записи в jbroot — пишите в /tmp или в /Library/MyLogs/
    //    Отметьте: это файл будет виден с правами пользователя приложения.
    NSString *logPath = JBROOT_PATH_NSSTRING(@"/tmp/paths-example-boot.log");
    NSString *msg = [NSString stringWithFormat:@"%@ loaded in %@\n",
                     [NSDate date],
                     [NSProcessInfo processInfo].processName];
    NSFileHandle *fh = [NSFileHandle fileHandleForWritingAtPath:logPath];
    if (!fh) {
        [[NSFileManager defaultManager] createFileAtPath:logPath contents:nil attributes:nil];
        fh = [NSFileHandle fileHandleForWritingAtPath:logPath];
    }
    if (fh) {
        [fh seekToEndOfFile];
        [fh writeData:[msg dataUsingEncoding:NSUTF8StringEncoding]];
        [fh closeFile];
    }

    // 6. Проверить наличие вспомогательного приложения
    //    Приложения в rootless — в /var/jb/Applications; в roothide — в randomized jbroot/Applications
    NSString *helperAppPath = JBROOT_PATH_NSSTRING(@"/Applications/PathsExampleHelper.app");
    BOOL hasHelper = [[NSFileManager defaultManager] fileExistsAtPath:helperAppPath];
    os_log(gLog, "helper app exists: %d (%{public}@)",
           (int)hasHelper, helperAppPath);
}

%hook UIViewController

- (void)viewDidLoad {
    %orig;

    // Пример использования bundle resources из твика
    if (gTweakBundle) {
        UIImage *icon = [UIImage imageNamed:@"my-icon"
                                   inBundle:gTweakBundle
              compatibleWithTraitCollection:nil];
        if (icon) {
            os_log(gLog, "loaded icon from tweak bundle: %p", icon);
        } else {
            os_log(gLog, "icon not found in bundle");
        }
    }
}

%end
```

Разбор:

- `libroot_get_jbroot_prefix()` → строковое значение префикса.
- `libroot_get_boot_uuid()` → UUID текущего cold boot.
- `JBROOT_PATH_NSSTRING(@"...")` → полный путь для `NSString *`.
- `JBROOT_PATH("...")` → полный путь для `const char *` (для C API).
- `NSBundle bundleWithPath:` — для доступа к ресурсам ВАШЕГО бандла (не приложения).
- `fileExistsAtPath:` — проверка наличия, корректно работает на обеих установках.
- Запись в `/tmp` — простая демонстрация; для production используйте специальную папку с ротацией.

## layout — ресурсы и config

Проект должен содержать:

```
layout/
└── Library/
    └── Application Support/
        └── PathsExample/
            ├── config.json              (default config)
            ├── my-icon.png              (ресурс)
            └── Info.plist               (для bundleWithPath:)
```

Пример `config.json`:

```json
{
    "enabled": true,
    "level": "info"
}
```

Пример `Info.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleIdentifier</key>
    <string>com.example.pathsexample.bundle</string>
    <key>CFBundleName</key>
    <string>PathsExample</string>
    <key>CFBundlePackageType</key>
    <string>BNDL</string>
</dict>
</plist>
```

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"
```

Проверка файлов на устройстве:

```bash
ssh mobile@<ip>
# config.json на месте?
ls -la "$(jbroot)/Library/Application Support/PathsExample/"
# icon.png на месте?

# Лог записан?
cat "$(jbroot)/tmp/paths-example-boot.log"
```

## Главные правила (повторение)

1. **`THEOS_PACKAGE_SCHEME = roothide`** в Makefile.
2. **`_LIBRARIES = root`** в Makefile.
3. **`libroot-dopamine`** в Depends control.
4. **`JBROOT_PATH(...)`** или **`JBROOT_PATH_NSSTRING(@"...")`** для путей.
5. **НИКОГДА** `/var/jb` в коде.
6. **`[NSBundle mainBundle]`** — это хост-приложение; для своего бандла — `bundleWithPath:` + `JBROOT_PATH`.

## Проверка .deb

```bash
# Пустое:
strings .theos/obj/PathsExample.dylib | grep /var/jb
# Если выдаёт — у вас хардкод; это ошибка.
```

## См. также

- Глава III.10 — libroot в деталях.
- Глава IV.9 — RootHide-совместимость.
- Шпаргалка 5 — libroot API.
- Шпаргалка 8 — таблица путей.

На этом **Cookbook полностью закрыт**. 20 рецептов — набор готовых паттернов для решения конкретных практических задач.
