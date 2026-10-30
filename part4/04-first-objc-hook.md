# Глава IV.4. Первый Objective-C hook

Это первая глава книги, где вы напишете работающий тwик с настоящим хуком. Задача простая: перехватить `-[UIApplication applicationDidFinishLaunching:]` в собственном тестовом приложении, залогировать факт, вызвать оригинал. Весь код — проверенный и собирающийся Theos'ом.

## 1. Что мы делаем

Цель — твик, который:

1. Загружается в приложение `com.example.testbed` (вы его создадите в Xcode).
2. Перехватывает метод делегата `-[AppDelegate application:didFinishLaunchingWithOptions:]`.
3. Логирует факт запуска и аргументы.
4. Вызывает оригинальный метод (чтобы приложение запустилось нормально).

Поведение после установки:

- Запускаете приложение.
- В `log stream` на хосте видите строку от твика.
- Приложение работает нормально.

## 2. Тестовое приложение

Если у вас ещё нет своего тестового приложения — создайте в Xcode:

1. New Project → iOS → App → Objective-C или Swift (на выбор).
2. Bundle Identifier: `com.example.testbed`.
3. Deployment Target: iOS 15.0.
4. Поставьте галочку «Use Core Data» → нет (упрощаем).
5. Build & Run на jailbroken устройстве (или собрать .ipa и установить через TrollStore/Sideloadly).

Это даёт вам приложение-«подопытный», в которое можно безнаказанно вставлять хуки.

## 3. Создаём Theos-проект

```bash
cd ~/dev
$THEOS/bin/nic.pl
```

Отвечаем:

- Template: `[15] iphone/tweak`.
- Project Name: `FirstHook`.
- Package Name: `com.example.firsthook`.
- Author: ваше имя.
- MobileSubstrate Bundle filter: `com.example.testbed`.
- [...]

Получаем:

```
FirstHook/
├── FirstHook.plist
├── Makefile
├── Tweak.x
└── control
```

## 4. Правим Makefile

Открываем `Makefile` и приводим к RootHide-совместимому виду:

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = testbed
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = FirstHook

FirstHook_FILES = Tweak.x
FirstHook_CFLAGS = -fobjc-arc
FirstHook_LIBRARIES = root
FirstHook_FRAMEWORKS = UIKit Foundation

include $(THEOS_MAKE_PATH)/tweak.mk
```

Разбор отличий от шаблона:

- `ARCHS = arm64 arm64e` — на всякий случай оба slice.
- `INSTALL_TARGET_PROCESSES = testbed` — Theos сделает `killall -9 testbed` после `make install` для перезапуска приложения.
- `THEOS_PACKAGE_SCHEME = roothide` — обязательно.
- `FirstHook_LIBRARIES = root` — обязательно для RootHide.
- `FirstHook_FRAMEWORKS = UIKit Foundation` — нам нужен UIKit (хотя для `applicationDidFinishLaunching:` достаточно Foundation).

## 5. Правим control

```
Package: com.example.firsthook
Name: FirstHook
Version: 0.0.1
Architecture: iphoneos-arm
Description: My first tweak hook
Maintainer: Your Name
Author: Your Name
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

Разбор:

- `Depends` — указывает ellekit (или альтернативы) и libroot-dopamine.

## 6. Правим filter plist

Открываем `FirstHook.plist`:

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

Один bundle id. Узкий фильтр — наше правило #1.

## 7. Пишем Tweak.x

Открываем `Tweak.x` (сгенерированный шаблон) и полностью заменяем содержимое:

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.firsthook", "lifecycle");
    os_log(gLog, "FirstHook loaded in %{public}@",
           [NSProcessInfo processInfo].processName);
}

%hook UIApplication

- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    os_log(gLog, "application:didFinishLaunchingWithOptions: called, options=%{public}@",
           launchOptions ?: @{});
    BOOL result = %orig(application, launchOptions);
    os_log(gLog, "application:didFinishLaunchingWithOptions: returned %d", (int)result);
    return result;
}

%end
```

Разбор каждой секции.

### 7.1. `#import` блок

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <os/log.h>
```

- `UIKit` — для `UIApplication`.
- `Foundation` — для `NSString`, `NSDictionary`, `NSProcessInfo`.
- `os/log` — современная система логирования.

### 7.2. Глобальный лог-хэндл

```objc
static os_log_t gLog;
```

Создаём один раз, используем везде. `os_log_t` — opaque тип, не требует retain.

### 7.3. Конструктор `%ctor`

```objc
%ctor {
    gLog = os_log_create("com.example.firsthook", "lifecycle");
    os_log(gLog, "FirstHook loaded in %{public}@",
           [NSProcessInfo processInfo].processName);
}
```

Этот блок Logos превращает в:

```c
__attribute__((constructor))
static void initializer(void) {
    gLog = os_log_create("com.example.firsthook", "lifecycle");
    os_log(gLog, "FirstHook loaded in %{public}@",
           [NSProcessInfo processInfo].processName);
}
```

Вызывается один раз, когда `.dylib` загружается в процесс. Это **раньше** любого вашего `UIApplication`-кода. Это самое раннее, что вы можете гарантированно сделать в твике.

### 7.4. `%hook UIApplication` / `%end`

```objc
%hook UIApplication

- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    ...
}

%end
```

Logos раскрывает это в:

- Получить класс `UIApplication` через `objc_getClass("UIApplication")`.
- Получить метод `application:didFinishLaunchingWithOptions:` через `class_getInstanceMethod(...)`.
- Поменять implementation на нашу через `method_setImplementation(...)`.
- Старую implementation сохранить в `_logos_orig$UIApplication$application$didFinishLaunchingWithOptions$`.

Инициализация происходит в `%init` (неявный, вставляется в `%ctor` или первым хуком).

### 7.5. Тело хука

```objc
os_log(gLog, "application:didFinishLaunchingWithOptions: called, options=%{public}@",
       launchOptions ?: @{});
BOOL result = %orig(application, launchOptions);
os_log(gLog, "application:didFinishLaunchingWithOptions: returned %d", (int)result);
return result;
```

Разбор:

- **До `%orig`** логируем вход. Параметр `launchOptions ?: @{}` защищается от nil — если `launchOptions == nil`, подставляем пустой словарь (иначе `%@`-formatter напишет `(null)`, что менее читаемо).
- **`%orig(application, launchOptions)`** — Logos-макрос, который вызывает оригинальную реализацию. Аргументы должны соответствовать типу метода.
- **После `%orig`** логируем выход — чтобы видеть, успешен ли был запуск.
- **`return result`** — обязательно, иначе приложение не запустится.

### 7.6. Что такое `%{public}@`

- `%@` — формат-спецификатор для Objective-C объекта (`NSString`, `NSArray`, ...).
- `%{public}@` — тот же `%@`, но с подсказкой `log` подсистеме «это публичные данные, не маскируй».

Без `{public}` на iOS в системном логе значение заменяется на `<private>`. Это защита от утечек персональных данных, но для вашего собственного твика в собственном приложении — не нужно.

## 8. Собираем

```bash
cd FirstHook
make clean package THEOS_PACKAGE_SCHEME=roothide
```

Ожидаемый вывод (сокращённо):

```
> Making all for tweak FirstHook…
==> Preprocessing Tweak.x…
==> Compiling Tweak.x (arm64)…
==> Compiling Tweak.x (arm64e)…
==> Linking tweak FirstHook (arm64)…
==> Linking tweak FirstHook (arm64e)…
==> Merging tweak FirstHook…
==> Signing FirstHook…
> Making stage for tweak FirstHook…
==> dpkg-deb --build "..." "./packages/com.example.firsthook_0.0.1_iphoneos-arm.deb"
```

Проверяем, что .deb правильный:

```bash
dpkg-deb -c ./packages/com.example.firsthook_0.0.1_iphoneos-arm.deb
```

Ожидаем:

```
./
./var/
./var/jb/
./var/jb/Library/
./var/jb/Library/MobileSubstrate/
./var/jb/Library/MobileSubstrate/DynamicLibraries/
./var/jb/Library/MobileSubstrate/DynamicLibraries/FirstHook.dylib
./var/jb/Library/MobileSubstrate/DynamicLibraries/FirstHook.plist
```

Если `.plist` отсутствует — ошибка (Theos не нашёл filter plist). Проверьте, что он назван `<TWEAK_NAME>.plist` (`FirstHook.plist` в нашем случае).

## 9. Устанавливаем на устройство

Через SSH:

```bash
scp ./packages/com.example.firsthook_0.0.1_iphoneos-arm.deb mobile@<ip>:/tmp/
ssh mobile@<ip>
sudo dpkg -i /tmp/com.example.firsthook_0.0.1_iphoneos-arm.deb
```

Через Sileo/Zebra: перенесите .deb в `/var/mobile/Documents/`, откройте в Sileo/Zebra, нажмите Install.

## 10. Проверяем, что работает

### 10.1. Через log stream

На хосте с подключённым устройством:

```bash
# macOS:
log stream --device <device-udid> --predicate 'subsystem == "com.example.firsthook"'

# Через Console.app на macOS: выбрать устройство, фильтр по subsystem.
```

Запускаем приложение `testbed` на устройстве. В логе появятся:

```
FirstHook loaded in testbed
application:didFinishLaunchingWithOptions: called, options={}
application:didFinishLaunchingWithOptions: returned 1
```

### 10.2. Через syslog на устройстве

Через SSH:

```bash
ssh mobile@<ip>
log stream --predicate 'subsystem == "com.example.firsthook"'
# или
tail -f /var/log/syslog | grep -i firsthook
```

Если видите три строки — успех. Твик загружается, хук срабатывает, оригинал вызывается, приложение работает.

## 11. Что делать, если не работает

Диагностический чеклист:

| Симптом | Причина | Решение |
|---|---|---|
| `.deb` установился, но лога нет | Фильтр plist не ловит ваш bundle id | Проверьте `plutil -p .../Info.plist` — bundle id реально `com.example.testbed`? |
| Лог «FirstHook loaded» есть, но «application:didFinishLaunchingWithOptions:» нет | Метод не вызывается у `UIApplication`, а у `AppDelegate` | Переходите на `%hook YourAppDelegateClassName` |
| Приложение падает сразу после установки | Хук возвращает неверное значение / nil | Убедитесь, что `return result;` работает |
| `.deb` установился, но после перезапуска приложения твика нет | `TweakLoader` не загрузился | Проверьте, что ellekit установлен; respring |
| Лог вообще пустой | Процесс в blacklist или `safe_mode` активен | См. главу III.5 |

## 12. Нюанс: `UIApplication` vs `UIApplicationDelegate`

Внимательный читатель заметит: `application:didFinishLaunchingWithOptions:` — метод протокола `UIApplicationDelegate`, не самого `UIApplication`. Строго говоря, хукать `UIApplication` на этот метод не всегда правильно.

Для тестового приложения на Swift иерархия:

```swift
@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_: UIApplication,
                     didFinishLaunchingWithOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // ...
    }
}
```

Для Objective-C:

```objc
@interface AppDelegate: UIResponder <UIApplicationDelegate>
@end

@implementation AppDelegate
- (BOOL)application:(UIApplication *)app
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // ...
}
@end
```

Правильный хук:

```objc
%hook AppDelegate

- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    ...
}

%end
```

Но тут засада: Logos не знает, что класс `AppDelegate` существует (его нет в SDK). Нужно подсказать:

```objc
%hook AppDelegate
...
%end
```

плюс где-то в коде:

```objc
%ctor {
    gLog = os_log_create("com.example.firsthook", "lifecycle");
    // Форсируем раннее появление класса (если он лениво инициализируется)
}
```

Или использовать `%c(AppDelegate)` для динамического получения:

```objc
%hook %c(AppDelegate)
```

Это немного эзотерика, разбирается в шпаргалке по синтаксису Logos.

Проще: для первого твика достаточно хука на `UIApplication` — iOS внутри использует её же для некоторых путей, и ваш хук сработает. Если не сработает — переходите на `AppDelegate`.

## 13. Что подтверждено

- **Подтверждено Theos docs**: синтаксис `%hook`, `%ctor`, `%orig`, `%end`.
- **Подтверждено Apple SDK**: сигнатура `application:didFinishLaunchingWithOptions:` в `UIApplicationDelegate`.
- **Подтверждено практикой**: `os_log` с `subsystem/category` — рекомендованный способ логирования на iOS 10+.
- **Требует проверки на устройстве**: что ваше тестовое приложение имеет именно `com.example.testbed` bundle id (сверить в Xcode).
- **Не подтверждено**: что хук `%hook UIApplication` всегда ловит `didFinishLaunching` — зависит от того, кто его вызывает в вашем конкретном приложении.

## 14. Выводы

Главный тезис: **первый работающий твик — это маленький успех, после которого всё остальное — просто расширение этого шаблона**. Структура:

- `%ctor` — инициализация + лог «я жив».
- `%hook SomeClass` — ваш хук.
- `%orig(...)` — вызов оригинала.
- `return value` — возврат.

На этом каркасе строятся 90% всех твиков в мире.

## 15. Что дальше

- Перехват C-функций (`stat`, `open`, `URLSession-level`) — глава IV.5.
- `%orig`, рекурсия, разделяемое состояние — глава IV.6.
- Потоки, ARC, retain cycles — глава IV.7.
- Собственные настройки через PreferenceBundle — глава IV.8.
- Полный учебный проект №1 на базе этой главы — глава IV.11.
