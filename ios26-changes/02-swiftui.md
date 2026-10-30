# SwiftUI в iOS 26: что изменилось

SwiftUI — framework от 2019 года, который к iOS 26 — зрелый. Для автора твика это значит: границы перехвата лучше понимать с самого начала.

## 1. Главное: SwiftUI сложно хукать

Это **основной** факт этой главы.

SwiftUI использует:

- **Static dispatch** — Swift функции без `@objc dynamic` компилируются в прямые вызовы. Fishhook не поможет.
- **Generics** — body View инстанцируются в compile-time. Нет единой точки перехвата.
- **Opaque return types** (`some View`) — тип тела известен только компилятору.
- **Property wrappers** (`@State`, `@Binding`, `@Observable`) — тесно связаны с runtime от Apple.

Результат: традиционный `%hook SwiftUIView` **не работает**.

## 2. Что можно хукать

### 2.1. ObjC-мостовые классы

Внутренне SwiftUI использует несколько Objective-C классов:

- `UIHostingController` — моста между UIKit и SwiftUI.
- Классы в `SwiftUIX`/`SwiftUIService` — private, iOS-internal.

`UIHostingController` — hookable:

```objc
%hook UIHostingController

- (void)viewDidLoad {
    os_log(gLog, "UIHostingController viewDidLoad");
    %orig;
}

- (void)viewWillAppear:(BOOL)animated {
    os_log(gLog, "UIHostingController viewWillAppear");
    %orig(animated);
}

%end
```

Это ловит момент, когда SwiftUI view монтируется в UIKit. Дальнейшее — внутри SwiftUI, недоступно для хука.

### 2.2. ObservableObject и @Observable

Приложение использует `ObservableObject` (iOS 13-17) или `@Observable` macro (iOS 17+). Эти **реализованы через Objective-C** в основе.

Если приложение имеет Model, объявленный как `class Model: ObservableObject` — Model — Swift class, но с `@objc` при inheriting from NSObject. Хукается:

```objc
@interface MyAppModel : NSObject
- (id)objectForKey:(NSString *)key;
@end

%hook MyAppModel
- (id)objectForKey:(NSString *)key {
    id result = %orig(key);
    os_log(gLog, "Model[%{public}@] = %{public}@", key, result);
    return result;
}
%end
```

**Условия**: класс должен быть `@objc` и объявлен через NSObject-наследование, или с явным `@objc dynamic` на методах.

### 2.3. @MainActor isolated методы

Swift 6 / iOS 17+ ввели `@MainActor` для thread isolation. Это не меняет хукаемость — метод всё ещё ObjC-dispatchable, если `@objc`.

## 3. Что НЕЛЬЗЯ хукать

| Что | Почему |
|---|---|
| `View.body` | Opaque return type, generic context |
| `View.func myModifier() -> some View` | То же |
| Private Swift internals (`SwiftUI._FrameLayoutModifier` и т. п.) | Не экспортированы, static dispatch |
| `@State` / `@Binding` property wrappers | Внутренняя магия Apple |

Это не ограничение твика — это дизайн SwiftUI. Для перехвата нужен другой подход.

## 4. Обходной путь — хук **моста**

Многие SwiftUI apps сохраняют data в:

- `UserDefaults` / `NSUserDefaults`.
- `URLSession` для сетевых запросов.
- `CoreData` / `SwiftData` для persistence.
- `FileManager` для файлов.

Все эти — ObjC или C API. Хукаются обычным способом. Для вашего задача:

- Хотите данные приложения — хукните `NSUserDefaults` или `URLSession`.
- Хотите UI изменения — хукните `UIHostingController` + модифицируйте layer / UIView-level.

Это **косвенный** подход, но работает надёжно.

## 5. iOS 26 изменения в SwiftUI (по открытым источникам)

Трэнд:

- **Больше macros** (`@Observable`, `#Preview`, `@Entry` возможно).
- **Interactive widgets** продолжают развиваться.
- **AppIntents** интеграция.
- **Live Activities** в SwiftUI.
- **Metal rendering** оптимизации (внутренние).

Для автора твика ни один из этих changes не open'ит новые hook-точки. SwiftUI остаётся «чёрным ящиком» для hook-based подхода.

## 6. Пример: твик для своего SwiftUI приложения

Задача: ваше приложение — SwiftUI; хотите хукать переход на конкретный экран.

### Подход A: ObjC Model

Если ваш code имеет `ObservableObject` Model в основе — хук её методов:

```swift
// Swift (ваше приложение):
class AppModel: ObservableObject {
    @Published var isAuthenticated: Bool = false

    @objc dynamic func navigateTo(_ screen: String) {
        // ...
    }
}
```

```objc
// Твик:
%hook AppModel

- (void)navigateTo:(NSString *)screen {
    os_log(gLog, "navigateTo: %{public}@", screen);
    %orig(screen);
}

%end
```

**Условие**: метод `@objc dynamic`. Без этого Swift делает static dispatch, хук не ловит.

### Подход B: UIHostingController

Если доступа к Model нет — хукните UIHostingController и работайте с его `.view`:

```objc
%hook UIHostingController

- (void)viewDidAppear:(BOOL)animated {
    %orig(animated);
    os_log(gLog, "SwiftUI screen appeared");

    // Добавить toast через наш %new метод (кейс 17):
    [self performSelector:@selector(showToast:) withObject:@"SwiftUI screen"];
}

%end
```

## 7. Таблица «SwiftUI vs UIKit» для хукинга

| Задача | SwiftUI | UIKit |
|---|---|---|
| Перехват появления экрана | `%hook UIHostingController.-viewDidAppear:` | `%hook UIViewController.-viewDidAppear:` |
| Перехват метода бизнес-логики | нужно `@objc dynamic` | обычный `%hook` |
| Изменение внешнего вида view | только через UIKit-level (layer) | свободно |
| Перехват navigation | `UINavigationController` работает | то же |
| Перехват data flow | через ObjC-мостовые Model | обычный `%hook` |

## 8. Что подтверждено

- **Подтверждено Apple docs**: SwiftUI как framework; `@objc dynamic` для interop.
- **Подтверждено практикой**: SwiftUI static dispatch не поддаётся fishhook/ellekit.
- **Подтверждено**: `UIHostingController` — обычный `UIViewController`, хукается нормально.
- **Требует проверки на устройстве**: конкретные internal selectors Apple SwiftUI runtime (если понадобятся) — зависят от iOS 26.

## 9. Выводы

Главный тезис: **SwiftUI — чёрный ящик для tweak-инструментов**. Хукайте границы (UIHostingController, ObjC Model, UserDefaults, URLSession) — не внутренности.

Для автора твика на SwiftUI-based target — больше работы по identification правильных мостовых точек. Но архитектура такова, что это **единственный** реалистичный путь.

## 10. См. также

- iOS 26 changes 1 — UIKit.
- Глава II.6 — UIKit, SwiftUI, Scene.
- Кейс 09 — Swift interop с ObjC.
- Глава I.3 — границы перехвата (ObjC/C/Swift).
