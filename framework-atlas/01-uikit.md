# Атлас: UIKit — что хукать, что — нет

UIKit — главный UI framework iOS. Стабильный с iOS 2. На iOS 26 все public классы работают так же, как на 18.

## 1. Классы, которые часто хукают

| Класс | Главные методы | Риск |
|---|---|---|
| `UIApplication` | `openURL:`, `canOpenURL:`, `openURL:options:completionHandler:`, `didFinishLaunching...` | Низкий |
| `UIViewController` | `viewDidLoad`, `viewWillAppear:`, `viewDidAppear:`, `viewWillDisappear:`, `dealloc` | Низкий |
| `UIView` | `layoutSubviews`, `drawRect:`, `hitTest:withEvent:`, `init(frame:)` | Низкий-средний |
| `UIWindow` | `makeKeyAndVisible`, `didMoveToWindow` | Низкий |
| `UIImageView` | `setImage:`, `image` | Низкий |
| `UILabel` | `setText:`, `text` | Низкий |
| `UITextField` | `setSecureTextEntry:`, `setText:` | Низкий |
| `UITableView` | `reloadData`, delegate methods (через класс делегата) | Средний |
| `UICollectionView` | аналогично | Средний |
| `UINavigationController` | `pushViewController:animated:`, `popViewController...` | Низкий |
| `UITabBarController` | `setSelectedIndex:`, `selectedIndex` | Низкий |
| `UIAlertController` | `addAction:`, `presentViewController:...` | Низкий |

## 2. Что опасно хукать

| Что | Почему |
|---|---|
| `objc_msgSend` | Горячая точка, ломает всё |
| `UIView.-init` (напрямую) | Вызывается тысячи раз в секунду |
| Нестандартные приватные методы `_privateUI*` | Нестабильны |

## 3. UIApplication — главный entry

```objc
%hook UIApplication

- (BOOL)application:(UIApplication *)app didFinishLaunchingWithOptions:(NSDictionary *)opts {
    os_log(gLog, "didFinishLaunchingWithOptions: %{public}@", opts);
    return %orig(app, opts);
}

- (BOOL)canOpenURL:(NSURL *)url { ... }
- (void)openURL:(NSURL *)url options:(NSDictionary *)options completionHandler:(void (^)(BOOL))completion { ... }

%end
```

Самый common entry point.

## 4. UIViewController — главный UI

```objc
%hook UIViewController

- (void)viewDidLoad { %orig; /* ... */ }
- (void)viewWillAppear:(BOOL)animated { %orig(animated); /* ... */ }
- (void)viewDidAppear:(BOOL)animated { %orig(animated); /* ... */ }

%end
```

Эти методы — для 90% UI-хуков.

## 5. UIView — только через `drawRect:`

`-init` UIView вызывается часто (тысячи раз на экран). **Не хукайте** `-init`. Используйте `drawRect:` или `layoutSubviews`:

```objc
%hook UIView

- (void)drawRect:(CGRect)rect {
    %orig(rect);
    // Логика после отрисовки
}

%end
```

## 6. Делегат-методы

Делегат — обычно класс вашего приложения (`AppDelegate`, `SceneDelegate`, custom `UITextFieldDelegate`, etc.). Для хука нужно знать класс:

```objc
@interface MyTextFieldDelegate : NSObject <UITextFieldDelegate>
@end

%hook MyTextFieldDelegate

- (BOOL)textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string {
    os_log(gLog, "shouldChange: %{public}@", string);
    return %orig(textField, range, string);
}

%end
```

Или fallback — хук на `NSObject` + `respondsToSelector:` check (глава V.2).

## 7. Scenes (iOS 13+)

На iOS 13+ часть ответственности переехала из `UIApplication` в `UIWindowScene`:

```objc
%hook UIWindowScene

- (void)setActivationState:(UISceneActivationState)state {
    os_log(gLog, "scene state → %ld", (long)state);
    %orig(state);
}

%end
```

iOS 26 продолжает this trend. Хукайте и Application-level, и Scene-level для полного покрытия.

## 8. Таблица «iOS 26 заметки»

| Класс | Статус на iOS 26 |
|---|---|
| `UIApplication` | Стабилен |
| `UIViewController` | Стабилен |
| `UIView` | Стабилен |
| `UIAlertController` | Стабилен (но visual может быть updated) |
| Новые iOS 17+ классы (`UIEditMenuInteraction`, etc.) | Проверять availability |
| iOS 26-specific — доступно через `@available(iOS 26.0, *)` в Swift / `__builtin_available(iOS 26.0, *)` в ObjC | Fresh |

## 9. Общая рекомендация

Для **90%** задач — хукайте UIKit public selectors. Они стабильны, безопасны, работают на всех iOS 15–26 без изменений.

## 10. См. также

- Глава IV.4 — первый ObjC hook.
- Глава II.6 — UIKit и Scene.
- iOS 26 changes 1 — UIKit на 26.
