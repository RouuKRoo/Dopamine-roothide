# Глава IV.12. Учебный проект №2: безопасный SpringBoard-твик

Второй учебный проект сложнее первого двумя факторами: мы вмешиваемся в системный процесс (SpringBoard), и мы работаем с **частным API**, точные имена которого на iOS 26 не задокументированы. Задача этой главы — показать, как это делать безопасно: с fallback-поведением, с процедурой поиска актуальных символов, с чётким откатом.

Главный честный факт: **точные селекторы, которые я показываю ниже, — ориентиры, снятые из открытых дампов класс-дампа SpringBoard на более ранних версиях iOS**. На iOS 26 они могут отличаться. Процедура адаптации — обязательная часть этой главы.

## 1. Цель

Твик, который к центральному элементу status bar (в районе отображения времени на экране блокировки или домашнем) добавляет короткое послесловие — скажем, « ⚡ ». Простая, видимая, безопасная модификация.

**Поведение после установки:**

- Разблокируйте экран → увидите время с суффиксом «⚡».
- На самом экране блокировки (lockscreen) — тоже.
- При ошибке (не найден нужный селектор) — твик **ничего не ломает**: вся работа покрыта `respondsToSelector:` и `@try`/`@catch`.

**Статус**: это учебный проект. Точные селекторы — ориентиры; на iOS 26.0 и 26.0.1 требуется их верификация через class-dump SpringBoard и SpringBoardFoundation.

## 2. Что требуется заранее

- Среда разработчика (глава IV.1).
- Установленный Dopamine с ellekit на устройстве A13 c iOS 26.0 или 26.0.1.
- Открытый SSH на устройстве.
- (опционально) class-dump или class-dump-dyld — для верификации символов.

## 3. Процедура поиска актуальных селекторов

Прежде чем писать код, снимите с устройства актуальные имена классов и методов, связанных с временем в status bar.

### 3.1. Через class-dump

Если у вас есть class-dump или class-dump-dyld, выгрузите headers:

```bash
# Через SSH на jailbroken устройстве:
ssh mobile@<ip>
mkdir -p /tmp/sbheaders
# Если установлен class-dump-dyld (пакет из репо)
class-dump-dyld -o /tmp/sbheaders -s \
  /System/Library/CoreServices/SpringBoard.app/SpringBoard

# Для фреймворка:
class-dump-dyld -o /tmp/sbfheaders -s \
  /System/Library/PrivateFrameworks/SpringBoardFoundation.framework/SpringBoardFoundation
```

Или через извлечение dyld_shared_cache (сложнее):

```bash
# Нужен ipsw CLI (ipsw.io) — на хосте
ipsw extract --dyld /path/to/iPhone_...ipsw
ipsw dyld dump --image SpringBoard dyld_shared_cache_arm64 > /tmp/sb-info.txt
```

### 3.2. Что искать

Ищем классы и методы, связанные с временем в status bar:

```bash
cd /tmp/sbheaders
grep -lrn "TimeDate\|TimeLabel\|TimeView\|statusBarTimeString\|SBFLSTimeStackView\|_SBUITimeViewController" .
```

Исторически (на iOS 15–18) интересными были:

- `_SBUITimeFormatter`, `-timeFormatForStyle:`
- `_SBUIDigitalClockView`
- `SBFLSTimeStackView`, `SBFLSTimeStackViewContentView`
- `_UIStatusBarForegroundView`
- `-[_UIStatusBarItemCustomView contentView]`

Для **нашей** задачи — изменение видимого текста времени — чаще всего работает хук на `-[NSDateFormatter stringFromDate:]` в контексте SpringBoard. Но правильнее искать специальный форматер времени на экране блокировки.

### 3.3. Если class-dump недоступен

Fallback-способ: использовать `strings` на SpringBoard бинарнике:

```bash
ssh mobile@<ip>
strings /System/Library/CoreServices/SpringBoard.app/SpringBoard | grep -i "timeFormat\|TimeLabel\|TimeView" | head -30
```

Это даст список потенциально интересных строк — отталкиваетесь от них для поиска методов.

### 3.4. Verification на устройстве

Perевязка — через runtime inspection. Напишите мини-твик, который перебирает все классы и ищет те, что содержат «Time» в имени:

```objc
%ctor {
    unsigned int count = 0;
    Class *classes = objc_copyClassList(&count);
    for (unsigned int i = 0; i < count; i++) {
        const char *name = class_getName(classes[i]);
        if (strcasestr(name, "time") && strcasestr(name, "SB")) {
            NSLog(@"[probe] class with 'time' and 'SB': %s", name);
        }
    }
    free(classes);
}
```

Запускаете на устройстве (filter plist = SpringBoard), смотрите лог. Это даст точный список классов, присутствующих в SpringBoard на вашей iOS.

## 4. Шаблон твика

Поскольку точный селектор на вашей iOS 26 я не могу гарантировать, пишу код в двух вариантах: **shadow-safe (ничего не делает, только логирует)** и **active (пытается модифицировать)**. Начинаете с первого, после класс-дампа — переходите на второй.

### 4.1. Структура проекта

```bash
cd ~/dev
$THEOS/bin/nic.pl
# Template: iphone/tweak
# Project Name: SBClockSuffix
# Package Name: com.example.sbclocksuffix
# Author: You
# Bundle filter: com.apple.springboard
```

Получаем:

```
SBClockSuffix/
├── Makefile
├── SBClockSuffix.plist
├── Tweak.x
└── control
```

### 4.2. Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = SpringBoard
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = SBClockSuffix

SBClockSuffix_FILES = Tweak.x
SBClockSuffix_CFLAGS = -fobjc-arc
SBClockSuffix_FRAMEWORKS = UIKit Foundation
SBClockSuffix_PRIVATE_FRAMEWORKS = SpringBoardFoundation
SBClockSuffix_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

Разбор:

- `INSTALL_TARGET_PROCESSES = SpringBoard` — после `make install` Theos сделает respring.
- `SBClockSuffix_PRIVATE_FRAMEWORKS = SpringBoardFoundation` — линкуемся с private framework, чтобы были доступны все символы.

### 4.3. SBClockSuffix.plist

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

### 4.4. control

```
Package: com.example.sbclocksuffix
Name: SBClockSuffix
Version: 0.0.1
Architecture: iphoneos-arm
Description: Append "⚡" after SpringBoard clock time
Maintainer: You
Author: You
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

### 4.5. Tweak.x — SHADOW вариант (безопасный)

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <os/log.h>
#import <objc/runtime.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.sbclocksuffix", "clock");
    os_log(gLog, "SBClockSuffix loaded in SpringBoard");

    // Probe: напечатать классы, связанные с временем
    unsigned int count = 0;
    Class *classes = objc_copyClassList(&count);
    for (unsigned int i = 0; i < count; i++) {
        const char *name = class_getName(classes[i]);
        if (strcasestr(name, "time") &&
            (strncmp(name, "SB", 2) == 0 || strncmp(name, "_SB", 3) == 0)) {
            os_log(gLog, "[probe] class: %{public}s", name);
        }
    }
    free(classes);
}

%hook NSDateFormatter

- (NSString *)stringFromDate:(NSDate *)date {
    NSString *result = %orig(date);
    // Логируем — но ничего не меняем
    if (result.length > 0 && result.length < 20) {
        os_log(gLog, "[probe] stringFromDate: %{public}@", result);
    }
    return result;
}

%end
```

Разбор:

- `%ctor` перебирает все классы и логирует те, что начинаются с `SB`/`_SB` и содержат `time`. Это даёт нам реальный список классов в SpringBoard на вашей iOS 26.
- `%hook NSDateFormatter stringFromDate:` — пассивно логирует (без изменений). SpringBoard зовёт этот метод для форматирования времени; из лога увидим реальные форматы.

Сборка → установка → respring:

```bash
make clean package install THEOS_PACKAGE_SCHEME=roothide \
     THEOS_DEVICE_IP=<ip>
ssh mobile@<ip> "killall -9 SpringBoard"
```

Смотрим лог:

```bash
log stream --predicate 'subsystem == "com.example.sbclocksuffix"' --device <udid>
```

Увидите имена реальных классов и формат времени. Это и есть материал для следующего шага.

### 4.6. Tweak.x — ACTIVE вариант (с модификацией)

Предположим, из probe-лога вы увидели класс `_SBUITimeFormatter` с методом `-timeString`. Внимание — **конкретные имена требуют верификации на iOS 26**. Шаблон:

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <os/log.h>
#import <objc/runtime.h>

static os_log_t gLog;
static NSString * const kSuffix = @" ⚡";

%ctor {
    gLog = os_log_create("com.example.sbclocksuffix", "clock");
    os_log(gLog, "SBClockSuffix (active) loaded");
}

// Fallback: простой хук на NSDateFormatter, который работает на любой iOS,
// но ловит и много не-временных дат. Это компромисс в сторону безопасности.
%hook NSDateFormatter

- (NSString *)stringFromDate:(NSDate *)date {
    NSString *result = %orig(date);
    // Простая эвристика: короткая строка, содержит ':' или форматер — это время
    if (result.length > 0 && result.length < 10 &&
        ([result containsString:@":"] || [result containsString:@"."])) {
        // На всякий случай — не добавляем суффикс дважды
        if (![result hasSuffix:kSuffix]) {
            return [result stringByAppendingString:kSuffix];
        }
    }
    return result;
}

%end
```

Этот вариант **работает**, но затрагивает слишком много мест: любой другой код в SpringBoard, который форматирует время с ':' — тоже получит суффикс. Это компромисс.

Более точный вариант (**требует верификации класса**):

```objc
%hook _SBUITimeFormatter   // имя гипотетическое, подтвердите!

- (NSString *)timeString {
    NSString *result = %orig;
    if (result.length > 0) {
        return [result stringByAppendingString:kSuffix];
    }
    return result;
}

%end

// Если класс не существует — Logos просто ничего не сделает (хук не инициализируется).
// Это защитный механизм — твик не упадёт.
```

### 4.7. Защита от неизвестного класса

Logos при `%hook ClassName` **требует**, чтобы класс был в SDK или объявлен через `@interface`. Если класс private — объявляем вручную:

```objc
@interface _SBUITimeFormatter : NSObject
- (NSString *)timeString;
@end

%hook _SBUITimeFormatter
- (NSString *)timeString {
    NSString *result = %orig;
    if (result && result.length > 0 && ![result hasSuffix:kSuffix]) {
        return [result stringByAppendingString:kSuffix];
    }
    return result;
}
%end
```

Если на устройстве **нет** такого класса, Logos при `%init` ничего не сделает (внутренне `objc_getClass("_SBUITimeFormatter")` вернёт `NULL`), и ваш хук молча пропустится — твик не упадёт.

### 4.8. Защита через `%c()`

Альтернатива для динамического получения класса:

```objc
%ctor {
    gLog = os_log_create("com.example.sbclocksuffix", "clock");
    Class cls = NSClassFromString(@"_SBUITimeFormatter");
    if (cls) {
        os_log(gLog, "Found target class, hooking");
        %init(TimeFormatter = cls);
    } else {
        os_log(gLog, "Target class not found — tweak idle");
    }
}

%hook TimeFormatter   // placeholder имя

- (NSString *)timeString {
    NSString *result = %orig;
    ...
}

%end
```

Это **самый безопасный** путь: если класс не найден — твик загрузился, залогировал факт, и мирно спит.

## 5. Сборка, установка, респринг

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide FINALPACKAGE=1
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 SpringBoard"
```

Theos `make install` делает `killall` сам. Если ставите вручную — делайте респринг через SSH после `dpkg -i`.

**Внимание**: не ставьте SpringBoard-твик, не имея откатного плана. Если твик падает SpringBoard'у — вы увидите черный экран, или бесконечный respring loop.

## 6. План отката (критично!)

Если SpringBoard упал и ушёл в loop:

### 6.1. Через SSH

```bash
ssh mobile@<ip>
# подождать, пока SpringBoard не поднимется хоть на секунду
sudo dpkg -r com.example.sbclocksuffix
killall -9 SpringBoard
```

Если SSH **не отвечает** (возможен в самом тяжёлом случае):

### 6.2. Через safe mode

Dopamine поддерживает «safe mode»: создание файла `<jbroot>/basebin/.safe_mode` отключает **все** твики. См. главу III.5.

Если устройство в loop и SSH не работает:

- Hold Volume Up + Volume Down + lock-кнопку → force restart.
- При загрузке — не запускать Dopamine Jailbreak сразу; через Dopamine.app есть опция «Enable Safe Mode» → твики отключаются до следующей загрузки.

### 6.3. Через iFile/Filza

Если устройство грузится до home screen, но SpringBoard крэшится при каждом открытии — не ставьте этот твик без заранее установленного Filza. Через Filza можно удалить `.dylib` твика напрямую.

## 7. Процедура проверки

### 7.1. Shadow-вариант

1. Установили `.deb`.
2. Респринг (`killall -9 SpringBoard`).
3. SpringBoard поднимается, выглядит нормально.
4. Через SSH смотрим лог:

```bash
log stream --predicate 'subsystem == "com.example.sbclocksuffix"' --device <udid>
```

Ожидаемо:

```
SBClockSuffix loaded in SpringBoard
[probe] class: _SBUITimeFormatter (или другое — что реально в iOS 26)
[probe] class: SBFLSTimeStackView
[probe] class: _SBUIFolderIconListTitleViewLabel  (пример)
[probe] stringFromDate: 10:30
[probe] stringFromDate: четверг, 30 октября
```

Вы увидите:
- Какие классы с «time» есть в вашей iOS 26.
- Какие строки форматируются и возвращаются.

### 7.2. Active-вариант

1. Поменяли код на реальный класс (из probe).
2. Собрали, установили, респринг.
3. Проверили — есть ли «⚡» рядом с часами на экране блокировки / статусбаре.
4. Если нет — вернулись к probe, уточнили класс/метод.
5. Повторили.

## 8. Типовые ошибки

| Ошибка | Причина | Решение |
|---|---|---|
| SpringBoard уходит в respring loop | Хук возвращает `nil` где не должен | В хуке `return result ?: @""` вместо просто `return result` |
| Класс не найден | Имя неверное | Используйте пробу (раздел 3.4) |
| Суффикс добавляется, но исчезает через секунду | SpringBoard форматирует время несколько раз, последний вызов — без хука | Хукнуть **все** вовлечённые классы или более высокий уровень |
| Суффикс дублируется (`⚡⚡`) | Хук вызывает `%orig`, который сам возвращает уже-с-суффиксом | Проверка `hasSuffix:kSuffix` обязательна |
| Твик загружается, но ничего не делает | Хук на класс, которого нет на iOS 26 | Logos молча пропустит — используйте safe pattern из раздела 4.8 |

## 9. Что подтверждено и что нет

- **Подтверждено**: правила работы `%hook` при отсутствии класса — Logos инициализирует через `objc_getClass`, если NULL — ничего не произойдёт.
- **Подтверждено**: `NSDateFormatter stringFromDate:` вызывается в SpringBoard для форматирования времени (многократно).
- **Требует проверки на устройстве** (критически!): **конкретные имена классов и методов** в SpringBoard на iOS 26.0 и 26.0.1. То, что я приводил (`_SBUITimeFormatter`, `-timeString`), — ориентиры с более ранних версий; на 26 могут отличаться. Процедура адаптации — обязательная часть этой главы.
- **Не подтверждено**: что любой пример класса в probe-логе — то, что вам нужно. Может потребоваться итерация.

## 10. Выводы

Главный тезис этой главы: **для твиков на private классы нужна защита от «класс не найден»**. Это достигается тремя приёмами:

1. Декларация `@interface` вручную — не падает на компиляции.
2. `NSClassFromString` + `%init(x = cls)` — не падает в рантайме.
3. Fallback на публичный API (`NSDateFormatter stringFromDate:`) — работает всегда, пусть и грубее.

Второй главный тезис: **для SpringBoard-твика всегда готовьте план отката**. SSH, safe mode, Filza — любой из этих способов должен быть доступен до того, как вы делаете `dpkg -i` на устройстве.

## 11. Что дальше

- Проект №3 — системная подсистема с риском — глава IV.13.
- Патчи самого jailbreak — глава V.1.
- Патчи существующих твиков — глава V.2.
- Процедура диагностики «твик упал SpringBoard'у» — глава VII.1.
