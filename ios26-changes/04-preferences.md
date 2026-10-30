# Settings.app в iOS 26

Settings.app — отдельное iOS приложение, его bundle id `com.apple.Preferences`. Для автора твика важно по трём причинам: регистрация PreferenceBundle, хуки UI настроек, cfprefsd-perенаправление.

## 1. Три точки интереса для автора твика

1. **Регистрация своего PreferenceBundle** — чтобы Settings.app показал вашу страницу.
2. **Хуки в UI Settings.app** — редкий случай, но возможный.
3. **Чтение/запись настроек через `CFPreferences`** — работает везде, не только в Settings.app.

Главы III.9 и IV.8 покрыли (2) и (3). Эта глава — (1) и общий контекст iOS 26.

## 2. Что заявляет Apple для Settings.app в iOS 26

По открытым источникам — направления развития:

- **Переорганизация** разделов (периодически происходит в major версиях).
- **Больше AI-related опций** (Siri, Image Playground, Writing Tools).
- **Privacy & Security** расширяется.
- **Apple Intelligence** controls.

Для **автора твика**:

- Структура Settings.app почти всегда стабильна в term'ах того, как добавлять свою страницу.
- PreferenceLoader (сторонний пакет) интегрируется в Settings.app через `/Library/PreferenceLoader/Preferences/*.plist` — на iOS 26 это ровно так же.

## 3. PreferenceLoader на iOS 26

PreferenceLoader — утилита, которая хукает Settings.app и позволяет сторонним PreferenceBundles появляться в списке.

На iOS 26 работает тот же механизм:

- `.plist` в `/Library/PreferenceLoader/Preferences/<Tweak>.plist` описывает entry.
- `.bundle` в `/Library/PreferenceBundles/<Tweak>.bundle/` содержит controller.
- Settings.app → scroll вниз → ваш твик появляется.

Пакет на Dopamine: `preferenceloader`. Устанавливается из того же репозитория.

Если PreferenceLoader не установлен — ваш PreferenceBundle **не отобразится**. Это проверка #1 при диагностике «моя страница не появилась».

## 4. PreferenceBundle как чёрный ящик

С точки зрения Apple API, PreferenceBundle — это `.bundle` с `PSListController` subclass как `NSPrincipalClass`. На iOS 26 эти classes всё ещё в `Preferences.framework`:

```objc
#import <Preferences/PSListController.h>
#import <Preferences/PSSpecifier.h>
#import <Preferences/PSCell.h>

@interface MyPrefs : PSListController
@end

@implementation MyPrefs

- (NSArray *)specifiers {
    if (!_specifiers) {
        _specifiers = [self loadSpecifiersFromPlistName:@"Root" target:self];
    }
    return _specifiers;
}

@end
```

Этот код работает на iOS 15–26 **одинаково**. Это стабильная поверхность.

## 5. Cells, доступные на iOS 26

Стандартный набор PS*Cells:

| Cell | Назначение | Пример в Root.plist |
|---|---|---|
| `PSGroupCell` | Заголовок группы | `cell=PSGroupCell, label="Title"` |
| `PSSwitchCell` | Toggle | `cell=PSSwitchCell, key, defaults, default` |
| `PSSliderCell` | Slider | `cell=PSSliderCell, key, defaults, default, min, max` |
| `PSEditTextCell` | Text field | `cell=PSEditTextCell, key, defaults, default, placeholder` |
| `PSSegmentCell` | Segmented control | `cell=PSSegmentCell, key, defaults, validTitles, validValues` |
| `PSButtonCell` | Button | `cell=PSButtonCell, label, action` |
| `PSLinkCell` | Переход в subpage | `cell=PSLinkCell, label, detail` |
| `PSTitleValueCell` | Title + value readonly | `cell=PSTitleValueCell, label, default` |
| `PSStaticTextCell` | Static text | `cell=PSStaticTextCell, label` |

Все работают на iOS 26 (verify через simple test). Эта таблица — референс.

## 6. Dynamic specifiers

Если нужны динамические cells (например, список установленных твиков):

```objc
- (NSArray *)specifiers {
    NSMutableArray *result = [NSMutableArray array];

    // Group header
    PSSpecifier *group = [PSSpecifier groupSpecifierWithName:@"Tweaks"];
    [result addObject:group];

    // Один switch per установленный tweak:
    for (NSString *name in [self installedTweakNames]) {
        PSSpecifier *s = [PSSpecifier preferenceSpecifierNamed:name
                                                          target:self
                                                             set:@selector(setPreferenceValue:specifier:)
                                                             get:@selector(readPreferenceValue:)
                                                          detail:nil
                                                            cell:PSSwitchCell
                                                            edit:nil];
        [s setProperty:name forKey:@"key"];
        [s setProperty:@"com.example.manager" forKey:@"defaults"];
        [s setProperty:@YES forKey:@"default"];
        [result addObject:s];
    }

    return [result copy];
}
```

Это даёт flexible PreferenceBundle, не ограниченный static plist.

## 7. Хук Settings.app самого

Редкий случай — хотите вмешаться в поведение Settings.app:

```make
INSTALL_TARGET_PROCESSES = Preferences
```

```xml
<key>Filter</key>
<dict><key>Bundles</key><array>
    <string>com.apple.Preferences</string>
</array></dict>
```

Пример — скрыть определённый пункт из списка. Это требует хука в `PSSystemConfigurationDataSource` или подобное — private API, нестабильно между версиями.

Для автора твика в 99% — лучше **не** хукать Settings.app. Добавляйте свою страницу через PreferenceBundle, не меняйте чужие.

## 8. cfprefsd на iOS 26

Напомним (глава III.9): Dopamine хукает `cfprefsd` через `rootlesshooks/cfprefsd.x` для перенаправления не-apple preferences в jbroot.

На iOS 26 это **работает так же**. Для автора твика это невидимо — вы используете `CFPreferences*` API и получаете правильные пути автоматически.

## 9. Что подтверждено

- **Подтверждено исходниками**: cfprefsd хук в Dopamine (`BaseBin/rootlesshooks/cfprefsd.x`).
- **Подтверждено Apple docs**: Preferences.framework классы `PSListController`, `PSSpecifier`, PS*Cells.
- **Подтверждено практикой**: PreferenceLoader — community tool, работает на всех rootless+roothide установках.
- **Требует проверки на устройстве**: конкретный layout Settings.app на iOS 26 (разделы могут быть реорганизованы, но добавление вашей страницы работает всегда).

## 10. Миграция PreferenceBundle с 18 на 26

Если ваш PreferenceBundle работал на iOS 18:

1. Пересобрать с `THEOS_PACKAGE_SCHEME=roothide` (глава IV.9).
2. Проверить Root.plist — все cells валидны.
3. Проверить, что `preferenceloader` установлен на устройстве.
4. Запустить на iOS 26, открыть Settings.app → scroll → ваш твик.

Обычно работает без изменений.

## 11. Выводы

Главный тезис: **PreferenceBundle API — очень стабилен**. Код из iOS 15 работает на iOS 26 без изменений. Infrastructure (cfprefsd, PreferenceLoader) — тоже стабильна.

Для автора твика на iOS 26 — settings — самая "спокойная" часть системы.

## 12. См. также

- Глава II.7 — Settings и preferences.
- Глава III.9 — cfprefsd хуки Dopamine.
- Глава IV.8 — настройки твика (CFPreferences).
- Кейс 02 — полный tweak с PreferenceBundle.
- Рецепт 09 — PreferenceBundle с libroot.
