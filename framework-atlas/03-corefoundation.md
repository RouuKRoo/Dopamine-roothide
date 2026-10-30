# Атлас: CoreFoundation

C-уровневый фундамент под Foundation. Все `CF*` типы.

## 1. Toll-free bridging

Многие CF-типы «мостятся» с ObjC-эквивалентом:

| CoreFoundation | Foundation |
|---|---|
| `CFStringRef` | `NSString *` |
| `CFArrayRef` | `NSArray *` |
| `CFMutableArrayRef` | `NSMutableArray *` |
| `CFDictionaryRef` | `NSDictionary *` |
| `CFDataRef` | `NSData *` |
| `CFURLRef` | `NSURL *` |
| `CFNumberRef` | `NSNumber *` |
| `CFDateRef` | `NSDate *` |

Можно свободно cast'ить с помощью `__bridge`.

## 2. Функции для хука

### 2.1. CFPreferences

Главная подсистема настроек:

```c
CFPreferencesCopyAppValue(key, applicationID);
CFPreferencesSetAppValue(key, value, applicationID);
CFPreferencesAppSynchronize(applicationID);
CFPreferencesGetAppBooleanValue(key, applicationID, &exists);
CFPreferencesGetAppIntegerValue(key, applicationID, &exists);
```

Хукаются через fishhook / ellekit (`%hookf`):

```objc
%hookf(CFPropertyListRef, CFPreferencesCopyAppValue, CFStringRef key, CFStringRef applicationID) {
    CFPropertyListRef v = %orig(key, applicationID);
    NSLog(@"CFPreferencesCopyAppValue(%@, %@) → %@",
          (__bridge NSString *)key,
          (__bridge NSString *)applicationID,
          (__bridge id)v);
    return v;
}
```

### 2.2. CFBundle

```c
CFBundleRef CFBundleGetMainBundle(void);
CFBundleRef CFBundleCreate(allocator, bundleURL);
CFURLRef CFBundleCopyResourceURL(bundle, name, type, subdir);
```

Используется под капотом `NSBundle`.

### 2.3. CFNotificationCenter

```c
CFNotificationCenterRef CFNotificationCenterGetDarwinNotifyCenter(void);
void CFNotificationCenterAddObserver(...);
void CFNotificationCenterPostNotification(...);
```

Darwin notifications (глава IV.8).

### 2.4. CFRunLoop

```c
CFRunLoopRef CFRunLoopGetCurrent(void);
void CFRunLoopRun(void);
void CFRunLoopPerformBlock(...);
```

Хукать осторожно — hot path.

## 3. Memory management

CoreFoundation использует **manual retain/release**:

```c
CFStringRef s = CFStringCreateWithCString(NULL, "hi", kCFStringEncodingUTF8);
// use s
CFRelease(s);
```

Правило:

- `Create*`/`Copy*` → владеете, `CFRelease` обязателен.
- `Get*` → не владеете, не release'те.

Утечки через забытый CFRelease — одна из самых частых ошибок в CF-коде.

## 4. iOS 26 заметки

- CoreFoundation стабилен десятилетиями.
- Minor изменения на 26 — нет breaking.
- Все перечисленные функции работают.

## 5. См. также

- Глава II.7 — Settings/Preferences.
- Глава IV.8 — настройки через CFPreferences.
- Шпаргалка 5 — libroot API.
