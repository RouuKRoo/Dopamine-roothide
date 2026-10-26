# Глава II.1. SpringBoard: домашний экран, иконки, папки

В этой главе разбирается главный пользовательский процесс iOS — SpringBoard — с точки зрения автора твиков под Dopamine RootHide на A12/A13 (iOS 26.0 и 26.0.1, PPL, подтверждено в `Application/Dopamine/Jailbreak/DOEnvironmentManager.m:254`). Мы ответим на три практических вопроса. Какой именно процесс запускает «домашний экран» и из каких фреймворков собран. Какие точки в нём можно трогать публичным API, а какие требуют приватных заголовков и хуков. Как сам Dopamine уже встроен в SpringBoard — это материал для подражания: его хуки написаны на тех же `Logos`-примитивах, что и твик пользователя, и опираются на тот же `TweakLoader`, который в вашем проекте будет грузить ваш `.dylib`. Повторяемость будет такой: каждое утверждение о коде Dopamine сопровождается ссылкой «путь/файл:строка», и каждое утверждение о приватных API iOS явно помечено как подтверждённое или требующее проверки на устройстве.

## 1.1. Процесс и бинарник

На iOS SpringBoard — это не демон и не framework. Это полноценное UIKit-приложение, живущее по пути `/System/Library/CoreServices/SpringBoard.app`, с главным исполняемым файлом `/System/Library/CoreServices/SpringBoard.app/SpringBoard`. Этот путь — не предположение: Dopamine напрямую сравнивает `argv[0]` именно с ним в двух местах. В `BaseBin/systemhook/src/main.c:492` — чтобы в процессе SpringBoard подгрузить `roothidehooks.dylib`. В `BaseBin/launchdhook/src/jbserver/jbdomain_systemwide.c:325` — чтобы после перезапуска SpringBoard дополнительно убить `kbd` (иначе ломаются AutoFill и клавиатура, см. комментарий там же, строки 322–337).

```c
if (!strcmp(gExecutablePath, "/usr/sbin/cfprefsd") ||
    !strcmp(gExecutablePath, "/System/Library/CoreServices/SpringBoard.app/SpringBoard") ||
    !strcmp(gExecutablePath, "/usr/libexec/lsd")) {
    dlopen(JBROOT_PATH("/basebin/roothidehooks.dylib"), RTLD_NOW);
}
```

Ключевые строки: это код в конструкторе `systemhook`, который выполняется в каждом процессе после отработки Mach-O-загрузчика (загрузчик `dyld`), но до функции `main`. По имени исполняемого файла — а не по `bundle id`, не по `launchd label` — здесь принимается решение, в какой из вспомогательных хук-библиотек Dopamine будет исполнен его собственный `%ctor`. Это важное наблюдение для автора твика: ваш `.dylib`, если он должен сработать именно в SpringBoard, тоже будет опознаваться по пути `argv[0]`, а не по чему-либо ещё — это удобно, потому что от перезагрузки имя файла не меняется.

SpringBoard — долгоживущий процесс. Он запускается `launchd`-ом как `com.apple.SpringBoard`, умирает редко и обычно только после `respring` (перезапуска пользовательского пространства). После каждого перезапуска он снова проходит через `systemhook` и, следовательно, через ваш хук, если тот установлен корректно.

## 1.2. Карта фреймворков

Клиентский код домашнего экрана распределён между несколькими private frameworks. Таблица ниже показывает, какой фреймворк за что отвечает, где лежит, и какую примерно роль он играет. Пути подтверждены как общеизвестные для современных версий iOS; наличие именно таких имён файлов на устройстве iOS 26.0/26.0.1 — утверждение, которое читатель обязан проверить сам: см. раздел 1.7 про процедуру с `class-dump` на устройстве.

| Framework | Расположение (private) | За что отвечает |
| --- | --- | --- |
| `SpringBoardFoundation` | `/System/Library/PrivateFrameworks/SpringBoardFoundation.framework` | фундаментальные типы SpringBoard, базовые классы иконок и моделей |
| `SpringBoardServices` | `/System/Library/PrivateFrameworks/SpringBoardServices.framework` | клиентский слой для IPC с SpringBoard-ом (XPC/mach), опрос состояния |
| `SpringBoardUI` | `/System/Library/PrivateFrameworks/SpringBoardUI.framework` | UI-примитивы, доли контроллеров, разделяемые с другими процессами |
| `SBAssistantInterface` | `/System/Library/PrivateFrameworks/...` | интеграция с Siri и Assistant UI (поверхностный интерфейс для сервиса) |
| `SBLockScreenBuildVersion` | артефакт сборки | версионирование lock screen; детали не подтверждены исходниками Dopamine |
| `FrontBoard` | `/System/Library/PrivateFrameworks/FrontBoard.framework` | фронтенд `BoardServices` для `SpringBoard`-процесса — классы `FBSApplicationLibrary`, `FBSystemService` и т. п. |
| `BoardServices` | `/System/Library/PrivateFrameworks/BoardServices.framework` | низкоуровневый слой IPC между frontboardd-подобными процессами и клиентскими приложениями, базовый класс `BSServiceConnection` |

Фреймворки `FrontBoard` и `BoardServices` особенно важны: именно в них живут классы `FBSApplicationLibrary` и `FBSystemService`, которые Dopamine перехватывает (см. раздел 1.4). Класс `BSServiceConnection` используется для получения PID вызывающего клиента — это стандартный способ для SpringBoard понять, «кто меня дёргает» (`BaseBin/roothidehooks/src/springboard.x:94`).

Что отсюда следует для автора твика. Если задача — изменить вид иконки, значок бэйджа, папку, dock, страницы — вам понадобятся классы из `SpringBoardFoundation`/`SpringBoardUI` (и, в более поздних версиях iOS, из `SpringBoardHome.framework`, которого на старых iOS не было; имя подтверждено только по сторонним source-dump'ам). Если задача — повлиять на запуск сторонних приложений, порядок открытия, их метаданные — это уровень `FBSApplicationLibrary`/`FBSystemService`. Если задача — реагировать на блокировку/разблокировку, перезапуск, фоновые события — это `SBAssistant*`, `SBLockScreen*` и демоны `assertiond`/`runningboardd`.

## 1.3. Публичное против приватного

На пользовательской стороне iOS Apple предоставляет только две формально публичных «ручки» к SpringBoard:

- Подсистема `UIApplication` (UIKit): открыть URL (`-[UIApplication openURL:options:completionHandler:]`), поставить бэйдж приложения, запросить фоновую задачу. Всё это доступно из обычного app sandbox без всяких хуков и не требует Dopamine.
- Подсистема `Launch Services` (`LSApplicationWorkspace`, `LSApplicationProxy`): перечислить приложения, узнать Bundle ID, получить плейсхолдер иконки. Часть API формально private, но исторически вызывалась и без прав — именно поэтому Dopamine проходится по ней дополнительно: `BaseBin/roothidehooks/src/lsd.x` блокирует ряд методов `lsd`, чтобы «засекреченные» приложения не раскрывались чувствительным клиентам.

Для настоящей работы с домашним экраном этих API мало. Чтобы перерисовать иконку, перехватить тап по иконке, добавить новый контроллер страницы, нужен метод из приватного заголовка. В сообществе ходят следующие общеизвестные имена классов (их наличие и прежняя сигнатура подтверждены для iOS 14–16; **для iOS 26.0/26.0.1 — требует проверки на устройстве через `class-dump`**):

- `SBIconView` — визуальное представление иконки (ячейка CoreAnimation поверх `UIView`), селекторы вроде `-setIcon:`, `-icon`, `-setHighlighted:`.
- `SBIconController` — синглтон, управляющий страницами, dock, папками.
- `SBHomeScreenViewController` — контроллер самого экрана «Домой» (на последних iOS большая часть логики переехала в вспомогательные классы `SBH*`).
- `SBIconModel` — модель данных: массив иконок, папок, положение, бэйджи.
- `SBFolderController`, `SBFolder` — папки.

Эти имена **в репозитории Dopamine напрямую не фигурируют**, что проверяется поиском `grep -r "SBIcon" BaseBin Application`. Поэтому для книги оговорка жёсткая: приведённый список — это общеизвестные имена из сторонних дампов (`iOSOnPC/Headers`, `theos/headers`), и перед первым хуком их обязательно надо верифицировать `class-dump`-ом на целевой версии. Селекторы и даже имена классов между major-версиями iOS мигрируют: Apple периодически рефакторит иконочный слой, перенося функционал в новые `SBH*`-классы.

## 1.4. Как Dopamine встраивается в SpringBoard

Dopamine делает в SpringBoard ровно то, ради чего он туда идёт: прячет jailbreak-данные от сторонних приложений, перенаправляет служебные пути приложений в `jbroot`, фильтрует запросы к Launch Services. Инъекция происходит из двух мест (разные пакетные схемы: `rootless` и `roothide`), поэтому важно сначала развести, какая версия Dopamine что делает.

```mermaid
flowchart LR
    A[launchd] --> B[SpringBoard]
    B --> C[systemhook.dylib<br>DYLD_INSERT_LIBRARIES]
    C --> D[Проверка argv[0]<br>systemhook/main.c:492]
    D --> E[dlopen roothidehooks.dylib<br>или rootlesshooks.dylib]
    E --> F[%ctor в main.x]
    F --> G[sbInit / springboardInit]
    G --> H[%init  для %hook<br>XBSnapshotContainerIdentity,<br>FBSApplicationLibrary,<br>FBSystemService, fcntl]
```

Что здесь показано: SpringBoard поднимается `launchd`-ом, в его адресное пространство попадает `systemhook` (как `DYLD_INSERT_LIBRARIES`-библиотека), конструктор `systemhook` на основе `argv[0]` подгружает одну из двух хук-библиотек — `roothidehooks.dylib` в схеме `roothide` или `rootlesshooks.dylib` в схеме `rootless`. У каждой из них есть собственный `%ctor` (`BaseBin/roothidehooks/src/main.x` или `BaseBin/rootlesshooks/main.x`), который по имени процесса вызывает соответствующую функцию инициализации. Для SpringBoard это `sbInit()` (roothide) или `springboardInit()` (rootless). Эти функции, в свою очередь, активируют `%hook`-блоки.

Ниже разбираются все хуки по порядку.

### 1.4.1. Rootless-схема: `BaseBin/rootlesshooks/SpringBoard.x`

Файл короткий — 72 строки, два хука:

Хук первый — на класс `XBSnapshotContainerIdentity`, метод `-snapshotContainerPath` (`BaseBin/rootlesshooks/SpringBoard.x:30-37`):

```objc
%hook XBSnapshotContainerIdentity
- (NSString *)snapshotContainerPath
{
    NSString *path = %orig;
    if([path hasPrefix:@"/var/mobile/Library/SplashBoard/Snapshots/"] &&
       ![self.bundleIdentifier hasPrefix:@"com.apple."]) {
        return JBROOT_PATH_NSSTRING(path);
    }
    return path;
}
%end
```

Ключ: `%orig` возвращает исходный путь, который iOS выбрала для сохранения снапшота переключателя задач. Если это путь внутри `/var/mobile/Library/SplashBoard/Snapshots/` и приложение не системное (`com.apple.*`), путь перенаправляется в `jbroot` — физически файл поедет не в системную `SplashBoard`-директорию, а в её зеркало внутри джейлбрейкованного корня. Иначе — поведение штатное, снапшот Apple сохраняется там, где Apple предполагает.

Хук второй — на С-функцию `fcntl` (`BaseBin/rootlesshooks/SpringBoard.x:41-66`):

```objc
%hookf(int, fcntl, int fildes, int cmd, ...) {
    if (cmd == F_SETPROTECTIONCLASS) {
        char filePath[PATH_MAX];
        if (fcntl(fildes, F_GETPATH, filePath) != -1) {
            if (string_has_prefix(filePath,
                 JBROOT_PATH_CSTRING("/var/mobile/Library/SplashBoard/Snapshots"))) {
                return 0;
            }
        }
    }
    /* ... прокидывание остальных команд через va_list ... */
    return %orig(fildes, cmd, arg1, ...);
}
```

Разбор. Это не Objective-C-хук, а `%hookf` — Logos-макрос для перехвата функции C (fishhook-style ребинда через Substrate). Поведение: если SpringBoard пытается выставить защитный класс (Data Protection) на дескриптор, указывающий внутрь нашего зеркала каталога `SplashBoard/Snapshots`, вернуть `0` без вызова оригинала. Комментарий в коде поясняет: `//Skip setting protection class on jailbreak apps, this doesn't work and causes snapshots to not be saved correctly` (строка 45). Это обход известной особенности iOS: системный ключ класса защиты не доступен для файлов на `jbroot`-разделе, и попытка его выставить роняет сохранение снапшота.

### 1.4.2. Roothide-схема: `BaseBin/roothidehooks/src/springboard.x`

Файл шире и важнее — 118 строк, четыре блока. Первый `%hookf fcntl` (`BaseBin/roothidehooks/src/springboard.x:6-31`) логически идентичен rootless-варианту; разница только в проверке пути: вместо префиксного сравнения строки используется `isSubPathOf(filePath, jbroot(...))`, функция из `libroothide`. То есть roothide-схема делает ту же защиту снапшотов, но через свой API маршрутизации пути.

Второй блок — тот же `XBSnapshotContainerIdentity`, но с расширенным условием (`BaseBin/roothidehooks/src/springboard.x:48-57`):

```objc
-(NSString *)snapshotContainerPath {
    NSString* path = %orig;
    if([path hasPrefix:@"/var/mobile/Library/SplashBoard/Snapshots/"] &&
       (![self.bundleIdentifier hasPrefix:@"com.apple."] ||
        is_apple_internal_identifier(self.bundleIdentifier.UTF8String))) {
        NSLog(@"snapshotContainerPath redirect %@ : %@", self.bundleIdentifier, path);
        path = jbroot(path);
    }
    return path;
}
```

Отличие от rootless-версии: условие перенаправления шире — помимо «не `com.apple.*`» перенаправляются и некоторые внутренние Apple-идентификаторы, подпадающие под функцию `is_apple_internal_identifier`. Её список фиксирован в `BaseBin/libroothider/src/common.h:92-96`:

```objc
#define APPLE_INTERNAL_IDENTIFIERS @[\
    @"com.apple.atrun",\
    @"com.apple.kdumpd",\
    @"com.apple.Terminal",\
]
```

Это Apple-инструментальные бандлы, которые пользователь может поставить в режиме «внутренний разработчик». Dopamine обращается с ними как со сторонними: их снапшоты тоже отправляются в `jbroot`-зеркало, чтобы иерархия системных снапшотов не содержала следов их активности.

Третий блок — `%hook FBSApplicationLibrary -applicationInfoForBundleIdentifier:` (`BaseBin/roothidehooks/src/springboard.x:63-87`):

```objc
%hook FBSApplicationLibrary
-(id)applicationInfoForBundleIdentifier:(NSString*)bundleIdentifier
{
    id result = %orig;
    NSURL* executableURL = [result performSelector:@selector(executableURL)];
    NSNumber* tag = objc_getAssociatedObject(bundleIdentifier, kDenyQueryTagKey);
    if(tag && tag.boolValue) {
        if(is_sensitive_app_identifier(bundleIdentifier.UTF8String)) { return nil; }
        if(result && executableURL &&
           isLikelyJailbrokenBundlePath(executableURL.path.fileSystemRepresentation)) {
            return nil;
        }
    }
    return result;
}
%end
```

Разбор. `FBSApplicationLibrary` — класс из `FrontBoard`, через который SpringBoard смотрит, какое приложение по данному Bundle ID установлено (и возвращает информационный объект, обычно именованный `SBApplicationInfo`, но мы его трогаем абстрактно, по `id` + `performSelector:`). Dopamine решает: если текущий запрос помечен как «подозрительный» (associated object `kDenyQueryTagKey`, см. ниже), и либо приложение в списке `SENSITIVE_APP_IDENTIFIERS` (`BaseBin/libroothider/src/common.h:99-105`: Dopamine сам, TrollInstaller, Alfie и так далее), либо исполняемый файл лежит на пути, который `isLikelyJailbrokenBundlePath` считает «джейлбрейковым», — вернуть `nil`, то есть «такого приложения не существует».

Что считает `isLikelyJailbrokenBundlePath` джейлбрейковым, видно в `BaseBin/roothidehooks/src/common.m:10-41`: если файловая система, на которой лежит бандл, не `/` и не `/private/preboot`, и это не «обычное» removable-приложение без TrollStore-маркера — такой путь считается подозрительным. Это компактная эвристика, которая покрывает классические и TrollStore-установки, но не притягивает за уши системные приложения.

Четвёртый блок — `%hook FBSystemService -openApplication:...` (`BaseBin/roothidehooks/src/springboard.x:89-111`). Этот хук перехватывает запросы «открыть приложение по Bundle ID», откуда бы они ни шли. Через `BSServiceConnection currentContext` достаётся PID и Bundle ID вызвавшего (строки 94–101), и далее:

```objc
if(jbclient_blacklist_check_pid(pid)==true) {
    NSLog(@"openApplication deny request from %@", _bundleID);
    objc_setAssociatedObject(bundleIdentifier, kDenyQueryTagKey, @YES,
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
return %orig;
```

Это PID-tracker, о котором мы предупреждали. `jbclient_blacklist_check_pid` (`BaseBin/libjailbreak/src/jbclient_roothide.c:110-128`) посылает XPC-запрос `JBS_ROOTHIDE_BLACKLIST_CHECK` джейлбрейк-демону и получает `blacklisted: true/false`. Если PID вызывающего в чёрном списке (банковское приложение, антиджейл-клиент), Dopamine не отменяет вызов — он только помечает конкретный `bundleIdentifier` (объект `NSString`) ассоциативным тегом «отказывать в последующих запросах». Отказ состоится позже, в `applicationInfoForBundleIdentifier:` третьего блока: вернётся `nil`, и защищённое приложение «не увидит» наличие Dopamine в системе.

Приём элегантен: Dopamine не атакует напрямую обнаружение, а делает слой `FBSApplicationLibrary`/`FBSystemService` «забывчивым» ровно по тем приложениям, которые должны остаться невидимыми ровно для тех клиентов, которых надо обмануть.

## 1.5. Таблица «что надо автору твика → куда вмешиваться → как проверить»

| Задача | Куда вмешиваться | Как проверить на целевой версии |
| --- | --- | --- |
| Поменять изображение иконки на лету | `%hook SBIconView` или `%hook SBIconImageView`, метод отрисовки иконки | `class-dump` SpringBoardFoundation/SpringBoardHome, найти селекторы `-setIcon:`, `-iconImage`, `-updateIconImage*` |
| Добавить бэйдж/счётчик поверх иконки | `%hook SBIconView` + добавить `UILabel` как subview; либо настоящая модель — через `SBIcon`-метод для бэйджа | подтвердить селектор `-badgeValue` или аналогичный `class-dump`-ом |
| Перехватить тап по иконке | `%hook SBIconController -iconTapped:` или современный аналог | подтвердить селектор, затем проверить, что `respring` действительно отрабатывает ваш `%ctor` |
| Запретить открытие конкретного приложения | `%hook FBSystemService -openApplication:withOptions:originator:requestID:completion:` (видно по `BaseBin/roothidehooks/src/springboard.x:90`) | селектор уже подтверждён исходниками Dopamine; аргументный список подтверждён на 26.0 там же |
| Спрятать приложение из списка | `%hook FBSApplicationLibrary -applicationInfoForBundleIdentifier:` (`springboard.x:64`); доп. — скрыть из `LSApplicationWorkspace` в lsd | подтверждено исходниками, см. раздел 1.4.2 |
| Поменять фон домашнего экрана | скорее `SBWallpaperController` / `BBWallpaperServer`, не SpringBoard как таковой | **требует проверки на устройстве**, в репозитории Dopamine нет |
| Поменять верстку страниц, dock | `SBIconListView`, `SBDockView`, `SBRootFolderView` или их `SBH*`-наследники | **требует проверки на устройстве через `class-dump`** |
| Залогировать все `openApplication` | как Dopamine: `%hook FBSystemService -openApplication:…`, читать PID из `BSServiceConnection currentContext` | pattern из `springboard.x:94-101` работает «как есть» |
| Сохранить снапшот не туда | `%hook XBSnapshotContainerIdentity -snapshotContainerPath` | подтверждён, см. `rootlesshooks/SpringBoard.x:30`, `roothidehooks/src/springboard.x:48` |

Таблица умышленно короткая и честная: шесть строк — это код, который можно писать прямо сейчас по готовым примерам из Dopamine; три нижние — то, что требует самостоятельного `class-dump`-а на устройстве с iOS 26.0 или 26.0.1.

## 1.6. Правила хорошего тона для хуков в SpringBoard

Поскольку SpringBoard — не ваш процесс, а системный, несколько правил, которые видно по архитектуре Dopamine и которые стоит соблюдать.

Первое — всегда ставьте `%init()` явно и внутри своей функции. В Dopamine это делается в `sbInit()`/`springboardInit()`, которые вызываются из `%ctor` в `main.x` только если `getProcessName()` вернул `"SpringBoard"` (`BaseBin/roothidehooks/src/main.x:38-41`, `BaseBin/rootlesshooks/main.x:28-31`). Это защита от того, что ваша библиотека случайно получит `%ctor` в другом процессе и попробует встать на классы, которых в нём нет. Для автора твика с одним `.dylib`, который загружается только в SpringBoard, можно обойтись одним `%ctor { %init(); }` — но проверять `getProcessName()` всё равно полезно, если один и тот же код используется в нескольких процессах.

Второе — пользуйтесь `objc_getAssociatedObject`/`objc_setAssociatedObject` для прокидывания состояния между хуками (`BaseBin/roothidehooks/src/springboard.x:70, 106`). Это стандартный Objective-C-механизм и единственный безопасный способ пометить «именно этот объект обрабатывать особо», не трогая внутренние поля чужих классов и не рискуя сломать PAC-подпись `isa` на arm64e.

Третье — не вызывайте приватные селекторы напрямую, если они могли переехать между major-версиями. Dopamine делает это осознанно: `performSelector:@selector(executableURL)` в `applicationInfoForBundleIdentifier:` (`springboard.x:67`) — это способ достать поле у объекта, тип которого формально `id` (`SBApplicationInfo` в Objective-C-типизации). Если на iOS 26 селектор по какой-либо причине пропал, `performSelector:` вернёт `nil`, а не упадёт на этапе компиляции с `[result executableURL]`. Это уклонение от жёсткой зависимости от хедеров — паттерн, который имеет смысл перенять. Проверять существование селектора лучше через `respondsToSelector:` до вызова; Dopamine в приведённом месте этого не делает, потому что заранее знает, что селектор был на всех iOS, где эта схема запускается.

Четвёртое — помните, что крэш SpringBoard — это крэш всей системы (пользователь увидит `respring`, потеряет открытые приложения в task switcher, разблокированные Face ID-сессии). Отладка должна быть щадящей: в репо Dopamine логи завёрнуты в `NSLog(...)`, который в релизной сборке превращён в пустой макрос (`#define NSLog(args...)` в `BaseBin/roothidehooks/src/main.x:7-9`). Это паттерн, который стоит использовать в своём коде: единая обёртка над логированием, которую легко вычистить в финальной сборке.

## 1.7. Как найти реальный селектор на iOS 26.0

В главе II.0 (о Theos и сборке) предполагается, что вы уже умеете собрать `.deb` под целевую версию. Теперь — процедура для поиска имени и подписи приватного метода, которую нельзя подменить публичной документацией. Процедура описывается без выполнения на реальной iOS 26 (в этой книге мы не можем это сделать) и опирается только на известный тулчейн:

1. Снимите с устройства с iOS 26.0 (не с симулятора!) бинарник целевого фреймворка — SpringBoardFoundation и, если применимо, SpringBoardHome:

```bash
# на устройстве с установленным Dopamine, под root/jbroot
cp /System/Library/PrivateFrameworks/SpringBoardFoundation.framework/SpringBoardFoundation \
   /var/jb/var/mobile/sb_fnd.bin
```

Почему именно устройство, а не симулятор. Симулятор содержит бинарники под x86_64/arm64 для macOS, а не arm64e-slice iOS; слои `SBH*`, PAC-дискриминаторы, список селекторов между симулятором и устройством расходятся. Для разработки под реальный A13 имеет смысл только дамп с живого устройства.

2. Снимите образ dyld_shared_cache (DSC), если бинарник оказался чисто stub-ом (стандартное поведение private frameworks на iOS 15+). Для этого вытащите DSC с устройства (`/System/Library/Caches/com.apple.dyld/dyld_shared_cache_arm64e`) и воспользуйтесь `ipsw` или `dsc_extractor` для извлечения отдельных dylib. Процедура подробно описана в документации проекта `ipsw`; здесь мы ограничиваемся констатацией, что без неё `class-dump` увидит пустой Mach-O.

3. Запустите `class-dump` или `ClassDumpDyld` на извлечённом бинарнике:

```bash
class-dump -H /путь/к/SpringBoardFoundation -o ./headers_sbfnd
# либо
ClassDumpDyld -o ./headers_sbfnd /путь/к/SpringBoardFoundation
```

Что искать. Для перехвата таппа по иконке в хедерах будет что-то вроде `@interface SBIconView ... - (void)iconTapped:(id)sender;` или аналогичный селектор. Если имя класса `SBIconView` отсутствует — ищите наследников: `grep -rE 'IconView|IconImageView' headers_sbfnd/`. На современных iOS SpringBoard активно делится функционалом с `SpringBoardHome.framework`, поэтому дамп его бинарника отдельно даст ещё одну пачку классов.

4. Проверьте сигнатуру метода. Objective-C-сигнатура включает типы аргументов и возвращаемого значения; `class-dump` по возможности их выводит, но для precise-типизации лучше воспользоваться `otool -ov` на том же бинарнике или `nm --defined-only | c++filt`. Для `%hook` на Logos сигнатура должна совпасть; расхождение в типе одного аргумента приведёт либо к warning-у Logos, либо к тихому `calling convention mismatch` на arm64e и, следовательно, к падению при первой же сборке стекового кадра.

5. Проверьте, что селектор действительно вызывается. Для этого в своём `.dylib` сначала не меняйте поведение — только `NSLog(@"-[%@ %@] called", self, NSStringFromSelector(_cmd))` и `return %orig;`. Поставьте пакет, сделайте `respring`, посмотрите `log stream --predicate 'process == "SpringBoard"'` на компьютере через `idevice_syslog` или `log collect`. Если вызов не идёт — выбран не тот класс или не тот селектор, вернитесь к шагу 3.

Эта процедура универсальна: ровно так же находятся селекторы в `FBSApplicationLibrary` (мы разбирали их по готовому коду Dopamine, но если бы хотели пойти дальше — делали бы `class-dump` `FrontBoard.framework`), `BSServiceConnection` (`BoardServices.framework`), `SBAssistantInterface` и прочее.

## 1.8. Разница между iOS 26.0 и 26.0.1

Поддержка обеих версий декларируется в `Application/Dopamine/Exploits/DarkSword/Info.plist` и `Application/Dopamine/Exploits/ClearSword/Info.plist` одним и тем же диапазоном `19.0`–`26.0.1` с перечислением билдов `23B5044l`, `23B5059e`, `23B5064e`. Применительно к коду, разобранному в этой главе — а это два файла, `BaseBin/rootlesshooks/SpringBoard.x` и `BaseBin/roothidehooks/src/springboard.x`, плюс поддерживающий `BaseBin/systemhook/src/main.c:492` и `BaseBin/launchdhook/src/jbserver/jbdomain_systemwide.c:325` — различий между 26.0 и 26.0.1 в этом месте проекта не выявлено. Хуки собираются одной и той же сборкой, активируются одинаково на обеих версиях.

Это утверждение узкое. Оно не говорит, что в самих приватных классах SpringBoard между 26.0 и 26.0.1 нет сдвигов селекторов, иерархий, подписей. Подобные сдвиги между минорными версиями Apple делает регулярно, и единственный способ их увидеть — повторить процедуру из раздела 1.7 на обеих версиях и сравнить дампы. В репозитории Dopamine такого сравнения нет: ему не нужны конкретные иконочные селекторы, он работает слоем выше (`FBS*`, snapshot container, fcntl).

## Выводы

Первое. SpringBoard — это UIKit-приложение, живущее по строгому пути `/System/Library/CoreServices/SpringBoard.app/SpringBoard`; Dopamine опознаёт его по `argv[0]` в двух местах кода (`BaseBin/systemhook/src/main.c:492`, `BaseBin/launchdhome/src/jbserver/jbdomain_systemwide.c:325`). Это же будет точкой опознания и для вашего `.dylib`.

Второе. Клиентский слой SpringBoard распределён между `SpringBoardFoundation`, `SpringBoardUI`, `FrontBoard`, `BoardServices` и ещё несколькими private frameworks. Публичного API (UIKit, Launch Services) недостаточно для реальных изменений в вид/поведение домашнего экрана — нужны приватные классы `SB*`, `SBH*` и явные хуки. Имена `SBIconView`, `SBIconController`, `SBHomeScreenViewController` — общеизвестны, но для iOS 26.0/26.0.1 их существование и сигнатуры **требуют проверки на устройстве через `class-dump`**.

Третье. Сам Dopamine хукает в SpringBoard только то, что относится к его собственной задаче скрытия джейлбрейка и перенаправления снапшотов: `XBSnapshotContainerIdentity.snapshotContainerPath`, `FBSApplicationLibrary.applicationInfoForBundleIdentifier:`, `FBSystemService.openApplication:...`, функция `fcntl` с командой `F_SETPROTECTIONCLASS`. Все четыре перехвата — отличный шаблон для автора твика: короткий `%ctor` → проверка `getProcessName()` → `%init()` в выделенной функции → `%hook`/`%hookf` на минимально необходимых местах.

Четвёртое. PID-tracker в хуке `FBSystemService` (`BaseBin/roothidehooks/src/springboard.x:90-111`) показывает «правильный» способ протащить состояние между вызовами разных методов разных классов — через `objc_setAssociatedObject` на объекте-аргументе. Этот приём снимает необходимость в собственной map-таблице и переживает любые сброс/перезапуск SpringBoard-процесса.

Пятое. В пределах кода, разобранного в этой главе, различий между iOS 26.0 и 26.0.1 не выявлено. Это узкое утверждение: оно не отменяет возможных сдвигов в приватных классах SpringBoard между этими версиями, которые надо проверять `class-dump`-ом отдельно.

## Что дальше

Следующая глава II.2 возьмёт за объект tweak-а не SpringBoard, а конкретное приложение — покажет, как написать `.dylib`, который загружается только в целевом бандле и работает с его `UIViewController`/`UIApplicationDelegate`. Глава II.3 разберёт системные демоны, похожие по роли на `cfprefsd`, `lsd` и `runningboardd` — ровно те, в которые Dopamine вставляет свои `roothidehooks` помимо SpringBoard (`BaseBin/roothidehooks/src/main.x`). Глава I.2 даёт процессное устройство iOS, в контексте которого SpringBoard — лишь одно из многих запускаемых `launchd`-ом UIKit-приложений; глава I.3 задаёт теорию перехватов, на которой построены все `%hook`-блоки, разобранные здесь.
