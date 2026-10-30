# Daemon: apsd (Apple Push Service)

Приёмник APNS push-уведомлений. **Не хукается** Dopamine.

## 1. Что это

- **Путь**: `/System/Library/PrivateFrameworks/ApplePushService.framework/apsd` (на старых iOS) или `/usr/sbin/apsd`
- **Executable name**: `apsd`
- **Запуск**: launchd
- **Привилегии**: root + extensive network entitlements

## 2. Что делает

- Поддерживает **постоянное TLS соединение** с Apple APNS gateway.
- Принимает входящие push-уведомления.
- Распределяет push в целевые приложения через `usernotificationsd`.
- Хранит push tokens.
- Обеспечивает delivery acknowledgments.
- Обрабатывает silent push (с `content-available: 1`).

Это один из критических демонов. iOS в идле держит apsd живым для получения push.

## 3. Framework

- `ApplePushService.framework` — public (ограниченно).
- `APSService.framework` — private.
- Классы: `APSConnection`, `APSMessage`.

## 4. Хуки Dopamine

**НЕТ** (глава II.14).

apsd имеет:

- Жёсткие entitlements для APNS traffic.
- Permanent TLS to Apple.
- Критическая роль для системы.

Хук = риск разорвать push для всех приложений. Dopamine отказывается.

## 5. Что хукать можно

### 5.1. Public API в приложении

```objc
%hook UIApplicationDelegate

- (void)application:(UIApplication *)application
    didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken
{
    os_log(gLog, "didRegisterForRemoteNotifications with token: %{public}@",
           [deviceToken description]);
    %orig(application, deviceToken);
}

- (void)application:(UIApplication *)application
    didReceiveRemoteNotification:(NSDictionary *)userInfo
             fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler
{
    os_log(gLog, "didReceiveRemoteNotification: %{public}@", userInfo);
    %orig(application, userInfo, completionHandler);
}

%end
```

Это **в приложении**, не в apsd. Хукается standard-way.

### 5.2. Notification Service Extension

Для перехвата push до display — ваш собственный `UNNotificationServiceExtension` (глава iOS 26 changes 6). Запускается при push с `mutable-content: 1`.

### 5.3. UserNotifications public API

```objc
%hook UNUserNotificationCenter

- (void)addNotificationRequest:(UNNotificationRequest *)request ... { }

%end
```

(Глава iOS 26 changes 6 для детальных рецептов.)

## 6. Что хукать НЕЛЬЗЯ

### 6.1. apsd сам

Невозможно — Dopamine в него не инжектируется (apsd — platform process, часто с restricted injection).

Даже если бы было возможно — не делайте.

### 6.2. `usernotificationsd`

Аналогично. Распределяет push в приложения. Хрупкий.

### 6.3. SpringBoard-side banner rendering

Возможно теоретически (BB*/SBF* классы), но нестабильно и этически — вы начинаете controlling push behavior для всех приложений.

## 7. Диагностика

```bash
ssh mobile@<ip>
log stream --process apsd --device <udid>

# Или только для вашего app:
log stream --predicate 'eventMessage CONTAINS "your-topic"' --device <udid>
```

APNS topic — обычно bundle id вашего приложения.

## 8. Push token management

Для автора твика в своём app:

```objc
%hook UIApplication

- (void)registerForRemoteNotifications {
    os_log(gLog, "registerForRemoteNotifications called");
    %orig;
}

%end
```

Это ловит момент регистрации. Token приходит в `didRegister...` delegate.

## 9. Что подтверждено

- **Подтверждено Dopamine**: apsd не хукается.
- **Подтверждено Apple docs**: Public UserNotifications + UIApplicationDelegate push methods.
- **Требует проверки**: конкретный путь apsd — варьируется между iOS versions.

## 10. Выводы

Главный тезис: **apsd — табу (так же, как mediaserverd, installd)**. Весь push management — в приложении через public API.

## 11. См. также

- Глава II.3 — Notifications.
- iOS 26 changes 6 — Notifications на 26.
- Атлас UserNotifications (framework-atlas).
