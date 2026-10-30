# Рецепт 6. Логирование Keychain доступа внутри своего приложения

**Цель**: логировать все `SecItemCopyMatching`/`SecItemAdd`/`SecItemUpdate`/`SecItemDelete` в собственном тестовом приложении — для диагностики безопасного хранения.

**Этика**: только для собственного приложения. Keychain чужого приложения — его private storage.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = KeychainTrace
KeychainTrace_FILES = Tweak.x
KeychainTrace_CFLAGS = -fobjc-arc
KeychainTrace_FRAMEWORKS = Security Foundation
KeychainTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## control

```
Package: com.example.keychaintrace
Name: KeychainTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace keychain access in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## KeychainTrace.plist

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
</dict>
```

## Tweak.x

```objc
#import <Foundation/Foundation.h>
#import <Security/Security.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.keychaintrace", "sec");
    os_log(gLog, "KeychainTrace loaded");
}

static NSString *redact_query(NSDictionary *query) {
    NSMutableDictionary *safe = [query mutableCopy];
    // Удаляем чувствительные поля перед логированием
    [safe removeObjectForKey:(__bridge id)kSecValueData];
    [safe removeObjectForKey:(__bridge id)kSecValueRef];
    [safe removeObjectForKey:(__bridge id)kSecAttrAccessGroup];
    return [safe description];
}

%hookf(OSStatus, SecItemCopyMatching, CFDictionaryRef query, CFTypeRef *result) {
    OSStatus r = %orig(query, result);
    @autoreleasepool {
        NSString *q = redact_query((__bridge NSDictionary *)query);
        os_log(gLog, "SecItemCopyMatching(query=%{public}@) → %d", q, (int)r);
    }
    return r;
}

%hookf(OSStatus, SecItemAdd, CFDictionaryRef attributes, CFTypeRef *result) {
    OSStatus r = %orig(attributes, result);
    @autoreleasepool {
        NSString *a = redact_query((__bridge NSDictionary *)attributes);
        os_log(gLog, "SecItemAdd(attrs=%{public}@) → %d", a, (int)r);
    }
    return r;
}

%hookf(OSStatus, SecItemUpdate, CFDictionaryRef query, CFDictionaryRef attributesToUpdate) {
    OSStatus r = %orig(query, attributesToUpdate);
    @autoreleasepool {
        NSString *q = redact_query((__bridge NSDictionary *)query);
        NSString *a = redact_query((__bridge NSDictionary *)attributesToUpdate);
        os_log(gLog, "SecItemUpdate(query=%{public}@, update=%{public}@) → %d", q, a, (int)r);
    }
    return r;
}

%hookf(OSStatus, SecItemDelete, CFDictionaryRef query) {
    OSStatus r = %orig(query);
    @autoreleasepool {
        NSString *q = redact_query((__bridge NSDictionary *)query);
        os_log(gLog, "SecItemDelete(query=%{public}@) → %d", q, (int)r);
    }
    return r;
}
```

Разбор:

- `redact_query` — удаляет чувствительные поля (`kSecValueData` — сам секрет; `kSecAttrAccessGroup` — group policy) перед логированием. Логируем **структуру** query, не содержимое.
- `OSStatus` — тип возврата Security API.
- `%hookf` — все 4 главные функции.
- `@autoreleasepool` — чтобы NSMutableDictionary не держать до next autorelease.

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

log stream --predicate 'subsystem == "com.example.keychaintrace"' --device <udid>
```

Триггер: в TestbedApp любое действие с keychain. В логе:

```
SecItemCopyMatching(query={kSecClass:kSecClassGenericPassword, ...}) → 0
SecItemAdd(attrs={kSecClass:kSecClassGenericPassword, kSecAttrAccount:myaccount, ...}) → 0
```

## Что НЕ логируем

Напомним: `redact_query` **удаляет** `kSecValueData` — там хранится сам секрет. Логирование секретов в system log = утечка. Если нужна диагностика конкретного значения — используйте **длину** и **хеш**, не само значение:

```objc
NSData *val = query[(__bridge id)kSecValueData];
if (val) {
    uint8_t hash[CC_SHA256_DIGEST_LENGTH];
    CC_SHA256(val.bytes, (CC_LONG)val.length, hash);
    os_log(gLog, "value: %lu bytes, sha256 prefix=%02x%02x%02x%02x",
           val.length, hash[0], hash[1], hash[2], hash[3]);
}
```

## Расширение: диагностика «почему вернулся ошибкой»

```objc
if (r != errSecSuccess) {
    const char *name = NULL;
    switch (r) {
        case errSecItemNotFound: name = "ItemNotFound"; break;
        case errSecDuplicateItem: name = "DuplicateItem"; break;
        case errSecAuthFailed:    name = "AuthFailed"; break;
        case errSecUserCanceled:  name = "UserCanceled"; break;
        case errSecInteractionNotAllowed: name = "InteractionNotAllowed"; break;
        default: name = "unknown";
    }
    os_log_error(gLog, "error %d (%s)", (int)r, name);
}
```

Расшифровка кодов ошибок — часто самое ценное.

## Этика

Это рецепт **для своего** приложения. Для чужого — нарушение privacy. Keychain содержит пароли, токены, биометрические шаблоны; логировать это = подвергать пользователя риску.

Если вы разработчик приложения и хотите увидеть, как keychain используется в проде — используйте этот рецепт в своём beta build на jailbroken test-device.

## См. также

- Глава II.14 — XPC и системные сервисы.
- Deep-dive 06 — codesign.
- Атлас фреймворков — Security (если будет написан).
