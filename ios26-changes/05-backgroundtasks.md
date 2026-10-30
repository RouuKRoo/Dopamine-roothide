# BackgroundTasks и жизненный цикл в iOS 26

Как iOS 26 позволяет приложению работать в фоне, и что из этого полезно для автора твика.

## 1. Текущий (iOS 13+) API — BGTaskScheduler

Стандартная поверхность для background work:

- `BGTaskScheduler` — главный класс.
- `BGAppRefreshTask` — короткая фоновая задача (refresh).
- `BGProcessingTask` — долгая фоновая задача (DB maintenance, large file ops).

iOS 26 — продолжает этот API. Можно ожидать новые task-types (например, `BGContinuedProcessingTaskRequest` — проверяйте Apple docs).

## 2. Для автора твика — три сценария

### 2.1. Логировать background активность в своём приложении

```objc
%hook BGTaskScheduler

- (BOOL)submitTaskRequest:(BGTaskRequest *)taskRequest error:(NSError **)error {
    os_log(gLog, "BGTaskScheduler submit: id=%{public}@ earliest=%{public}@",
           taskRequest.identifier,
           taskRequest.earliestBeginDate);
    return %orig(taskRequest, error);
}

- (void)cancelTaskRequestWithIdentifier:(NSString *)identifier {
    os_log(gLog, "BGTaskScheduler cancel: %{public}@", identifier);
    %orig(identifier);
}

- (void)cancelAllTaskRequests {
    os_log(gLog, "BGTaskScheduler cancel all");
    %orig;
}

%end

%hook BGTask

- (void)setExpirationHandler:(void (^)(void))expirationHandler {
    os_log(gLog, "BGTask setExpirationHandler for %{public}@", self.identifier);
    %orig(expirationHandler);
}

- (void)setTaskCompletedWithSuccess:(BOOL)success {
    os_log(gLog, "BGTask complete: %{public}@ success=%d", self.identifier, (int)success);
    %orig(success);
}

%end
```

Это даёт полную картину — что приложение submit'ит, когда task запускается, когда complete'ится.

### 2.2. Продлить фоновую работу

Публичное API: `UIApplication beginBackgroundTaskWithExpirationHandler:` — даёт короткое окно продолжения (обычно ~30 сек).

Твик может хукнуть этот метод, чтобы логировать:

```objc
%hook UIApplication

- (UIBackgroundTaskIdentifier)beginBackgroundTaskWithExpirationHandler:(void (^)(void))handler {
    UIBackgroundTaskIdentifier id = %orig(handler);
    os_log(gLog, "beginBackgroundTask → %lu", (unsigned long)id);
    return id;
}

- (void)endBackgroundTask:(UIBackgroundTaskIdentifier)identifier {
    os_log(gLog, "endBackgroundTask %lu", (unsigned long)identifier);
    %orig(identifier);
}

%end
```

**Нельзя** продлить окно насильно — iOS сама убивает процесс по watchdog. Можно только логировать.

### 2.3. Хук в runningboardd — продление жизни

Для **системного** продления (не внутри приложения, а на уровне всей iOS):

- Хук в `runningboardd` — опасно, может сломать жизненный цикл.
- Dopamine сам хукает runningboardd для своих целей (`BaseBin/roothidehooks/src/runningboardd.x`).

Автор твика **не должен** хукать runningboardd. Используйте public API — там всё, что доступно легально.

## 3. iOS 26 изменения (по открытым источникам)

Ожидаемые направления (**не** конкретика):

- **AI-related tasks** — background inference, maybe новый `BGMLTaskRequest`.
- **Energy budgets** — более жёсткие limits.
- **Continued processing** — для long-running (минуты, не секунды).

Для автора твика это не меняет fundamental подход:

- Для своего приложения — public API + hooks для логирования.
- Для чужого — нельзя продлить насильно; можно только observe.

## 4. UIApplicationDelegate методы

Жизненный цикл приложения отражён в делегат-методах:

| Метод | Когда вызывается |
|---|---|
| `-application:didFinishLaunchingWithOptions:` | Запуск |
| `-applicationDidEnterBackground:` | Переход в фон |
| `-applicationWillEnterForeground:` | Из фона на передний |
| `-applicationDidBecomeActive:` | Активный |
| `-applicationWillResignActive:` | Теряет активность |
| `-applicationWillTerminate:` | iOS killing (нечасто) |

Все хукаются стандартным Logos.

На iOS 13+ часть этого делегата переехала в `UIScene` / `UIWindowScene` (`-sceneDidEnterBackground:` etc.). Приложения, обновлённые на `UIScene` lifecycle, не используют старые методы. Хукайте оба — для совместимости.

```objc
%hook AppDelegate

- (void)applicationDidEnterBackground:(UIApplication *)application {
    os_log(gLog, "AppDelegate: didEnterBackground");
    %orig(application);
}

%end

%hook SceneDelegate   // или ваш actual class для scene

- (void)sceneDidEnterBackground:(UIScene *)scene {
    os_log(gLog, "SceneDelegate: sceneDidEnterBackground");
    %orig(scene);
}

%end
```

**Внимание**: `AppDelegate` и `SceneDelegate` — классы вашего приложения, не UIKit. Logos нуждается в декларации (`@interface AppDelegate: UIResponder <...>`).

## 5. Опасность: хуки на жизненный цикл в системных процессах

Хук `-applicationDidEnterBackground:` в SpringBoard — ломает систему. SpringBoard не «уходит в фон», он host для всего остального.

Правило: хуки `UIApplicationDelegate` **только** в твике для конкретного приложения (filter plist на bundle id), **никогда** на SpringBoard.

## 6. BGTaskScheduler Debug (на устройстве)

Для диагностики — Apple предоставляет private API, который даёт force-run background task:

```bash
# В Xcode debugger:
e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"com.example.task"]
```

На jailbroken device через LLDB (Advanced 8) — то же самое. Твик может хукнуть `_simulateLaunchForTaskWithIdentifier:` для автоматического run (но это debug-only path).

## 7. Что подтверждено

- **Подтверждено Apple docs**: `BGTaskScheduler`, `BGTaskRequest`, `UIApplication` lifecycle.
- **Подтверждено Dopamine**: `runningboardd` хуки — только Dopamine внутри (`BaseBin/roothidehooks/src/runningboardd.x`).
- **Требует проверки Apple docs**: конкретные новые task-types в iOS 26 — проверить release notes.
- **Не подтверждено**: что принципиально новое AI-task API появилось в 26 — пока это предположения.

## 8. Выводы

Главный тезис: **BGTaskScheduler + UIApplication lifecycle — стабильный публичный API**. Для автора твика — хукается стандартным Logos.

Для «продлить фоновую работу приложения» — публичного пути нет. Хуки на `runningboardd` опасны и уже заняты Dopamine. Довольствуйтесь наблюдением, не вмешивайтесь.

## 9. См. также

- Глава II.5 — переключатель приложений и жизненный цикл.
- Глава III.9 — runningboardd хуки Dopamine.
- Атлас BackgroundTasks — framework-atlas (если будет написан).
