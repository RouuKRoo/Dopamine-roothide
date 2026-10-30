# Кейс 12. Использование ellekit как рантайм-хукер напрямую

Logos `%hook` и `%hookf` — удобный синтаксический сахар над ellekit API. Иногда полезно работать с ellekit **напрямую**: когда условие хука вычисляется в рантайме, когда нужны специфические параметры, или когда проект построен без Theos. В этом кейсе — три полноценных примера.

## 1. Что такое ellekit

ellekit — hook runtime, устанавливается на устройстве через пакет `ellekit`. Предоставляет:

- `EKHookFunction(...)` — inline hook C-функций.
- `EKHookMessage(...)` — swizzle ObjC методов.
- `EKHookMemory(...)` — raw write в executable память.

Эти функции **включены в саму ellekit.dylib**; подключаются через `<ellekit.h>` + линковка с `-lellekit`.

## 2. Общая Makefile для «ellekit напрямую»

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = EKDirect

EKDirect_FILES = Tweak.m
EKDirect_CFLAGS = -fobjc-arc
EKDirect_FRAMEWORKS = UIKit Foundation
EKDirect_LIBRARIES = root ellekit

include $(THEOS_MAKE_PATH)/tweak.mk
```

Ключевое: `_FILES = Tweak.m` (не `.x`!) и `_LIBRARIES = root ellekit`.

## 3. Пример A: ObjC swizzle через EKHookMessage

Задача: хукнуть `-[UIViewController viewDidLoad]` без Logos.

```objc
// Tweak.m
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <os/log.h>
#import <ellekit.h>

static os_log_t gLog;

static void (*orig_viewDidLoad)(UIViewController *self, SEL _cmd);

static void my_viewDidLoad(UIViewController *self, SEL _cmd) {
    os_log(gLog, "viewDidLoad in %{public}@", NSStringFromClass(self.class));
    orig_viewDidLoad(self, _cmd);
}

__attribute__((constructor))
static void initializer(void) {
    gLog = os_log_create("com.example.ekdirect", "swizzle");

    Class cls = [UIViewController class];
    SEL sel = @selector(viewDidLoad);
    Method m = class_getInstanceMethod(cls, sel);
    IMP origIMP = method_getImplementation(m);

    EKHookMessage(cls, sel, (IMP)my_viewDidLoad, (IMP *)&orig_viewDidLoad);

    os_log(gLog, "hook installed, orig IMP = %p", origIMP);
}
```

Разбор:

- `EKHookMessage` — ellekit API: подменяет IMP метода, сохраняет оригинал.
- `orig_viewDidLoad` — указатель, через который можно вызвать оригинал.
- Параметры: `(Class, SEL, newIMP, *origIMPOut)`.

Это эквивалентно `%hook UIViewController` + `- (void)viewDidLoad { %orig; os_log(...); }`, но без Logos.

## 4. Пример B: C-функция через EKHookFunction

Задача: хукнуть `access` в собственном приложении.

```objc
// Tweak.m (продолжение или отдельный файл)
#import <unistd.h>
#import <ellekit.h>

static int (*orig_access)(const char *path, int mode);

static int my_access(const char *path, int mode) {
    int ret = orig_access(path, mode);
    os_log(gLog, "access(%{public}s, 0x%x) = %d", path, mode, ret);
    return ret;
}

__attribute__((constructor))
static void init_c_hooks(void) {
    gLog = os_log_create("com.example.ekdirect", "chook");
    EKHookFunction((void *)access, (void *)my_access, (void **)&orig_access);
}
```

Разбор:

- `EKHookFunction` — inline hook: записывает `BR` в начало `access`.
- Параметры: `(target, replacement, *origOut)`.
- `orig_access` указатель будет на trampoline, в котором восстановлены первые инструкции настоящей `access` плюс jump обратно.

Это эквивалентно `%hookf(int, access, ...)`.

## 5. Пример C: условный хук в рантайме

Задача: хукнуть метод только если класс действительно существует.

```objc
#import <objc/runtime.h>

static void (*orig_myMethod)(id self, SEL _cmd, NSString *arg);

static void my_myMethod(id self, SEL _cmd, NSString *arg) {
    os_log(gLog, "intercepted: %{public}@", arg);
    orig_myMethod(self, _cmd, arg);
}

__attribute__((constructor))
static void init_conditional(void) {
    gLog = os_log_create("com.example.ekdirect", "cond");

    const char *className = "RuntimeKnownClass";
    Class cls = objc_getClass(className);
    if (!cls) {
        os_log(gLog, "class %s not found, skip", className);
        return;
    }

    SEL sel = sel_registerName("myMethod:");
    Method m = class_getInstanceMethod(cls, sel);
    if (!m) {
        os_log(gLog, "method %s not found", sel_getName(sel));
        return;
    }

    EKHookMessage(cls, sel, (IMP)my_myMethod, (IMP *)&orig_myMethod);
    os_log(gLog, "hook installed on %s %s", className, sel_getName(sel));
}
```

Разбор:

- Класс и селектор определяются **в рантайме** (`objc_getClass`, `sel_registerName`).
- Условие — `cls && m` — защита от «класса нет на этой версии iOS».
- Эквивалент Logos: `%ctor { Class c = NSClassFromString(...); if (c) %init(Alias = c); }` + `%hook Alias`.

## 6. Пример D: raw memory patch

Задача: записать несколько байт по адресу (редко нужно, но ellekit предоставляет).

```c
#include <ellekit.h>

static void patch_bytes(void) {
    extern void *target_function;    // предположим, адрес известен
    uint8_t patch[] = {
        0x00, 0x00, 0x80, 0xD2,     // mov x0, #0
        0xC0, 0x03, 0x5F, 0xD6      // ret
    };
    EKHookMemory((void *)target_function, patch, sizeof(patch));
}
```

Это **агрессивный патч** — переписывает первые 8 байт функции на «return 0». После этого вызовов `target_function` возвращает 0.

Используйте только:

- В собственном приложении.
- Когда нужен **полный** no-op или подмена, а не хук с orig.
- Понимая ARM64 ABI (без изменения ABI, иначе crash).

## 7. Таблица «Logos vs ellekit direct»

| Задача | Logos | ellekit direct |
|---|---|---|
| Простой ObjC swizzle | `%hook Class` + method | `EKHookMessage(class, sel, imp, &orig)` |
| C-функция hook | `%hookf(ReturnType, name, args)` | `EKHookFunction(fn, replacement, &orig)` |
| Условный хук | `%init(Alias = cls)` | `if (cls) EKHookMessage(...)` |
| Hook в произвольный момент | — не поддерживается | `EKHookMessage` в любой точке |
| Raw memory patch | — не поддерживается | `EKHookMemory` |
| Boilerplate | Минимум | Больше кода |

## 8. Когда что выбирать

**Logos**:

- Простые случаи, 90% твиков.
- Удобный синтаксис, читаемость.
- Хочется автоматических `_logos_orig$...` переменных.

**ellekit direct**:

- Условные хуки в рантайме, когда условие сложное.
- Проект без Theos (своя build system).
- Нужны редкие ellekit-возможности (raw memory, run-time deconflict).
- Хотите видеть, что реально происходит под капотом.

Для нашей целевой среды — обычно Logos. Этот кейс — демонстрация, что доступен и прямой путь.

## 9. Инспекция

Проверить, что ellekit линкуется:

```bash
otool -L .theos/obj/EKDirect.dylib | grep ellekit
# @rpath/libellekit.dylib (compatibility version ...)
```

Если нет — проверьте `_LIBRARIES = ellekit` в Makefile.

## 10. Что проверить на устройстве

После установки — в `log stream`:

```bash
log stream --predicate 'subsystem == "com.example.ekdirect"' --device <udid>
```

Запустите TestbedApp:

```
hook installed, orig IMP = 0x...
viewDidLoad in ViewController
access(/some/path, 0x4) = 0
```

Если нет — проверьте, что ellekit установлен:

```bash
ssh mobile@<ip>
dpkg -l | grep ellekit
```

## 11. Полный Tweak.m

Соберём всё в один файл:

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import <unistd.h>
#import <os/log.h>
#import <ellekit.h>

static os_log_t gLog;

// ObjC swizzle
static void (*orig_viewDidLoad)(UIViewController *self, SEL _cmd);
static void my_viewDidLoad(UIViewController *self, SEL _cmd) {
    os_log(gLog, "viewDidLoad in %{public}@", NSStringFromClass(self.class));
    orig_viewDidLoad(self, _cmd);
}

// C-function hook
static int (*orig_access)(const char *path, int mode);
static int my_access(const char *path, int mode) {
    int ret = orig_access(path, mode);
    os_log(gLog, "access(%{public}s, 0x%x) = %d", path, mode, ret);
    return ret;
}

__attribute__((constructor))
static void initializer(void) {
    gLog = os_log_create("com.example.ekdirect", "init");
    os_log(gLog, "EKDirect loaded in %{public}@",
           [NSProcessInfo processInfo].processName);

    // ObjC swizzle
    EKHookMessage([UIViewController class],
                  @selector(viewDidLoad),
                  (IMP)my_viewDidLoad,
                  (IMP *)&orig_viewDidLoad);

    // C-function hook
    EKHookFunction((void *)access, (void *)my_access, (void **)&orig_access);
}
```

Это полноценный ellekit-direct твик без единой `%hook`-директивы.

## 12. Что подтверждено

- **Подтверждено**: ellekit — пакет на RootHide Dopamine, предоставляет API для хуков.
- **Подтверждено практикой**: `_LIBRARIES = root ellekit` в Makefile даёт `otool -L` показывающую зависимость.
- **Требует проверки**: точный API в вашей версии ellekit — может варьироваться между версиями.
- **Не подтверждено**: что `EKHookMemory` рекомендован для production — обычно использовать с осторожностью.

## 13. Выводы

Главный тезис: **ellekit — живая, доступная библиотека; работать с ней напрямую — так же легко, как через Logos**. Выбор между подходами — вопрос стиля и специфики задачи.

Для 95% твиков Logos удобнее. Для 5% (условная логика, custom build system) — ellekit напрямую.

## 14. См. также

- Кейс 13 — твик без Logos, только C/ObjC + ellekit.
- Глава IV.5 — перехват C-функций через `%hookf`.
- Шпаргалка 1 — синтаксис Logos.
