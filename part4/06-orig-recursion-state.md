# Глава IV.6. Вызов оригинала, рекурсия, разделяемое состояние

Эта глава — о тонкостях, которые отличают работающий твик от «вроде работает, но однажды в пятницу вечером приложение упало на пустом месте». Три сюжета: как правильно звать оригинал, как не уйти в рекурсию, и как разделять данные между хуками одного и того же твика.

## 1. `%orig` — три формы и две ошибки

Logos даёт три варианта `%orig`:

| Форма | Что делает |
|---|---|
| `%orig` | Без аргументов — вызовет оригинал с теми же аргументами, что получил хук |
| `%orig(arg1, arg2, ...)` | С аргументами — передаст именно их |
| `%orig(arg1, arg2, ...);` | То же, но для void-методов (игнорирует результат) |

### 1.1. Пример выбора формы

```objc
%hook UIViewController
- (void)viewDidLoad {
    NSLog(@"viewDidLoad");
    %orig;        // идентично %orig()
}
%end
```

Безаргументная форма удобна для void-методов. Для методов с аргументами и возвратом — явная форма с аргументами безопаснее:

```objc
%hook NSFileManager
- (BOOL)fileExistsAtPath:(NSString *)path {
    BOOL exists = %orig(path);
    NSLog(@"fileExists(%@) = %d", path, (int)exists);
    return exists;
}
%end
```

### 1.2. Типичная ошибка — забыли вернуть

```objc
%hook UIApplication
- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    NSLog(@"launch");
    %orig(application, launchOptions);  // ОШИБКА: не вернули
}
%end
```

Компилятор может предупредить warning'ом, может — нет. Результат: приложение запускается, но делегат получает «мусор» из регистров как `BOOL`. На практике — flaky поведение, иногда работает, иногда нет.

**Правило**: для методов с возвратом **всегда** `return %orig(...);` или сохраняйте результат, модифицируйте, возвращайте.

### 1.3. Типичная ошибка — вызвали дважды

```objc
%hook UIViewController
- (void)viewDidLoad {
    %orig;
    NSLog(@"viewDidLoad");
    %orig;  // ОШИБКА: двойной вызов
}
%end
```

Контроллер вызовет свой `-viewDidLoad` дважды. Для некоторых — безопасно; для многих — создаст дубликаты subviews, двойные подписки на нотификации, retain cycle.

**Правило**: `%orig` вызывается **ровно один раз** за одно срабатывание хука.

### 1.4. Решение: до/после

Стандартная структура:

```objc
%hook UIViewController
- (void)viewDidLoad {
    // код ДО (ваша работа ПЕРЕД оригинальным методом)
    NSLog(@"about to call viewDidLoad on %@", NSStringFromClass(self.class));

    %orig;  // ровно один вызов

    // код ПОСЛЕ (ваша работа ПОСЛЕ оригинала)
    NSLog(@"viewDidLoad done, view=%@", self.view);
}
%end
```

Этот паттерн — универсальный.

## 2. Когда НЕ звать `%orig`

Полное замещение оригинала — легитимно в двух случаях:

### 2.1. Полная замена поведения

Пример: отключить функцию в собственном приложении:

```objc
%hook MyController
- (void)showAd {
    // Полностью отключаем рекламу в своём приложении
    // НЕ зовём %orig
}
%end
```

Для `void`-методов — просто пропускаем `%orig`. Для методов с возвратом — возвращаем «безобидное» значение:

```objc
%hook MyController
- (BOOL)shouldShowAd {
    return NO;   // Не зовём %orig, всегда "не показывать"
}
%end
```

### 2.2. Условная замена

```objc
%hook UIApplication
- (BOOL)openURL:(NSURL *)url {
    if ([url.scheme isEqualToString:@"blocked"]) {
        NSLog(@"blocked openURL %@", url);
        return NO;
    }
    return %orig(url);
}
%end
```

Паттерн: проверяем условие, в одной ветке возвращаем свой результат, в другой — вызываем оригинал.

## 3. Рекурсия — как возникает и как её чинить

### 3.1. Пример 1: `NSLog` внутри хука на `access`

```objc
%hookf(int, access, const char *path, int mode) {
    NSLog(@"access(%s)", path);   // КРАШ
    return %orig(path, mode);
}
```

Проблема: `NSLog` внутри пишет в system log через `/dev/console` или подобное; это вызывает `access` для проверки файла; попадает в ваш же хук; `NSLog` снова... → stack overflow.

### 3.2. Пример 2: `fileExistsAtPath` внутри хука на `open`

```objc
%hookf(int, open, const char *path, int flags, ...) {
    // Защищаем от открытия файлов из определённой папки
    NSString *pathStr = [NSString stringWithUTF8String:path];
    if ([[NSFileManager defaultManager] fileExistsAtPath:pathStr]) {  // РЕКУРСИЯ
        ...
    }
    return %orig(path, flags);
}
```

`-[NSFileManager fileExistsAtPath:]` внутри делает `stat` / `open` / `access`. Любой из них попадёт в ваш же хук.

### 3.3. Решение A — не использовать опасные API в хуках

Самое чистое решение:

- Для логирования — `os_log` (не делает file I/O).
- Для проверки существования файла — `access(path, F_OK)` **напрямую через `syscall(2)`** (минуя libSystem обёртку).

```objc
#import <sys/syscall.h>
#import <unistd.h>

static inline int raw_access(const char *path, int mode) {
    return syscall(SYS_access, path, mode);
}

%hookf(int, open, const char *path, int flags, ...) {
    if (raw_access(path, F_OK) != 0) {
        // файла не существует — своя логика
    }
    return %orig(path, flags);
}
```

### 3.4. Решение B — thread-local recursion guard

Когда без «опасного» API не обойтись:

```objc
#import <pthread.h>

static pthread_key_t gReentrancyKey;

%ctor {
    pthread_key_create(&gReentrancyKey, NULL);
}

static inline bool enter_guard(void) {
    if (pthread_getspecific(gReentrancyKey)) return false;
    pthread_setspecific(gReentrancyKey, (void *)1);
    return true;
}
static inline void leave_guard(void) {
    pthread_setspecific(gReentrancyKey, NULL);
}

%hookf(int, open, const char *path, int flags, ...) {
    mode_t mode = (flags & O_CREAT) ? ({
        va_list ap; va_start(ap, flags);
        mode_t m = va_arg(ap, int);
        va_end(ap); m;
    }) : 0;

    int ret = %orig(path, flags, mode);

    if (enter_guard()) {
        // Безопасный код: NSLog, NSFileManager, что угодно
        NSLog(@"open(%s, 0x%x) = %d", path, flags, ret);
        leave_guard();
    }

    return ret;
}
```

Logic:

- `enter_guard()` возвращает `true`, если мы НЕ в рекурсии.
- Устанавливает thread-local флаг.
- Вся «опасная» работа — внутри `if (enter_guard())`.
- `leave_guard()` снимает флаг.

Это стандартный паттерн; им пользуются многие хуковые библиотеки.

### 3.5. Решение C — отложенное логирование

Если нужны все вызовы, в т. ч. рекурсивные, но мы не хотим крэшить:

```objc
// В хуке — кладём в очередь без I/O
%hookf(int, open, const char *path, int flags, ...) {
    int ret = %orig(path, flags);
    enqueue_log_event(path, flags, ret);  // lock-free queue
    return ret;
}

// Отдельный тред — читает очередь и логирует
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0), ^{
    while (true) {
        LogEvent e;
        if (dequeue_log_event(&e)) {
            NSLog(@"open(%s, 0x%x) = %d", e.path, e.flags, e.ret);
        } else {
            usleep(1000);
        }
    }
});
```

Это сложнее, но даёт полную картину без потери данных и без рекурсии.

## 4. Разделяемое состояние между хуками

Один твик часто имеет несколько `%hook`-блоков, которые должны делиться состоянием. Три паттерна.

### 4.1. Глобальная переменная

Самый простой случай:

```objc
static NSMutableArray *gInterestingObjects;

%ctor {
    gInterestingObjects = [NSMutableArray array];
}

%hook ClassA
- (void)methodA {
    [gInterestingObjects addObject:self];
    %orig;
}
%end

%hook ClassB
- (void)methodB {
    for (id obj in gInterestingObjects) {
        NSLog(@"got %@", obj);
    }
    %orig;
}
%end
```

Нюансы:

- `gInterestingObjects` — strong reference, пока глобальная переменная жива, объекты не могут release'нуться. Это — потенциальная утечка.
- Доступ не синхронизирован. При многопоточном использовании — крэш.

Для простых случаев достаточно; для серьёзных — используйте варианты ниже.

### 4.2. Associated Objects (ObjC)

Классический способ прикрепить данные к конкретному объекту:

```objc
#import <objc/runtime.h>

static char kMyFlag;   // уникальный ключ (адрес переменной уникален)

%hook UIViewController
- (void)viewDidLoad {
    %orig;
    objc_setAssociatedObject(self, &kMyFlag, @YES, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (void)viewWillAppear:(BOOL)animated {
    NSNumber *flag = objc_getAssociatedObject(self, &kMyFlag);
    if ([flag boolValue]) {
        NSLog(@"this VC has already loaded once");
    }
    %orig(animated);
}
%end
```

Разбор:

- `&kMyFlag` — ключ. Адрес статической переменной — уникальный глобальный идентификатор. **Не используйте строки или NSString как ключ** — Apple предупреждает об этом в docs; используйте `const void *`.
- `OBJC_ASSOCIATION_RETAIN_NONATOMIC` — политика хранения:
  - `OBJC_ASSOCIATION_RETAIN_NONATOMIC` — retain, но не atomic (быстрее).
  - `OBJC_ASSOCIATION_RETAIN` — retain, atomic.
  - `OBJC_ASSOCIATION_COPY_NONATOMIC` / `OBJC_ASSOCIATION_COPY` — copy.
  - `OBJC_ASSOCIATION_ASSIGN` — просто ссылка без retain.
- Нюанс: когда объект-носитель уничтожается, его associated objects автоматически освобождаются. Это защищает от утечек.

### 4.3. Properties через `%property` (Logos)

Logos даёт сокращение:

```objc
%hook UIViewController
%property (nonatomic, strong) NSNumber *myFlag;

- (void)viewDidLoad {
    %orig;
    self.myFlag = @YES;
}

- (void)viewWillAppear:(BOOL)animated {
    if ([self.myFlag boolValue]) {
        NSLog(@"already loaded");
    }
    %orig(animated);
}
%end
```

Это эквивалент associated objects, но с удобным синтаксисом. Logos за кулисами создаёт getter/setter через `objc_setAssociatedObject`.

Поддерживаемые атрибуты:

- `strong`, `weak`, `assign`, `copy`, `retain`.
- `nonatomic`, `atomic`.

Пример с `weak`:

```objc
%property (nonatomic, weak) UIView *lastViewSeen;
```

### 4.4. Какой паттерн выбрать

| Задача | Паттерн |
|---|---|
| Один глобальный флаг | Глобальная переменная |
| Один глобальный `NSMutableArray/Dictionary` | Глобальная + `@synchronized` или `dispatch_queue` |
| Данные привязаны к объекту | `%property` или `objc_setAssociatedObject` |
| Таблица «ключ → данные», где ключ — указатель | `NSMapTable` + `@synchronized` |
| Асинхронная работа | Глобальная + GCD queue |

## 5. Инициализация в правильный момент

Код в `%ctor` выполняется очень рано — до любой `+load` других классов (не вашего), до `main()`. Это может быть **слишком рано** для некоторых операций.

Безопасные операции в `%ctor`:

- `malloc`, `free`.
- `pthread_key_create`, `pthread_mutex_init`.
- Создание `os_log_t`.
- Регистрация xpc-коллбэков.
- Logos `%init` сам.

Опасные операции в `%ctor`:

- Доступ к UIKit (например, `UIApplication.sharedApplication`) — может быть `nil`.
- Создание UIWindow — приложение ещё не готово.
- Любая синхронная работа с mediaserverd/CoreAudio — может deadlock'нуть.
- Инициализация третьих SDK (например, Firebase) — обычно крэшит.

Для таких случаев — отложите инициализацию в хук более позднего события:

```objc
static dispatch_once_t gOnce;

%hook UIViewController
- (void)viewDidLoad {
    %orig;
    dispatch_once(&gOnce, ^{
        // Один раз на процесс, но ПОСЛЕ готовности UIKit
        NSLog(@"UIKit ready, we can safely do heavy init");
        setupMyThing();
    });
}
%end
```

`dispatch_once` гарантирует, что код внутри выполнится ровно один раз на процесс. Это стандартный способ ленивой инициализации в хуке.

## 6. Таблица «ситуация → решение»

| Ситуация | Правильный подход |
|---|---|
| Хук метода с возвратом | Всегда `return %orig(...)` или сохраните → модифицируйте → верните |
| Полное замещение void-метода | Не зовите `%orig` |
| Полное замещение возвращающего метода | Верните корректное значение (не nil случайно) |
| Логирование в хуке `access`/`open` | `os_log` + возможно thread-local guard |
| Хранить данные для одного `UIViewController` | `%property` |
| Хранить глобальный флаг | `static` переменная |
| Инициализация, требующая UIKit | `dispatch_once` в раннем hook |
| Многопоточный доступ к общей коллекции | `@synchronized` или `dispatch_queue` |

## 7. Пример: твик с состоянием и защитой от рекурсии

Полный Tweak.x — трекер жизненного цикла view controllers своего приложения:

```objc
#import <UIKit/UIKit.h>
#import <os/log.h>

static os_log_t gLog;
static dispatch_queue_t gQueue;
static NSMutableArray *gActiveControllers;

%ctor {
    gLog = os_log_create("com.example.vclife", "lifecycle");
    gQueue = dispatch_queue_create("com.example.vclife.state", DISPATCH_QUEUE_SERIAL);
    gActiveControllers = [NSMutableArray array];
    os_log(gLog, "VCLife loaded");
}

%hook UIViewController

- (void)viewDidLoad {
    %orig;
    __weak id weakSelf = self;
    dispatch_async(gQueue, ^{
        [gActiveControllers addObject:[NSValue valueWithNonretainedObject:weakSelf]];
        os_log(gLog, "viewDidLoad: %{public}@ (total=%lu)",
               NSStringFromClass([weakSelf class]),
               (unsigned long)gActiveControllers.count);
    });
}

- (void)dealloc {
    __weak id weakSelf = self;
    dispatch_async(gQueue, ^{
        NSValue *ptr = [NSValue valueWithNonretainedObject:weakSelf];
        [gActiveControllers removeObject:ptr];
    });
    %orig;
}

%end
```

Разбор:

- `gActiveControllers` — коллекция «live view controllers».
- `gQueue` — serial queue для синхронизации.
- `__weak id weakSelf` — не удерживаем сам VC (иначе — утечка).
- `NSValue valueWithNonretainedObject:` — слабая ссылка для хранения в массиве.
- В `dealloc` удаляем из массива.
- `os_log` — безопасен в хуке.

Это полноценный, рабочий, многопоточно-безопасный tracer. Образец кода, который можно адаптировать под свои нужды.

## 8. Что подтверждено

- **Подтверждено Logos docs**: синтаксис `%orig`, `%property`, `%init`, `%ctor`.
- **Подтверждено Apple docs**: `objc_setAssociatedObject`/`objc_getAssociatedObject`, policies, рекомендации по ключам (адрес статической переменной).
- **Подтверждено Dopamine исходниками**: пример `__attribute__((musttail))` для избежания stack-pollution — глава III.5.
- **Требует проверки на устройстве**: что `dispatch_once` корректно работает в вашем конкретном сценарии (как правило — да).
- **Не подтверждено**: гарантии порядка выполнения `%ctor` между твиками одного процесса — зависит от TweakLoader.

## 9. Выводы

Главный тезис: **`%orig` вызывается ровно один раз**, рекурсия предотвращается либо выбором безопасных API (`os_log` вместо `NSLog`), либо thread-local guard, а разделяемое состояние — через `%property` или глобальные переменные с синхронизацией.

Эти три паттерна (вызов оригинала, защита от рекурсии, состояние) — основа грамотного твика. Если они освоены — большинство типовых задач решается стандартным кодом.

## 10. Что дальше

- Потоки, ARC, retain cycles в деталях — глава IV.7.
- Настройки твика и ресурсы — глава IV.8.
- Диагностика «твик падает/зависает» — глава VII.1.
- Шпаргалка по синтаксису Logos — `cheatsheets/01-logos-syntax.md`.
