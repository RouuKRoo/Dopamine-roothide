# Глава V.5. Контроль качества патча: пути, dylib, init, фильтры, настройки, совместимость 26.0 ↔ 26.0.1

Эта глава — закрытие Части V. Если предыдущие четыре главы отвечали на вопрос «как написать патч», то эта — на вопрос «как убедиться, что патч не ломает ничего, и как мигрировать между 26.0 и 26.0.1». Это чеклист, который даёт право на релиз.

## 1. Матрица качества — семь измерений

Хороший патч проверяется по семи осям:

| № | Ось | Что проверить |
|---|---|---|
| 1 | **Пути** | нет хардкода `/var/jb`, всё через libroot |
| 2 | **dylib зависимости** | все `otool -L` корректно, нет повисших `@rpath` |
| 3 | **Порядок инициализации** | `%ctor` безопасен, тяжёлая работа отложена |
| 4 | **Process filters** | filter plist максимально узкий |
| 5 | **Обработка ошибок** | есть fallback, нет silent failures |
| 6 | **Настройки** | через CFPreferences, Darwin notification |
| 7 | **Совместимость** | работает на обеих версиях iOS, которые заявляете |

Пройдите каждую ось — ваш патч готов.

## 2. Ось 1: Пути

### 2.1. Автопроверка

```bash
# В исходниках
grep -rn '/var/jb\|/basebin' *.x *.xm *.m *.c *.h 2>/dev/null | grep -v 'JBROOT_PATH\|libroot_get'

# В собранной dylib
strings .theos/obj/MyTweak.dylib | grep '/var/jb'
```

Ожидаемо — обе пустые.

### 2.2. Что искать руками

```bash
grep -rn 'fopen\|open(\|stat(\|access(\|realpath(' *.x *.m *.c | head
```

Для каждого вызова проверьте: путь корректно построен через `JBROOT_PATH`?

### 2.3. Checkbox

```
[ ] grep /var/jb — пусто в исходниках
[ ] strings dylib — не содержит /var/jb
[ ] NSBundle пути через JBROOT_PATH
[ ] CFPreferences вызовы с правильным domain
```

## 3. Ось 2: dylib зависимости

### 3.1. Автопроверка

```bash
otool -L .theos/obj/MyTweak.dylib
```

Проверяйте:

- Первая строка — ваш install_name.
- Все `@rpath/*.dylib` должны иметь соответствующий `LC_RPATH`.
- Нет странных путей `/Users/<name>/...` или `/Volumes/...`.
- Нет `/Library/MobileSubstrate/MobileSubstrate.dylib` (это legacy — у нас ellekit).

```bash
otool -l .theos/obj/MyTweak.dylib | grep -A2 LC_RPATH
```

### 3.2. Отсутствующие зависимости

Для каждой `@rpath/*.dylib` зависимости в `otool -L`, проверьте, что она доступна в пространстве поиска:

- `/var/jb/libroot.dylib` → подхватывается через `LC_RPATH /var/jb`.
- `/var/jb/usr/lib/libellekit.dylib` → аналогично.

На самом устройстве через SSH:

```bash
find /var/jb -name "libroot.dylib" 2>/dev/null
find /var/jb -name "libellekit.dylib" 2>/dev/null
```

Если пусто — зависимости нет; dyld упадёт.

### 3.3. Checkbox

```
[ ] otool -L чистый, нет странных путей
[ ] Каждая @rpath зависимость разрешима через LC_RPATH
[ ] libroot-dopamine в Depends control
[ ] ellekit (или альтернатива) в Depends control
```

## 4. Ось 3: Порядок инициализации

### 4.1. Что проверить в `%ctor`

Запрещено в `%ctor`:

- Инициализация heavy frameworks (CoreData, SwiftUI).
- Создание UIKit-объектов.
- Чтение сетью.
- Любая синхронная блокировка.

Разрешено:

- `os_log_create`.
- `pthread_key_create`.
- `pthread_mutex_init`.
- `dispatch_queue_create`.
- Чтение CFPreferences (быстро).
- Регистрация Darwin notification.

### 4.2. Что отложить в `dispatch_once`

Для «тяжёлой» инициализации используйте один из поздних триггеров:

```objc
static dispatch_once_t gOnce;

%hook UIApplication
- (BOOL)application:(UIApplication *)app didFinishLaunchingWithOptions:(NSDictionary *)opts {
    dispatch_once(&gOnce, ^{
        // Безопасная тяжёлая инициализация
        [self heavyInit];
    });
    return %orig(app, opts);
}
%end
```

### 4.3. Checkbox

```
[ ] %ctor только лёгкие операции
[ ] Тяжёлая инициализация через dispatch_once в раннем хуке
[ ] Нет обращений к UIKit в %ctor
[ ] Нет синхронных I/O операций в %ctor
```

## 5. Ось 4: Process filters

### 5.1. Что проверить в filter plist

```bash
plutil -p .theos/_/var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.plist
```

Правила:

- **Bundles** — максимально узкий (одно приложение, не wildcard).
- **Classes** — только если **действительно** нужно (например, UIKit-wide хук).
- **Executables** — только для демонов.
- **CoreFoundationVersion** — для cutoff по iOS.

### 5.2. Антипаттерны

Плохо:

```xml
<key>Filter</key>
<dict>
    <key>Classes</key>
    <array>
        <string>NSObject</string>
    </array>
</dict>
```

Это загрузит ваш твик **во всё**, включая демонов (у которых есть `NSObject`). Один из этих демонов упадёт — и у вас проблема.

Хорошо (для своего приложения):

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
</dict>
```

### 5.3. Checkbox

```
[ ] Filter plist присутствует в .deb
[ ] Bundles или Executables указывают только нужный процесс
[ ] Нет Classes = UIApplication если это не нужно
[ ] CoreFoundationVersion задан если используете iOS 17+ API
```

## 6. Ось 5: Обработка ошибок

### 6.1. Что проверить

- `%orig` вызван для каждого хука? (Если не вызван — осознанно?)
- Return-value — не nil где не должно?
- `malloc`/`CFCreate` — есть `free`/`CFRelease`?
- Timer — есть `invalidate`?
- Observer — есть `removeObserver` в dealloc?

### 6.2. Silent failures

Опасный антипаттерн — спрятать ошибку:

```objc
%hookf(int, something, int arg) {
    int r = %orig(arg);
    if (r != 0) {
        return 0;   // ВМЕСТО того, чтобы вернуть ошибку
    }
    return r;
}
```

Теперь приложение считает, что всё ок, хотя `something` вернул ошибку. Хуже всего, когда дебагер видит «вроде работает, но что-то не так».

Правильно:

```objc
%hookf(int, something, int arg) {
    int r = %orig(arg);
    if (r != 0) {
        os_log_error(gLog, "something failed: r=%d", r);
    }
    return r;  // пропускаем ошибку как есть
}
```

### 6.3. Checkbox

```
[ ] Каждый %hook либо вызывает %orig, либо осознанно его пропускает
[ ] Return-value проверены на nil где возможно nil
[ ] Нет silent failures — ошибки логируются
[ ] malloc/free, retain/release, create/release — парные
```

## 7. Ось 6: Настройки

### 7.1. Что проверить

- Настройки хранятся через CFPreferences (не прямым fopen на .plist).
- Darwin notification постится при изменении.
- Твик подписан на эту notification и reload'ит префы.
- `.plist` имя — reverse-DNS вашего пакета.

### 7.2. Проверка на устройстве

```bash
ssh mobile@<ip>
plutil -p /var/mobile/Library/Preferences/com.example.mytweak.plist
# или на RootHide через libroot path
```

Файл должен содержать то, что вы ожидаете, и обновляться когда пользователь переключает switch.

### 7.3. Checkbox

```
[ ] CFPreferences вместо прямого fopen
[ ] Darwin notification при изменении настроек
[ ] Reload в твике при notification
[ ] Domain preferences = package identifier
```

## 8. Ось 7: Совместимость 26.0 vs 26.0.1

### 8.1. Что меняется между минорными версиями

- **private селекторы**: Apple часто добавляет/переименовывает их. Проверка — class-dump на обеих версиях или probe с `objc_copyClassList`.
- **CoreFoundationVersion**: может сдвинуться на несколько единиц.
- **Entitlements**: некоторые закрываются или открываются.
- **Поведение демонов**: `cfprefsd`, `lsd`, `SpringBoard` могут слегка меняться.
- **Dopamine поддерживает обе версии**: см. `Info.plist` эксплойтов DarkSword/ClearSword с `End=26.0.1` (глава I.5).

### 8.2. Процедура миграционного теста

Если у вас два устройства (26.0 и 26.0.1) или одно, которое вы можете обновить:

1. Собрать твик с текущим кодом.
2. Установить на 26.0.
3. Запустить тестовое приложение, пройтись по сценариям.
4. Записать логи, скриншоты.
5. Обновить устройство до 26.0.1 (если возможно без потери jailbreak).
6. Переустановить Dopamine, твик.
7. Повторить тесты.
8. Сравнить поведение.

Если расхождение — патч на условное поведение (см. ниже).

### 8.3. Условный код для разных минорных

```objc
#import <sys/sysctl.h>

static NSString *gOSVersion = nil;

%ctor {
    char buf[64] = {0};
    size_t sz = sizeof(buf);
    if (sysctlbyname("kern.osproductversion", buf, &sz, NULL, 0) == 0) {
        gOSVersion = @(buf);
    }
}

static BOOL is_26_0_1(void) {
    return [gOSVersion isEqualToString:@"26.0.1"];
}

%hook SomeClass
- (void)someMethod {
    if (is_26_0_1()) {
        // ветка для 26.0.1
    } else {
        // ветка для 26.0
    }
    %orig;
}
%end
```

Это грубое, но работающее разделение. Для more sophisticated — проверять конкретные селекторы:

```objc
Class cls = NSClassFromString(@"_SBUINewClassIn26_0_1");
if (cls) {
    // 26.0.1 ветка
} else {
    // 26.0 ветка
}
```

### 8.4. Checkbox

```
[ ] Твик собран и установлен на 26.0
[ ] Тесты пройдены на 26.0
[ ] Твик установлен на 26.0.1
[ ] Тесты пройдены на 26.0.1
[ ] Расхождения зафиксированы в CHANGELOG
[ ] Условный код там, где требуется
```

## 9. Полный чеклист релиза

Собранный:

```
# Пути
[ ] grep /var/jb — пусто в исходниках
[ ] strings dylib — пусто
[ ] CFPreferences с правильным domain
[ ] NSBundle через JBROOT_PATH

# dylib
[ ] otool -L чистый
[ ] LC_RPATH содержит /var/jb (или другой нужный)
[ ] libroot-dopamine в Depends
[ ] ellekit в Depends

# Инициализация
[ ] %ctor лёгкий
[ ] Тяжёлая работа через dispatch_once
[ ] Нет UIKit в %ctor
[ ] Нет синхронных I/O в %ctor

# Filter
[ ] Filter plist есть в .deb
[ ] Bundles узкий — только нужный процесс
[ ] Нет Classes = UIApplication если не нужно
[ ] CoreFoundationVersion если используете iOS 17+ API

# Ошибки
[ ] %orig вызван для каждого хука (либо осознанно пропущен)
[ ] Return-value проверены на nil
[ ] Нет silent failures
[ ] malloc/free парные

# Настройки
[ ] CFPreferences вместо fopen
[ ] Darwin notification работает
[ ] Reload в твике

# Совместимость
[ ] Работает на 26.0
[ ] Работает на 26.0.1
[ ] Расхождения зафиксированы

# Релизные
[ ] Version в control инкрементирована
[ ] FINALPACKAGE=1 при сборке
[ ] Changelog обновлён
[ ] README на репозитории актуален
[ ] Прошёл inspect.sh (глава IV.10)
[ ] Прошёл verify.sh (глава V.4)
```

## 10. План отката

Независимо от качества — патч может сломать что-то в проде. План отката:

### 10.1. Для твика

```bash
# На устройстве:
sudo dpkg -r com.example.mytweak
killall -9 <target>
# или через Sileo/Zebra -> Remove
```

### 10.2. Для патча jailbreak

- `.safe_mode` маркер (глава V.1).
- Rollback basebin через предыдущую версию `basebin.tar`.
- DFU restore как последнее средство.

### 10.3. Для binary patch

- Храните оригинальный `.deb` до того, как ставить патченный.
- При проблеме — переустановить оригинал поверх.

## 11. Таблица «не могу решить — что делать»

| Проблема | Ответ |
|---|---|
| На 26.0 работает, на 26.0.1 — нет | Condition по `kern.osproductversion` или по наличию класса |
| На RootHide работает, на rootless — нет | Условие по `strncmp(libroot_get_jbroot_prefix(), "/var/jb", 7)` |
| Иногда работает, иногда нет | Thread safety: добавить serial queue или `@synchronized` |
| Работает в своём приложении, в чужом — нет | Hook на private класс, которого нет; probe/fallback |
| После установки — респринг loop | Снять патч через SSH или safe mode; упростить `%ctor` |

## 12. Что подтверждено

- **Подтверждено Apple docs**: `sysctlbyname("kern.osproductversion", ...)` — публичный API.
- **Подтверждено проектом**: совместимость 26.0–26.0.1 на A12/A13 декларирована в `DOEnvironmentManager.m:254` + DarkSword/ClearSword `Info.plist`.
- **Подтверждено Apple docs**: `objc_copyClassList`, `NSClassFromString` — стандартные API для probe.
- **Требует проверки на устройстве**: реальное поведение на 26.0.1 (пока предполагается идентичным 26.0, но Apple может иметь minor fixes).
- **Не подтверждено**: что все минорные версии 26.x будут поддерживаться Dopamine одинаково; проект сейчас декларирует именно 26.0 и 26.0.1.

## 13. Выводы

Главный тезис: **патч без chekclist'а — это рулетка**. Семь осей качества (пути / dylib / init / фильтры / ошибки / настройки / совместимость) закрывают 95% типовых проблем. Пять минут чеклиста экономят неделю отладки у пользователей.

Три якорных правила этой главы:

1. **Инспекция до релиза**: `otool -L`, `strings | grep /var/jb`, `plutil -p filter.plist`.
2. **Тест на обеих версиях** iOS 26.0 и 26.0.1.
3. **Готовый план отката**: `.safe_mode`, rollback .deb, DFU — в разной серьёзности.

На этом Часть V закрывается. Вы знаете, как писать патчи любого вида, как контролировать их качество, как мигрировать между минорными версиями.

## 14. Что дальше

- Исследование неизвестной функциональности — глава VI.1.
- Статический анализ vs устройство — глава VI.2.
- Диагностика «не загружается / падает» — глава VII.1.
- Чеклисты и шпаргалки в приложениях — A, B, C, D.
