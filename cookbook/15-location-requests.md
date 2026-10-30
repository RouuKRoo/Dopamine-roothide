# Рецепт 15. Логирование запросов локации в своём приложении

**Цель**: увидеть все запросы `CLLocationManager` в тестовом приложении — authorization, start/stop, delegate callbacks.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = LocationTrace
LocationTrace_FILES = Tweak.x
LocationTrace_CFLAGS = -fobjc-arc
LocationTrace_FRAMEWORKS = CoreLocation Foundation
LocationTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## LocationTrace.plist

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
Package: com.example.locationtrace
Name: LocationTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace CoreLocation requests in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## Tweak.x

```objc
#import <CoreLocation/CoreLocation.h>
#import <Foundation/Foundation.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.locationtrace", "loc");
    os_log(gLog, "LocationTrace loaded");
}

%hook CLLocationManager

// Authorization
- (void)requestWhenInUseAuthorization {
    os_log(gLog, "[%p] requestWhenInUseAuthorization", self);
    %orig;
}

- (void)requestAlwaysAuthorization {
    os_log(gLog, "[%p] requestAlwaysAuthorization", self);
    %orig;
}

// Location updates
- (void)startUpdatingLocation {
    os_log(gLog, "[%p] startUpdatingLocation (desiredAccuracy=%.1f)",
           self, self.desiredAccuracy);
    %orig;
}

- (void)stopUpdatingLocation {
    os_log(gLog, "[%p] stopUpdatingLocation", self);
    %orig;
}

- (void)requestLocation {
    os_log(gLog, "[%p] requestLocation (one-shot)", self);
    %orig;
}

// Heading
- (void)startUpdatingHeading {
    os_log(gLog, "[%p] startUpdatingHeading", self);
    %orig;
}

- (void)stopUpdatingHeading {
    os_log(gLog, "[%p] stopUpdatingHeading", self);
    %orig;
}

// Region monitoring
- (void)startMonitoringForRegion:(CLRegion *)region {
    os_log(gLog, "[%p] startMonitoringForRegion: %{public}@", self, region);
    %orig(region);
}

- (void)stopMonitoringForRegion:(CLRegion *)region {
    os_log(gLog, "[%p] stopMonitoringForRegion: %{public}@", self, region);
    %orig(region);
}

// Significant location changes
- (void)startMonitoringSignificantLocationChanges {
    os_log(gLog, "[%p] startMonitoringSignificantLocationChanges", self);
    %orig;
}

- (void)stopMonitoringSignificantLocationChanges {
    os_log(gLog, "[%p] stopMonitoringSignificantLocationChanges", self);
    %orig;
}

// Visit monitoring
- (void)startMonitoringVisits {
    os_log(gLog, "[%p] startMonitoringVisits", self);
    %orig;
}

- (void)stopMonitoringVisits {
    os_log(gLog, "[%p] stopMonitoringVisits", self);
    %orig;
}

// Setters — важные параметры
- (void)setDesiredAccuracy:(CLLocationAccuracy)accuracy {
    os_log(gLog, "[%p] setDesiredAccuracy: %.1f", self, accuracy);
    %orig(accuracy);
}

- (void)setDistanceFilter:(CLLocationDistance)distance {
    os_log(gLog, "[%p] setDistanceFilter: %.1f", self, distance);
    %orig(distance);
}

- (void)setAllowsBackgroundLocationUpdates:(BOOL)allows {
    os_log(gLog, "[%p] setAllowsBackgroundLocationUpdates: %d", self, (int)allows);
    %orig(allows);
}

%end
```

Разбор:

- `self` в format → `[%p]` — для различения разных экземпляров CLLocationManager, если приложение имеет несколько.
- Все главные методы: authorization, location, heading, region, visit, significant changes.
- Setters для `desiredAccuracy`, `distanceFilter`, `allowsBackgroundLocationUpdates` — полезно знать настройки.

## Расширение: delegate callbacks

Чтобы увидеть, когда location **прилетает**:

```objc
// Хукаем сам протокол — для этого ищем делегат через associated:
%hook CLLocationManager
- (void)setDelegate:(id<CLLocationManagerDelegate>)delegate {
    os_log(gLog, "[%p] setDelegate: %{public}@", self, delegate);
    %orig(delegate);
    // Опционально — хукнуть конкретные методы делегата через isa-swizzling
    // (или добавить в delegate методы, если он — ваш)
}
%end
```

Для универсального перехвата delegate'а потребовался бы method_setImplementation на конкретном классе. Для своего приложения проще — добавить логирование прямо в делегат.

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

log stream --predicate 'subsystem == "com.example.locationtrace"' --device <udid>
```

Триггер — в TestbedApp любое действие с CLLocationManager. В логе:

```
[0x11a023100] requestWhenInUseAuthorization
[0x11a023100] setDesiredAccuracy: 10.0
[0x11a023100] setDistanceFilter: 100.0
[0x11a023100] startUpdatingLocation (desiredAccuracy=10.0)
```

## Этика

Для собственного приложения — valid. Для чужого — tracking location без разрешения = criminal.

## См. также

- Глава II.11 — power/battery/thermal (CoreLocation влияет на batteryusage).
- Атлас CoreLocation+CoreMotion (framework-atlas, если будет написан).
- Рецепт 8 — LAContext (аналогичный подход к hooking authorization).
