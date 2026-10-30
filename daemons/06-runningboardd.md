# Daemon: runningboardd

Жизненный цикл процессов — assertions, memory limits, state transitions.

## 1. Что это

- **Путь**: `/usr/libexec/runningboardd`
- **Executable name**: `runningboardd`
- **Запуск**: launchd (iOS 13+)
- **Привилегии**: root + system entitlements

## 2. Что делает

- Хранит state каждого запущенного процесса (`RBProcess` объект).
- Выдаёт **assertions** — обещания «этот процесс нужен live».
- Применяет memory limits через jetsam.
- Принимает решения о термальных ограничениях.
- Отслеживает foreground/background transitions.
- Координирует с SpringBoard при app switching.

## 3. Framework

- `RunningBoardServices.framework` — public interface для клиентов.
- `RunningBoard.framework` — внутренний.
- Класс: `RBProcess`, `RBSAssertion`.

## 4. Хуки Dopamine

Из `BaseBin/roothidehooks/src/runningboardd.x` (глава III.9):

- `fixAllowedPaths` — переписывает список `allowed locked file paths` для jailbreak-приложений, конвертируя rootfs-paths в jbroot-paths.
- Комментарий: «Fixed the RUNNINGBOARD 3735883980 issue».

Это **существующий** хук. Если ваш твик хочет вмешиваться в runningboardd — вы ломаете его.

## 5. Assertions API — публично

Приложение может получить assertion:

```swift
import BackgroundTasks

let token = ProcessInfo.processInfo.beginActivity(
    options: .userInitiated,
    reason: "my long operation")
// ... долгая работа ...
ProcessInfo.processInfo.endActivity(token)
```

Это public, работает везде включая iOS 26.

Для автора твика — хук `-beginActivity:reason:` в `NSProcessInfo` для диагностики активностей своего приложения.

## 6. Что хукать безопасно

```objc
%hook NSProcessInfo

- (id)beginActivityWithOptions:(NSActivityOptions)options reason:(NSString *)reason {
    id token = %orig(options, reason);
    os_log(gLog, "beginActivity: options=0x%lx reason=%{public}@",
           (unsigned long)options, reason);
    return token;
}

- (void)endActivity:(id)activity {
    os_log(gLog, "endActivity: %{public}@", activity);
    %orig(activity);
}

%end
```

Для своего приложения — полезно знать, что активно.

## 7. Что хукать опасно

- **`RBProcess`** методы — внутренние, нестабильные.
- **`RBSAssertion`** — то же.
- **runningboardd** XPC интерфейс напрямую — хрупко.

Dopamine сам ограничился единственным хуком (fixAllowedPaths); больше не нужно.

## 8. jetsam и память

runningboardd — главный клиент jetsam. При нехватке памяти:

1. jetsam отправляет warning приложениям.
2. Приложения получают `-didReceiveMemoryWarning:`.
3. runningboardd может kill процессы с низким приоритетом.

Для автора твика — чтобы не быть kill'нутым:

- Не держать в памяти большие caches без purge-on-warning.
- Отвечать на memory warnings реально (освобождать).
- Избегать `beginActivity` без `endActivity` (видимость runningboardd).

## 9. Диагностика

```bash
ssh mobile@<ip>
log stream --process runningboardd --device <udid>
```

Увидите state transitions, assertions, jetsam decisions.

Активные assertions:

```bash
ssh mobile@<ip>
# Через assertion-related CLI (может не быть на устройстве):
# runningboardctl status (not standard)
# Через log stream — лучше
```

## 10. Что подтверждено

- **Подтверждено исходниками**: `BaseBin/roothidehooks/src/runningboardd.x:7-45` — fixAllowedPaths.
- **Подтверждено Apple docs**: NSProcessInfo activity API.
- **Требует проверки на устройстве**: конкретные internal классы.

## 11. Выводы

Главный тезис: **runningboardd — критический demon, трогается только для observation через public NSProcessInfo**. Dopamine сам имеет один хук; больше — лишнее.

## 12. См. также

- Глава II.5 — переключатель приложений и жизненный цикл.
- Глава III.9 — rootlesshooks/roothidehooks.
- iOS 26 changes 5 — BackgroundTasks.
