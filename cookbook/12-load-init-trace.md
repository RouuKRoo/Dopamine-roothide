# Рецепт 12. Трейс `+load` и `-init` методов

**Цель**: увидеть, какие классы `+load`-инициализируются и какие объекты `-init`-ализируются в тестовом приложении на старте.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = LoadInitTrace
LoadInitTrace_FILES = Tweak.x
LoadInitTrace_CFLAGS = -fobjc-arc
LoadInitTrace_FRAMEWORKS = Foundation
LoadInitTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## LoadInitTrace.plist

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
</dict>
```

## control

```
Package: com.example.loadinittrace
Name: LoadInitTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace +load and -init in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## Tweak.x

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import <os/log.h>

static os_log_t gLog;
static pthread_key_t gReentrancy;
static NSMutableSet<NSString *> *gLoggedInits;    // set, чтобы не спамить одним классом
static dispatch_queue_t gSetQueue;

%ctor {
    gLog = os_log_create("com.example.loadinittrace", "obj");
    pthread_key_create(&gReentrancy, NULL);
    gLoggedInits = [NSMutableSet set];
    gSetQueue = dispatch_queue_create("com.example.loadinittrace.set", DISPATCH_QUEUE_SERIAL);
    os_log(gLog, "LoadInitTrace loaded");
}

static inline bool enter(void) {
    if (pthread_getspecific(gReentrancy)) return false;
    pthread_setspecific(gReentrancy, (void *)1);
    return true;
}
static inline void leave(void) { pthread_setspecific(gReentrancy, NULL); }

// Хук на -init у NSObject — ловит все init экземпляров
%hook NSObject

- (instancetype)init {
    id result = %orig;
    if (enter()) {
        NSString *className = NSStringFromClass(object_getClass(result));
        __block BOOL isNew = NO;
        dispatch_sync(gSetQueue, ^{
            if (![gLoggedInits containsObject:className]) {
                [gLoggedInits addObject:className];
                isNew = YES;
            }
        });
        if (isNew) {
            os_log(gLog, "first -init: %{public}@", className);
        }
        leave();
    }
    return result;
}

%end
```

Разбор:

- `pthread_key_t gReentrancy` — защита от рекурсии: `os_log` + `NSStringFromClass` могут вызывать `-init` других объектов, что приведёт к stack overflow без guard.
- `gLoggedInits` — set «уже залогированных классов». Иначе будет тысячи записей одного и того же.
- `dispatch_sync(gSetQueue, ...)` — thread-safe доступ к set.
- Логируем **только первый** `-init` каждого класса — для компактности.

## Трейс `+load`

`+load` вызывается dyld'ом при загрузке образа, **до** `main()` и даже до `%ctor` ваших Logos. Логично его хукать невозможно (вы уже позже). Что можно — логировать классы с `+load` **статически**, через `otool`:

```bash
otool -ov /path/to/TestbedApp.app/TestbedApp | grep -B1 "+load" | head -30
```

Или через runtime inspection после загрузки:

```objc
%ctor {
    gLog = os_log_create("com.example.loadinittrace", "obj");
    unsigned int count = 0;
    Class *classes = objc_copyClassList(&count);
    for (unsigned int i = 0; i < count; i++) {
        Class cls = classes[i];
        Method load = class_getClassMethod(cls, @selector(load));
        if (load) {
            // Класс имеет +load — но мы не можем сказать, вызвался ли уже;
            // можем только зафиксировать наличие:
            Class metaCls = object_getClass(cls);
            if (class_getClassMethod(metaCls, @selector(load)) == load) {
                os_log(gLog, "class with +load: %{public}s", class_getName(cls));
            }
        }
    }
    free(classes);
}
```

Выдаст список классов, объявляющих `+load`.

## `+initialize` трейс

В отличие от `+load`, `+initialize` вызывается лениво — при первом обращении. Его можно хукнуть:

```objc
%hook NSObject

+ (void)initialize {
    os_log(gLog, "+initialize: %{public}@", NSStringFromClass(self));
    %orig;
}

%end
```

Это **широкий** хук — ловит `+initialize` всех классов в процессе. Выдаст список порядка «кто инициализировался в каком порядке».

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

log stream --predicate 'subsystem == "com.example.loadinittrace"' --device <udid>
```

Запустить TestbedApp. В логе — порядка 50-200 первых `-init` разных классов.

## См. также

- Глава IV.6 — `%orig`, рекурсия, разделяемое состояние.
- Глава IV.7 — потоки, ARC, lifecycle.
- Рецепт 11 — dlsym tracing.
