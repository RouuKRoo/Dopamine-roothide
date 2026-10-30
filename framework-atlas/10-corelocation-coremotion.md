# Атлас: CoreLocation и CoreMotion

Local-sensor APIs. Privacy-sensitive, высокая этическая ответственность.

## 1. CoreLocation

### 1.1. CLLocationManager

Главный класс:

```objc
CLLocationManager *manager = [CLLocationManager new];
manager.delegate = self;
manager.desiredAccuracy = kCLLocationAccuracyBest;
manager.distanceFilter = 10;

[manager requestWhenInUseAuthorization];
[manager startUpdatingLocation];
```

Авторизация:

- `-requestWhenInUseAuthorization` — только когда приложение активно.
- `-requestAlwaysAuthorization` — всегда (требует justification).

### 1.2. Delegate methods

```objc
- (void)locationManager:(CLLocationManager *)manager
         didUpdateLocations:(NSArray<CLLocation *> *)locations;

- (void)locationManager:(CLLocationManager *)manager
             didFailWithError:(NSError *)error;

- (void)locationManager:(CLLocationManager *)manager
    didChangeAuthorizationStatus:(CLAuthorizationStatus)status;
```

### 1.3. Хуки

Рецепт 15 cookbook — полный перечень 13 методов (authorization, updates, heading, region, visit, significant changes).

### 1.4. iOS 14+ fine-grained accuracy

- **Full** — precise coordinates.
- **Reduced** — ~5km accuracy (user privacy option).

Приложение запрашивает конкретный accuracy:

```objc
if (manager.accuracyAuthorization == CLAccuracyAuthorizationReducedAccuracy) {
    [manager requestTemporaryFullAccuracyAuthorizationWithPurposeKey:@"MyPurpose"
                                                               completion:^(NSError *err) { ... }];
}
```

## 2. CoreMotion

### 2.1. CMMotionManager

Акселерометр, gyroscope, magnetometer, device motion:

```objc
CMMotionManager *mgr = [CMMotionManager new];
if (mgr.isAccelerometerAvailable) {
    mgr.accelerometerUpdateInterval = 0.1;
    [mgr startAccelerometerUpdatesToQueue:[NSOperationQueue mainQueue]
                                 withHandler:^(CMAccelerometerData *data, NSError *err) {
        // use data.acceleration (x, y, z)
    }];
}
```

Классы:

- `CMAccelerometerData` — ускорение.
- `CMGyroData` — rotation rate.
- `CMMagnetometerData` — magnetic field.
- `CMDeviceMotion` — fused motion.

### 2.2. CMPedometer

Шагомер:

```objc
CMPedometer *pedo = [CMPedometer new];
[pedo startPedometerUpdatesFromDate:[NSDate date]
                           withHandler:^(CMPedometerData *data, NSError *err) {
    // data.numberOfSteps
}];
```

### 2.3. CMMotionActivityManager

Activity recognition (walking, running, driving):

```objc
CMMotionActivityManager *activity = [CMMotionActivityManager new];
[activity startActivityUpdatesToQueue:[NSOperationQueue mainQueue]
                                withHandler:^(CMMotionActivity *act) {
    if (act.walking) { ... }
    if (act.running) { ... }
    if (act.automotive) { ... }
}];
```

## 3. Хуки

### 3.1. Своё app

Хукать для:

- Логирование запросов (как много sampling).
- Подмены значений для тестов (fake GPS, fake motion).

```objc
%hook CLLocationManager

- (void)startUpdatingLocation {
    os_log(gLog, "startUpdatingLocation (accuracy=%.1f)", self.desiredAccuracy);
    %orig;
}

%end

%hook CMMotionManager

- (void)startAccelerometerUpdatesToQueue:(NSOperationQueue *)queue
                                 withHandler:(CMAccelerometerHandler)handler
{
    os_log(gLog, "startAccelerometerUpdates");
    %orig(queue, handler);
}

%end
```

### 3.2. Этика

Подмена **своей** location — valid test.

Подмена **чужой** location в чужом приложении:

- Технически возможно через hook delegate `-didUpdateLocations:`.
- Этически — нарушение privacy приложения.
- Юридически — часто обман servisов (uber, dating apps) = нарушение TOS.

Для **личного** использования (например, подмена location в своём test build) — OK.

## 4. iOS 26 заметки

- CoreLocation / CoreMotion API стабильны годами.
- iOS 14+ fine-grained accuracy — продолжает работать.
- Нет breaking changes на 26.

## 5. Battery impact

Напоминание (глава II.11):

- `kCLLocationAccuracyBest` + continuous updates → огромное drain.
- Для своего app — используйте минимально нужный accuracy.
- Для хука — не продлевайте уже-запущенные updates за пределы времени, когда они нужны.

## 6. См. также

- Рецепт 15 — location logging.
- iOS 26 changes 8 — Privacy.
- Глава II.11 — power/battery.

На этом **framework-atlas секция закрыта**. 10 глав — главные frameworks для автора твика.
