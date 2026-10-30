# Shortcuts и Intents в iOS 26

Shortcuts (приложение) + App Intents (framework) — путь, через который приложения экспортируют свои действия в Siri, Spotlight, Widgets, StandBy, и прочее. Для автора твика — важно понимать, что это **публично** и работает **без** хуков.

## 1. Что было до iOS 16

На iOS 10–15 приложения использовали `IntentsUI.framework` + `.intentdefinition` файлы. Это Objective-C API. Для автора — хукается стандартно.

## 2. Что в iOS 16+: App Intents

С iOS 16 Apple ввёл **App Intents** — Swift-first API. Это тот же самый цель (экспорт действий), но:

- Чистый Swift.
- Декларативный (структуры, не методы).
- Автоматически интегрируется в Shortcuts, Siri, Spotlight, Focus.

Для нашего целевого iOS 26 — App Intents — главный API.

### 2.1. Как выглядит

В Swift-приложении:

```swift
import AppIntents

struct OpenMyScreenIntent: AppIntent {
    static var title: LocalizedStringResource = "Open My Screen"
    static var description = IntentDescription("Opens the main screen")

    func perform() async throws -> some IntentResult {
        // Открыть экран
        return .result()
    }
}
```

iOS автоматически:

- Показывает это в Shortcuts.app.
- Доступно через Siri.
- Можно добавить в Widget / Home Screen.

## 3. Для автора твика

### 3.1. Экспортировать свой action

Если ваш твик имеет companion app — добавьте App Intent через public Swift. Не требует tweak-инфраструктуры.

### 3.2. Хукнуть App Intent выполнение

Класс `AppIntent` — Swift struct, но iOS runtime оборачивает в ObjC-мостовой класс (`AppIntentsExtension` или подобное). Хукать сложно (Swift struct ≠ ObjC class).

**Подход**: хукнуть промежуточный ObjC класс, если он известен. Или хукнуть ваш own helper, который вызывается из intent'а.

Для 99% задач — проще не хукать intents, а делать свою логику напрямую.

### 3.3. Логирование shortcut запусков

Shortcut запускает ваше приложение через `openURL` → `-application:continueUserActivity:`:

```objc
%hook AppDelegate   // или ваш class

- (BOOL)application:(UIApplication *)application
   continueUserActivity:(NSUserActivity *)userActivity
     restorationHandler:(void (^)(NSArray *))restorationHandler
{
    os_log(gLog, "continueUserActivity: type=%{public}@ title=%{public}@",
           userActivity.activityType,
           userActivity.title);
    return %orig(application, userActivity, restorationHandler);
}

%end
```

Это универсальный путь: любое Shortcut-event приходит через этот метод.

## 4. Что заявляет Apple для iOS 26

Направления:

- **Больше AppIntent types** — для разных контекстов (Widgets, StandBy, CarPlay).
- **AI-related intents** (Apple Intelligence).
- **Automation improvements** в Shortcuts.app.

Для автора твика — public API, стабильно за счёт того, что Apple сам инвестирует в stability.

## 5. Siri integration

Siri использует App Intents автоматически. Для хука:

- `-handleIntent:` в приложении — но это обёртка.
- Через Shortcuts path — лучше.

## 6. Что подтверждено

- **Подтверждено Apple docs**: App Intents framework (iOS 16+).
- **Подтверждено Apple docs**: `NSUserActivity`, `-continueUserActivity:` — стабильный путь.
- **Требует проверки**: конкретные новые intent types на iOS 26.

## 7. Выводы

Главный тезис: **Shortcuts + App Intents — public, стабильный, "безопасный"**. Для автора твика — наблюдение через `-continueUserActivity:` в своём приложении. Полный контроль над expresion через App Intents.

Для хука чужого App Intents — Swift makes it hard; обычно не стоит.

## 8. См. также

- Глава II.5 — жизненный цикл приложений.
- Атлас Foundation / UIKit (framework-atlas).
- iOS 26 changes 1 — UIKit.
