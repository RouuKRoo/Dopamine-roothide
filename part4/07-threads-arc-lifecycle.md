# Глава IV.7. Потоки, ARC, жизненный цикл

Три слова, которые отвечают за 80% труднодиагностируемых крэшей и утечек в твиках. В этой главе разберём каждое отдельно и как они сочетаются, с конкретными примерами «что не делать».

## 1. Потоки — где выполняется ваш хук

Главное правило: **ваш хук выполняется на том же потоке, что и перехваченный метод**. Если iOS вызывает `-[UIViewController viewDidLoad]` на main thread — ваш хук тоже на main thread. Если callback прилетает на background thread — хук там же.

Это даёт две категории проблем.

### 1.1. Не трогайте UIKit с background thread

```objc
%hookf(int, SomeC_FunctionCalledFromBackground, ...) {
    [UIApplication.sharedApplication keyWindow].backgroundColor = UIColor.redColor;
    //    ^^^^^^^^^^^^^^^^^ ТРОГАТЬ UIKit С BACKGROUND — КРАШ (или undefined)
    return %orig(...);
}
```

Правило: всё, что начинается с `UI*`, — только main thread. Чтобы безопасно вызвать UI-операцию из любого треда:

```objc
dispatch_async(dispatch_get_main_queue(), ^{
    UIApplication.sharedApplication.keyWindow.backgroundColor = UIColor.redColor;
});
```

### 1.2. Долгие операции на main thread — зависание UI

```objc
%hook UIViewController
- (void)viewDidLoad {
    %orig;
    // ОШИБКА: синхронная сеть на main thread
    NSData *d = [NSData dataWithContentsOfURL:[NSURL URLWithString:@"https://example.com/big.json"]];
    NSLog(@"got %lu bytes", d.length);
}
%end
```

Хук выполняется на main thread (viewDidLoad), сетевой запрос блокирует UI, iOS убивает приложение по watchdog.

Правильно:

```objc
%hook UIViewController
- (void)viewDidLoad {
    %orig;
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSData *d = [NSData dataWithContentsOfURL:[NSURL URLWithString:@"https://example.com/big.json"]];
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"got %lu bytes", d.length);
        });
    });
}
%end
```

Серьёзная работа — на background queue; UI-работа после — обратно на main.

### 1.3. Multiple threads одновременно в вашем хуке

```objc
static NSMutableArray *gList;  // БЕЗ синхронизации

%hook SomeClass
- (void)methodA {
    [gList addObject:@"a"];  // Если methodA вызывается из разных threads — крэш
    %orig;
}
%end
```

`NSMutableArray` **не** thread-safe. Два потока одновременно `addObject:` → неопределённое поведение, часто crash с «Collection was mutated while being enumerated».

Решения:

**Serial queue** (рекомендован):

```objc
static NSMutableArray *gList;
static dispatch_queue_t gQueue;

%ctor {
    gList = [NSMutableArray array];
    gQueue = dispatch_queue_create("com.example.list.queue", DISPATCH_QUEUE_SERIAL);
}

%hook SomeClass
- (void)methodA {
    dispatch_async(gQueue, ^{
        [gList addObject:@"a"];
    });
    %orig;
}
%end
```

**`@synchronized`** (проще, но медленнее):

```objc
%hook SomeClass
- (void)methodA {
    @synchronized (gList) {
        [gList addObject:@"a"];
    }
    %orig;
}
%end
```

**`NSLock`** (легаси, редко стоит использовать).

### 1.4. Потокобезопасный Objective-C vs не

Общее правило:

- **Thread-safe**: `NSArray`, `NSDictionary`, `NSString` (immutable). `dispatch_*` queues. `OSAtomic*` (deprecated — используйте `atomic_*` C11).
- **НЕ thread-safe**: `NSMutableArray`, `NSMutableDictionary`, `NSMutableString`. Любой `UIView`/`UIViewController`. CoreData managed objects (вне своего контекста).

## 2. ARC и твики

### 2.1. Что такое ARC

Automatic Reference Counting — компилятор автоматически вставляет `retain`/`release`/`autorelease` в объекты Objective-C. Включается через `-fobjc-arc` в CFLAGS (мы это делаем в Makefile).

Правило: **включайте ARC**, если нет особых причин не включать. ARC убирает 90% ошибок памяти, оставляя только retain cycles.

### 2.2. Retain cycles в твиках

Основная проблема — «блок захватывает self, self держит блок» или «associated object удерживает хост-объект».

**Пример цикла с блоком**:

```objc
%hook UIViewController
%property (nonatomic, strong) id something;

- (void)viewDidLoad {
    %orig;
    self.something = ^{
        [self doStuff];   // self в блоке → блок retain'ит self, self retain'ит блок
    };
}
%end
```

Правильно:

```objc
- (void)viewDidLoad {
    %orig;
    __weak typeof(self) weakSelf = self;
    self.something = ^{
        [weakSelf doStuff];
    };
}
```

### 2.3. Associated objects и жизненный цикл

```objc
static char kMyData;

%hook UIViewController
- (void)viewDidLoad {
    %orig;
    NSMutableArray *data = [NSMutableArray new];
    objc_setAssociatedObject(self, &kMyData, data, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    // data живёт, пока жив self; dealloc self автоматически освободит
}
%end
```

Это **безопасно**: когда VC уничтожается, `data` релизится.

**Опасный паттерн**:

```objc
%hook SomeObject
- (instancetype)init {
    self = %orig;
    if (self) {
        objc_setAssociatedObject(self, &kMyData, self, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
        //                                       ^^^^ объект держит ссылку на самого себя через associated
    }
    return self;
}
%end
```

Objект никогда не освободится — вы создали retain cycle через associated object.

**Решение**: используйте `OBJC_ASSOCIATION_ASSIGN` или `weak` через враппер.

### 2.4. Блоки в хуках

Если ваш хук запускает блок на будущее:

```objc
%hook UIViewController
- (void)viewDidLoad {
    %orig;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 5 * NSEC_PER_SEC),
                   dispatch_get_main_queue(), ^{
        // блок захватит self; но мы не хотим мешать dealloc
        __weak UIViewController *weakSelf = self;
    // здесь weakSelf может быть nil, если VC закрылся
    });
}
%end
```

Паттерн тот же: `__weak` на self, проверка на nil в блоке.

## 3. Жизненный цикл объектов

### 3.1. `+load` vs `+initialize`

ObjC runtime имеет два «статических инициализатора» для классов:

- **`+load`** вызывается при загрузке образа (image load). Для вашей dylib — вскоре после `%ctor`.
- **`+initialize`** вызывается при первом обращении к классу или его подклассу.

В твиках чаще используйте `%ctor`, но иногда полезно:

```objc
%hook MyClass
+ (void)initialize {
    %orig;
    NSLog(@"MyClass initialized");
    // здесь можно безопасно сделать [[self alloc] init] без рекурсии в +initialize
}
%end
```

Правила:

- **НЕ** используйте `+load` в хуках — за вас это делает `%ctor`.
- **Можно** использовать `+initialize`, если нужно «один раз на класс».

### 3.2. `-dealloc` в хуках

```objc
%hook SomeClass
- (void)dealloc {
    NSLog(@"deallocating %@", self);
    %orig;  // ВАЖНО: %orig в конце
}
%end
```

Правила `dealloc`:

- `%orig` — **в конце**, иначе работа идёт с частично освобождённым объектом.
- **Не** вызывайте методы `self` в `dealloc` после `%orig` — объект уже уничтожен.
- **Не** создавайте в `dealloc` блок, захватывающий `self` — объект мёртв.
- Associated objects автоматически освобождаются **после** `dealloc` — так что внутри `dealloc` они ещё доступны.

### 3.3. `-init` в хуках

```objc
%hook SomeClass
- (instancetype)init {
    self = %orig;
    if (self) {
        // инициализация ваших associated objects
        self.myCounter = @0;
    }
    return self;
}
%end
```

Правила:

- `%orig` **первой**.
- Проверка `if (self)` — на случай возврата `nil` от parent init.
- Возврат `self` обязателен.

Нюанс: не каждый `init*` метод вызывает другой `init`. У `UIViewController` есть `-initWithNibName:bundle:`, `-initWithCoder:`, `-init`. Хукать только `-init` может пропустить объекты, созданные через nib.

Решение — хукать `+allocWithZone:` или `+alloc`, но это ломает runtime. Проще — хукнуть каждый init-вариант отдельно.

## 4. Dispatch queues в твиках

### 4.1. Какие queues использовать

| Queue | Когда |
|---|---|
| `dispatch_get_main_queue()` | UIKit, UI изменения |
| `dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)` | Общая фоновая работа |
| `dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0)` | Низкоприоритетное (I/O, кеш) |
| свой `dispatch_queue_create("...", DISPATCH_QUEUE_SERIAL)` | Защита shared state |
| свой concurrent | Редко в твиках |

### 4.2. Не блокируйте main queue

```objc
dispatch_sync(dispatch_get_main_queue(), ^{
    // если мы уже на main — DEADLOCK
});
```

Если нужно гарантированно на main:

```objc
if ([NSThread isMainThread]) {
    // уже на main — вызываем прямо
} else {
    dispatch_sync(dispatch_get_main_queue(), ^{...});
}
```

Или используйте `dispatch_async` вместо `dispatch_sync` всегда, когда не нужен немедленный ответ.

## 5. Утечки и как их видеть

### 5.1. Типовые источники утечек в твиках

1. **Associated objects с strong**, которые ссылаются обратно на хост.
2. **Блоки, захватывающие self**, которые хранятся в property.
3. **NSTimer** без invalidate (таймер держит target).
4. **NSNotificationCenter observer** без removeObserver (observer держит target).
5. **CF* объекты без CFRelease**.

### 5.2. Инструменты диагностики

- **Instruments на Xcode** — Allocations, Leaks шаблоны. Работает, если вы можете подключить debugger к процессу.
- **vmmap** через SSH:

```bash
ssh mobile@<ip>
sudo vmmap <pid> | grep -i object
```

Показывает регионы памяти процесса.

- **`task_info`** через свой helper + `MACH_TASK_BASIC_INFO` — можно читать resident_size процесса.
- **os_signpost** вокруг подозрительных операций — потом анализируете в Instruments.

### 5.3. Профилактика

Правила «не создавать утечку»:

1. `__weak typeof(self) weakSelf = self;` для блоков с self.
2. `[center removeObserver:self]` в `dealloc` для `NSNotificationCenter`.
3. `[timer invalidate]` для `NSTimer`.
4. `CFRelease(...)` для `Create`/`Copy` функций.
5. Associated objects на self самому — только `OBJC_ASSOCIATION_ASSIGN`.

## 6. Пример: твик с правильной работой с потоками и памятью

Твик-metric — считает количество `-[UIView drawRect:]` для своих `UIView` в разных контроллерах, безопасно:

```objc
#import <UIKit/UIKit.h>
#import <os/log.h>
#import <objc/runtime.h>

static os_log_t gLog;
static dispatch_queue_t gQueue;
static NSMutableDictionary<NSString *, NSNumber *> *gCounters;

%ctor {
    gLog = os_log_create("com.example.drawmetric", "metric");
    gQueue = dispatch_queue_create("com.example.drawmetric.q", DISPATCH_QUEUE_SERIAL);
    gCounters = [NSMutableDictionary dictionary];
}

%hook UIView

- (void)drawRect:(CGRect)rect {
    %orig(rect);

    NSString *className = NSStringFromClass(self.class);
    dispatch_async(gQueue, ^{
        NSNumber *c = gCounters[className] ?: @0;
        gCounters[className] = @(c.intValue + 1);
    });
}

%end

%hook UIApplication

- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    BOOL ret = %orig(application, launchOptions);
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 10 * NSEC_PER_SEC),
                   dispatch_get_main_queue(), ^{
        dispatch_sync(gQueue, ^{
            os_log(gLog, "drawRect counters: %{public}@", gCounters);
        });
    });
    return ret;
}

%end
```

Разбор:

- `gCounters` — mutable dictionary, доступ только через `gQueue` (serial queue).
- `drawRect:` — любые UIView вызывают на main thread, но мы перебрасываем работу на свою queue для thread-safety.
- `dispatch_after` + `dispatch_sync` для чтения — доступ синхронизирован.
- `os_log` безопасен из любого треда.
- `NSStringFromClass` возвращает кэшированную строку, не выделяет новую память.

Этот код **потокобезопасен**, **без утечек**, **не блокирует UI**.

## 7. Таблица частых ошибок

| Ошибка | Проявление | Фикс |
|---|---|---|
| UIKit с background thread | Непонятные крэши в UIKit internals | `dispatch_async(main_queue, ...)` |
| Synchronous сеть на main | Watchdog kill через 10с | Background queue + async |
| `NSMutableArray` без синхронизации | «Collection mutated while enumerated» | Serial queue или `@synchronized` |
| Retain cycle через блок | Утечка объектов | `__weak typeof(self) weakSelf` |
| `%orig` первым в `dealloc` | Крэш после первого dealloc | `%orig` **в конце** `dealloc` |
| Доступ к `self` после `%orig` в dealloc | Крэш (dangling pointer) | Не делать работу с `self` после `%orig` |
| `+load` в хуке | Непредсказуемый порядок | Используйте `%ctor` или `+initialize` |

## 8. Что подтверждено

- **Подтверждено Apple docs**: `-fobjc-arc`, `dispatch_*` functions, `objc_setAssociatedObject` policies.
- **Подтверждено практикой**: `__weak typeof(self)` — паттерн из Apple sample code.
- **Подтверждено Dopamine исходниками**: пример `pthread_mutex_t` защиты в разных местах `BaseBin/`.
- **Требует проверки на устройстве**: конкретное поведение watchdog на iOS 26 (точный порог блокировки main thread) — ориентир 10 секунд, но может варьироваться.

## 9. Выводы

Главный тезис: **в твике нет «безнаказанного» кода**. Любая `NSMutableArray` без синхронизации, любой блок без `__weak`, любая UIKit-операция с background thread — потенциальный краш в проде.

Три правила, которые решают 80% проблем:

1. UIKit — только main thread; тяжёлая работа — background queue.
2. Shared mutable state — через serial queue или `@synchronized`.
3. Блоки, захватывающие self — через `__weak`.

Остальное — детали, но критичные детали.

## 10. Что дальше

- Настройки твика через PreferenceBundle — глава IV.8.
- Совместимость с RootHide (пути в разных местах) — глава IV.9.
- Сборка и инспекция `.deb` — глава IV.10.
- Полный учебный проект с тестируемой логикой — глава IV.13.
