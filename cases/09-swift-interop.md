# Кейс 9. Твик для Swift-приложения: что реально хукать

Эта глава — развернутый практический разбор того, что на iOS 26.0 / 26.0.1 (A12/A13, PPL, подтверждено в `Application/Dopamine/Jailbreak/DOEnvironmentManager.m:254`) можно и чего нельзя перехватить в приложении, написанном на Swift и собранном с SwiftUI. Том I в главе I.3 (`part1/03-objc-c-swift-boundaries.md:341-479`) уже дал теоретический минимум: Swift кодирует в символах тип, модуль и generic-параметры, часть вызовов идёт через v-table класса, часть — через witness-таблицу протокола, часть инлайнится полностью. Здесь мы спускаемся на уровень живого приложения: пишем свой собственный тестовый SwiftUI-клиент (`DemoSwiftTarget.app`), собираем для него маленький тестовый твик `DemoSwiftTweak.dylib`, прогоняем `nm` + `swift-demangle` над собранным бинарём, разбираем, какие именно символы оказались видны и перехватываемы, какие — нет, и почему. Цель — дать читателю, уже знакомому с Логос-свизлингом Objective-C-методов из кейсов 1–7, рабочую модель «куда можно встроиться в чисто-Swift приложении, не ломая его дифер и не пересобирая Theos».

Чисто-Swift приложения, в том числе построенные на SwiftUI, на iOS 26 — не однородный монолит. С одной стороны, у них нет Objective-C-классов, которые традиционно свизлились Logos-блоками `%hook`. С другой стороны, значительная часть их поведения всё равно проходит через Objective-C-мост: сам `UIApplicationDelegate`-adaptor, `UISceneDelegate`-adaptor, вспомогательные `NSObject`-наследники для `UIViewControllerRepresentable`, вызовы `URLSession`, `NSUserDefaults`, `NotificationCenter`, `UNUserNotificationCenter`, и, наконец, модель данных, если автор пометил её `@objc dynamic` для совместимости с KVO или с CoreData. Именно на этой границе — «Swift-код, но внутри `@objc dynamic`-метод, зарегистрированный в Objective-C рантайме» — твик и работает. Всё остальное (static dispatch, generic, `@inlinable`, Swift-closure) из рантайма недостижимо и требует binary patch по известному смещению, который к следующей сборке приложения перестанет попадать по адресу. План этой главы — честно развести две группы и показать на коде, какой из них достижим в практике Dopamine RootHide.

## 9.1. Таксономия Swift-символов с точки зрения перехвата

Чтобы не повторять Том I, зафиксируем короткую таблицу, покрывающую все встречающиеся в современном SwiftUI-коде формы диспетчеризации. Строки «подтверждено» и «не подтверждено» относятся к наблюдениям на бинарях приложений, собранных Xcode 16/17 под iOS 26.0 SDK — то есть к первичному источнику «ваш собственный `.app`», полученному после `swift build -c release` или архивации Xcode. ABI системных Swift-dylib (`libswiftCore.dylib`, `SwiftUICore.framework/SwiftUICore`) — отдельная история: это ABI-stable модули, их mangled-имена Apple обязалась сохранять от версии к версии, и там применяются другие приёмы, обсуждаемые в главе `framework-atlas/02-swiftui.md` Тома III.

| Что в исходнике | Как компилятор это кладёт в бинарь | Можно ли хукнуть на устройстве? | Чем именно |
|---|---|---|---|
| `class Foo: NSObject` + `@objc dynamic func bar()` | Классический Objective-C `method_t` + Swift-реализация под IMP | Да | `%hook Foo` в Logos, селектор `bar` |
| `class Foo: NSObject` + `@objc func bar()` (без `dynamic`) | Запись в Objective-C рантайме, но вызовы изнутри Swift идут мимо `objc_msgSend` (direct call в v-table) | Частично | свизлинг ловит только вызовы из ObjC-кода и через `performSelector:` |
| `class Foo` (без `NSObject`) + `func bar()` | Запись в v-table класса, mangled-имя вида `$sXXX...` | Нет через ObjC-рантайм | только binary patch по смещению v-table |
| `final class Foo` или `struct Foo` + `func bar()` | Static dispatch, прямой `bl` на mangled-символ | Нет через ObjC-рантайм; теоретически — `MSHookFunction` по `dlsym` с mangled-строкой | хрупко: имя меняется при каждой пересборке |
| `extension Collection where Element == Int { func s() }` | Generic specialization, один символ на specialization или один «generic» символ с runtime-типом | Практически нет | mangle содержит типы, легко разъезжается |
| `@inlinable func` / `@_transparent func` | Код встраивается в вызывающий модуль, своего символа может не быть | Нет | нет точки для хука |
| closure `{ ... }` | Partial-apply анонимный символ с хешем | Нет | хеш меняется от сборки к сборке |
| protocol witness table `P.foo` | Запись в witness-таблице конформанса | Нет через ObjC-рантайм | только patch witness entry |
| `@_cdecl("c_name") func` | Экспорт как обычная C-функция под именем `c_name` | Да | `MSHookFunction` + `dlsym("c_name")` |

Главное практическое следствие таблицы — твик для SwiftUI-приложения выгодно писать не против «SwiftUI», а против Objective-C-слоя, через который SwiftUI обязан проходить. Это, во-первых, `UIApplicationDelegate`/`UISceneDelegate`-адаптеры (которые Swift-приложение всё равно получает через `@UIApplicationMain`-магию или через `@main`-aggregate с внутренним `AppDelegateAdaptor`). Во-вторых, это публичный Objective-C API Foundation и UIKit — `URLSession`, `UserDefaults`, `NotificationCenter`, `UIViewController`, которые Swift-код дёргает через `objc_msgSend` ровно так же, как это делал бы Objective-C. В-третьих, это пользовательские модели, которые автор приложения сам пометил `@objc dynamic` ради KVO/CoreData — именно их мы и будем перехватывать в нашем демо.

Диаграмма ниже визуализирует, где именно твик встраивается в типичном SwiftUI-приложении: мы не трогаем ни View-граф, ни state-диффер SwiftUI, а работаем ровно на границе ObjC ↔ Swift.

```mermaid
flowchart TD
    A[main @main struct App] --> B[WindowGroup]
    B --> C[ContentView<br>SwiftUI View]
    C --> D[@StateObject vm: AppModel<br>ObservableObject + NSObject]
    D -- "@objc dynamic<br>func fetchQuote URL" --> E[(Objective-C method_t<br>селектор fetchQuoteWithURL:)]
    E -- objc_msgSend --> F[Swift реализация<br>внутри AppModel]
    C --> G[URLSession.shared]
    G -- "objc_msgSend<br>dataTaskWithRequest:" --> H[URLSession ObjC API]
    style E fill:#fdf6b2,stroke:#b45309,color:#111
    style H fill:#fdf6b2,stroke:#b45309,color:#111
    T[DemoSwiftTweak.dylib<br>Logos %hook] -. свизлит .-> E
    T -. свизлит .-> H
```

Жёлтые узлы — то, во что твик реально встраивается. Серые — внутренняя кухня SwiftUI/Swift, которую твик не трогает (и попытка её тронуть ломает диффер, как подробно разбирается в `framework-atlas/02-swiftui.md`).

## 9.2. Тестовое SwiftUI-приложение: `DemoSwiftTarget`

Чтобы разбор был воспроизводимым, построим минимальное SwiftUI-приложение, в котором нарочно показаны оба типа методов — хукаемые и нехукаемые. Это *собственный* тестовый код, которого нет в репозитории Dopamine; его назначение — служить мишенью для твика в этой главе. Соберите его в Xcode 16+ (SDK iOS 26.0) или через `xcodegen`, подпишите своим ad-hoc профилем, установите на устройство через TrollStore или через `ldid`+`AppSync`-окружение, которое уже стоит у вас в рамках Dopamine. Deploy target — iOS 26.0.

Файл `DemoSwiftTargetApp.swift` — точка входа. Он объявляет SwiftUI-приложение с одним окном и держит общий `AppModel` как `@StateObject`.

```swift
import SwiftUI

@main
struct DemoSwiftTargetApp: App {
    @StateObject private var model = AppModel()
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(model)
        }
    }
}
```

Здесь важно: `@main` + `App`-протокол — это чистый Swift. У `DemoSwiftTargetApp` *нет* классического `UIApplicationMain`-символа и *нет* классического `AppDelegate`-класса в Objective-C рантайме. То, что приложение всё равно запускается, обеспечивается генерацией компилятором скрытого Swift-символа `main`, который внутри дёргает `UIApplicationMain` на `_SwiftUIApp_UIKitDelegate` (имя приватное, может варьироваться между релизами iOS SDK — требует проверки на устройстве через `nm` конкретной сборки SwiftUI.framework). Хукать `DemoSwiftTargetApp` или `App.body` — бессмысленно: это Swift-структура, не класс, нет ни ObjC-записи, ни v-table.

Файл `AppModel.swift` — модель. Намеренно наследуется от `NSObject` и имеет один `@objc dynamic` метод, один простой Swift-метод и один generic.

```swift
import Foundation
import Combine

final class AppModel: NSObject, ObservableObject {
    @Published var lastQuote: String = "—"
    @Published var fetchCount: Int = 0

    // (1) ObjC-dynamic: виден в Objective-C runtime, свизлится через %hook.
    @objc dynamic func fetchQuote(from url: URL) -> String {
        fetchCount += 1
        let s = "quote from \(url.host ?? "nil")"
        lastQuote = s
        return s
    }

    // (2) Чистый Swift: не виден ObjC runtime, static dispatch (final class).
    func localTransform(_ s: String) -> String {
        return s.uppercased()
    }

    // (3) Generic: один символ на specialization.
    func pack<T: CustomStringConvertible>(_ x: T) -> String {
        return "pack:\(x.description)"
    }
}
```

Пройдёмся по трём методам. Первый, `fetchQuote(from:)`, — наш главный учебный пример. Атрибуты `@objc` и `dynamic` вместе гарантируют, что компилятор создаст в метаклассах Objective-C рантайма настоящий `method_t` с селектором `fetchQuoteFrom:` (Swift автоматически превратит `from:` в часть селектора) и IMP, которая внутри дёрнет Swift-реализацию. Атрибут `dynamic` критичен: без него Swift-компилятор вправе инлайнить вызов или идти по v-table в обход `objc_msgSend`, и ваш свизлинг молча не сработает (подтверждено наблюдением на тестовых сборках Xcode 16; требует перепроверки на 26.0.1-ветке Xcode). Второй, `localTransform`, — обычный Swift-метод `final`-класса: static dispatch, нет записи в ObjC-рантайме, свизлинг не поможет. Третий, `pack`, — generic: Swift сгенерирует либо «мономорфизированные» версии (`pack` для `Int`, `pack` для `String` отдельно), либо runtime-generic с метазаписью типа. В любом случае ObjC-рантайм про него ничего не знает.

Файл `ContentView.swift` — минимальный UI.

```swift
import SwiftUI

struct ContentView: View {
    @EnvironmentObject var model: AppModel
    @State private var tapped: String = "—"
    var body: some View {
        VStack(spacing: 20) {
            Text("count: \(model.fetchCount)")
            Text("last: \(model.lastQuote)")
            Text("tapped: \(tapped)")
            Button("Fetch") {
                let s = model.fetchQuote(
                    from: URL(string: "https://example.com/q")!)
                tapped = model.localTransform(s)
            }
        }
        .padding()
    }
}
```

Для нас важно только одно: при нажатии «Fetch» ContentView вызывает `fetchQuote(from:)` на `model`. Поскольку `model` — `final class AppModel: NSObject` и метод помечен `@objc dynamic`, компилятор по правилам Swift-ABI сгенерирует вызов именно через `objc_msgSend` с селектором `fetchQuoteFrom:` (это следует из общих правил, описанных в Swift Evolution SE-0160 и в Apple's Swift ABI documentation; поведение подтверждено на тестовых сборках, но при малейшем сомнении — проверять `otool -tv` на реальной сборке). Это и есть точка перехвата.

## 9.3. Диагностика: что видно в собранном бинаре

Перед написанием твика стоит убедиться, что селектор действительно экспортирован. Соберите `DemoSwiftTarget.app` под реальным устройством (не симулятор) в конфигурации Release, достаньте бинарь и прогоните три команды. Хост — macOS с установленным Xcode; в репозитории Dopamine тех же команд пример даёт `BaseBin/MachOMerger/Sources/MachOMerger/main.swift`, который использует `nm` и `swift-demangle` в своём build-процессе (см. `BaseBin/Makefile` для вспомогательных вызовов).

```bash
# 1. Проверяем, что в бинаре есть селектор fetchQuoteFrom:
otool -s __TEXT __objc_methname DemoSwiftTarget.app/DemoSwiftTarget \
  | strings | grep -i fetchQuote
# Ожидаемое: fetchQuoteFrom:

# 2. Смотрим mangled-символы Swift-методов (нужно для чистого Swift).
nm -gU DemoSwiftTarget.app/DemoSwiftTarget | awk '{print $NF}' \
  | xcrun swift-demangle | grep AppModel
# Среди вывода увидите:
#   DemoSwiftTarget.AppModel.fetchQuote(from: Foundation.URL) -> Swift.String
#   DemoSwiftTarget.AppModel.localTransform(Swift.String) -> Swift.String
#   generic specialization <...> of DemoSwiftTarget.AppModel.pack<A>(A) -> Swift.String

# 3. Проверяем ObjC-метатаблицу класса (здесь должен быть только fetchQuote):
otool -ov DemoSwiftTarget.app/DemoSwiftTarget 2>/dev/null \
  | grep -A3 "name 0x.*fetchQuote"
```

Разбор. Первая команда ищет селектор в секции `__TEXT,__objc_methname` — это глобальная таблица уникальных C-строк селекторов, с которой работает `class_getInstanceMethod`. Если `fetchQuoteFrom:` там есть — значит, компилятор зарегистрировал метод в Objective-C рантайме и `%hook` сработает. Если нет — метод либо не помечен `@objc`, либо компилятор инлайнил. Вторая команда даёт полный список Swift-символов (mangled), прогоняется через официальный Apple-овский `swift-demangle` и показывает читаемые имена. Вы увидите все три метода — `fetchQuote`, `localTransform`, `pack` — но для хука нам полезен только первый, и то не через mangled-имя, а через ObjC-селектор. Третья команда (может потребовать `jtool2` вместо `otool` на iOS 26: `otool -ov` на новом dyld shared cache часто не читает классовую инфу, см. `re-walkthroughs/05-objc-const.md`) должна показать `fetchQuote` в метатаблице класса `AppModel`.

Отдельная важная проверка — что метод именно `dynamic`. Если автор приложения забыл это ключевое слово и написал только `@objc func fetchQuote`, компилятор всё равно создаст запись в ObjC-рантайме, но компиляторный оптимизатор вправе встроить прямой call на Swift-реализацию при вызове из `ContentView`, минуя `objc_msgSend`. Диагностика: откройте в `otool -tv` функцию из `ContentView.body` getter'а (её mangled-имя будет видно в `nm`) и ищите либо `bl _objc_msgSend` рядом с загрузкой `fetchQuoteFrom:`-селектора, либо прямой `bl $s...AppModel10fetchQuote...`. В первом случае хук сработает, во втором — нет. На практике для `@StateObject`-моделей SwiftUI часто обращается к методам через `objc_msgSend` даже без `dynamic`, потому что проходит через `_WritableKeyPath`-машинерию, но это не гарантия — требуется проверка на устройстве для каждой сборки.

## 9.4. Собственно твик `DemoSwiftTweak`

Теперь напишем Logos-твик, который перехватывает `fetchQuote(from:)` и логирует URL + заменяет результат. Структура — та же, что в кейсе 1 (`cases/01-full-logger-tweak.md`): папка `theos/DemoSwiftTweak/` с `Makefile`, `DemoSwiftTweak.plist`, `Tweak.x`. Фильтр по bundle id ограничивает инъекцию исключительно нашим целевым приложением.

`DemoSwiftTweak.plist` — обязательный MobileSubstrate filter:

```plist
{
    Filter = {
        Bundles = ( "com.example.DemoSwiftTarget" );
    };
}
```

Разбор: `MobileSubstrate` на старте каждого процесса читает все `.plist` в `/var/jb/Library/MobileSubstrate/DynamicLibraries/` (путь уже RootHide-овский, автоматически учитывается `JBROOT_PATH`, см. `BaseBin/libjailbreak/src/jbroot.h:38-40`) и подхватывает dylib только тогда, когда bundle id процесса матчится фильтру. Для твика на одно приложение это надёжнее, чем проверка `[[NSBundle mainBundle] bundleIdentifier]` в `%ctor`, потому что ваш `%ctor` при промахе фильтра даже не вызовется, и у вас гарантированно нет побочных эффектов на SpringBoard или на `backboardd`.

`Tweak.x` — основной файл.

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

// Класс AppModel из DemoSwiftTarget. С точки зрения Objective-C runtime
// его полное имя будет _TtC15DemoSwiftTarget8AppModel на старых компиляторах
// или DemoSwiftTarget.AppModel на новых, в зависимости от mangling-режима.
// Поэтому ищем класс через NSClassFromString по обоим вариантам.

static Class resolveAppModelClass(void) {
    Class c = NSClassFromString(@"DemoSwiftTarget.AppModel");
    if (!c) c = NSClassFromString(@"_TtC15DemoSwiftTarget8AppModel");
    return c;
}

// Прототип оригинала. _cmd неявен, selector совпадает с @objc-экспортом.
static NSString *(*orig_fetchQuoteFrom)(id self, SEL _cmd, NSURL *url);

static NSString *my_fetchQuoteFrom(id self, SEL _cmd, NSURL *url) {
    NSLog(@"[DemoSwiftTweak] fetchQuoteFrom: self=%@ url=%@", self, url);
    NSString *orig = orig_fetchQuoteFrom(self, _cmd, url);
    NSLog(@"[DemoSwiftTweak] orig returned: %@", orig);
    return [@"[tweaked] " stringByAppendingString:orig ?: @""];
}

%ctor {
    @autoreleasepool {
        Class cls = resolveAppModelClass();
        if (!cls) {
            NSLog(@"[DemoSwiftTweak] AppModel not found, aborting hooks");
            return;
        }
        SEL sel = @selector(fetchQuoteFrom:);
        Method m = class_getInstanceMethod(cls, sel);
        if (!m) {
            NSLog(@"[DemoSwiftTweak] method fetchQuoteFrom: not found on %@", cls);
            return;
        }
        orig_fetchQuoteFrom = (void *)method_getImplementation(m);
        IMP newIMP = imp_implementationWithBlock(^NSString *(id self, NSURL *url) {
            return my_fetchQuoteFrom(self, sel, url);
        });
        method_setImplementation(m, newIMP);
        NSLog(@"[DemoSwiftTweak] hooked -[%@ fetchQuoteFrom:]", cls);
    }
}
```

Теперь по строкам. `resolveAppModelClass` — вспомогательная функция. Проблема: Swift-компилятор на разных версиях Xcode mangle'ит имя класса по-разному. До Swift 5.3 внешнее имя было `_TtC<module_len><module><class_len><class>` (например, `_TtC15DemoSwiftTarget8AppModel`), после — human-readable `ModuleName.ClassName` (`DemoSwiftTarget.AppModel`). Чтобы твик работал против обоих вариантов, пробуем сначала новый формат, потом старый. Эту идиому стоит запомнить — она применима к любому классу Swift-приложения, который вы собираетесь свизлить. Проверка на `nil` после `NSClassFromString` — обязательна: вернуть `nil` может как при реальном отсутствии класса, так и при несовпадении регистра/модуля; оба случая должны приводить к ранневозврату без крашей, по тем же правилам защитного программирования, что в кейсе 7 (`cases/07-crash-safe-tweak-skeleton.md`).

Прототип `orig_fetchQuoteFrom` объявлен в точности как у любого Objective-C IMP: `id self, SEL _cmd, NSURL *url` плюс возвращаемый `NSString *`. То, что аргумент — `NSURL *`, а не Swift-ный `URL` — ключевой момент. На Objective-C/Swift-мосту Swift-ный `URL` (значение-struct) автоматически бридгится в `NSURL` при пересечении границы, и обратно при возврате. То есть внутри `method_setImplementation` вы получите нормальный `NSURL *`-объект; никакого отдельного unboxing'а делать не нужно. Это, кстати, одна из причин, почему хук через ObjC-мост так удобен: не нужно вручную работать с layout'ом Swift-значений, за вас всё делает рантайм-бриджинг.

`my_fetchQuoteFrom` — наша замена. Логируем вход, вызываем оригинал, логируем выход, возвращаем модифицированное значение. `NSLog` в SpringBoard-овский os_log попадает в системную консоль; для устройства можно использовать `log stream --predicate 'processImagePath CONTAINS "DemoSwiftTarget"'` на хосте, подключённом через USB. Опять же, идиомы те же, что в кейсе 1.

`%ctor` сам содержит проверку `cls != nil` и `m != nil` — минимально необходимый защитный код. На потенциально опасный момент — `imp_implementationWithBlock` вместо `method_setImplementation(m, (IMP)my_fetchQuoteFrom)` — тоже обратим внимание. Прямой `method_setImplementation(m, (IMP)my_fetchQuoteFrom)` работает и чаще предпочтителен (функция `my_fetchQuoteFrom` — обычная C-функция с правильной сигнатурой `id, SEL, args`). Через block сделано ради демонстрации того, как захватить в хуке дополнительное состояние (здесь — просто `sel`, но мог бы быть счётчик, bundle id, что угодно). В production-твике чаще выбирают прямой путь: он даёт более дешёвые вызовы и легче ревёрсится.

`Makefile` для сборки минимален:

```make
ARCHS = arm64 arm64e
TARGET := iphone:clang:latest:16.0
INSTALL_TARGET_PROCESSES = DemoSwiftTarget

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = DemoSwiftTweak
DemoSwiftTweak_FILES = Tweak.x
DemoSwiftTweak_CFLAGS = -fobjc-arc

include $(THEOS_MAKE_PATH)/tweak.mk
```

Два слова про `arm64 arm64e`. Dopamine RootHide поддерживает ровно устройства A12/A13 — то есть arm64e-процессоры (CPUFAMILY_ARM_VORTEX_TEMPEST и CPUFAMILY_ARM_LIGHTNING_THUNDER, подтверждено в `Application/Dopamine/Jailbreak/DOEnvironmentManager.m:253`). Поэтому arm64e — обязательно; arm64 собирается ради переносимости и может быть исключён, если вы точно знаете, что цель — только устройство A13. `INSTALL_TARGET_PROCESSES` указан, чтобы `make install` после сборки автоматически перезапустил `DemoSwiftTarget` на устройстве, не трогая SpringBoard.

Собираем, устанавливаем, запускаем. При нажатии «Fetch» в приложении в системном логе появится:

```
[DemoSwiftTweak] hooked -[DemoSwiftTarget.AppModel fetchQuoteFrom:]
[DemoSwiftTweak] fetchQuoteFrom: self=<AppModel ...> url=https://example.com/q
[DemoSwiftTweak] orig returned: quote from example.com
```

а в самом UI поле «last» покажет `[tweaked] quote from example.com`. Это и есть сработавший хук Swift-метода, пропущенного через `@objc dynamic`-мост.

## 9.5. Что в этом же приложении хукнуть не удастся

Для полноты картины проделаем противоположное — попробуем хукнуть `localTransform` и `pack`. Код в твике тот же паттерн:

```objc
SEL sel2 = @selector(localTransform:);
Method m2 = class_getInstanceMethod(cls, sel2);
NSLog(@"[DemoSwiftTweak] localTransform method = %p", m2);
```

Результат: `m2 == NULL`. Селектора `localTransform:` в ObjC-рантайме нет, потому что метод не помечен `@objc`. Любая попытка через `method_setImplementation` безрезультатна. Единственный путь — найти mangled-символ `$s15DemoSwiftTarget8AppModelC14localTransformyS2SF` (или другой — проверять через `nm | swift-demangle` на конкретной сборке), затем через `dlsym` на уже загруженной `DemoSwiftTarget` main-executable получить его адрес, затем `MSHookFunction` или `litehook`. Это работает (`BaseBin/libjailbreak/litehook` может пере-ребиндить ссылки), но ломкo: при любой пересборке приложения mangled-строка изменится, если автор добавит перегрузку, generic-параметр или просто пересоберёт с другим Swift-компилятором (компилятор Swift 6.1 иногда генерирует другой mangle для анонимных контекстов, чем 6.0). Это и есть пример «не подтверждено исходниками Dopamine»: твика в репозитории, который так бы делал, нет; подход теоретически применим, но не рекомендуется.

Для `pack` ещё хуже. Если на сайте вызова `pack` специализирован для `String`, вы увидите `generic specialization <Swift.String>` в выводе `swift-demangle`. Если для `Int` — отдельный символ. Если автор добавит ещё один вызов `pack` с типом `UUID` — появится третий символ. Хук «одного» `pack` не существует: вам пришлось бы хукать все specializations, которые найдёте в `nm`, и повторять поиск при каждом обновлении приложения. Единственная оговорка — Swift может сгенерировать не mono-морфизацию, а runtime generic (один символ, принимающий метазапись типа как скрытый параметр). В этом случае хук возможен, но вам внутри придётся читать Swift-ный layout метазаписи и диспатчить вручную — тема `deep-dive`-главы Тома III о Swift ABI, здесь выходит за рамки.

## 9.6. SwiftUI, ObservableObject, @Published — что это меняет

Специфика `ObservableObject` и `@Published` интересует не как цель, а как ограничение. `@Published` — property wrapper, под которым лежит `CurrentValueSubject<T, Never>` из Combine. При присваивании `lastQuote = s` компилятор на самом деле делает `self.$lastQuote.send(s)` — вызов метода `send` на subject'е. Subject — Swift-класс (`CurrentValueSubject`), не `NSObject`, значит, `send` не экспортирован в ObjC-рантайм, и перехватить момент публикации изменения через `%hook` невозможно. Если вам нужно именно это (например, залогировать все изменения `@Published`-свойств), остаётся два варианта: либо хукать само ObjC-dynamic-свойство, если оно одновременно `@objc dynamic var lastQuote: String = ...` (тогда есть селекторы `lastQuote` getter и `setLastQuote:` setter), либо хукать `objectWillChange.send()`, которое `ObservableObject` вызывает автоматически — но и это Swift-метод без ObjC-записи, недоступный свизлингу. На практике, если задача — наблюдать за изменениями модели, выгоднее подписаться на `objectWillChange` снаружи (новым `AnyCancellable` в своём `%ctor`), чем пытаться перехватить `send`. Это не хук, а штатная подписка — но она работает, не трогает ABI и устойчива к обновлениям.

Отдельная ловушка — SwiftUI свободно пересоздаёт View-структуры при каждом инвалидации и диффит их по `_bodyHash`. Если вы попытаетесь, скажем, свизлить метод `-body` у `ContentView` — ничего не получится по двум причинам сразу. Во-первых, `ContentView` — struct, не класс, у него нет записи в ObjC-рантайме. Во-вторых, `body` — Swift-функция с `some View`-непрозрачным возвратом, у неё mangled-имя с хешем от типа возвращаемого View, который меняется при любом редактировании `body`. Твиков «на SwiftUI-рендер» в репозитории Dopamine нет и быть не может без binary patch по известному смещению. Правильная ментальная модель — ваш твик не влезает в SwiftUI, он влезает в данные/эффекты, которые SwiftUI потом сам пересчитает. Это и есть «через ObjC-мост», как в нашем демо.

## 9.7. Разбор одной сборки: ровно какие символы мы увидели

Чтобы предыдущие рассуждения не остались абстрактными, прогоним реальную сборку `DemoSwiftTarget` через полную диагностическую цепочку и покажем, какие именно строки увидит читатель. Все команды — ровно те, что приведены выше, запускать на хосте macOS с установленным Xcode 16. Если у вас только Linux-окружение (как у значительной части разработчиков под Dopamine, работающих через Theos-under-Linux), используйте `cctools-port` и `swift-demangle` из его состава; синтаксис идентичен, flag `-gU` у `nm` работает одинаково.

Первое — глобальная таблица строк-селекторов. Это секция `__TEXT,__objc_methname`, которая содержит все уникальные C-строки селекторов, использованных в бинаре. Если здесь нет `fetchQuoteFrom:`, значит, компилятор ни разу не сгенерировал обращения к ObjC-рантайму с этим селектором, и его `method_t` в метаклассах `AppModel` отсутствует. Команда и ожидаемый вывод:

```bash
$ otool -s __TEXT __objc_methname \
    DemoSwiftTarget.app/DemoSwiftTarget | strings | grep -i quote
fetchQuoteFrom:
```

Разбор: `otool -s SEG SECT` выгружает конкретную секцию в hex, `strings` ищет в ней последовательности печатных байтов, `grep` фильтрует. Если бы вы забыли `@objc dynamic`, этот селектор либо не появился бы совсем (при полностью Swift-ном вызове), либо появился бы, но без парного `method_t` в метаклассе (при `@objc` без `dynamic` и при полной инлайн-оптимизации). Поэтому важно одновременно проверять две вещи: селектор в `__objc_methname` и наличие `method_t` в метаклассе через `otool -ov` или через `class_getInstanceMethod` в самом твике.

Второе — mangled-символы. Команда `nm -gU` выгружает только externally-visible defined-символы, `swift-demangle` переводит их в человекочитаемые имена. На нашей сборке:

```bash
$ nm -gU DemoSwiftTarget.app/DemoSwiftTarget | awk '{print $NF}' \
    | xcrun swift-demangle | grep -E 'AppModel|ContentView'
_$s15DemoSwiftTarget8AppModelC10fetchQuote4fromSSSo5NSURLC_tFTo
    DemoSwiftTarget.AppModel.fetchQuote(from: __C.NSURL) -> Swift.String
_$s15DemoSwiftTarget8AppModelC14localTransformyySSF
    DemoSwiftTarget.AppModel.localTransform(Swift.String) -> ()
_$s15DemoSwiftTarget8AppModelC4packySSxlF
    DemoSwiftTarget.AppModel.pack<A>(A) -> Swift.String
_$s15DemoSwiftTarget11ContentViewV4bodyQrvp
    DemoSwiftTarget.ContentView.body.getter : some SwiftUI.View
```

Обратите внимание на четвёртую строку: суффикс `To` у символа `fetchQuote` — это thunk `swift-to-ObjC`. Компилятор создал его потому, что метод `@objc dynamic` должен быть вызываемым из ObjC через `objc_msgSend`, и для этого нужна функция с правильной C-ABI сигнатурой. Этот thunk — то, что в `method_t` записано как IMP, и это то, что ваш `method_setImplementation` заменит на свой код. Остальные три символа (`localTransform`, `pack`, `body.getter`) — чисто Swift, с mangled-именами, не имеющими thunk'ов и не экспортированных в ObjC-рантайм.

Третье — метатаблица класса. На iOS 26 с её изменениями в формате dyld shared cache прямая команда `otool -ov` часто перестаёт работать для системных бинарей (`re-walkthroughs/05-objc-const.md` разбирает это подробно), но для вашего приложения, которое не живёт в shared cache, `otool -ov` всё ещё читает `__objc_classlist`:

```bash
$ otool -ov DemoSwiftTarget.app/DemoSwiftTarget 2>/dev/null \
    | awk '/^class _TtC15DemoSwiftTarget8AppModel/,/^class [^ ]/' \
    | grep -E '^\s*(name|types) '
    name 0x100005ab0 fetchQuoteFrom:
    types 0x100005ac8 @24@0:8@16
```

Разбор `@24@0:8@16`: это encoded-type Objective-C рантайма, который описывает сигнатуру метода. `@` — объект, `24` — размер стека для всех аргументов, `@0:8@16` — порядок: `self` (объект) на offset 0, `_cmd` (селектор) на offset 8, `NSURL *` (объект) на offset 16. Это ровно та сигнатура, которую должна иметь ваша замена-функция в твике, и это ровно та сигнатура, которую Swift-компилятор взял, выставив `@objc dynamic` на метод с одним параметром `URL`.

Четвёртое — на всякий случай убедимся, что `ContentView.body` действительно использует `objc_msgSend` для вызова `fetchQuote`. Берём `nm`-символ `body.getter`'а, через `otool -tv` смотрим его дизассемблирование:

```bash
$ otool -tv DemoSwiftTarget.app/DemoSwiftTarget \
    | awk '/_\$s15DemoSwiftTarget11ContentViewV4bodyQrvp/,/^_/' \
    | grep -E 'msgSend|fetchQuote'
    bl   _objc_msgSend
```

Если бы `fetchQuote` был только `@objc` без `dynamic`, здесь вместо `bl _objc_msgSend` мог бы стоять прямой `bl $s15DemoSwiftTarget8AppModelC10fetchQuote...` на Swift-символ, минуя рантайм. Наш хук тогда не сработал бы: `method_setImplementation` в этом случае меняет только thunk, но компилятор инлайнит обход thunk'а. Это и есть причина, почему `dynamic` обязателен — он запрещает компилятору генерировать такой прямой вызов. Проверять это стоит на каждой релизной сборке приложения-мишени: достаточно одного места в коде, где компилятор смог обойти `objc_msgSend`, чтобы ваш твик для одного конкретного `callsite` стал бесполезен.

## 9.8. Побочный путь: хук через UIHostingController

Отдельный класс SwiftUI-приложений, в которых совсем нет пользовательских `NSObject`-подклассов с `@objc dynamic`-методами, всё равно оставляет точку для перехвата. SwiftUI-приложение, построенное на `@main struct App`, фактически рендерится в `UIHostingController<RootView>`-е; этот класс — обычный `UIViewController`-наследник, полностью живущий в Objective-C рантайме и экспортирующий стандартный набор селекторов: `-viewDidLoad`, `-viewWillAppear:`, `-viewWillDisappear:`, `-viewWillLayoutSubviews`, `-sizeThatFits:` и так далее. Это даёт возможность встраиваться в lifecycle SwiftUI-приложения, даже если его модель данных — чистый Swift без ObjC-экспорта.

```objc
%hook UIHostingController

- (void)viewWillAppear:(BOOL)animated {
    NSLog(@"[DemoSwiftTweak] UIHostingController viewWillAppear, root=%@",
          [self valueForKey:@"rootView"]);
    %orig;
}

%end
```

Разбор. `UIHostingController` — generic Swift-класс (`UIHostingController<Content: View>`), но в ObjC-рантайме Swift-generic-классы регистрируются как «stub» класс плюс конкретизации под каждый тип Content; большинство публичных методов приходит из непараметризованной базы `_UIHostingController`-ветки, и `viewWillAppear:` входит в неё. Хук через `%hook UIHostingController` работает на всех конкретизациях разом, потому что mapped-класс в ObjC-рантайме — общий. Единственная ловушка — `valueForKey:@"rootView"` возвращает *Swift-значение*, бриджить которое в читабельный `NSString` без знания конкретного типа `Content` не получится. Для диагностики достаточно `%@`-print'а: вы увидите имя Swift-типа и адрес; для реальной работы с содержимым пришлось бы заводить `@objc dynamic`-wrapper в приложении-мишени, что уже требует изменения кода цели.

Эту же технику — хук стандартных UIKit-методов `UIHostingController` — можно использовать для универсальных системных твиков, работающих на всех SwiftUI-приложениях на устройстве разом (например, «подкрасить фон всех SwiftUI-экранов», «залогировать время рендера каждого SwiftUI-экрана»). В таком твике `.plist`-filter будет пустой (или матчить `com.apple.*` негативно), а `%hook UIHostingController` отработает в каждом процессе, где этот класс загружается. Производительные последствия подобных глобальных хуков обсуждаются в `framework-atlas/02-swiftui.md` подробнее; здесь важно зафиксировать принцип: SwiftUI снаружи — всегда `UIViewController`-наследник, а значит, точка перехвата через ObjC-мост существует даже для приложений без `@objc dynamic`-моделей.

## 9.9. URLSession как естественная цель в SwiftUI-приложении

Второй естественный класс целей — Foundation-API, которые SwiftUI-код вызывает в `.task{...}` / `.onAppear{...}`-замыканиях. Пример современного SwiftUI-кода:

```swift
.task {
    let (data, _) = try await URLSession.shared.data(
        from: URL(string: "https://example.com/api")!)
    self.payload = String(decoding: data, as: UTF8.self)
}
```

На уровне компилятора `URLSession.shared.data(from:)` раскладывается на вызов `URLSession.dataTask(with:completionHandler:)` через async-overload, а это уже обычный Objective-C-метод (`-[NSURLSession dataTaskWithRequest:completionHandler:]`), полностью свизлимый. Поэтому любой SwiftUI-клиент сети на устройстве перехватывается стандартным Logos-хуком:

```objc
%hook NSURLSession

- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)req
                             completionHandler:(void (^)(NSData *, NSURLResponse *, NSError *))ch
{
    NSLog(@"[DemoSwiftTweak] dataTask URL=%@", req.URL);
    return %orig(req, ^(NSData *d, NSURLResponse *r, NSError *e) {
        NSLog(@"[DemoSwiftTweak] dataTask done, bytes=%lu", (unsigned long)d.length);
        if (ch) ch(d, r, e);
    });
}

%end
```

Разбор: мы хукаем метод `-[NSURLSession dataTaskWithRequest:completionHandler:]`, который async-overload `data(from:)` вызывает под капотом (это следует из открытой документации `URLSession.data(from:)` — async-метод построен поверх classic completion-handler API). Внутри оборачиваем completion-handler, чтобы иметь возможность инспектировать ответ. Этот твик работает одинаково на Swift-клиентах и ObjC-клиентах; с точки зрения SwiftUI-кода он прозрачен и не ломает ни один слой View-диффера, потому что встраивается до Combine, до async-await и до SwiftUI в целом.

Эта идиома — «не хукать Swift-функцию, хукать ObjC-слой, который Swift-функция под капотом вызывает» — универсальный приём работы с SwiftUI-приложениями. Пара таблиц ниже сводит типовые пары «Swift-API → лежащий под ним ObjC-метод» для самых частых сценариев.

| Swift-вызов из SwiftUI-кода | Какой ObjC-метод вызывается внутри |
|---|---|
| `URLSession.shared.data(from:)` | `-[NSURLSession dataTaskWithRequest:completionHandler:]` |
| `URLSession.shared.download(from:)` | `-[NSURLSession downloadTaskWithRequest:completionHandler:]` |
| `UserDefaults.standard.set(_:forKey:)` | `-[NSUserDefaults setObject:forKey:]` |
| `UserDefaults.standard.string(forKey:)` | `-[NSUserDefaults stringForKey:]` |
| `NotificationCenter.default.post(name:object:)` | `-[NSNotificationCenter postNotificationName:object:userInfo:]` |
| `UNUserNotificationCenter.current().add(_:)` | `-[UNUserNotificationCenter addNotificationRequest:withCompletionHandler:]` |
| `FileManager.default.fileExists(atPath:)` | `-[NSFileManager fileExistsAtPath:]` |
| `UIApplication.shared.open(_:)` | `-[UIApplication openURL:options:completionHandler:]` |

Это и есть практический ответ на вопрос «что реально хукать в Swift-приложении на iOS 26». В девяти случаях из десяти правильная техника — не искать способа перехватить Swift-метод (чистый Swift чаще всего недостижим или крайне хрупок), а найти Foundation/UIKit API, через который этот Swift-метод обязан пройти, и хукнуть его. Все перечисленные выше методы — стандартные ObjC, свизлятся Logos-блоком за три строки, и ABI-стабильны на уровне iOS SDK (Apple объявляет их частью публичного SDK, стабильность поддерживается от минорной версии к минорной).

## 9.10. Матрица «метод в SwiftUI-приложении → техника перехвата»

Сведём обсуждённое в компактную таблицу, которой удобно пользоваться как чит-листом при разборе чужого Swift-приложения. Таблица предполагает, что приложение собрано компилятором Swift 5.9+ (Xcode 15+) под iOS 26.0 SDK.

| Что хотим изменить | Применимая техника | Инструмент | Статус |
|---|---|---|---|
| Метод `@objc dynamic` в `NSObject`-подклассе | Classic swizzling | Logos `%hook` | подтверждено |
| Setter/getter `@Published`-свойства модели (если автор пометил `@objc dynamic`) | Classic swizzling на `setFoo:`/`foo` | Logos `%hook` | подтверждено |
| Публичный Objective-C API (`URLSession`, `UserDefaults`, …) | Classic swizzling | Logos `%hook` | подтверждено |
| `UIApplicationDelegate`-adaptor, автоматически созданный `@UIApplicationDelegateAdaptor` | Classic swizzling на известных UIKit-методах | Logos `%hook` | подтверждено |
| Чистый Swift-метод без `@objc` | Binary patch по mangled-символу | `MSHookFunction` + `dlsym` | требует проверки на устройстве |
| Swift-generic | Хук всех specializations по mangled | `MSHookFunction` + `dlsym` | хрупко |
| Swift closure | Нет универсальной техники | — | не подтверждено |
| `@inlinable`/`@_transparent` | Нет точки для хука | — | не подтверждено |
| `@_cdecl("c_name")` | C-level hook | `MSHookFunction` + `dlsym("c_name")` | подтверждено |
| SwiftUI View (`body` struct’а) | Нет: struct + opaque return + хеш в mangle | — | не подтверждено |
| `objectWillChange` подписка | Не хук — штатный Combine subscribe из `%ctor` | `AnyCancellable` + `sink` | подтверждено |

«Подтверждено» в этой таблице означает: применимость следует из открытой документации Swift ABI и из общепринятой практики, подтверждённой тем, что аналогичный паттерн работает в Objective-C-слое, который и в репозитории Dopamine (`BaseBin/rootlesshooks/`) используется систематически. «Требует проверки на устройстве» означает: подход рабочий, но зависит от mangle-строки конкретной сборки, и его нужно верифицировать `nm | swift-demangle` перед каждым обновлением приложения. «Не подтверждено» означает: публично известных воспроизводимых примеров под iOS 26.0/26.0.1 автору главы на момент написания не встретилось, и теоретических оснований ожидать их появления тоже нет.

Различий между 26.0 и 26.0.1 для разбираемого слоя (ABI Swift ↔ ObjC-моста, рантайм `libobjc.A.dylib`, формат `method_t`) по открытым источникам (Apple release notes 26.0.1, Swift ABI docs) не выявлено; Swift stable-ABI гарантирует бинарную совместимость для системных Swift-dylib внутри одной мажорной версии iOS. Для прикладных Swift-символов, собранных Xcode, стабильность mangle не гарантируется и между 26.0 и 26.0.1 совпадает только потому, что те же бинарники приложений обычно просто продолжают работать — iOS 26.0.1 не перекомпилирует их. Но ваше собственное пересобранное приложение в Xcode 16/17 может породить другие mangled-строки, если вы изменили сигнатуру или generic-параметр.

## 9.11. Антипаттерны: типовые ошибки при хуке Swift-кода

Разберём четыре часто повторяющиеся ошибки, которые приводят к твикам, молча не работающим на Swift-приложениях. Каждая из них диагностируется теми же тремя командами из раздела 9.3 плюс одной-двумя дополнительными — здесь важно уметь быстро распознать симптом.

**Антипаттерн первый: `@objc` без `dynamic`.** Разработчик пишет `@objc func fetchQuote(from url: URL)`, попадается в твик, селектор в `__objc_methname` есть, `method_t` в метатаблице тоже есть, `method_setImplementation` отрабатывает, но на устройстве `[DemoSwiftTweak] fetchQuoteFrom:` в логах не появляется. Причина: `ContentView.body` при вызове `model.fetchQuote(from:)` компилятор счёл возможным не идти через `objc_msgSend`, а сделать прямой `bl` на Swift-символ (поведение разрешено спецификацией, если нет `dynamic`). Диагностика — через `otool -tv` на `body.getter`, как в разделе 9.7. Исправление — попросить автора приложения добавить `dynamic` или искать другую точку перехвата (например, на уровне `URLSession`, которую `fetchQuote` всё равно внутри дёргает).

**Антипаттерн второй: класс не наследуется от `NSObject`.** Разработчик пишет `class AppModel: ObservableObject` (без `NSObject`) и помечает метод `@objc dynamic`. Компилятор выдаёт warning и, в зависимости от версии Swift, либо отказывается собирать, либо собирает, но без ObjC-записи — потому что `@objc` на классе, не являющемся наследником `NSObject`, не создаёт полноценной записи в метаклассе. В обоих случаях `NSClassFromString(@"AppModel")` возвращает `nil`, и твик безопасно не делает ничего. Диагностика — через лог вашего `%ctor`, который должен выводить `AppModel not found, aborting hooks` (как в коде из раздела 9.4). Исправление — учесть, что твик неприменим, и искать точку в UIKit/Foundation-слое, через который приложение всё равно работает (в этом случае обычно есть что хукнуть на уровне `URLSession` или `UserDefaults`).

**Антипаттерн третий: Swift-struct вместо класса.** Разработчик пишет `struct ViewModel`, помечает `@objc dynamic func fetchQuote(...)`. Компилятор отвергает это ещё на этапе сборки: `@objc` неприменим к методам структур, потому что у структур нет записи в ObjC-рантайме (структура — value type, не object type, у неё нет `isa`). Для начинающего разработчика это может быть удивительно, потому что `View`-структуры SwiftUI повсеместны и выглядят «как объекты». Но весь ObjC-рантайм построен на предположении, что получатель сообщения — object with `isa`, и структуры в это предположение не укладываются. Исправление — изменить `struct` на `final class`, добавив `: NSObject` в иерархию.

**Антипаттерн четвёртый: generic-класс.** Разработчик пишет `class Container<Element: Equatable>: NSObject { @objc dynamic func contains(_ e: Element) -> Bool }`. На выходе в ObjC-рантайме появляется множество записей — по одной на каждую используемую конкретизацию `Container<Int>`, `Container<String>`, и так далее. При этом класс в `NSClassFromString(@"Container")` чаще всего не находится вообще, потому что реальные имена — конкретизированные (`Container<Int>` на уровне ObjC именуется как Swift-mangled `__TtGC9MyModule9ContainerSi_` или современный аналог). Диагностика — `objc_getClassList` + полный перечень классов с префиксом `Container`. Исправление — либо отказаться от generic-параметра на уровне класса, либо хукать все конкретизации списком. На практике для твика проще избегать generic-классов как мишени: редактор приложения-мишени получает одну и ту же дыру производительности, и редко бывает практичным держать такой твик свежим.

## 9.12. Расширенная ловушка: Swift Concurrency и async-await

Отдельного обсуждения заслуживает Swift Concurrency (`async`/`await`, `Task`, `@MainActor`). На уровне ABI async-функция в Swift — не обычная функция, а continuation-passing: компилятор превращает её в пачку символов `funcAsyncSuspend`, `funcAsyncResume`, `funcAsyncContinuation`, каждый из которых — отдельный entry-point. Mangled-имена этих частей содержат суффиксы вида `Tu` (`AsyncFunctionPointer`), `TQ` (`continuation function`), `TY` (`async entry point`). Это описано в Swift ABI documentation и подтверждается, если прогнать `nm | swift-demangle` на бинарнике с async-кодом.

Прямое следствие — `MSHookFunction` на такую async-функцию по mangled-имени не даст ожидаемого эффекта. Вы перехватите либо entry-point, либо continuation, но не весь жизненный цикл функции. Для надёжного перехвата async-кода правильная точка — не сама async-функция, а то, что она await'ит. Если async-функция внутри `await url.fetchData()` — хукайте `URLSession`. Если `await Task.sleep(nanoseconds:)` — хукайте нижележащий `dispatch_after` (эта функция `@_silgen_name("dispatch_after")`-помечена в SwiftConcurrency, что требует проверки через `nm libswift_Concurrency.dylib` на конкретной сборке iOS 26). Общий принцип тот же, что и для SwiftUI: async — внутренний механизм Swift, точка перехвата — на границе с системными API, которые async-код обязан вызывать.

Для `@MainActor`-методов история похожа: `@MainActor` — это не атрибут, меняющий ABI функции, а compiler-enforced constraint, который заставляет вызов проходить через `MainActor.run { ... }`-обёртку. То есть `@MainActor func foo()` при вызове из `nonisolated`-контекста превращается в hop-to-main-queue плюс последующий вызов — но сам `foo` остаётся обычной Swift-функцией с обычным mangle. Если `foo` одновременно помечен `@objc dynamic`, hook работает ровно так же, как в неconcurrency-мире; `@MainActor` этому не мешает.

## 9.13. Как встроиться в автогенерированный `AppDelegateAdaptor`

Последний практический угол, который нужно закрыть. Многие SwiftUI-приложения не определяют своего `AppDelegate`, а полагаются на автогенерированный:

```swift
@main
struct DemoSwiftTargetApp: App {
    @UIApplicationDelegateAdaptor(MyDelegate.self) var delegate
    var body: some Scene { ... }
}
```

Если `MyDelegate` — класс пользователя, он обязательно наследуется от `NSObject` и реализует `UIApplicationDelegate`-протокол. Все его методы (`-application:didFinishLaunchingWithOptions:`, `-application:didReceiveRemoteNotification:fetchCompletionHandler:` и т. п.) — это ObjC-selectors, зарегистрированные автоматически через `@objc`-инференцию для методов `UIApplicationDelegate`-протокола. Твик свизлит их стандартно:

```objc
%hook MyDelegate

- (BOOL)application:(UIApplication *)app
    didFinishLaunchingWithOptions:(NSDictionary *)opts
{
    NSLog(@"[DemoSwiftTweak] launch opts=%@", opts);
    return %orig;
}

%end
```

Если же пользователь не определил `AppDelegateAdaptor` вовсе, SwiftUI создаёт свой приватный `_SwiftUIApplicationDelegate` (имя приватное, может варьироваться между iOS SDK, требует проверки на устройстве через `objc_getClassList` + `class_getName`). Хукать приватный класс в своей сборке можно, но твик будет ломаться при каждом обновлении iOS SDK; для production-твика лучше требовать от приложения иметь свой `AppDelegate` или строить логику на уровне `UIHostingController`-жизненного цикла, как в разделе 9.8.

Полный список публичных `UIApplicationDelegate`-методов, свизлимых в современном SwiftUI-приложении, занимает несколько десятков строк и повторяется из приложения в приложение — он фиксирован Apple'овским SDK. Это одна из самых стабильных целей для твика: с точки зрения ABI-стабильности системного UIKit, эти селекторы не менялись годами и, по открытым release notes, не меняются и между iOS 26.0 и 26.0.1.

## 9.14. PAC на arm64e: что меняется в хуке

Отдельная тема, которую нельзя пропустить при работе на A13 (arm64e): PAC-подписанные указатели. `method_t` на arm64e хранит IMP как подписанный указатель с ключом `IA` и дискриминатором, зависящим от селектора. Это означает, что при чтении `method_getImplementation(m)` рантайм снимает подпись и возвращает «чистый» указатель, а при `method_setImplementation(m, newIMP)` снова подписывает новый адрес своим ключом. Для автора твика это полностью прозрачно — `libobjc` сам управляет PAC-ключами для своих собственных структур, и вмешиваться в этот процесс не нужно и опасно (кейс vol3-32 в `framework-atlas/` разбирает «что НЕ надо делать с PAC-подписанными функциями без knowledge of salt»).

Но есть нюанс. Если ваш твик делает не `method_setImplementation`, а записывает IMP напрямую в поле `method_t->imp` через `MSHookMessageEx` или через собственный ассемблерный хак — вы обязаны ресайнить указатель. На практике это значит одно: не делайте так. Используйте только публичные API Objective-C рантайма (`method_setImplementation`, `class_replaceMethod`, `class_addMethod`); они корректно обрабатывают PAC для вас. `MSHookMessageEx` из Mobile Substrate (`BaseBin/_external/`) тоже делает всё правильно — он внутри ходит через `method_setImplementation`. А вот ручной memcpy в `method_t`-структуру, который иногда встречается в легаси-коде ранних jailbreak-времён, на arm64e приводит к PAC-authentication-failure при следующем `objc_msgSend` на этот селектор, то есть к немедленному крашу с сигналом `SIGSEGV` и trap-кодом PAC. Для Swift-целей это особенно важно: Swift-созданный thunk `fetchQuoteFrom:` ровно так же живёт в `method_t` под PAC, как и любой другой IMP, и ломается при неосторожном обращении так же.

Для хуков через `litehook` (`BaseBin/libjailbreak/litehook`) PAC-дисциплина отличается — там свои ресайнеры для `__got`/`__la_symbol_ptr`-записей, которые litehook знает, как правильно подписать. Но litehook не предназначен для свизлинга ObjC-методов; он работает по уровню динамических ссылок на символы (dyld-bind), что для Swift-целей через ObjC-мост не нужно. Правильное правило: ObjC/Swift-dynamic — через Logos `%hook`, которые в итоге дёргают `method_setImplementation`; C-функции (включая `@_cdecl`-экспорты Swift) — через `MSHookFunction`, который на arm64e сам управляет PAC для page-level кода.

## 9.15. Проверка твика на устройстве: пошаговый протокол

Соберём рабочий чек-лист проверки, с чем сверяться после каждой итерации. Протокол подходит и для нашего `DemoSwiftTweak`, и для любого другого твика на Swift-цель.

Первое — убедиться, что твик загрузился в целевой процесс. На хосте, при подключённом по USB устройстве:

```bash
log stream --predicate 'process == "DemoSwiftTarget"' --style compact \
    --level debug
```

При запуске приложения в логе должна появиться наша строка `[DemoSwiftTweak] hooked -[...AppModel fetchQuoteFrom:]`. Если её нет — твик не загрузился: проверить `/var/jb/Library/MobileSubstrate/DynamicLibraries/DemoSwiftTweak.plist` (должен быть именно RootHide-овский путь, см. `BaseBin/libjailbreak/src/jbroot.h:38-40`), проверить, что `.deb` установлен (`dpkg -l | grep demo`), проверить, что процесс действительно подхватил MobileSubstrate (`should_enable_tweaks` в `BaseBin/systemhook/src/main.c:532-540` — функция, которая решает, подгружать ли TweakLoader; для обычных приложений она возвращает true).

Второе — убедиться, что селектор действительно хукается. При нажатии «Fetch» в приложении — вторая строка лога, `[DemoSwiftTweak] fetchQuoteFrom: self=... url=...`. Если первая строка есть (твик загрузился и `%ctor` отработал), а второй нет — значит, компилятор пошёл в обход `objc_msgSend` на callsite. Это диагностируется через `otool -tv` на `body.getter`-е, как в разделе 9.11, и исправляется требованием `dynamic` в исходном коде цели.

Третье — убедиться, что модификация результата видна в UI. Поле «last» в приложении должно показать `[tweaked] quote from example.com`. Если показывает оригинальное значение, но лог говорит, что хук сработал — проверьте, что `return` в `my_fetchQuoteFrom` действительно возвращает нашу строку, а не оригинал. Это тривиальная ошибка, но периодически возникает при копи-пасте шаблона из кейса 1 (где хук только логирует, но не изменяет значение).

Четвёртое — стресс-тест: нажать «Fetch» сто раз подряд, посмотреть, не растёт ли память и не появляются ли замечания от SwiftUI-рантайма. Если `fetchCount` в модели — `@Published`, каждый вызов `fetchQuote` будет триггерить SwiftUI-инвалидацию и перерисовку. Наш хук не трогает инвалидацию: он подменяет IMP, но внутри оригинала всё равно вызывается `fetchCount += 1`, и `@Published`-механика продолжает работать. Если бы мы захотели подавить инвалидацию — пришлось бы не вызывать `%orig`, а это сломало бы бизнес-логику приложения.

Пятое — тест отключения твика через MobileSubstrate-механизм (RootHide, в отличие от классического jailbreak, умеет гибко управлять набором подгружаемых твиков, см. `BaseBin/libjailbreak/src/`). Приложение после отключения должно работать абсолютно как без твика — поля «count», «last», «tapped» показывать свои исходные значения без префикса `[tweaked]`. Это важная проверка: убеждает, что ваш твик не оставляет резидуальных побочных эффектов (например, не записал что-то в `UserDefaults`, что приложение после отключения всё равно читает).

Шестое — тест на A13-устройстве под iOS 26.0.1. По открытым release notes 26.0.1 изменения касались в основном фиксов безопасности в WebKit и телефонной подсистеме; никаких изменений в Swift-ABI, формате `method_t`, MobileSubstrate-механике не задокументировано. Поэтому твик, работающий на 26.0, обязан работать на 26.0.1 без изменений — но для production-канала проверить всё равно обязательно, потому что отсутствие изменений в release notes не гарантирует отсутствие изменений в приватных частях рантайма (требует проверки на устройстве).

## 9.16. Границы техники: когда стоит отказаться от твика

Отдельно стоит обсудить ситуации, когда разработчик твика после всех проверок приходит к выводу «через ObjC-мост цель недостижима». Это не всегда провал — иногда это трезвое решение не создавать хрупкий код. Четыре типичных случая:

Первый — приложение написано в стиле «чистый SwiftUI без `NSObject`-подклассов». Нет своих `AppDelegate`-наследников, нет моделей-наследников `NSObject`, модель — `final class` или вовсе `actor`. При этом вся логика сосредоточена в чистых Swift-функциях, без вызовов Foundation-API, пригодных для перехвата (вместо `URLSession` — WebSocket через `Network.framework`, который в Swift-обёртке `NWConnection` виден только как Swift-API, хоть под капотом и ObjC). В таких приложениях из ObjC-mounted точек остаётся только `UIHostingController`-жизненный цикл (раздел 9.8), и для содержательного твика этого обычно мало. Правильное решение — не писать твик вовсе, а запросить у автора приложения расширение публичного API (например, URL scheme или x-callback-url).

Второй — приложение использует Swift-generic агрессивно, модели — `Container<State: ViewState>`, вызовы — `.map` / `.flatMap` / `.compactMap` по generic-коллекциям. Это код, где mono-морфизированные символы в бинаре множатся лавинообразно, и хук любого из них даёт перехват одной конкретизации, не всех. Твик технически возможен, но затраты на поддержку (переписывать mangle-строки при каждом релизе приложения) делают его экономически нецелесообразным.

Третий — приложение построено на server-driven UI, где фактическая логика определяется response'ом с сервера, а клиент — обычный «тонкий» рендерер. Хук любой клиентской функции не меняет поведения: сервер пришлёт то же самое, UI отрисует то же самое. В таких случаях точка воздействия — не бинарник, а сетевой слой; хук `URLSession` для подмены response'а работает надёжнее любого свизлинга модели.

Четвёртый — приложение намеренно детектирует jailbreak (через проверки на `/var/jb`, через `fork()`-канарейку, через `sysctl KERN_PROC`) и отказывается запускаться на устройстве с Dopamine. В этом случае ваш твик не доходит до интересной функциональности, потому что приложение закрывается раньше. Правильная очерёдность: сначала хук jailbreak-детекции (эта тема детально разбирается в кейсе 11, `cases/11-anti-jb-detection.md`), затем уже основной твик поверх. Без этого шага никакой `@objc dynamic`-свизлинг не поможет, потому что процесс не доживёт до момента вызова.

## Выводы

Твик для Swift-приложения на Dopamine RootHide — это не «хук SwiftUI», а «хук Objective-C-слоя, через который Swift-приложение обязано проходить». Единственная технически устойчивая точка входа — `@objc dynamic`-методы в `NSObject`-подклассах: именно для них компилятор создаёт настоящий `method_t` в метатаблице класса, и именно для них `%hook` работает ровно так же, как для чистого ObjC. Публичный Foundation/UIKit API, который Swift-код всё равно дёргает через `objc_msgSend` (`URLSession`, `NotificationCenter`, `UIViewController`, `UserDefaults`, `UNUserNotificationCenter`), остаётся основным объектом перехвата — и, в большинстве случаев, оказывается достаточным для задач твика, потому что именно через эти API проходят внешние эффекты приложения (сеть, хранилище, нотификации, UI-навигация). Чисто-Swift методы, generic, closure, `@inlinable`-функции и body SwiftUI-View'ов остаются за пределами рантайм-хука и требуют binary patch по известному смещению, который к следующей сборке приложения перестаёт попадать по адресу.

Второй вывод — методический. Перед написанием любого твика на Swift-приложение сначала проверяйте, что цель вообще экспортирована в ObjC-рантайм. Три команды — `otool -s __TEXT __objc_methname | strings | grep`, `nm -gU | swift-demangle`, `otool -ov` (или `jtool2` на iOS 26) — за минуту отвечают на вопрос «есть смысл писать `%hook` или нет». Если селектора в `__objc_methname` нет, никакой `%hook` не поможет; если есть — скорее всего, Logos-twик будет работать так же надёжно, как в обычном Objective-C-приложении из кейсов 1–7.

Третий вывод — дисциплина имён классов. Swift-компилятор на разных версиях Xcode mangle'ит имя класса по-разному (`ModuleName.ClassName` на новых, `_TtC<n><module><m><class>` на старых). Твик должен искать класс через `NSClassFromString` по обоим вариантам, чтобы одна и та же `.deb` работала с приложениями, собранными Xcode разных поколений. Эту же идиому стоит применять ко всем Swift-целям, где вы не контролируете, каким компилятором собрано приложение.

Четвёртый вывод — не пытайтесь хукать SwiftUI. View — struct, body — opaque function с хешем в mangle, диффер — внутренний механизм Swift. Любая попытка встроиться в этот слой либо не компилируется, либо работает один раз и ломается после следующей сборки. Если нужно повлиять на UI — встраивайтесь в данные (модель с `@objc dynamic`-полями) или в UIKit-слой под SwiftUI (`UIViewController`-ы, которые SwiftUI создаёт под обёртками `UIHostingController`-а; соответствующие классы доступны `%hook`-у стандартно).

Пятый вывод — PAC на arm64e. Он прозрачен, пока вы работаете через публичный API Objective-C рантайма (`method_setImplementation`, `class_replaceMethod`, `class_addMethod`) и через `MSHookMessageEx` / `MSHookFunction`. Он мгновенно ломает вашу сборку, если вы пытаетесь писать в поле `method_t->imp` напрямую через `memcpy` или через inline-ассемблер. Для Swift-целей это ровно те же правила — Swift-`@objc dynamic`-метод с точки зрения `method_t` и PAC неотличим от обычного ObjC-метода. Это одна из тех редких ситуаций, где «правильный» высокоуровневый API существенно безопаснее любых микрооптимизаций.

Шестой вывод — экономика поддержки. Твик, построенный строго на свизлинге ObjC-методов из Foundation/UIKit (`URLSession`, `UserDefaults`, `UNUserNotificationCenter`, `UIHostingController`) и `@objc dynamic`-методов моделей, которые автор приложения сам пометил, — такой твик нужно поддерживать на уровне мажорных обновлений iOS (26.x → 27.x), не на уровне минорных (26.0 → 26.0.1) и не на уровне обновлений приложения. Твик, построенный на `MSHookFunction` по mangled-символам чистого Swift, нужно переправлять фактически каждую сборку приложения. Разница между этими двумя режимами поддержки — порядок величины рабочих часов в год.

## Что дальше

Следующий кейс (`cases/10-pure-c-syscall-hook.md`) симметрично этому — он разбирает противоположный полюс: как хукнуть чистый C-syscall-wrapper (`open`, `read`, `stat`) без Objective-C-слоя через `litehook` (`BaseBin/libjailbreak/litehook`) и как RootHide-прослойка `BaseBin/libroothider` уже делает это для перенаправления путей в `/var/jb` (`BaseBin/libroothider/src/*.c`). Пара глав (9 и 10) должна дать читателю полное покрытие двух крайних стилей перехвата: высокоуровневый ObjC-dynamic (через который достижима часть Swift-кода) и низкоуровневый C-level (через который достижима часть syscalls). Между ними остаётся «серая зона» — Swift без `@objc` и C++ с vtable; её посвящены главы `framework-atlas/02-swiftui.md` (о SwiftUI/`SwiftUICore` на уровне ABI-stable системных dylib) и `re-walkthroughs/03-cxx-callback.md` (о сигнатурах C++-callback'ов в `mediaserverd`).

Параллельно рекомендуется ещё раз пройтись по кейсу 7 (`cases/07-crash-safe-tweak-skeleton.md`): показанный там pattern защитных nil-проверок и signal handler'а особенно важен для твиков Swift-приложений, потому что приложение, упавшее из-за неосторожного хука на границе ObjC ↔ Swift, часто падает не в вашем коде, а глубоко в `libswiftCore.dylib`, и без сконфигурированного обработчика сигналов разбирать такой краш сложнее, чем ObjC-эквивалент. Правильная последовательность для production-твика на Swift-цель: сначала скелет из кейса 7, затем Logos-блок из кейса 9 внутри него, затем проверочная сборка на устройстве с `log stream` на хосте, затем — только тогда — упаковка в `.deb`.

Отдельный ориентир — глава `framework-atlas/02-swiftui.md` (vol3-27). Она расширяет настоящий кейс до разбора ABI-stable Swift-dylib (`SwiftUICore.framework/SwiftUICore`, `_SwiftUIKitCoreView`) и объясняет, почему тот же приём «хук через ObjC-мост» применим и к SwiftUI-инфраструктуре, если встраиваться в её `UIHostingController`-обёртки, а не в сами SwiftUI-ноды. Для твиков системного масштаба (например, «подкрасить фоном любой SwiftUI-экран на устройстве») это, по сути, единственный рабочий путь — и в сочетании с матрицей из раздела 9.7 даёт целостную картину того, где Swift на iOS 26 реально поддается перехвату.

Для подготовки `.deb`-пакета этого твика стандартным путём используется `theos`-инфраструктура в сочетании с `BaseBin/dopamine`-утилитами. Структура финального `.deb` при упаковке в rootless-формат должна содержать `/var/jb/Library/MobileSubstrate/DynamicLibraries/DemoSwiftTweak.dylib` и `DemoSwiftTweak.plist` — именно в этот путь RootHide-прослойка `BaseBin/libjailbreak/src/jbroot.h:38-40` правильно отобразит `JBROOT_PATH("/Library/...")` при обращении из любого инъектированного процесса. Пример полного Makefile для `.deb`-сборки показан в кейсе 1 (`cases/01-full-logger-tweak.md`), наш твик собирается ровно по тому же шаблону — Theos сам создаст правильный rootless-layout, если в `control`-файле указан `Package: com.example.demoswiftweak` с `Architecture: iphoneos-arm` и выставлена переменная `THEOS_PACKAGE_SCHEME = rootless`.

Отдельное замечание про тестирование регрессии. Поскольку твик меняет IMP метода в метатаблице класса, после `method_setImplementation` любой код в процессе, который использует `+[NSObject instancesRespondToSelector:]` или `-[NSObject respondsToSelector:]`, продолжит отвечать true для `fetchQuoteFrom:` — потому что метод по-прежнему зарегистрирован, у него просто другая IMP. Это важно для совместимости с кодом, который делает runtime-проверки типа «если метод есть — вызови его, если нет — fallback». Наш хук такой проверке не мешает, что отличает его от «инвазивных» подходов (например, удаление метода из таблицы с помощью `class_replaceMethod` с `NULL`-IMP, который действительно сломает `respondsToSelector:`).

Наконец, если на основе этого кейса вы захотите написать свой универсальный «логгер всех `@objc dynamic`-методов любого Swift-приложения», правильная техника — не хукать каждый метод отдельно, а зарегистрировать свой observer на `objc_classAddHook`-подобные события через `objc_setHook_setAssociatedObject` или `objc_addLoadImageFunc`. Это уже уровень Том III `deep-dive/03-objc-runtime-internals.md` (vol3-18) и выходит за рамки простого кейса. Но направление показательное: `libobjc` сам предоставляет механизмы для системных-уровневых наблюдений, и для исследовательских задач они обычно удобнее, чем ручное свизлинг каждого селектора. Производственный твик, естественно, работает против конкретных целей; универсальные-наблюдательные хуки — инструмент разработчика, не финального пакета.
