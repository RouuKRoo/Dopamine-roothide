# Атлас: UserNotifications

Public framework для push и local notifications. Стабилен с iOS 10.

## 1. Главные классы

### 1.1. UNUserNotificationCenter

Singleton:

```objc
UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
```

Методы:

- `-requestAuthorizationWithOptions:completionHandler:` — запрос разрешения.
- `-getNotificationSettingsWithCompletionHandler:` — текущие settings.
- `-addNotificationRequest:withCompletionHandler:` — добавить notification.
- `-getPendingNotificationRequestsWithCompletionHandler:` — список pending.
- `-removePendingNotificationRequestsWithIdentifiers:` — удалить pending.
- `-removeAllDeliveredNotifications` — очистить delivered.

### 1.2. UNNotificationRequest

```objc
UNMutableNotificationContent *content = [UNMutableNotificationContent new];
content.title = @"Hello";
content.body = @"World";
content.sound = UNNotificationSound.defaultSound;

UNTimeIntervalNotificationTrigger *trigger =
    [UNTimeIntervalNotificationTrigger triggerWithTimeInterval:60 repeats:NO];

UNNotificationRequest *request =
    [UNNotificationRequest requestWithIdentifier:@"myId"
                                           content:content
                                           trigger:trigger];
```

### 1.3. UNNotificationServiceExtension

Для перехвата входящих push с `mutable-content: 1`:

```objc
@interface MyNotificationService : UNNotificationServiceExtension
@end

@implementation MyNotificationService

- (void)didReceiveNotificationRequest:(UNNotificationRequest *)request
                     withContentHandler:(void (^)(UNNotificationContent *))contentHandler
{
    UNMutableNotificationContent *bestAttemptContent = [request.content mutableCopy];
    // Модифицировать:
    bestAttemptContent.body = [bestAttemptContent.body stringByAppendingString:@" [modified]"];
    contentHandler(bestAttemptContent);
}

@end
```

## 2. Хуки

Рецепты из cookbook:

- `UNUserNotificationCenter -addNotificationRequest:` — что приложение планирует.
- `UNUserNotificationCenterDelegate -willPresentNotification:` — что показывается.
- `UNUserNotificationCenterDelegate -didReceiveNotificationResponse:` — user action.

## 3. Что НЕ хукать

- `UNNotificationServiceExtension` чужого приложения из вашего app — invalid process boundary.
- Private BB* classes (BulletinBoard) — нестабильно.

## 4. Push vs Local

| Push (APNS) | Local |
|---|---|
| Прилетает от Apple | Приложение планирует |
| Требует push token | Нет |
| apsd + usernotificationsd | UNUserNotificationCenter напрямую |
| Internet connection нужен | Нет |

Для автора твика — API идентичен.

## 5. Live Activities (iOS 16+)

```objc
#import <ActivityKit/ActivityKit.h>

// Public API iOS 16+:
// (ActivityKit в Swift обычно)
```

На iPhone 11 (A13) — работает, но без Dynamic Island (нет cutout). Отображается только на Lock Screen.

## 6. iOS 26 заметки

- UNUserNotificationCenter — стабилен.
- Interactive notifications (buttons, text input) — продолжают работать.
- AI-related features (summarize) — на A13 ограниченно (нет Neural Engine performance).

## 7. См. также

- Глава II.3 — Notifications.
- iOS 26 changes 6 — Notifications на 26.
- Daemon: apsd.
