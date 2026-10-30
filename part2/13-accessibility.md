# Глава II.13. Доступность

Доступность — зона с особыми техническими и этическими правилами. Технически она даёт в руки инструменты, которые превращают приложение из «чёрного ящика» в описываемую через API карту элементов интерфейса — именно то, что соблазняет авторов «следилок». Этически — это подсистема, созданная для людей, у которых альтернатив нет. В этой главе мы разделим: что сам Dopamine в accessibility делает (почти ничего), какая публичная поверхность доступна автору твика, и где начинается серая зона.

## 1. Что это за подсистема

iOS 26 продолжает модель, зафиксированную ещё в 2010-х, с рядом добавлений последних лет:

- **AX Server** — семейство процессов `assistantd`, `axassetsd`, `axavservicesd`, `AXUIAutomation` (имена могут слегка варьироваться между минорными версиями; на iOS 26.0/26.0.1 проверяется `launchctl list | grep -i ax` на устройстве).
- **VoiceOver** — главный клиент AX; собственный процесс (`SpringBoard` остаётся контейнером UI, но распознавание и озвучивание — отдельные процессы и/или потоки).
- **Switch Control, Voice Control** — альтернативные способы ввода, также клиенты AX.
- **AssistiveTouch** — overlay-кнопка от `SpringBoard`.
- **Audio graph для VoiceOver** — отдельный путь, см. также II.10.

Публичный SDK для автора приложения — модуль `UIAccessibility`:

- Свойства/протокол `UIAccessibilityElement`, `isAccessibilityElement`, `accessibilityLabel`, `accessibilityHint`, `accessibilityTraits`, `accessibilityIdentifier`.
- Нотификации: `UIAccessibilityIsVoiceOverRunning`, `UIAccessibilityVoiceOverStatusDidChangeNotification`, `UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, ...)`.
- Custom actions, custom rotors (с iOS 10+), и в iOS 26 — расширенные API для действий (`UIAccessibilityCustomAction` → `UIAccessibilityAction`, точные API подтверждаются по release notes 26).

Private сторона — класс `AXUIElement` (ветка AppKit на macOS; на iOS в приложениях юзерспейса не экспортируется публично), и внутренние AX-протоколы, живущие в `AXRuntime.framework`, `AXCommon.framework`, `AccessibilityUtilities.framework`.

## 2. Что в accessibility делает Dopamine

Проверка простая:

```bash
grep -rln -iE "accessibility|voiceover|AXRuntime|UIAccess" \
    BaseBin/ Application/Dopamine/Jailbreak/ Application/Dopamine/UI/ 2>/dev/null
```

Результат — upstream Dopamine не хукает accessibility-подсистему. Отдельный grep по `UIAccess`:

- в `Application/Dopamine/UI/` могут быть упоминания `UIAccessibility` в контексте UI приложения-установщика (чтобы его собственный интерфейс был доступен), но не как хуки системного AX.

Практический вывод для автора твика: от jailbreak-прослойки ты не унаследуешь ни полезных примитивов, ни помех. Dopamine здесь полностью «прозрачен».

## 3. Публичная поверхность — что хукается легко

Таблица «частые задачи и методы перехвата»:

| Задача | Метод | Риск | Комментарий |
|---|---|---|---|
| Логировать, кто запрашивает accessibility-метки в твоём приложении | `-[UIView accessibilityLabel]` getter | Низкий | Полезно для отладки собственного UI. |
| Подменить `accessibilityLabel` для скринридера | тот же getter | Низкий | Этично только для своего приложения. |
| Отследить, когда включается VoiceOver | подписка на `UIAccessibilityVoiceOverStatusDidChangeNotification` | Низкий | Через `NSNotificationCenter`, хука не требует. |
| Логировать озвучиваемые тексты | `UIAccessibilityPostNotification` | Низкий | Через fishhook/ellekit, работает в своём приложении. |
| Снять accessibility-дерево чужого приложения системно | — | **Этическая зона** | Технически возможно, см. раздел 5. |

### 3.1. Пример: трейсер accessibilityLabel в своём приложении

```objc
#import <UIKit/UIKit.h>
#import <os/log.h>

static os_log_t gAXLog;

%ctor { gAXLog = os_log_create("com.example.axtrace", "label"); }

%hook UIView

- (NSString *)accessibilityLabel
{
    NSString *label = %orig;
    if (label.length > 0) {
        os_log(gAXLog, "accessibilityLabel on %{public}@: %{public}@",
               NSStringFromClass(self.class), label);
    }
    return label;
}

%end
```

Разбор:

- `%hook UIView` + метод `accessibilityLabel` — работает, потому что UIKit многие view-классы наследуют от `UIView` и реализуют свойство здесь.
- Проверка `label.length > 0` отфильтровывает пустые ответы (многие views не задают метку).
- `%{public}@` — критично, иначе на устройстве значения уйдут в приват-формат `<private>`.

Это легитимный диагностический инструмент: отладка собственного UI на предмет его реальной «читаемости» для VoiceOver.

### 3.2. Пример: нотификация, когда включается VoiceOver

Для реактивного твика (например, темнее экран, когда VoiceOver активен — чисто ваше приложение) — без хука:

```objc
%ctor {
    [[NSNotificationCenter defaultCenter] addObserverForName:UIAccessibilityVoiceOverStatusDidChangeNotification
                                                      object:nil
                                                       queue:[NSOperationQueue mainQueue]
                                                  usingBlock:^(NSNotification *n) {
        BOOL on = UIAccessibilityIsVoiceOverRunning();
        NSLog(@"VoiceOver is now %@", on ? @"ON" : @"OFF");
    }];
}
```

Публичный API `UIAccessibilityIsVoiceOverRunning()` возвращает актуальное состояние без хуков.

## 4. Пограничные случаи: live transcription и Siri suggestions

iOS 26 расширяет роль AX в двух направлениях:

- **Live Caption** — автоматическая субтитрация системного аудио. По открытым источникам (Apple release notes WWDC-эпохи) это отдельная функциональность, которая работает поверх AX-инфраструктуры. На iOS 26.0 конкретные имена классов и демонов — требуют проверки на устройстве.
- **Siri Intelligence / Writing Tools** — работают поверх AX на уровне чтения контента приложения. Для автора твика это важно, потому что твой twik может оказаться «параллельно» Siri, который тоже читает accessibility-дерево.

Правило: не считай AX-дерево «своей собственностью». В iOS 26 его читают сразу несколько системных подсистем (VoiceOver, Siri, Writing Tools), и твои хуки на `accessibilityLabel` будут срабатывать на вызовах от всех них.

## 5. Этическая зона — твики, читающие с чужого экрана

Технически у автора твика есть возможность пройтись по иерархии views (`UIApplication.keyWindow.subviews` рекурсивно) в чужом приложении и зачитать все `accessibilityLabel`/`text`/`attributedText`. Это даёт эффект, похожий на screen-reader на максимальных скоростях.

Три причины, почему это **плохая идея**:

1. **Юридически** в большинстве юрисдикций чтение данных пользователя без согласия (включая содержимое мессенджера или баннера) — это нарушение privacy-регуляций.
2. **Экосистемно** такие твики убивают доверие к jailbreak-сообществу: Apple получает повод для усиления песочницы и mitigation'ов в следующих версиях iOS.
3. **Технически** AX-дерево — не стабильный интерфейс. Приложения часто его намеренно не заполняют (чтобы не светить внутреннее состояние), и твой «следилочный» твик даёт неполные данные, которые пользователь воспринимает как достоверные.

Если ваш usecase легитимен — тестирование UI, отладка accessibility собственного приложения, автоматизация for yourself — пусть он останется в границах filter plist вашего приложения.

## 6. API, доступные через UIAccessibility — полный список для авторов твиков

Для удобства:

| Класс / метод | Что даёт |
|---|---|
| `UIAccessibilityIsVoiceOverRunning()` | Boolean, активен ли VoiceOver |
| `UIAccessibilityIsSwitchControlRunning()` | Switch Control |
| `UIAccessibilityIsReduceMotionEnabled()` | Включён ли «Reduce Motion» |
| `UIAccessibilityIsReduceTransparencyEnabled()` | «Reduce Transparency» |
| `UIAccessibilityIsBoldTextEnabled()` | «Bold Text» |
| `UIAccessibilityIsDarkerSystemColorsEnabled()` | «Increase Contrast» |
| `UIAccessibilityIsInvertColorsEnabled()` | «Invert Colors» (legacy) |
| `UIAccessibilityIsClosedCaptioningEnabled()` | Closed captions |
| `UIAccessibilityIsAssistiveTouchRunning()` | AssistiveTouch |
| `UIAccessibilityPostNotification(...)` | Отправить announcement/screen-change/layout-change |
| `UIAccessibilityCustomAction` / `-Rotor` | Кастомные действия для VoiceOver |

Полезное наблюдение: все эти функции — не методы класса, а C-функции. Хукать их через fishhook/ellekit — проще, чем через Logos. Пример:

```c
#include <dlfcn.h>

static BOOL (*orig_IsVO)(void);
static BOOL fake_IsVO(void)
{
    BOOL real = orig_IsVO();
    // "Форсируем VoiceOver" в своём приложении — только для теста
    return YES;
}

__attribute__((constructor))
static void init_hooks(void)
{
    // Через ellekit API, подробности — в главе IV.5
    // EKHookFunction((void*)&UIAccessibilityIsVoiceOverRunning, (void*)fake_IsVO, (void**)&orig_IsVO);
}
```

## 7. Таблица «что хукать / что не стоит»

| Задача | Способ | Статус |
|---|---|---|
| Логировать запросы `accessibilityLabel` в моём приложении | `%hook UIView` | Безопасно |
| Подменить `accessibilityLabel` для скринридера в своём приложении | То же | Безопасно, но прежде подумай — не ломаешь ли ты доступность |
| Форсировать VoiceOver on/off для теста | fishhook `UIAccessibilityIsVoiceOverRunning` | Безопасно |
| Подавить все announcements в своём приложении | `%hookf UIAccessibilityPostNotification` | Безопасно |
| Снять AX-дерево другого приложения | обход views + `object_getIvar` | **Этически — нет** |
| Хукать AX Server (`assistantd`, `axassetsd`) | — | **Технически возможно, но ни одной причины так делать для стороннего твика** |

## 8. Что подтверждено и что нет

- **Подтверждено исходниками**: Dopamine не вмешивается в accessibility-подсистему (пустой `grep` по BaseBin). Это означает, что любое поведение AX в системе на jailbroken-устройстве — чисто iOS'ное.
- **Подтверждено документацией Apple**: все перечисленные функции `UIAccessibility*` — публичный SDK.
- **Требует проверки на устройстве**: точные имена процессов AX на iOS 26.0/26.0.1 (их легко получить через `launchctl list | grep -i ax` на устройстве); конкретные классы, обслуживающие Live Caption на 26.
- **Не подтверждено**: наличие в iOS 26 конкретных private AX-протоколов из более ранних версий (`AXRemoteElement`, `AXSpeechSynthesizer`), это артефакты, меняющиеся между минорными версиями, и требуют class-dump на устройстве.

## 9. Выводы

Главный тезис: accessibility — это публичный, стабильный, легко хукаемый интерфейс, который Dopamine оставляет полностью в покое. Автор твика получает нулевое вмешательство со стороны jailbreak и полную ответственность за этические последствия. Безопасный маршрут — делать твики, которые помогают доступности или облегчают отладку собственного UI; «следилки», читающие AX-дерево чужого приложения, выходят за рамки инженерной задачи этой книги.

Если нужно подменять поведение VoiceOver у конкретного приложения-хоста (моего же приложения) — весь стек работает ровно так, как описан в Apple SDK, без сюрпризов, с теми же `%hook UIView` или fishhook'ами C-функций `UIAccessibility*`.

## 10. Что дальше

- Для диагностики «пропал VoiceOver после установки твика» — глава VII.1.
- Для деталей `assistantd` и подобных демонов — глава «Daemons: AX» (планируется).
- Для cookbook рецепта «логирование accessibility в своём приложении» — соответствующий рецепт раздела cookbook.
- Для этики и границ — глава I.5 (что jailbreak даёт и чего не стоит делать).
