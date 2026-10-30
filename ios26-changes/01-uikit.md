# Изменения UIKit в iOS 26 с точки зрения автора твика

Эта глава — не пересказ Apple release notes (они обновляются и книга устаревает). Это — **методология** для определения, что изменилось в UIKit на iOS 26, и как это влияет на твики.

**Важная оговорка**: конкретные API изменения в UIKit iOS 26 проверяются против актуальной Apple documentation (`developer.apple.com/documentation/uikit`). Этот текст даёт framework, который работает в любой момент.

## 1. Что проверять

Минор iOS 26.0 → 26.0.1 обычно **не меняет** публичный UIKit API. Патч-релизы — это bug fixes. Проверять нужно:

1. **Deprecations** — какие методы помечены `API_DEPRECATED` в заголовках SDK.
2. **New APIs** — какие классы и методы добавлены.
3. **Private classes renaming** — какие SpringBoard/UIKit internal имена могли переименоваться.
4. **Behavioural changes** — что ведёт себя иначе при тех же вызовах.

## 2. Как увидеть deprecations

На хосте с Xcode:

```bash
grep -rn "API_DEPRECATED.*26\.0\|API_DEPRECATED.*19\.0" \
    /Applications/Xcode.app/Contents/.../iPhoneOS<ver>.sdk/System/Library/Frameworks/UIKit.framework/Headers/*.h
```

(Напомним — Apple internal number iOS 26 = «19». Grep по обоим стоит делать.)

Это выдаст список: селектор + сообщение «replaced by X» или «no replacement».

## 3. Как увидеть новые API

Diff headers между SDK 16.x и 18.x+:

```bash
# Два SDK установлены:
diff -r \
    /path/to/iPhoneOS16.5.sdk/.../UIKit.framework/Headers \
    /path/to/iPhoneOS18.0.sdk/.../UIKit.framework/Headers \
    | less
```

Новые заголовочные файлы, новые классы в существующих — всё видно.

## 4. Что исторически менялось в UIKit между major версиями

Чтобы ориентироваться — трендовые направления:

| Период | Что добавлялось |
|---|---|
| iOS 13 | SwiftUI (отдельный framework), UIScene, DarkMode |
| iOS 14 | WidgetKit, UIApplicationShortcutItem |
| iOS 15 | async/await в UIKit APIs, UIButton configuration, UISheetPresentationController |
| iOS 16 | UIEditMenuInteraction, UIAction menu improvements |
| iOS 17 | SF Symbols animations, Observation framework (Swift), SwiftData |
| iOS 18 | Interactive Live Activities, Control Center customization |
| iOS 26 | (новые features — проверяйте Apple docs) |

Для автора твика это даёт понимание: iOS 26 — очередной шаг эволюции. Фундаментальные классы (UIApplication, UIViewController, UIView, UIWindow) стабильны годами. Peripheral framework (SwiftUI, WidgetKit) — меняются быстрее.

## 5. Стабильные точки хука

На любой iOS, включая 26, эти селекторы стабильны:

| Класс.Метод | Статус |
|---|---|
| `UIApplication.-openURL:options:completionHandler:` | Стабилен с iOS 10 |
| `UIViewController.-viewDidLoad` | Стабилен с iOS 2 |
| `UIViewController.-viewWillAppear:` | Стабилен |
| `UIViewController.-viewDidAppear:` | Стабилен |
| `UIView.-drawRect:` | Стабилен |
| `UIView.-layoutSubviews` | Стабилен |
| `UIImageView.-setImage:` | Стабилен |
| `UITextField.-setSecureTextEntry:` | Стабилен с iOS 2 |
| `UITableViewDataSource.-tableView:cellForRowAtIndexPath:` | Стабилен |

На эти селекторы ваш твик может рассчитывать на iOS 26 так же, как на iOS 15.

## 6. Нестабильные точки

| Класс | Стабильность |
|---|---|
| `_UIStatusBar...` (private) | Переименовано в iOS 13 и ещё раз в iOS 17; на 26 — проверяйте |
| `_UINavigationBar...` (private, часть) | Нестабильно |
| `SBF*` SpringBoard (глава II.1) | Нестабильно между minor версиями |
| `_UIFloatingContentView` и подобные | Нестабильно |
| Classes with underscore prefix | Все private, стабильность не гарантирована |

Для таких — `NSClassFromString` + probe (глава V.2, кейс 05).

## 7. SwiftUI в UIKit приложениях

iOS 26 продолжает курс на SwiftUI integration. В UIKit-based приложении:

- `UIHostingController` для встраивания SwiftUI view.
- `UIViewRepresentable` для обратного.
- iOS 18/26 — `#Observable` macro для observation.

Это **не меняет** fundamental hook workflow — вы хукаете UIHostingController (который — обычный UIViewController).

## 8. Interactive Widgets (из iOS 17/18)

Виджеты получили интерактивность — `Button`/`Toggle` в WidgetKit. Для автора твика:

- Пользовательский widget — не tweak, а app extension (`.appex`).
- Можно хукать его bundle id = `app.bundleid.widgetkit-extension`.
- Код extension-а работает в отдельном процессе.

## 9. Live Activities

iOS 16+ дало Live Activities. На iOS 26 они продолжают работать через `ActivityKit`:

- Tweak hook в SpringBoard — для изменения внешнего вида Live Activity.
- Хук в app-extension (ActivityConfiguration provider) — для подмены контента.

Это нестабильные классы — требуют class-dump на iOS 26.

## 10. Что подтверждено

- **Подтверждено Apple docs** (как framework, актуальность — читатель проверяет сам): UIKit API — публичный, стабильный, задокументированный.
- **Подтверждено историей**: public selectors UIView/UIViewController/UIApplication стабильны годами.
- **Требует проверки на устройстве**: конкретные imena private классов на iOS 26.0/26.0.1.
- **Не подтверждено**: что в iOS 26 нет новых deprecations, которые ломают ваши хуки — проверяйте SDK headers.

## 11. Процедура адаптации твика к iOS 26

```
1. Собрать с SDK (latest).
2. Запустить на iOS 26 устройстве.
3. Проверить log — твик загружается?
4. Триггер сценария — хуки срабатывают?
5. Если нет — probe (глава V.2):
   - objc_copyClassList — есть ли класс?
   - class_copyMethodList — есть ли селектор?
6. Если класс/селектор переименован — обновить код через NSClassFromString.
7. Протестировать на 26.0 и 26.0.1 (глава V.5).
```

## 12. Выводы

Главный тезис: **UIKit public API очень стабилен**. 90% ваших хуков на iOS 26 будут работать без изменений, если вы использовали public selectors. 10% — private — требуют адаптации.

Для минимизации работы — хукайте максимально высокий public уровень. Чем ниже / privateer — тем чаще придётся адаптировать.

## 13. См. также

- Глава II.6 — UIKit, SwiftUI, Scene.
- Глава VI.2 — static vs device анализ.
- Глава V.2 — починка существующего твика.
- iOS 26 changes 2 — SwiftUI.
