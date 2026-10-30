# SpringBoard и Home Screen в iOS 26 — по открытым источникам

SpringBoard — главный UI-процесс iOS, хост домашнего экрана и системных UI (lockscreen, Control Center, StatusBar). Эта глава — чего ожидать на iOS 26.

**Оговорка**: конкретные private классы и методы на iOS 26 — требуют верификации на устройстве через class-dump. Текст даёт framework.

## 1. Что заявляет Apple для iOS 26

По открытым источникам (release notes, WWDC presentations) — развитие тем, начатых в iOS 17/18:

- **Interactive widgets** на Home Screen — продолжают эволюционировать.
- **Live Activities** в различных контекстах.
- **Dynamic Island** на устройствах с cutout — расширенные возможности.
- **Lock Screen customization** — больше опций.
- **Control Center** customization — модули перераспределяются.
- **StandBy** mode (iPhone в горизонтальной ориентации) — углубление.

Для iPhone 11 (A13) часть этих features недоступна (нет Dynamic Island — нет cutout). Но изменения в SpringBoard — глобальные.

## 2. Процессы в SpringBoard

На iOS 26 набор ключевых процессов не меняется радикально:

| Процесс | Что делает |
|---|---|
| `SpringBoard` | UI главный |
| `backboardd` | События ввода, haptics, окна |
| `SpringBoardServices` | XPC сервис для других процессов |
| `runningboardd` | Жизненный цикл процессов |

Для автора твика это те же цели, что были на 15-18.

## 3. Что хукает сам Dopamine в SpringBoard

Напомним (глава III.9):

- `BaseBin/rootlesshooks/SpringBoard.x`:
  - `XBSnapshotContainerIdentity.snapshotContainerPath` — snapshot пути jailbreak-приложений.
  - `fcntl F_SETPROTECTIONCLASS` — обход для jbroot snapshots.
- `BaseBin/roothidehooks/src/springboard.x`:
  - RootHide-specific хуки (точный список — в коде).

Эти хуки — **на iOS 26 присутствуют** (без них jailbreak не работает). Если ваш твик хочет вмешаться в snapshot flow — вы будете в цепочке с хуками Dopamine. Не ломайте их.

## 4. Нестабильные private классы

Исторически менялись между minor:

| Класс (гипотетически) | Что делает |
|---|---|
| `SBIconView` | Иконка на home screen |
| `SBIconController` | Контроллер иконок |
| `SBIconListView` | Страница иконок |
| `SBHomeScreenViewController` | Главный VC home screen |
| `SBFolderViewController` | Папки |
| `_SBUITimeFormatter` или подобное | Форматирование времени в статус-баре |
| `SBFLSStatus*` | Lockscreen status |
| `_SBUIAlertController` | Системные alert'ы |
| `SBControlCenterController` | Control Center |

На iOS 26 **имена могут отличаться**. Проверка — обязательно через `objc_copyClassList` probe (глава V.2).

## 5. Процедура верификации на iOS 26

Пустой шаблон probe-твика:

```objc
#import <objc/runtime.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.sbprobe", "classes");

    unsigned int count = 0;
    Class *classes = objc_copyClassList(&count);
    for (unsigned int i = 0; i < count; i++) {
        const char *name = class_getName(classes[i]);
        // Фокусируемся на SB*/_SB* классах — SpringBoard-specific
        if (strncmp(name, "SB", 2) == 0 ||
            strncmp(name, "_SB", 3) == 0 ||
            strncmp(name, "SBF", 3) == 0 ||
            strstr(name, "StatusBar") ||
            strstr(name, "IconView") ||
            strstr(name, "HomeScreen")) {
            os_log(gLog, "class: %{public}s", name);
        }
    }
    free(classes);
    os_log(gLog, "total: %u classes", count);
}
```

Собрать, установить, filter plist `com.apple.springboard`:

```xml
<key>Filter</key><dict><key>Bundles</key><array>
    <string>com.apple.springboard</string>
</array></dict>
```

Запуск + respring + log stream → получите **точный** список SB* классов на вашей iOS 26.

## 6. Таблица «что хукать безопасно»

Относительно безопасные на любой iOS:

| Класс.Метод | Задача |
|---|---|
| `UIScreen.-mainScreen.-brightness` | Яркость |
| `UIApplication.-sharedApplication.-statusBarOrientation` | Ориентация |
| `UIDevice.+currentDevice.-orientation` | Device orientation |
| `UIScreen.-captureDidChangeNotification` | Запись экрана |
| `UIAlertController.-presentOnTopmost` (private) | Системные alert'ы |

Для **SpringBoard-only** функциональности (replacing iconы, скрытие индикаторов) — нестабильные private классы, probe обязателен.

## 7. План отката при SpringBoard-крэше

Повтор критически важного (глава IV.12):

### 7.1. Через SSH

```bash
ssh mobile@<ip>
sudo dpkg -r com.example.yourtweak
killall -9 SpringBoard
```

### 7.2. Safe mode

```bash
touch /var/jb/basebin/.safe_mode
killall -9 SpringBoard
# В safe mode все tweaks выключены (systemhook проверяет этот файл).
```

### 7.3. Через Dopamine.app

На свежей загрузке — Dopamine.app → "Enable Safe Mode" → твики не грузятся.

## 8. Что подтверждено

- **Подтверждено исходниками Dopamine**: хуки в SpringBoard от Dopamine — `BaseBin/rootlesshooks/SpringBoard.x` и `BaseBin/roothidehooks/src/springboard.x`.
- **Подтверждено Apple docs**: публичные UI/screen API в UIKit.
- **Подтверждено практикой**: private SpringBoard классы — нестабильны между versions.
- **Требует проверки на устройстве**: ВСЕ конкретные private имена на iOS 26 — через probe.

## 9. Приоритеты автора твика

Для SpringBoard-tweak на iOS 26:

1. **Защитный код** — `NSClassFromString`, `respondsToSelector:`, `@try`.
2. **Minimal footprint** — хукните как можно меньше.
3. **План отката** готов до установки.
4. **Тестирование на обеих версиях** (26.0, 26.0.1).
5. **Fallback behaviour** — если класс не найден, твик молчит.

## 10. Выводы

Главный тезис: **SpringBoard — рискованная зона**. Public API стабилен; private — нет. Для iOS 26 — probe обязательный шаг, не optional.

Dopamine сам работает в SpringBoard; ваш твик должен сосуществовать с его хуками, не ломая существующий flow.

## 11. См. также

- Глава II.1 — SpringBoard подсистема.
- Глава III.9 — rootlesshooks/roothidehooks.
- Глава IV.12 — учебный SpringBoard-твик.
- Кейс 05 — замена иконки.
- Глава V.2 — починка через probe.
