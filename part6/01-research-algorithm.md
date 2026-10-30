# Глава VI.1. Воспроизводимый алгоритм «от наблюдения до работающего хука»

Это ядро Части VI. Задача главы — дать вам систематический процесс, по которому можно взять любое наблюдение «iOS делает X» и свести к работающему твику. Это не «озарение», это **механика**: пройти по шагам, записать на каждом промежуточный результат, и либо получить ответ, либо чёткое «нет, недоступно».

## 1. Алгоритм высокого уровня

```mermaid
flowchart TD
    Obs[1. Наблюдение в одной строке]
    Proc[2. Какой процесс делает X]
    FW[3. Какой framework за это отвечает]
    Sym[4. Какие символы / классы участвуют]
    Sig[5. Проверить сигнатуру статически]
    Mini[6. Минимальный эксперимент (shadow)]
    Impl[7. Реальная реализация]
    Side[8. Проверка побочных эффектов]
    Obs --> Proc
    Proc --> FW
    FW --> Sym
    Sym --> Sig
    Sig --> Mini
    Mini --> Impl
    Impl --> Side
    Side -- found issue --> Impl
    Side -- clean --> Done[Готов к релизу]
```

Восемь шагов. Каждый обязателен, но некоторые занимают 10 секунд, другие — часы. Проведём их на конкретном примере.

## 2. Пример: подавить банер «Low Battery» в своём приложении

Представим задачу: ваше тестовое приложение запускается в фоне, когда батарея мало — iOS показывает системный alert «20% Battery Remaining». Вы хотите, чтобы в вашем приложении этот alert не появлялся.

Пройдём по алгоритму.

### 2.1. Шаг 1: Наблюдение в одной строке

**Задача**: когда батарея на 20% или 10%, iOS показывает модальный alert с текстом «Low Battery». Я хочу подавить его в моём приложении.

Хорошая формулировка — одна строка, конкретная. Плохая — «батарея странно работает».

### 2.2. Шаг 2: Какой процесс делает X

Два варианта:

**Вариант A — SpringBoard**. Большинство системных alert'ов рисует SpringBoard.

**Вариант B — ваше приложение**. Иногда UI отрисовывается в контексте приложения через iOS-shim.

Как определить? Через `log stream`:

```bash
log stream --predicate 'eventMessage CONTAINS "Low Battery" OR eventMessage CONTAINS "LowBattery"' --device <udid>
```

При срабатывании alert в логе появляются строки с префиксом процесса. Если видите `SpringBoard[...]` — вариант A. Если `TestbedApp[...]` — вариант B.

Для нашего примера предположим — SpringBoard.

### 2.3. Шаг 3: Какой framework за это отвечает

Пройти по SpringBoard-процессу через `otool`:

```bash
# На устройстве:
ssh mobile@<ip>
otool -L /System/Library/CoreServices/SpringBoard.app/SpringBoard | grep -i batt
# или
otool -L /System/Library/CoreServices/SpringBoard.app/SpringBoard | grep -i power
```

Возможный вывод (гипотетический, точные имена требуют верификации на iOS 26):

```
    /System/Library/PrivateFrameworks/PowerUI.framework/PowerUI
    /System/Library/PrivateFrameworks/BatteryUIKit.framework/BatteryUIKit
```

Или через strings:

```bash
strings /System/Library/CoreServices/SpringBoard.app/SpringBoard | grep -i "lowbattery\|low battery" | head -10
```

Это даст подсказку о контексте, где этот текст используется.

### 2.4. Шаг 4: Какие символы / классы участвуют

Class-dump или probe. Пример probe-кода:

```objc
#import <objc/runtime.h>

%ctor {
    unsigned int count = 0;
    Class *classes = objc_copyClassList(&count);
    for (unsigned int i = 0; i < count; i++) {
        const char *name = class_getName(classes[i]);
        if (strcasestr(name, "lowbattery") || strcasestr(name, "lowpower")) {
            os_log(OS_LOG_DEFAULT, "candidate: %s", name);
        }
    }
    free(classes);
}
```

Запускаете, смотрите лог. Пример гипотетического результата:

```
candidate: SBLowBatteryViewController
candidate: SBLowBatteryAlertController
candidate: _SBUILowBatteryAlertView
candidate: SBPowerUILowBatteryAlertManager
```

Теперь у нас 4 кандидата.

Далее — найти **метод**, который реально вызывается при показе alert. Через `class_copyMethodList`:

```objc
Class cls = objc_getClass("SBLowBatteryAlertController");
unsigned int mc = 0;
Method *methods = class_copyMethodList(cls, &mc);
for (unsigned int i = 0; i < mc; i++) {
    SEL sel = method_getName(methods[i]);
    os_log(OS_LOG_DEFAULT, "  method: %s", sel_getName(sel));
}
free(methods);
```

Пример вывода:

```
  method: init
  method: show
  method: showAnimated:
  method: dismissAnimated:completion:
  method: alertViewForBatteryLevel:
```

Хорошие кандидаты для хука: `-show` и `-showAnimated:`.

### 2.5. Шаг 5: Проверить сигнатуру статически

Проверить, что метод реально принимает те аргументы, что мы думаем:

```objc
Method m = class_getInstanceMethod(cls, @selector(showAnimated:));
if (m) {
    char *types = method_copyReturnType(m);
    os_log(OS_LOG_DEFAULT, "return type: %s", types);
    free(types);
    for (unsigned int i = 0; i < method_getNumberOfArguments(m); i++) {
        char *argType = method_copyArgumentType(m, i);
        os_log(OS_LOG_DEFAULT, "  arg[%u]: %s", i, argType);
        free(argType);
    }
}
```

Пример вывода:

```
return type: v    (void)
arg[0]: @          (id = self)
arg[1]: :          (SEL = _cmd)
arg[2]: B          (BOOL = animated)
```

Отлично — метод `-showAnimated:(BOOL)animated` без возврата. Хук будет простой.

### 2.6. Шаг 6: Минимальный эксперимент (shadow)

Пассивный хук, который только логирует:

```objc
@interface SBLowBatteryAlertController : NSObject
- (void)showAnimated:(BOOL)animated;
@end

%hook SBLowBatteryAlertController

- (void)showAnimated:(BOOL)animated {
    os_log(gLog, "LowBatteryAlert.showAnimated:%d — passing through", (int)animated);
    %orig(animated);
}

%end
```

Filter plist: `com.apple.springboard`.

Сборка, установка, respring. Когда alert появляется — в логе видим «passing through». Это подтверждает: наш хук действительно вызывается.

### 2.7. Шаг 7: Реальная реализация

Теперь блокируем:

```objc
%hook SBLowBatteryAlertController

- (void)showAnimated:(BOOL)animated {
    // Проверить, активно ли наше тестовое приложение
    UIApplication *app = [UIApplication sharedApplication];
    NSString *frontmostBundleId = ???;  // Как получить?
    if ([frontmostBundleId isEqualToString:@"com.example.testbed"]) {
        os_log(gLog, "suppressing LowBatteryAlert for testbed");
        return;   // не зовём %orig — alert не показывается
    }
    %orig(animated);
}

%end
```

**Проблема**: как узнать в SpringBoard, какое приложение активно? В SpringBoard нет `UIApplication.sharedApplication` в привычном смысле. Нужен другой API.

Через probe на SpringBoard:

```objc
%ctor {
    Class fbs = objc_getClass("FBSceneManager");
    // искать методы, возвращающие front app
    // (не буду выдумывать — покажу подход)
}
```

Или найти private API — например, `[(SpringBoard *)[UIApplication sharedApplication] _accessibilityFrontMostApplication]`. Но это **private**, и точное имя надо верифицировать class-dump'ом.

Если точное имя неизвестно, fallback-подход:

```objc
%hook SBLowBatteryAlertController

- (void)showAnimated:(BOOL)animated {
    // Условно: активировать подавление через глобальный тwик-switch
    if (/* наш switch "suppress low battery" включён */) {
        return;
    }
    %orig(animated);
}

%end
```

И через PreferenceBundle — пользователь сам решает «подавлять ли LowBattery».

Это демонстрирует важный приём: **когда полный контроль недоступен, предоставьте toggle пользователю**. Хук не судит сам, когда ему срабатывать; это делает пользователь.

### 2.8. Шаг 8: Проверка побочных эффектов

- Если alert теперь не показывается — приложение знает, что батарея мало? Что произойдёт, если устройство выключится неожиданно?
- Если пользователь забыл отключить suppression — он потеряет данные.
- Если alert блокируется в неподходящий момент (например, во время покупки) — проблема.

Mitigation:

- Логировать каждый suppressed alert.
- Разрешить auto-off (например, suppress выключается при <5% battery).
- Документировать для пользователя.

## 3. Общая таблица «инструмент → что даёт → ограничение»

| Инструмент | Что даёт | Ограничение |
|---|---|---|
| `log stream` | Увидеть, какой процесс делает X | Фильтр по тексту — иногда ложные совпадения |
| `ps`, `launchctl list` | Увидеть все процессы | Не даёт сигнатур методов |
| `otool -L` на процессе | Список frameworks | Не говорит, какой из них делает X |
| `strings` на бинарнике | Строки — подсказка о контексте | Много мусора |
| `objc_copyClassList` probe | Все классы на устройстве | Только ObjC, не C-функции |
| `class_copyMethodList` | Методы класса | Только публичные (которые класс декларирует) |
| class-dump / class-dump-dyld | Полный header-dump класса | Иногда не работает на новых iOS |
| `method_copyReturnType`/`copyArgumentType` | Сигнатура метода | ObjC encoding — учитесь читать |
| `nm` на бинарнике | Все символы (C + ObjC) | Длинный список, нужен grep |
| `otool -tV` | Дизассемблер инструкций | Нужен reverse engineering skill |
| Hopper / Ghidra / IDA | Decompiler | Высокий порог входа |

## 4. Общий алгоритм — развёрнутая версия

### 4.1. Входная ясность

Прежде чем начинать, зафиксируйте:

- **Что** вы хотите изменить (одно поведение, одна строка).
- **Где** вы хотите этого добиться (конкретное приложение или системно).
- **Какая цена ошибки** (моё приложение упадёт — ок; SpringBoard — чуть страшнее; `logd` — крайне плохо).

### 4.2. Процессная карта

1. Запустите целевое действие в iOS.
2. Через `log stream` найдите, какой процесс логирует связанное с действием.
3. Запишите bundle id или имя процесса.

### 4.3. Framework hunting

1. `otool -L` на процессе → список фреймворков.
2. `strings` на процессе → поиск характерных строк.
3. `grep` по именам фреймворков с характерными словами (например, «BatteryUIKit» — очевидно).

### 4.4. Class hunting

1. Либо class-dump процесса/framework.
2. Либо runtime probe с `objc_copyClassList`.
3. Либо `nm` на framework бинарнике.

### 4.5. Method hunting

1. `class_copyMethodList(cls, &count)` — все методы класса.
2. Или class-dump → header → список.
3. Или `nm -gU` → selectors (format `_<Class>__<Selector>:`).

### 4.6. Sig check

1. `method_copyReturnType` / `copyArgumentType`.
2. Или по header'у из class-dump.

### 4.7. Shadow

Пассивный хук, который только логирует. Три задачи:

- Подтвердить, что хук срабатывает.
- Увидеть реальные аргументы.
- Увидеть частоту вызовов.

### 4.8. Impl

Реализовать логику. Проверить на устройстве.

### 4.9. Side effects

Запустить много сценариев. Что ломается?

## 5. Пример: от нуля до «работает» за один сеанс

Задача: логировать все `-[UIApplication openURL:options:completionHandler:]` в своём приложении.

**1. Observation**: приложение открывает deep link, я хочу увидеть URL.

**2. Process**: ваше приложение.

**3. Framework**: UIKit.

**4. Classes**: `UIApplication` — публичный.

**5. Signature**: `openURL:options:completionHandler:` — Apple docs говорят `-(void)openURL:(NSURL *)url options:(NSDictionary *)options completionHandler:(void (^)(BOOL))completion`.

**6. Shadow**:

```objc
%hook UIApplication
- (void)openURL:(NSURL *)url options:(NSDictionary *)options completionHandler:(void (^)(BOOL))completion {
    os_log(gLog, "openURL: %{public}@", url.absoluteString);
    %orig(url, options, completion);
}
%end
```

**7. Impl**: тот же код + сохранение в файл если нужно.

**8. Side**: completion handler вызывается нормально → приложение не повисает.

Весь цикл — 15 минут от нуля до работающего твика.

## 6. Антипаттерны

### 6.1. «Начать с кода»

Пишут `%hook ClassX` без того, чтобы сначала проверить, существует ли `ClassX` на iOS 26. Это гарантия потерянного дня.

Правильно: сначала probe, потом код.

### 6.2. «Угадать имя класса»

«Я уверен, это называется `SBAppIconView`!» — и `objc_getClass("SBAppIconView")` возвращает `NULL`.

Правильно: pробe с перебором, либо class-dump.

### 6.3. «Пропустить shadow»

Сразу реализовать активный хук. Если он падает — не ясно: сам хук падает или `%orig` падает после модификаций?

Правильно: shadow сначала, активный патч после.

### 6.4. «Не проверить побочки»

Хук работает в одном сценарии, забыли проверить остальные. Релиз → жалобы пользователей.

Правильно: прошли минимум 10 вариантов использования.

## 7. Чеклист перед «реальным хуком»

```
[ ] Задача сформулирована в одной строке
[ ] Процесс определён
[ ] Framework определён
[ ] Класс найден и верифицирован (NSClassFromString не NULL)
[ ] Метод найден
[ ] Сигнатура проверена (return type + args)
[ ] Shadow-хук написан и подтвердил срабатывание
[ ] Impl реализован
[ ] Протестировано на 5+ сценариев
```

## 8. Что подтверждено

- **Подтверждено Apple docs**: `objc_copyClassList`, `class_copyMethodList`, `method_copyReturnType`, `method_copyArgumentType` — публичный Objective-C runtime API.
- **Подтверждено Apple docs**: ObjC type encoding ("@" = id, ":" = SEL, "v" = void, "B" = BOOL).
- **Подтверждено практикой**: алгоритм «observation → process → framework → class → method → sig → shadow → impl → side» — стандартный для reverse engineering.
- **Требует проверки**: конкретные имена (`SBLowBatteryAlertController`, `BatteryUIKit`) в примерах выше — иллюстративные, точные имена на iOS 26 требуют верификации на устройстве.

## 9. Выводы

Главный тезис: **есть системный алгоритм из 8 шагов, который работает для любой задачи «изменить поведение iOS»**. Это не магия, а механика.

Три якорных правила:

1. **Shadow перед impl**. Пассивное логирование до активной модификации.
2. **Verify every assumption**. `NSClassFromString` проверка, `respondsToSelector:` проверка — на каждом шагу.
3. **Side effects обязательны** — хук, прошедший `%orig` в happy path, может не пройти в edge case.

## 10. Что дальше

- Статический анализ vs устройство — глава VI.2.
- Диагностика «не загружается / падает» — глава VII.1.
- Chaedy reports — глава VII.2.
- Матрица 26.0 vs 26.0.1 для твика — глава VII.3.
