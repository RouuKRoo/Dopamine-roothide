# Атлас: BackgroundTasks

iOS 13+ framework для фоновой работы приложения. Public, стабильный.

## 1. Классы

### 1.1. BGTaskScheduler

Singleton:

```objc
BGTaskScheduler *scheduler = BGTaskScheduler.sharedScheduler;
```

Методы:

- `-submitTaskRequest:error:` — запросить фоновую task.
- `-cancelTaskRequestWithIdentifier:` — отменить.
- `-cancelAllTaskRequests` — отменить все.
- `-registerForTaskWithIdentifier:usingQueue:launchHandler:` — зарегистрировать handler.
- `-getPendingTaskRequestsWithCompletionHandler:` — список pending.

### 1.2. Request types

| Класс | Назначение | Timeout |
|---|---|---|
| `BGAppRefreshTaskRequest` | Короткий refresh (данные для UI) | ~30 сек |
| `BGProcessingTaskRequest` | Длительная обработка (DB, large files) | до минут, но ограничено |
| `BGHealthResearchTaskRequest` | HealthKit (iOS 16+) | специфично |

### 1.3. Task

Внутри handler:

```objc
[scheduler registerForTaskWithIdentifier:@"com.example.refresh" usingQueue:nil launchHandler:^(BGTask *task) {
    task.expirationHandler = ^{
        // iOS сообщает — время вышло
        [task setTaskCompletedWithSuccess:NO];
    };

    // Делать работу...
    [task setTaskCompletedWithSuccess:YES];
}];
```

## 2. Для автора твика

Хуки — для диагностики (рецепт — глава iOS 26 changes 5):

```objc
%hook BGTaskScheduler

- (BOOL)submitTaskRequest:(BGTaskRequest *)request error:(NSError **)error {
    os_log(gLog, "submitTaskRequest: id=%{public}@ earliest=%{public}@",
           request.identifier, request.earliestBeginDate);
    return %orig(request, error);
}

%end
```

## 3. Что НЕ хукать

- `runningboardd` — критичен для lifecycle (Dopamine уже имеет хук).
- Internal `_BGManager*` — нестабильно.

## 4. Debug

Через LLDB (Advanced 8):

```
e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"com.example.refresh"]
```

Это запускает task на устройстве без ждать iOS schedule. Private API, но доступен в debug контексте.

## 5. iOS 26 заметки

- `BGContinuedProcessingTaskRequest` (если появился на 26) — для долгой работы.
- AI-related: `BGMLTaskRequest` — спекулятивно.
- Public API стабилен с iOS 13.

## 6. См. также

- Глава II.5 — жизненный цикл процессов.
- iOS 26 changes 5 — BackgroundTasks на 26.
- Daemon: runningboardd.
