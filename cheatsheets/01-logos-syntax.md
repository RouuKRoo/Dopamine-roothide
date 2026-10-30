# Шпаргалка 1. Синтаксис Logos

Logos — препроцессор Theos, превращающий `.x`/`.xm` файлы в чистый `.m`/`.mm`. Все директивы Logos начинаются с `%`.

## Директивы

| Директива | Назначение |
|---|---|
| `%hook ClassName ... %end` | Хук экземплярных методов класса |
| `%hook ClassName.className ... %end` | Hook class-методов (вариант, `+`-методы внутри `%hook`) |
| `%end` | Закрыть `%hook`, `%group`, `%subclass` |
| `%orig` | Вызов оригинала (без аргументов для void) |
| `%orig(a, b)` | Вызов оригинала с аргументами |
| `%hookf(ReturnType, func, args...)` | Хук C-функции (fishhook/ellekit под капотом) |
| `%new` | Новый метод (добавляется в класс) |
| `%new(encoding)` | Новый метод с явным ObjC type encoding |
| `%property` | Associated-object property внутри `%hook` |
| `%class` / `%c(Class)` | Получить класс в рантайме |
| `%init` | Инициализировать все hooks текущей group |
| `%init(Group)` | Инициализировать named group |
| `%init(ClassAlias = cls)` | Инициализировать `%hook ClassAlias` с реальным классом cls |
| `%group Name ... %end` | Группа условно-инициализируемых хуков |
| `%ctor { ... }` | Constructor функция |
| `%dtor { ... }` | Destructor функция |
| `%subclass Child : Parent ... %end` | Создать subclass во время загрузки |
| `%log` | Логировать текущий вызов (помещается внутри хука) |

## Полный Tweak.x каркас

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>

@interface MyPrivateClass : NSObject
- (void)privateMethod;
@end

%group iOS15_18
%hook UIViewController
- (void)viewDidLoad {
    NSLog(@"iOS 15-18 hook");
    %orig;
}
%end
%end

%group iOS26
%hook MyPrivateClass
%property (nonatomic, strong) NSString *myData;

- (void)privateMethod {
    self.myData = @"iOS 26 variant";
    %orig;
}

%new
- (void)myHelper {
    NSLog(@"новый метод");
}
%end
%end

%ctor {
    if (@available(iOS 26.0, *)) {
        Class cls = NSClassFromString(@"MyPrivateClass");
        if (cls) %init(iOS26);
    } else {
        %init(iOS15_18);
    }
}
```

## %hook по условию

Для private классов, не известных на этапе компиляции:

```objc
%hook ClassAlias
- (void)method { /*...*/ %orig; }
%end

%ctor {
    Class realClass = NSClassFromString(@"RealClassName");
    if (realClass) %init(ClassAlias = realClass);
}
```

## %new — пример

```objc
%hook UIViewController
%new
- (void)appendCustomBehavior {
    NSLog(@"мой метод добавлен ко всем UIViewController");
}
%end

// В другом коде:
[someVC appendCustomBehavior];
```

## %hookf — пример

```objc
%hookf(int, open, const char *path, int flags, ...) {
    mode_t mode = 0;
    if (flags & O_CREAT) {
        va_list ap; va_start(ap, flags);
        mode = va_arg(ap, int); va_end(ap);
    }
    int ret = %orig(path, flags, mode);
    NSLog(@"open(%s) = %d", path, ret);
    return ret;
}
```

Если нужно задать имя функции динамически:

```objc
%hookf(int, _unused_name)(const char *path, int flags) {
    return %orig(path, flags);
}

%ctor {
    void *fn = MSFindSymbol(image, "__some_symbol");
    %init(_unused_name = fn);
}
```

## Типовые ошибки

| Симптом | Причина |
|---|---|
| `undefined reference to _logos_orig$...` | Logos не отработал — вызовите `make clean` |
| `use of undeclared identifier` на имени класса | Класс не известен — объявите `@interface` вручную |
| `use of %orig outside of hook` | `%orig` может использоваться только внутри `%hook` метода |
| Хук не срабатывает после установки | Забыли `%init` для named group |
| `%new` не компилируется | Укажите type encoding: `%new(v@:)` для `void foo()` |

## См. также

- Theos docs: `$THEOS/vendor/logos/documentation/`.
- Глава IV.4 — первый `%hook`.
- Глава IV.5 — `%hookf`.
- Глава IV.6 — `%orig`, рекурсия, состояние.
