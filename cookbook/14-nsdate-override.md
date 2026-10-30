# Рецепт 14. Подмена `NSDate` для теста автономно от системного времени

**Цель**: заставить приложение «видеть» заданное время (например, «1 Jan 2027 00:00 UTC»), независимо от реального часов устройства. Полезно для unit-testing приложений, зависящих от текущего времени (expiry, scheduling).

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = FakeDate
FakeDate_FILES = Tweak.x
FakeDate_CFLAGS = -fobjc-arc
FakeDate_FRAMEWORKS = Foundation
FakeDate_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## FakeDate.plist

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
Package: com.example.fakedate
Name: FakeDate
Version: 0.0.1
Architecture: iphoneos-arm
Description: Force a specific date in TestbedApp for testing
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## Tweak.x

```objc
#import <Foundation/Foundation.h>
#import <os/log.h>
#import <time.h>
#import <sys/time.h>

static os_log_t gLog;

// Целевое время — 1 Jan 2027 00:00:00 UTC
// Unix timestamp: 1798761600
static const NSTimeInterval kFakeEpoch = 1798761600.0;

// Alternative: offset от реального времени (сдвиг)
// static const NSTimeInterval kFakeOffset = 365.0 * 24 * 60 * 60;  // +1 год

%ctor {
    gLog = os_log_create("com.example.fakedate", "date");
    NSDate *fake = [NSDate dateWithTimeIntervalSince1970:kFakeEpoch];
    os_log(gLog, "FakeDate loaded, pretending it is %{public}@", fake);
}

// 1. NSDate +now и +date
%hook NSDate

+ (NSDate *)now {
    return [NSDate dateWithTimeIntervalSince1970:kFakeEpoch];
}

+ (instancetype)date {
    return [self dateWithTimeIntervalSince1970:kFakeEpoch];
}

// Подменить timeIntervalSince1970 у текущего date — опасно, т. к. метод у всех NSDate
// Не делаем — хукаем только +now/+date.

%end

// 2. CFAbsoluteTime (CoreFoundation)
%hookf(CFAbsoluteTime, CFAbsoluteTimeGetCurrent) {
    // CFAbsoluteTime — секунды с 1 Jan 2001
    const NSTimeInterval kCFAbsoluteTimeIntervalSince1970 = 978307200.0;
    return kFakeEpoch - kCFAbsoluteTimeIntervalSince1970;
}

// 3. gettimeofday (POSIX)
%hookf(int, gettimeofday, struct timeval *tv, void *tz) {
    int r = %orig(tv, tz);
    if (r == 0 && tv) {
        tv->tv_sec = (time_t)kFakeEpoch;
        tv->tv_usec = 0;
    }
    return r;
}

// 4. time (POSIX)
%hookf(time_t, time, time_t *out) {
    time_t t = (time_t)kFakeEpoch;
    if (out) *out = t;
    return t;
}
```

Разбор:

- **`NSDate +now` и `+date`** — главные источники времени для Objective-C кода.
- **`CFAbsoluteTimeGetCurrent`** — для CoreFoundation API; `CFAbsoluteTime` считает от 1 Jan 2001, нужно вычесть константу `kCFAbsoluteTimeIntervalSince1970`.
- **`gettimeofday`** — POSIX, часто используется внутри libSystem.
- **`time`** — то же.

Этого набора обычно достаточно, чтобы приложение «поверило» в фальшивое время.

## Что НЕ хукаем

- **`clock_gettime`** — для `CLOCK_MONOTONIC`/`CLOCK_UPTIME_RAW` (не зависит от wall clock). Эти часы — для измерения интервалов; если их тоже подменить, break timers.
- **`mach_absolute_time`** — то же, monotonic.

Правило: подменяем **wall clock** (реальная дата), не **monotonic clock** (используется для таймеров).

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"
```

Запустить TestbedApp. Любой вызов `[NSDate now]` вернёт 1 Jan 2027. Проверка в приложении:

```swift
print(Date())
// 2027-01-01 00:00:00 +0000
```

## Расширение: offset вместо абсолютного значения

Если нужен сдвиг (+365 дней от реального сейчас, а не фиксированная дата):

```objc
static const NSTimeInterval kOffset = 365.0 * 24 * 60 * 60;

+ (NSDate *)now {
    NSTimeInterval realNow = %orig.timeIntervalSince1970;
    return [NSDate dateWithTimeIntervalSince1970:realNow + kOffset];
}

%hookf(int, gettimeofday, struct timeval *tv, void *tz) {
    int r = %orig(tv, tz);
    if (r == 0 && tv) {
        tv->tv_sec += (time_t)kOffset;
    }
    return r;
}
```

## Расширение: Toggle через настройки

```objc
static BOOL gEnabled;

static void reloadPrefs(void) {
    Boolean exists = false;
    Boolean b = CFPreferencesGetAppBooleanValue(CFSTR("enabled"),
                                                 CFSTR("com.example.fakedate"),
                                                 &exists);
    gEnabled = exists ? b : NO;
}

%hook NSDate
+ (NSDate *)now {
    if (!gEnabled) return %orig;
    return [NSDate dateWithTimeIntervalSince1970:kFakeEpoch];
}
%end
```

Плюс PreferenceBundle с switch (рецепт 9).

## См. также

- Рецепт 9 — PreferenceBundle.
- Глава IV.5 — перехват C-функций.
- Атлас Foundation — глава II.6.
