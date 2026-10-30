# Глава IV.3. Выбор процесса и фильтры инъекции

Правильный filter plist — половина успеха твика. Неверно указали процесс — твик не загружается вовсе; слишком широко — ломаете систему или получаете непонятные крэши в случайных местах. В этой главе разберём, как узнать bundle id целевого приложения, какая разница между `Bundles`, `Classes` и `Executables`, и как не хукнуть всю систему случайно.

## 1. Правило номер один — узкий фильтр

Самое важное правило разработки твиков:

> **Начинайте с самого узкого фильтра, который решает задачу.**

Если задача — хукнуть одно приложение, filter — на bundle id этого приложения. Если задача — хукнуть SpringBoard, filter — `com.apple.springboard`. **Никогда** не начинайте с filter'а, который ловит все процессы — вы либо завесите систему, либо получите случайные крэши у приложений, которые не ожидали ваш хук.

Расширение фильтра — всегда поздняя оптимизация, когда вы точно знаете, что ваш хук стабилен.

## 2. Четыре вида фильтра — что когда применять

Напомним из главы IV.2:

| Фильтр | Что проверяет | Когда использовать |
|---|---|---|
| **Bundles** | Bundle identifier приложения | Твики для конкретного приложения или системного процесса с bundle id |
| **Executables** | Имя исполняемого файла (без пути) | Твики для демонов без bundle id |
| **Classes** | Наличие ObjC-класса в адресном пространстве | «Загрузить во всё, что использует UIKit» или «всё, что имеет SBSpringBoard» |
| **CoreFoundationVersion** | Диапазон версий CF | Cutoff по версии iOS |

Правильные комбинации — чаще всего `Bundles` + (опционально) `CoreFoundationVersion`.

## 3. Как узнать bundle id приложения

Четыре способа.

### 3.1. Через SSH на устройстве

Самый точный способ. Залогиниться по SSH, найти .app:

```bash
# Все установленные приложения
ls /var/containers/Bundle/Application/*/*.app

# Для системных приложений
ls /Applications/*.app

# Прочитать Info.plist
plutil -p /var/containers/Bundle/Application/<uuid>/Example.app/Info.plist | grep BundleIdentifier
```

Пример вывода:

```
"CFBundleIdentifier" => "com.example.my-app"
```

### 3.2. Через Settings → General → iPhone Storage

На устройстве откройте Settings → General → iPhone Storage → <приложение>. Внутри будет написан bundle id (на iOS 26 — в разделе «About this app»).

Для системных приложений этот путь не работает — bundle id нужно смотреть через SSH.

### 3.3. Через Mac с подключённым устройством

```bash
ideviceinstaller --list-apps
```

Для этого нужен `libimobiledevice` (из homebrew). Выдаст список bundle id + name всех установленных приложений.

### 3.4. Через `log stream` на хосте

Если устройство подключено и в Console/log stream, при запуске приложения можно увидеть строки с bundle id:

```
default    11:23:45  com.example.my-app[12345]  <message>
```

Иногда — самый быстрый путь, если вы уже отлаживаетесь.

## 4. Bundle ID системных процессов — таблица

Для частых целей держите в голове:

| Процесс | Bundle ID / Executable |
|---|---|
| SpringBoard | `com.apple.springboard` |
| Settings.app | `com.apple.Preferences` |
| Safari | `com.apple.mobilesafari` |
| App Store | `com.apple.AppStore` |
| Messages | `com.apple.MobileSMS` |
| Phone | `com.apple.mobilephone` |
| Maps | `com.apple.Maps` |
| Camera | `com.apple.camera` |
| Photos | `com.apple.mobileslideshow` |
| Music | `com.apple.Music` |
| Mail | `com.apple.mobilemail` |
| Calendar | `com.apple.mobilecal` |
| Reminders | `com.apple.reminders` |
| Weather | `com.apple.weather` |
| Clock | `com.apple.mobiletimer` |
| Calculator | `com.apple.calculator` |
| FaceTime | `com.apple.facetime` |
| Shortcuts | `com.apple.shortcuts` |

Для демонов (у которых нет bundle id) используйте `Executables`:

| Демон | Executable |
|---|---|
| backboardd | `backboardd` |
| cfprefsd | `cfprefsd` |
| installd | `installd` (но помните — не хукать! Глава II.12) |
| lsd | `lsd` |
| runningboardd | `runningboardd` |
| apsd | `apsd` |
| nehelper | `nehelper` |
| usernotificationsd | `usernotificationsd` |
| mediaserverd | `mediaserverd` (но не хукать! Глава II.10) |
| powerd | `powerd` |
| trustd | `trustd` |
| debugserver | `debugserver` |
| logd | `logd` (но на iOS 16+ `systemhook` исключает их — глава III.5) |

## 5. Как твик реагирует на расширения

Расширения приложений (`.appex`) — это **отдельные процессы**. Твик, вставленный в хост-приложение, автоматически в его расширения **не попадает**.

Пример: вы написали твик для WhatsApp. Хост-приложение — `net.whatsapp.WhatsApp`. Но у WhatsApp есть Share Extension (для «Поделиться» из других приложений) с собственным bundle id, например, `net.whatsapp.WhatsApp.ShareExtension`.

Чтобы твик загружался и в расширение, filter plist должен быть:

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>net.whatsapp.WhatsApp</string>
        <string>net.whatsapp.WhatsApp.ShareExtension</string>
        <string>net.whatsapp.WhatsApp.Intents</string>
        <!-- и т. д. — все .appex -->
    </array>
</dict>
```

Чтобы узнать все расширения приложения, через SSH:

```bash
ls /var/containers/Bundle/Application/<uuid>/<App>.app/PlugIns/
```

Пример вывода:

```
ShareExtension.appex/
NotificationServiceExtension.appex/
IntentsExtension.appex/
```

Внутри каждой .appex — свой Info.plist с собственным bundle id.

## 6. Classes-фильтр — когда полезен

Пример применения: твик, который должен работать «во всех приложениях, которые используют UIKit» — то есть во всех пользовательских приложениях.

```xml
<key>Filter</key>
<dict>
    <key>Classes</key>
    <array>
        <string>UIApplication</string>
    </array>
</dict>
```

Разбор:

- `UIApplication` — класс, который есть в любом UIKit-приложении.
- Фильтр ловит все приложения с UIKit, но не ловит демонов (у них нет UIApplication).

**Внимание**: этот фильтр очень широкий. Ваш твик загрузится во все мессенджеры, все соцсети, все игры, все браузеры. Любой баг — и у вас 20 крэшей сразу.

Используйте только если:

- хук **действительно** нужен во всех приложениях (например, глобальный tracer);
- хук **защищён** проверками (например, `self.class != UIApplication` в начале, чтобы отвергать всё, кроме конкретных сценариев);
- хук **минимален** и предсказуем.

## 7. CoreFoundationVersion — cutoff по iOS

Если твик использует API, которых нет в старых iOS, добавьте `CoreFoundationVersion` чтобы не загружаться вовсе:

```xml
<key>CoreFoundationVersion</key>
<array>
    <real>1800.0</real>  <!-- min -->
    <real>9999.0</real>  <!-- max (бесконечность) -->
</array>
```

Пример для «только iOS 15+»:

```xml
<key>CoreFoundationVersion</key>
<array>
    <real>1800.0</real>
    <real>9999.0</real>
</array>
```

Для «только iOS 26.0 и 26.0.1» (если точное значение известно):

```xml
<!-- Пока точное значение CFVersion для iOS 26.0 требует проверки на устройстве -->
<!-- Пример: ~2200.0, но ТРЕБУЕТ ПРОВЕРКИ -->
<key>CoreFoundationVersion</key>
<array>
    <real>2200.0</real>
    <real>2300.0</real>
</array>
```

Чтобы снять актуальное значение с устройства:

```bash
# Через SSH на jailbroken-устройстве
sysctl -n kern.osproductversion    # покажет "26.0" или "26.0.1"
# В коде процесса:
# CFCoreFoundationVersionNumber (глобальная переменная libCoreFoundation)
```

Или через `otool -L` на любом бинарнике — там есть `compatibility version` для CoreFoundation.framework, но это не то же самое, что `CFCoreFoundationVersionNumber`. Самый надёжный путь — запустить short program:

```c
#include <CoreFoundation/CoreFoundation.h>
#include <stdio.h>
int main() {
    printf("CF version: %.1f\n", CFCoreFoundationVersionNumber);
    return 0;
}
```

Собрать и запустить на устройстве.

## 8. Executables — для демонов

Если bundle id нет (демон), используйте `Executables`:

```xml
<key>Filter</key>
<dict>
    <key>Executables</key>
    <array>
        <string>debugserver</string>
    </array>
</dict>
```

Фильтр матчит имя файла (не путь, не bundle). Для `/usr/bin/debugserver` → `debugserver`.

Важные нюансы:

- Фильтр чувствителен к регистру.
- Match — exact match базнейма, не suffix.
- Для процесса, у которого есть и bundle, и executable — Bundles имеет приоритет над Executables (если указаны оба, достаточно одного совпадения).

## 9. Пример: твик для своего тестового приложения

Шагов разработки:

1. Создать собственное iOS-приложение `com.example.testbed` (простая Xcode-проект с одной кнопкой).
2. Собрать и перенести на устройство (через TrollStore или Xcode для jailbreak-dev).
3. Написать твик с filter:

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
</dict>
```

4. В Tweak.x — хуки, которые работают только в этом приложении.

Этот паттерн — главный в этой книге для безопасных экспериментов. Всё, что в учебных проектах глав IV.11–IV.13 — делается именно так.

## 10. Debug + Release filter

Иногда полезно иметь разные filter plist для debug и release:

```make
# В Makefile
ifeq ($(FINALPACKAGE),1)
    FILTER_FILE = HelloTweak.plist
else
    FILTER_FILE = HelloTweak-debug.plist
endif

HelloTweak_FILTER = $(FILTER_FILE)
```

Debug-plist может быть шире (для тестирования), release — узким.

Это редкая оптимизация, но полезная для крупных проектов.

## 11. Таблица «запах» — когда filter подозрителен

Проверьте свой filter plist перед релизом:

| Признак | Диагноз |
|---|---|
| `Bundles` пуст, но есть только `Classes = UIApplication` | Супершироко — переделайте |
| `Bundles` содержит `com.apple.*` без выбора | Опасно — чётко укажите нужные |
| Нет `CoreFoundationVersion` при использовании iOS 17+ API | Твик будет крэшить на iOS 15/16 |
| `Executables` без `Bundles` для не-демона | Проверьте — вероятно, вы хотели `Bundles` |
| Нет filter plist вообще | **Критическая ошибка**, твик загрузится **во ВСЕ** процессы |

Последнее — реальная трагедия начинающих. Без filter plist Theos создаёт `.deb`, который устанавливается, и твик загружается в каждый процесс системы, включая `cfprefsd`, `logd`, `launchd` (через `systemhook`). Любая ошибка — bootloop. **Всегда проверяйте, что filter plist в .deb есть**:

```bash
dpkg-deb -c your.deb | grep plist
# Должен показать .plist рядом с .dylib
```

## 12. Что подтверждено

- **Подтверждено Logos docs**: четыре типа фильтра (`Bundles`, `Classes`, `Executables`, `CoreFoundationVersion`) — стандартный формат.
- **Подтверждено практикой проекта**: Dopamine сам использует filter plist в своих internal dylib (`BaseBin/roothidehooks/roothidehooks.plist` — пустой, что означает «не стандартный Logos-механизм», а прямое ldload).
- **Требует проверки на устройстве**: конкретные bundle id сторонних приложений — зависят от того, что установлено.
- **Не подтверждено**: конкретное значение `CFCoreFoundationVersionNumber` для iOS 26.0 — требует запуска на устройстве.

## 13. Выводы

Главный тезис: **узкий filter — безопасный filter**. Начинайте с одного bundle id (своё приложение или конкретный системный процесс), и только когда убедитесь в стабильности, расширяйте.

Если у вашего твика нет filter plist вовсе — пересоберите с ним. Это критическая настройка, а не опциональная.

## 14. Что дальше

- Первый `%hook` + `%orig` — глава IV.4.
- Перехват C-функций через fishhook/ellekit — глава IV.5.
- Шпаргалка по filter plist со всеми вариантами — `cheatsheets/04-filter-plist.md`.
- Для первого полного учебного проекта — глава IV.11.
