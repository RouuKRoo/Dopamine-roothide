# RE 1. Как найти частный селектор в SpringBoard iOS 26

Пошаговый workflow: от «я хочу подменить время в status bar» до работающего хука.

## 1. Задача

Подменить формат времени в status bar SpringBoard — например, добавить секунды.

Публичного API для этого нет. Нужно private селектор.

## 2. Шаг 1: Извлечь SpringBoard framework из dyld_shared_cache

На хосте, где есть IPSW для iOS 26.0 на вашем устройстве:

```bash
# ipsw CLI (blacktop/ipsw):
ipsw extract --dyld /path/to/iPhone11,2_26.0_23B5044l_Restore.ipsw

# Выгрузить SpringBoardFoundation framework:
ipsw dyld extract dyld_shared_cache_arm64 \
    --image "/System/Library/PrivateFrameworks/SpringBoardFoundation.framework/SpringBoardFoundation" \
    -o /tmp/sbf/
```

Теперь у вас настоящий бинарник из iOS 26.

## 3. Шаг 2: class-dump

```bash
# Если установлен class-dump-dyld:
class-dump-dyld /tmp/sbf/SpringBoardFoundation -o /tmp/sbfheaders

# Или через ClassDump-dyld-dsc (альтернативная версия)
# Или вручную через nm:
nm -gU /tmp/sbf/SpringBoardFoundation | grep -i "time\|clock" | head -30
```

Результат — заголовочные файлы для каждого класса.

## 4. Шаг 3: grep для нужных слов

```bash
cd /tmp/sbfheaders

# Классы с «Time» в имени:
ls | grep -i time
# _SBUITimeFormatter.h
# SBTimeDataSource.h
# ...

# Методы, связанные с временем:
grep -rl "timeString\|timeFormat\|hours\|minutes" .
```

Это даёт список кандидатов. Обычно их 5-10.

## 5. Шаг 4: Прочитать заголовки

Открываете каждый кандидат:

```bash
cat _SBUITimeFormatter.h
```

Содержимое (пример, гипотетический):

```objc
@interface _SBUITimeFormatter : NSObject {
    NSDateFormatter *_formatter;
    NSString *_cachedString;
}
+ (instancetype)sharedInstance;
- (NSString *)timeString;
- (void)invalidateTime;
- (void)setFormatStyle:(unsigned long)style;
@end
```

Хороший кандидат — метод `-timeString`.

## 6. Шаг 5: Runtime verification

На устройстве — pробe этого класса:

```objc
%ctor {
    Class cls = NSClassFromString(@"_SBUITimeFormatter");
    os_log(gLog, "class: %p", cls);

    if (cls) {
        SEL sel = @selector(timeString);
        Method m = class_getInstanceMethod(cls, sel);
        os_log(gLog, "method timeString: %p", m);

        // Проверить сигнатуру:
        if (m) {
            char *returnType = method_copyReturnType(m);
            os_log(gLog, "return type: %{public}s", returnType);
            free(returnType);
        }
    }
}
```

Собрать твик, установить в SpringBoard, смотреть log. Если:

- `class: 0x...` — класс есть.
- `method timeString: 0x...` — метод есть.
- `return type: @` — возвращает ObjC object (NSString *).

Готово — можно хукать.

## 7. Шаг 6: Хук

```objc
@interface _SBUITimeFormatter : NSObject
- (NSString *)timeString;
@end

%hook _SBUITimeFormatter

- (NSString *)timeString {
    NSString *orig = %orig;
    NSDate *now = [NSDate date];
    NSDateFormatter *fmt = [NSDateFormatter new];
    fmt.dateFormat = @"HH:mm:ss";
    return [fmt stringFromDate:now];
}

%end
```

Собрать, установить, respring. Status bar теперь показывает секунды.

## 8. Шаг 7: Edge cases

Private класс может иметь multiple callsites. Проверить, что хук работает везде:

- Lockscreen status bar.
- Home screen status bar.
- Control Center.
- In-app status bar (если приложения используют thet же formatter).

Если где-то не работает — probe на класс + метод в этом context'е.

## 9. Если на iOS 26 класс переименован

Scenario: `_SBUITimeFormatter` в iOS 18 → `_SBFTimeFormatter` в iOS 26.

Решение — fallback chain:

```objc
%ctor {
    Class cls = NSClassFromString(@"_SBUITimeFormatter")
             ?: NSClassFromString(@"_SBFTimeFormatter")
             ?: NSClassFromString(@"SBFLSTimeFormatter");

    if (cls) {
        %init(TimeFormatter = cls);
    } else {
        os_log(gLog, "no time formatter class found!");
    }
}

%hook TimeFormatter
- (NSString *)timeString { ... }
%end
```

Это переносимо через minor и major версии.

## 10. Пропущенные шаги для тонких случаев

Если method не найден в class — возможно он в superclass:

```objc
Class cls = NSClassFromString(@"_SBUITimeFormatter");
while (cls) {
    Method m = class_getInstanceMethod(cls, @selector(timeString));
    if (m) {
        os_log(gLog, "found in %s", class_getName(cls));
        break;
    }
    cls = class_getSuperclass(cls);
}
```

Хукайте тот класс, где нашли.

## 11. Workflow summary

```
Static (host):
  1. Extract framework из dyld_shared_cache
  2. class-dump
  3. grep по заголовкам → кандидаты

Runtime (device):
  4. Probe: NSClassFromString + class_getInstanceMethod
  5. Verify signature

Development:
  6. Hook с %hook или %init(Alias=cls)
  7. Build + install + test

Fallback:
  8. Multi-class fallback chain для переносимости
```

## 12. Что подтверждено

- **Подтверждено**: `objc_copyClassList`, `class_getInstanceMethod`, `method_copyReturnType` — standard runtime API.
- **Подтверждено Apple docs**: ObjC type encoding (`@` для id, etc.).
- **Требует проверки**: конкретные классы (`_SBUITimeFormatter`, etc.) на iOS 26 — через probe на устройстве.
- **Не подтверждено** (намеренно): точные internals iOS 26 — верификация на устройстве.

## 13. См. также

- Глава V.2 — починка через probe.
- Глава VI.1 — алгоритм исследования.
- Глава VI.2 — static vs device.
- Шпаргалка 1 — Logos (`%init(Alias = cls)`).
