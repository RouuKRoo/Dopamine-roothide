# Глава V.3. Runtime-патчи поведения приложения

Третья категория патчей — **runtime**: вы не меняете исходный код приложения, не меняете бинарник на диске; вы меняете поведение в памяти загруженного процесса. Это и есть то, что делает Logos `%hook`, но стоит разобрать это в терминах патча — чтобы понимать границы, когда это уместно и когда — нет.

## 1. Что это такое

Runtime-патч — изменение поведения уже-работающего кода без модификации артефакта на диске.

Три подвида:

- **ObjC method swizzling** — подмена `IMP` в таблице методов класса. Это то, что делает `%hook ClassA` + метод.
- **C-функции** — подмена через fishhook (GOT) или inline (ellekit).
- **Прямое изменение памяти** — через `vm_write` (через `hookd`) или через ваш собственный код в адресном пространстве.

Для автора твика это три разные техники с разными границами применимости.

## 2. Когда уместен runtime-патч

Таблица «ситуация → подход»:

| Ситуация | Подход |
|---|---|
| Исправить поведение в приложении, которое не ваше | Runtime-патч (вы не можете собрать его исходники) |
| Добавить логирование в стороннее приложение | Runtime-патч |
| Отключить jailbreak-detection в стороннем приложении | Runtime-патч; но подумайте об этике |
| Исправить баг в твике, исходники которого есть | Патч исходников (глава V.2) |
| Исправить баг в самом Dopamine | Патч исходников (глава V.1) |
| Изменить логику приложения, но хотите, чтобы приложение могло быть удалено и переустановлено без потери патча | Runtime-патч (патч на диск теряется при переустановке) |
| Нужна максимальная производительность с минимумом overhead | Иногда binary patch (глава V.4), иногда runtime |

## 3. ObjC method swizzling в деталях

Основной инструмент — `%hook` из Logos. Разбор того, что происходит.

### 3.1. Было

```objc
@implementation MyClass
- (BOOL)isAuthorized {
    return [self checkCredentials];
}
@end
```

При загрузке:

- `MyClass` зарегистрирован в ObjC runtime.
- Селектор `isAuthorized` имеет IMP, указывающий на нужный метод.

### 3.2. После вашего `%hook`

```objc
%hook MyClass
- (BOOL)isAuthorized {
    NSLog(@"intercepted");
    return YES;   // всегда authorized
}
%end
```

При `%init`:

1. Logos вызывает `class_getInstanceMethod(objc_getClass("MyClass"), @selector(isAuthorized))`.
2. Получает `Method m` (структура с указателем IMP).
3. Через `method_setImplementation(m, new_imp)` подменяет IMP.
4. Старая IMP сохраняется в `_logos_orig$MyClass$isAuthorized`.

С этого момента **все** вызовы `-isAuthorized` на объектах `MyClass` идут в ваш код.

### 3.3. Как перехватить одного объекта, не всего класса

ObjC runtime поддерживает **isa-swizzling**: создать подкласс на лету и заменить `isa` у одного экземпляра.

Пример:

```objc
#import <objc/runtime.h>

- (void)patchSingleObject:(id)obj {
    Class origClass = object_getClass(obj);
    const char *subclassName = [[NSString stringWithFormat:@"%s_Patched_%p",
                                 class_getName(origClass), obj] UTF8String];

    Class subclass = objc_allocateClassPair(origClass, subclassName, 0);
    if (!subclass) return;  // уже создан

    // Добавить переопределённый метод
    SEL sel = @selector(isAuthorized);
    IMP newImp = imp_implementationWithBlock(^BOOL(id self) {
        return YES;
    });
    class_addMethod(subclass, sel, newImp, "B@:");  // B = BOOL, @ = id, : = SEL
    objc_registerClassPair(subclass);

    // Поменять isa у объекта
    object_setClass(obj, subclass);
}
```

Теперь только `obj` имеет этот override; остальные экземпляры `MyClass` работают как обычно.

Это — очень тонкая работа, применяется редко (например, в KVO внутри ObjC runtime).

## 4. C-функции через fishhook/ellekit

Детальный разбор был в главе IV.5. Напомним: `%hookf` для функций из других dylib (GOT-based), `EKHookFunction` для any direct call (inline-based).

**Пример применения для runtime-патча**: исправить баг в чужой dylib через хук на её функцию. 

Сценарий: приложение использует библиотеку `libExampleSDK.dylib`, в ней функция `check_license()`, возвращающая `int`. На новой iOS она возвращает `-1` из-за проверки, которая стала строже. Нам нужно, чтобы она возвращала `0` (успех) в контексте нашего собственного тестового приложения.

```objc
%hookf(int, check_license) {
    int ret = %orig;
    // Логируем настоящий результат
    os_log(gLog, "check_license returned %d (overriding)", ret);
    return 0;  // всегда успех
}
```

**Этическое правило**: такой runtime-патч легитимен только для **собственного** приложения или в учебных целях. Патч лицензии в чужом коммерческом приложении — это DMCA-нарушение; не рекомендую.

## 5. Прямое изменение памяти

Самый агрессивный путь — патч инструкций в памяти процесса.

### 5.1. Случай: in-process (ваш же процесс)

Ваш твик работает в том же процессе, где функция, которую вы хотите пропатчить. Пример — изменить несколько байт функции:

```c
#include <sys/mman.h>
#include <mach/mach.h>

bool patch_in_place(void *addr, const void *bytes, size_t len) {
    // Разрешить RW на страницу
    uintptr_t page = (uintptr_t)addr & ~(getpagesize() - 1);
    size_t pageSpan = ((uintptr_t)addr + len - page + getpagesize() - 1) & ~(getpagesize() - 1);

    kern_return_t kr = vm_protect(mach_task_self(), page, pageSpan, false,
                                   VM_PROT_READ | VM_PROT_WRITE | VM_PROT_COPY);
    if (kr != KERN_SUCCESS) return false;

    memcpy(addr, bytes, len);

    kr = vm_protect(mach_task_self(), page, pageSpan, false,
                    VM_PROT_READ | VM_PROT_EXECUTE);
    return (kr == KERN_SUCCESS);
}

// Применение:
void *target = (void *)check_license;  // адрес функции
uint8_t nopReturn[] = {
    0x00, 0x00, 0x80, 0xD2,  // mov x0, #0
    0xC0, 0x03, 0x5F, 0xD6   // ret
};
patch_in_place(target, nopReturn, sizeof(nopReturn));
```

Это: подменили прологовые инструкции `check_license` на «return 0». Функция перестала делать проверку, всегда возвращает 0.

**Это агрессивный патч**. Он работает, но:

- Не универсален: инструкции ABI-specific (arm64).
- Хрупок: любое обновление целевой функции сломает патч.
- Не отменяем в обычном `%orig`-style — вы потеряли оригинальный пролог.

Лучшая альтернатива — fishhook/ellekit, который делает это внутри «культурно» (сохраняет оригинал в trampoline).

### 5.2. Случай: in другой process (через `hookd`)

Если нужно пропатчить **чужой** процесс (куда ваш твик не загружается), используйте `hookd` (глава III.7):

```c
#include <libjailbreak/hookd.h>
#include <mach/mach.h>

int patch_remote_process(pid_t pid, uint64_t addr, uint8_t *bytes, size_t len) {
    task_port_t taskPort = MACH_PORT_NULL;
    if (task_for_pid(mach_task_self(), pid, &taskPort) != KERN_SUCCESS) return -1;

    kern_return_t kr = hookd_hook(taskPort, addr, bytes, len);

    mach_port_deallocate(mach_task_self(), taskPort);
    return (kr == KERN_SUCCESS) ? 0 : -2;
}
```

Детали — глава III.7.

## 6. Пример: учебный runtime-патч isAuthorized

Задача: в собственном тестовом приложении (`com.example.testbed`) ваш `MyService` имеет метод `-isAuthorized`, который по бизнес-логике иногда возвращает NO. Для unit-тестирования UI вы хотите, чтобы он **всегда** возвращал YES.

### 6.1. Код твика

Полный Tweak.x:

```objc
#import <Foundation/Foundation.h>
#import <os/log.h>

static os_log_t gLog;

@interface MyService : NSObject
- (BOOL)isAuthorized;
@end

%ctor {
    gLog = os_log_create("com.example.alwaysauth", "patch");
    os_log(gLog, "AlwaysAuth loaded");
}

%hook MyService

- (BOOL)isAuthorized {
    BOOL original = %orig;
    os_log(gLog, "isAuthorized original=%d, forcing YES", (int)original);
    return YES;
}

%end
```

Разбор:

- `@interface MyService : NSObject` — декларация класса, чтобы Logos знал его сигнатуру.
- `-isAuthorized` всегда возвращает YES, но мы сохраняем настоящее значение в лог — для диагностики.

### 6.2. Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = AlwaysAuth

AlwaysAuth_FILES = Tweak.x
AlwaysAuth_CFLAGS = -fobjc-arc
AlwaysAuth_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

### 6.3. Filter plist

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
</dict>
```

### 6.4. control

```
Package: com.example.alwaysauth
Name: AlwaysAuth
Version: 0.0.1
Architecture: iphoneos-arm
Description: Force isAuthorized=YES in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

### 6.5. Проверка

Собрать, установить, запустить тестовое приложение. Везде, где раньше `[service isAuthorized]` возвращал NO, теперь YES. В логе:

```
AlwaysAuth loaded
isAuthorized original=0, forcing YES
isAuthorized original=0, forcing YES
```

## 7. Контракт поведения — что ломается при подмене

Критическая мысль: **ваш runtime-патч должен сохранять контракт поведения метода**.

### 7.1. Что это значит

Если метод возвращает `BOOL`, вы можете вернуть любой `BOOL`. Но:

- Если метод документирован как «возвращает NO, если X» — приложение может **полагаться** на этот контракт.
- Если контракт нарушен, последующий код приложения ведёт себя непредсказуемо.

Пример опасной подмены:

```objc
%hook NSFileManager
- (BOOL)removeItemAtPath:(NSString *)path error:(NSError **)error {
    return YES;  // ВСЕГДА "успех"
}
%end
```

Проблема: приложение после «успеха» ожидает, что файла нет. Но файл есть. Следующие операции (например, запись в тот же путь) ведут себя странно.

### 7.2. Защитные правила

1. **Логируйте настоящее значение** — чтобы знать, когда ваш override маскирует проблему.
2. **Проверяйте побочные эффекты** — если `%orig` **должен** был что-то сделать (создать файл, добавить в коллекцию), и вы не зовёте `%orig` — вы ломаете контракт.
3. **Выбирайте, когда патчить** — лучше условный патч (например, только для определённого объекта или в определённом сценарии), чем безусловный.

### 7.3. Пример условного патча

```objc
%hook NSFileManager

- (BOOL)removeItemAtPath:(NSString *)path error:(NSError **)error {
    // Патчим только для нашего "магического" пути
    if ([path containsString:@"/tmp/magic-test"]) {
        if (error) *error = nil;
        return YES;  // имитация успеха
    }
    // Остальное — нормально
    return %orig(path, error);
}

%end
```

Это намного безопаснее, чем безусловный `return YES`.

## 8. Таблица «что патчить, что — нет»

| Что патчить | Что НЕ патчить |
|---|---|
| `-isAuthorized` вашего собственного MyService | `-authorized` системного account manager |
| Debug UI показатели в собственном приложении | Телеметрия/аналитика (ethic risk) |
| Поведение в собственном UI тесте | Antifraud проверки чужого приложения |
| Логирование не влияющее на возврат | Return-value методов, от которых зависит безопасность |

## 9. Отладка «патч вроде работает, но что-то ломается»

Таблица симптомов:

| Симптом | Диагноз | Фикс |
|---|---|---|
| Патч работает, но приложение падает позже | Нарушен контракт метода | Логировать настоящее значение, проверить, зависит ли app от этого |
| Патч применяется не всегда | Хук на класс, но метод реализован в subclass | Хукать subclass, или всю иерархию |
| Хук перехватывает слишком много объектов | Классовый хук затронул «чужие» объекты | isa-swizzling или проверка `self.class` в начале |
| После обновления iOS патч не работает | Селектор переименован или метод inlined | Переделать на другую точку перехвата (глава V.2) |

## 10. Чеклист перед релизом runtime-патча

```
[ ] Патч точечный — только нужный метод, не вся иерархия
[ ] Условие активации проверено — не меняет поведение везде
[ ] Настоящее значение сохранено в лог (для диагностики)
[ ] Контракт метода сохранён (не возвращаем nil где не должно)
[ ] Нет глобальных побочных эффектов
[ ] Патч не нарушает безопасность других пользователей
[ ] Патч не маскирует системную проблему (которую стоит починить иначе)
[ ] Для внешнего приложения — проверена этика
```

## 11. Что подтверждено

- **Подтверждено Apple docs**: `method_setImplementation`, `objc_allocateClassPair`, `object_setClass`, `imp_implementationWithBlock`.
- **Подтверждено Dopamine**: `hookd_hook` API — глава III.7, `BaseBin/libjailbreak/src/hookd.h`.
- **Подтверждено Logos**: `%hook` реализован через `method_setImplementation` плюс сохранение оригинала в `_logos_orig$...`.
- **Требует проверки на устройстве**: что конкретный приём (isa-swizzling, in-place patch) работает для вашего целевого класса/функции.
- **Не подтверждено**: что все C-функции в iOS 26 стабильно хукаются inline — Apple периодически inline'ит некоторые.

## 12. Выводы

Главный тезис: **runtime-патч — это точечное изменение поведения в живом процессе**. Самый безопасный путь — `%hook` для ObjC и `%hookf` для C. Самый агрессивный — прямая запись в память (через `hookd` для чужих процессов или через `vm_protect`+`memcpy` для своего).

Три якорных правила:

1. **Логируйте оригинальный результат** — всегда, в любом хуке.
2. **Условный патч лучше безусловного** — проверка `if (specific_condition)` экономит нервы.
3. **Контракт метода — священен** — не возвращайте `nil` там, где метод обещал не-nil; не возвращайте `YES` без побочного эффекта, который метод должен был создать.

## 13. Что дальше

- Mach-O / метаданные (binary patch) — глава V.4.
- Контроль качества патча — глава V.5.
- Для внешних приложений и анти-detect — соответствующие рецепты Cookbook.
- Для исследования поведения перед патчем — глава VI.1.
