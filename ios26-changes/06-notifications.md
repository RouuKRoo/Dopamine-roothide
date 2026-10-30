# Уведомления в iOS 26

Notifications — one of the heaviest subsystems iOS (SpringBoard + apsd + usernotificationsd + BB (BulletinBoard) + сами apps). Для автора твика важно знать безопасные и опасные зоны.

## 1. Три уровня notifications в iOS

| Уровень | API | Процессы |
|---|---|---|
| **Public API** (приложения) | `UNUserNotificationCenter`, `UNNotificationRequest` | Собственное приложение |
| **Delivery infrastructure** | APNS (Apple Push), silent push | `apsd`, `usernotificationsd` |
| **UI presentation** | SpringBoard banner, Notification Center UI | SpringBoard, BB*** private classes |

Для автора твика:

- Уровень 1 — хукается стандартным Logos.
- Уровень 2 — **нельзя**. Private, системное, часто сломается.
- Уровень 3 — через SpringBoard хуки, нестабильно между minor версиями.

## 2. UNUserNotificationCenter API (стабильный)

Главная public точка. На iOS 26 стабилен.

```objc
%hook UNUserNotificationCenter

// Запрос разрешения
- (void)requestAuthorizationWithOptions:(UNAuthorizationOptions)options
                        completionHandler:(void (^)(BOOL, NSError *))completionHandler
{
    os_log(gLog, "requestAuthorization: options=0x%lx", (unsigned long)options);
    %orig(options, completionHandler);
}

// Добавление notification
- (void)addNotificationRequest:(UNNotificationRequest *)request
            withCompletionHandler:(void (^)(NSError *))completionHandler
{
    os_log(gLog, "addNotificationRequest: id=%{public}@ title=%{public}@",
           request.identifier, request.content.title);
    %orig(request, completionHandler);
}

// Удаление
- (void)removePendingNotificationRequestsWithIdentifiers:(NSArray<NSString *> *)identifiers {
    os_log(gLog, "removePendingNotificationRequests: %{public}@", identifiers);
    %orig(identifiers);
}

- (void)removeAllDeliveredNotifications {
    os_log(gLog, "removeAllDeliveredNotifications");
    %orig;
}

%end
```

Для **собственного** приложения — полная картина, что делает ваше приложение с уведомлениями.

## 3. UNUserNotificationCenterDelegate методы

Приложение имеет delegate, который получает callbacks. Для хука нужно знать класс делегата:

```objc
@interface MyNotificationDelegate : NSObject <UNUserNotificationCenterDelegate>
@end

%hook MyNotificationDelegate

- (void)userNotificationCenter:(UNUserNotificationCenter *)center
         willPresentNotification:(UNNotification *)notification
            withCompletionHandler:(void (^)(UNNotificationPresentationOptions))completionHandler
{
    os_log(gLog, "willPresent: id=%{public}@ title=%{public}@",
           notification.request.identifier,
           notification.request.content.title);
    %orig(center, notification, completionHandler);
}

- (void)userNotificationCenter:(UNUserNotificationCenter *)center
         didReceiveNotificationResponse:(UNNotificationResponse *)response
                withCompletionHandler:(void (^)(void))completionHandler
{
    os_log(gLog, "didReceiveResponse: id=%{public}@ action=%{public}@",
           response.notification.request.identifier,
           response.actionIdentifier);
    %orig(center, response, completionHandler);
}

%end
```

**Условие**: класс делегата должен быть известен — обычно это `AppDelegate` или специальный `NotificationDelegate` в вашем приложении.

## 4. Notification Service Extension

Для перехвата **входящих** push до их отображения — ваше приложение может иметь `UNNotificationServiceExtension`. Это `.appex`:

- Отдельный процесс.
- Запускается iOS при получении push с `mutable-content: 1`.
- Может модифицировать content (image, title, body) до display.

Автор твика может хукнуть `NotificationService` расширение чужого приложения — это **отдельный** process, filter plist на bundle id `.appex`.

Это полезно для:

- Логирования приходящих push.
- Модификации (для своего приложения).

## 5. Что нельзя или опасно

### 5.1. Хук apsd / usernotificationsd

```make
INSTALL_TARGET_PROCESSES = apsd
```

```xml
<key>Filter</key><dict><key>Executables</key><array>
    <string>apsd</string>
</array></dict>
```

**Опасно**. Эти процессы делают огромную работу — все push на устройстве. Крэш = потеря push для всех приложений до перезагрузки.

Dopamine сам в apsd/usernotificationsd не вмешивается (глава II.14).

### 5.2. Хук SpringBoard для изменения внешнего вида banner

Теоретически возможно (private classes `BBB*`, `SBF*`). Практически — нестабильно между minor версиями. Если очень нужно — probe (глава V.2) и готовьте план отката (SpringBoard crash = respring loop).

### 5.3. Подавление чужих уведомлений

- Через хук apsd — опасно (см. выше).
- Через хук в SpringBoard — нестабильно.
- Через подмену своего собственного delegate — работает только для своего app.

Для **своего** app — просто используйте `UNUserNotificationCenter removePendingNotificationRequestsWithIdentifiers:`.

## 6. iOS 26 изменения (по открытым источникам)

Направления:

- **Live Activities** продолжают эволюционировать — notification как progress indicator.
- **Interactive notifications** — больше actions в banner.
- **AI-related** — summarize notifications (Apple Intelligence).
- **Scheduled notifications** — scheduled push.

Для автора твика — public API на 26 стабилен. Private classes — probe обязателен.

## 7. Таблица «что хукать»

| Задача | Способ | Безопасность |
|---|---|---|
| Логировать push в своём app | `UNUserNotificationCenter.-addNotificationRequest:` | Безопасно |
| Логировать push callback в своём app | `UNUserNotificationCenterDelegate` методы | Безопасно (нужен класс делегата) |
| Модифицировать incoming push в своём app | `UNNotificationServiceExtension` + hook | Безопасно (своя .appex) |
| Подавить все push своего app | `removePendingNotificationRequests...` или не позволять `addNotificationRequest:` | Безопасно |
| Логировать push чужого app | hook в `apsd` | **Опасно** (не делайте) |
| Изменить вид banner на SpringBoard | hook `BBB*`/`SBF*` private | Нестабильно, probe обязателен |
| Подавить все push на устройстве | — | **Нельзя** без мощных hooks в системе |

## 8. Live Activities

iOS 16+ дал Live Activities (push, который держится в Dynamic Island / Lock Screen). На iPhone 11 (A13) **нет** Dynamic Island (нет cutout), но Live Activity всё равно показывается на Lock Screen.

API: `ActivityKit` в вашем приложении. Для автора твика:

- Hook `ActivityManager` — public, работает в своём app.
- Hook SpringBoard-side rendering — private, нестабильно.

## 9. Что подтверждено

- **Подтверждено Apple docs**: `UNUserNotificationCenter`, `UNNotificationRequest`, `UNUserNotificationCenterDelegate`.
- **Подтверждено Dopamine**: `apsd`/`usernotificationsd` не хукаются (глава II.14).
- **Требует проверки на устройстве**: конкретные `BB*`/`SBF*` классы для UI banner на iOS 26.
- **Не подтверждено**: все изменения в Apple Intelligence summarize feature для notifications на 26.

## 10. Выводы

Главный тезис: **public UserNotifications API — стабильно, хукается легко. SpringBoard-level — нестабильно. apsd-level — табу**.

Для своего приложения — полный контроль над push через public API. Для чужого — только observation через `UNNotificationServiceExtension` или UI-level хуки (с рисками).

## 11. См. также

- Глава II.3 — Notifications.
- Глава II.1 — SpringBoard.
- Атлас UserNotifications — framework-atlas (если будет).
