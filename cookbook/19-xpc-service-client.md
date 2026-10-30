# Рецепт 19. Клиент к XPC-сервису приложения

**Цель**: twик, который перехватывает `NSXPCConnection` и `xpc_connection_*` C-API в своём приложении — чтобы видеть, с какими сервисами и какими сообщениями.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = XPCTrace
XPCTrace_FILES = Tweak.x
XPCTrace_CFLAGS = -fobjc-arc
XPCTrace_FRAMEWORKS = Foundation
XPCTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## XPCTrace.plist

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
Package: com.example.xpctrace
Name: XPCTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace XPC connections in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## Tweak.x

```objc
#import <Foundation/Foundation.h>
#import <xpc/xpc.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.xpctrace", "xpc");
    os_log(gLog, "XPCTrace loaded");
}

// ============ Objective-C API: NSXPCConnection ============

%hook NSXPCConnection

- (instancetype)initWithServiceName:(NSString *)serviceName {
    os_log(gLog, "NSXPCConnection initWithServiceName: %{public}@", serviceName);
    return %orig(serviceName);
}

- (instancetype)initWithMachServiceName:(NSString *)name options:(NSXPCConnectionOptions)options {
    os_log(gLog, "NSXPCConnection initWithMachServiceName: %{public}@ options=0x%lx",
           name, (unsigned long)options);
    return %orig(name, options);
}

- (instancetype)initWithListenerEndpoint:(NSXPCListenerEndpoint *)endpoint {
    os_log(gLog, "NSXPCConnection initWithListenerEndpoint: %{public}@", endpoint);
    return %orig(endpoint);
}

- (void)resume {
    os_log(gLog, "NSXPCConnection resume: service=%{public}@",
           self.serviceName ?: @"<unnamed>");
    %orig;
}

- (void)invalidate {
    os_log(gLog, "NSXPCConnection invalidate: service=%{public}@",
           self.serviceName ?: @"<unnamed>");
    %orig;
}

- (id)remoteObjectProxy {
    os_log(gLog, "NSXPCConnection remoteObjectProxy requested for %{public}@",
           self.serviceName ?: @"<unnamed>");
    return %orig;
}

- (id)remoteObjectProxyWithErrorHandler:(void (^)(NSError *))handler {
    os_log(gLog, "NSXPCConnection remoteObjectProxyWithErrorHandler for %{public}@",
           self.serviceName ?: @"<unnamed>");
    return %orig(handler);
}

%end

// ============ C API: xpc_connection_* ============

%hookf(xpc_connection_t, xpc_connection_create, const char *name, dispatch_queue_t targetq) {
    xpc_connection_t result = %orig(name, targetq);
    os_log(gLog, "xpc_connection_create: name=%{public}s", name ?: "<null>");
    return result;
}

%hookf(xpc_connection_t, xpc_connection_create_mach_service, const char *name, dispatch_queue_t targetq, uint64_t flags) {
    xpc_connection_t result = %orig(name, targetq, flags);
    os_log(gLog, "xpc_connection_create_mach_service: name=%{public}s flags=0x%llx",
           name ?: "<null>", flags);
    return result;
}

%hookf(void, xpc_connection_send_message, xpc_connection_t conn, xpc_object_t msg) {
    @autoreleasepool {
        const char *name = xpc_connection_get_name(conn);
        char *desc = xpc_copy_description(msg);
        os_log(gLog, "xpc send to %{public}s: %{public}s",
               name ?: "<unnamed>", desc ?: "<null>");
        if (desc) free(desc);
    }
    %orig(conn, msg);
}

%hookf(xpc_object_t, xpc_connection_send_message_with_reply_sync, xpc_connection_t conn, xpc_object_t msg) {
    @autoreleasepool {
        const char *name = xpc_connection_get_name(conn);
        char *desc = xpc_copy_description(msg);
        os_log(gLog, "xpc send+reply to %{public}s: %{public}s",
               name ?: "<unnamed>", desc ?: "<null>");
        if (desc) free(desc);
    }

    xpc_object_t reply = %orig(conn, msg);

    @autoreleasepool {
        if (reply) {
            char *rdesc = xpc_copy_description(reply);
            os_log(gLog, "xpc reply: %{public}s", rdesc ?: "<null>");
            if (rdesc) free(rdesc);
        }
    }

    return reply;
}

%hookf(void, xpc_connection_resume, xpc_connection_t conn) {
    const char *name = xpc_connection_get_name(conn);
    os_log(gLog, "xpc resume: %{public}s", name ?: "<unnamed>");
    %orig(conn);
}
```

Разбор:

- **NSXPCConnection** — обёртка над C-API; удобна для type-safe вызовов.
- **C API** — используется ниже, в system frameworks и в некоторых приложениях напрямую.
- `xpc_copy_description(msg)` — возвращает `char *` с дампом xpc_object (dict/int/string/data). После использования — `free()`.
- `@autoreleasepool` — чтобы NSString, созданные для os_log, не висели.

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

log stream --predicate 'subsystem == "com.example.xpctrace"' --device <udid>
```

При запуске приложения — минимум одно `xpc_connection_create_mach_service` (на `com.apple.system.logger` или подобное) + несколько `send`.

Пример полезного вывода:

```
NSXPCConnection initWithServiceName: com.example.helper
NSXPCConnection resume: service=com.example.helper
xpc_connection_create_mach_service: name=com.apple.cfprefsd.agent flags=0
xpc send to com.apple.cfprefsd.agent: <dictionary>{ "key1" => "value1" }
```

## Расширение: фильтр по имени сервиса

```objc
static NSSet *gInteresting;

%ctor {
    gLog = os_log_create("...", "xpc");
    gInteresting = [NSSet setWithArray:@[@"com.example.helper",
                                          @"com.apple.cfprefsd.agent"]];
}

%hookf(void, xpc_connection_send_message, xpc_connection_t conn, xpc_object_t msg) {
    const char *cname = xpc_connection_get_name(conn);
    NSString *name = cname ? @(cname) : @"<unnamed>";
    if ([gInteresting containsObject:name]) {
        char *desc = xpc_copy_description(msg);
        os_log(gLog, "xpc send to %{public}@: %{public}s", name, desc);
        if (desc) free(desc);
    }
    %orig(conn, msg);
}
```

## Примечание: jbserver

Dopamine использует XPC внутри (глава III.4). Твик, инъектированный в процесс, может видеть вызовы `jbclient_*` — это нормальный трафик к `launchdhook` → jbserver. Не удивляйтесь, если лог содержит обращения к mach-сервисам с именем-подобным «com.apple.security...» или internal domains.

## См. также

- Глава II.14 — XPC и системные сервисы.
- Deep-dive внутренностей jbserver — глава III.4.
- Атлас Foundation (если будет написан).
