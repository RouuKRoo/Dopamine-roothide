# RE 3. Как изучить внутренние структуры (ivars) класса

Иногда для хука нужно знать внутренние поля класса — например, adjust state без вызова setter.

## 1. Задача

Класс `SBIconView` (гипотетический) имеет private ivar `_badgeNumberOverride`. Хотим его прочитать/изменить.

## 2. Шаг 1: static — class-dump

```bash
# Класс dump:
class-dump-dyld /path/to/SpringBoard -o /tmp/dump

cat /tmp/dump/SBIconView.h
```

Пример (гипотетический):

```objc
@interface SBIconView : UIView {
    SBIcon *_icon;
    UIImage *_iconImage;
    NSString *_label;
    NSInteger _badgeNumberOverride;
    BOOL _highlighted;
    // ... ещё поля
}
@property (retain) SBIcon *icon;
@property (copy) NSString *label;
// ...
@end
```

Вот оно: `_badgeNumberOverride` — type `NSInteger`.

## 3. Шаг 2: Runtime verification

```objc
#import <objc/runtime.h>

%ctor {
    Class cls = NSClassFromString(@"SBIconView");
    if (!cls) {
        os_log_error(gLog, "SBIconView class not found");
        return;
    }

    unsigned int count = 0;
    Ivar *ivars = class_copyIvarList(cls, &count);
    for (unsigned int i = 0; i < count; i++) {
        const char *name = ivar_getName(ivars[i]);
        const char *type = ivar_getTypeEncoding(ivars[i]);
        ptrdiff_t offset = ivar_getOffset(ivars[i]);
        os_log(gLog, "ivar %s: type=%s offset=%ld",
               name, type, (long)offset);
    }
    free(ivars);
}
```

Результат — реальный list ivars на вашем устройстве:

```
ivar _icon: type=@"SBIcon" offset=48
ivar _iconImage: type=@"UIImage" offset=56
ivar _label: type=@"NSString" offset=64
ivar _badgeNumberOverride: type=q offset=72
ivar _highlighted: type=B offset=80
```

- `@"SBIcon"` — ObjC object типа SBIcon.
- `q` — long long (64-bit integer).
- `B` — bool.

Offset — байт offset от начала объекта. Можно использовать для direct access.

## 4. Шаг 3: Чтение ivar

Два способа:

### 4.1. Через `object_getIvar` (safe)

```objc
Class cls = NSClassFromString(@"SBIconView");
Ivar iv = class_getInstanceVariable(cls, "_badgeNumberOverride");
if (iv) {
    id result = object_getIvar(someIconView, iv);
    // Для primitive int — не работает (возвращает object)
    // Для primitive — нужен direct offset access.
}
```

**Проблема**: `object_getIvar` работает только для **object** ivars (`@"ClassName"`). Для `int`/`BOOL` — возвращает wrapped NSNumber или undefined.

### 4.2. Через direct offset (unsafe но для primitives)

```objc
NSInteger badge = *(NSInteger *)((uint8_t *)someIconView + 72);
// или через value:
void *val = ivar_getOffset(iv) + (uint8_t *)someIconView;
```

**Опасно**: layout может отличаться между версиями iOS. Используйте только после runtime verification.

### 4.3. Самый безопасный — KVO

```objc
NSInteger badge = [[someIconView valueForKey:@"_badgeNumberOverride"] integerValue];
```

Objective-C KVO работает даже с private ivars (если ObjC2 runtime).

## 5. Шаг 4: Запись в ivar

### 5.1. Через `object_setIvar`

```objc
Class cls = NSClassFromString(@"SBIconView");
Ivar iv = class_getInstanceVariable(cls, "_badgeNumberOverride");
object_setIvar(someIconView, iv, @(42));  // boxed
```

### 5.2. Через KVC

```objc
[someIconView setValue:@(42) forKey:@"_badgeNumberOverride"];
```

Самый safe способ.

### 5.3. Direct

```objc
*((NSInteger *)((uint8_t *)someIconView + 72)) = 42;
```

Fastest, но unsafe к version changes.

## 6. Что подтверждено

- **Подтверждено Apple docs**: `class_copyIvarList`, `object_getIvar`, `object_setIvar`, `class_getInstanceVariable`, `valueForKey:`/`setValue:forKey:`.
- **Подтверждено практикой**: KVO работает с private ivars.
- **Требует проверки на устройстве**: конкретный layout — offset может отличаться между iOS.

## 7. Ограничения

- **PAC** на arm64e — некоторые ivar указатели подписаны. Direct read возвращает signed pointer; нужно strip (`ptrauth_strip`).
- **ObjC runtime optimizations** — некоторые ivars могут быть **synthesized runtime**, не в layout. Такие недоступны через ivar APIs; только через `valueForKey:` (если setter/getter экспортирован).

Для A13 (arm64) PAC не касается userspace — direct access работает без ptrauth.

## 8. См. также

- Глава V.3 — runtime patches.
- Глава V.2 — починка существующего твика.
- RE 1 — поиск селектора.
- Deep-dive 03 — PAC.
