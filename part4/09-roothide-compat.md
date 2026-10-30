# Глава IV.9. Совместимость с RootHide

Этa глава — концентрат того, что нужно знать, чтобы ваш твик работал одинаково хорошо и на классическом rootless (Dopamine 3 без roothide), и на RootHide-установке. Правильный ответ почти всегда один: `libroot` + корректный `THEOS_PACKAGE_SCHEME`. Но нюансов — больше.

## 1. Что меняет RootHide

Напомним, кратко:

- **Rootless Dopamine**: все jailbreak-данные живут под `/var/jb/`. Это предсказуемый и видимый путь.
- **RootHide Dopamine**: jbroot имеет **случайный префикс** `/var/<rand1>/<rand2>/.jbroot`, плюс симлинк `/basebin → <jbroot>/basebin`. Приложения-детекторы, проверяющие `/var/jb/`, видят, что его нет.

Для автора твика это означает:

- **Хардкод пути `/var/jb/...`** в вашем коде → не работает на RootHide.
- **Чтение `$JBROOT`** из окружения → не работает (переменной нет).
- **Разбор `realpath("/basebin")`** → работает, но не является официальным API.

Единственный правильный путь — через **`libroot`**.

## 2. THEOS_PACKAGE_SCHEME и его роль

`THEOS_PACKAGE_SCHEME=roothide` в Makefile влияет на три вещи:

1. **Install path** — куда внутри `.deb` кладутся файлы. При `roothide` Theos использует внутренние маркеры, которые dpkg на устройстве раскрывает в правильный randomized путь.
2. **Control-файл** — автоматически подставляется правильная Architecture и зависимости.
3. **rpath и install_name** — в собранном `.dylib` они указывают на пути, которые работают при любом префиксе jbroot.

Таблица трёх схем:

| Scheme | Install path в .deb | Для чего |
|---|---|---|
| не задано (legacy) | `/Library/MobileSubstrate/DynamicLibraries/` | Старый jailbreak до iOS 15 |
| `rootless` | `/var/jb/Library/MobileSubstrate/DynamicLibraries/` | Классический rootless Dopamine |
| `roothide` | Внутренние маркеры + корректный rpath | **RootHide Dopamine (наш целевой)** |

**Для нашей целевой среды — всегда `roothide`**. Пакет, собранный с `roothide`, работает и на RootHide, и на rootless (dpkg расставит пути правильно). Пакет, собранный с `rootless`, может сломаться на RootHide, потому что хардкодит путь `/var/jb/`.

## 3. Чек-лист RootHide-совместимости

Семь пунктов, которые нужно проверить перед релизом:

| № | Проверка | Как |
|---|---|---|
| 1 | `THEOS_PACKAGE_SCHEME = roothide` в Makefile | grep «roothide» Makefile |
| 2 | `*_LIBRARIES = root` в Makefile | grep «LIBRARIES» Makefile |
| 3 | В `control` зависимость от `libroot-dopamine` | grep «libroot» control |
| 4 | В коде нет хардкода `/var/jb` | `grep -rn '/var/jb' *.x *.m *.c` — пусто |
| 5 | В коде нет чтения `$JBROOT` | `grep 'JBROOT' *.x *.m *.c` — только через libroot API |
| 6 | Макросы `JBROOT_PATH(...)` или `libroot_get_jbroot_prefix()` используются | grep «JBROOT_PATH\|libroot_get» |
| 7 | В собранном `.dylib` нет строк `/var/jb` | `strings MyTweak.dylib | grep /var/jb` — пусто или только служебные |

Пункт 7 — финальная автоматическая проверка. Если `strings` показывает жёсткие пути — у вас есть скрытый хардкод.

## 4. Антипаттерн 1: хардкод путей в коде

Плохо:

```objc
NSString *path = @"/var/jb/usr/share/mytweak/data.json";
NSData *data = [NSData dataWithContentsOfFile:path];
```

На RootHide этого файла по пути `/var/jb/usr/share/mytweak/data.json` **нет**. Есть `/var/<rand>/<rand>/.jbroot/usr/share/mytweak/data.json`.

Хорошо:

```objc
#import <libroot.h>
NSString *path = @(JBROOT_PATH("/usr/share/mytweak/data.json"));
NSData *data = [NSData dataWithContentsOfFile:path];
```

Или напрямую через libroot API:

```objc
const char *jbroot = libroot_get_jbroot_prefix();
NSString *path = [NSString stringWithFormat:@"%s/usr/share/mytweak/data.json", jbroot];
```

## 5. Антипаттерн 2: `fopen("/var/jb/...")` в C-коде

Плохо:

```c
FILE *f = fopen("/var/jb/Library/MyData/config.cfg", "r");
```

Хорошо:

```c
#include <libroot.h>
FILE *f = fopen(JBROOT_PATH("/Library/MyData/config.cfg"), "r");
```

`JBROOT_PATH` раскрывается в inline-функцию, которая конкатенирует правильный префикс и путь на стеке через `alloca`. Zero-allocation, zero-overhead.

## 6. Антипаттерн 3: прямое чтение `/basebin`

Плохо:

```c
char buf[PATH_MAX];
ssize_t n = readlink("/basebin", buf, sizeof(buf)-1);
if (n > 0) {
    buf[n] = '\0';
    // отрезать "/basebin" и использовать как jbroot
}
```

Это **случайно работает** на RootHide-установке (`/basebin` действительно symlink в `<jbroot>/basebin`). Но:

- Это не публичный контракт.
- На rootless `/basebin` **нет** — ваш код выдаст ошибку.
- Код хрупок к будущим изменениям.

Хорошо: `libroot_get_jbroot_prefix()`.

## 7. Антипаттерн 4: `access("/var/jb", F_OK)` как проверка jailbreak

Классическая проверка «есть ли jailbreak»:

```c
if (access("/var/jb", F_OK) == 0) {
    // jailbroken
}
```

Это:

1. **Ломается на RootHide** — файла по этому точному пути нет (есть `/basebin` symlink, но не `/var/jb`).
2. **Детектит jailbreak**, если жб-приложение закидывает такое — это то, против чего RootHide и был создан.
3. **Не правильный способ** в принципе: правильный — `jbclient_dopamine_is_jailbroken(NULL)` из `libjailbreak`.

Хорошо для стороннего твика (который и так знает, что он в jailbreak-окружении — он же работает в твике):

- Просто доверяйте тому, что вы в jailbreak-окружении.
- Если нужна проверка — `jbclient_dopamine_is_jailbroken(NULL)` (сторонний твик обычно и не вызывает её).

## 8. Антипаттерн 5: NSBundle с хардкодом

Плохо:

```objc
NSBundle *b = [NSBundle bundleWithPath:@"/var/jb/Library/Application Support/MyTweak"];
```

Хорошо:

```objc
NSString *bundlePath = @(JBROOT_PATH("/Library/Application Support/MyTweak"));
NSBundle *b = [NSBundle bundleWithPath:bundlePath];
```

Детальнее — глава IV.8 про ресурсы.

## 9. Запуск внешних бинарников твиком

Если ваш твик хочет запустить `jbctl` или собственный helper из `/usr/bin/`:

Плохо:

```c
system("/var/jb/usr/bin/mytweak-helper --do-stuff");
```

Хорошо:

```c
char cmd[PATH_MAX + 100];
snprintf(cmd, sizeof(cmd), "%s/usr/bin/mytweak-helper --do-stuff",
         libroot_get_jbroot_prefix());
system(cmd);
```

Или через `posix_spawn` для большего контроля (не советуется из `%ctor` — рано).

Для редких случаев, когда вам нужен путь ровно к `uicache`:

```c
#include <libjailbreak/jbroot.h>
exec_cmd(JBROOT_PATH("/usr/bin/uicache"), "-a", NULL);
```

(Пример из `BaseBin/rootlesshooks/lsd.x` — см. главу III.9.)

## 10. Шрифты, SQLite, другие ресурсы

Эти ресурсы часто имеют свои пути. Таблица «что ожидают стандартные API» и «как подправить»:

| API | Путь по умолчанию | Что делать в твике |
|---|---|---|
| `UIFont fontWithName:...` | ищет в бандле приложения | загружайте через `CTFontManagerRegisterFontsForURL` с `JBROOT_PATH` |
| `sqlite3_open(...)` | именно тот путь, что передали | `JBROOT_PATH("/Library/MyData/db.sqlite")` |
| `[NSBundle mainBundle]` | бандл приложения-хоста (не твика!) | используйте `bundleWithPath:` с `JBROOT_PATH` для своего бандла |
| `[NSUserDefaults standardUserDefaults]` | defaults приложения-хоста | используйте `initWithSuiteName:` с domain вашего твика |

Главный источник ошибок — `[NSBundle mainBundle]`. В хуке `mainBundle` — это бандл **хост-приложения**, не вашего твика. Если вам нужен ваш `MyTweak.bundle` — передавайте путь явно.

## 11. Автотест совместимости

Небольшой скрипт, который можно запустить до релиза:

```bash
#!/usr/bin/env bash
set -e
DEB="$1"
echo "=== Checking $DEB for RootHide compat ==="

# 1. Правильный путь внутри
echo "-- paths inside .deb --"
dpkg-deb -c "$DEB" | grep -q "/var/jb/Library/MobileSubstrate/DynamicLibraries/" \
    && echo "  OK: rootless path"
dpkg-deb -c "$DEB" | grep -v "^d" | grep -q "\.dylib$" \
    && echo "  OK: dylib present"
dpkg-deb -c "$DEB" | grep -v "^d" | grep -q "\.plist$" \
    && echo "  OK: filter plist present"

# 2. control зависимости
TMP=$(mktemp -d)
dpkg-deb -e "$DEB" "$TMP"
grep -q "libroot-dopamine" "$TMP/control" \
    && echo "  OK: depends on libroot-dopamine" \
    || echo "  WARN: no libroot-dopamine in Depends"

# 3. Инспекция dylib
DYLIB=$(dpkg-deb -c "$DEB" | grep "\.dylib$" | head -1 | awk '{print $NF}' | sed 's|^\./||')
mkdir -p "$TMP/x"
dpkg-deb -x "$DEB" "$TMP/x"
DYLIB_PATH="$TMP/x/${DYLIB#./}"

echo "-- otool -L --"
otool -L "$DYLIB_PATH" | sed 's/^/  /'

echo "-- strings | grep /var/jb --"
if strings "$DYLIB_PATH" | grep -q '/var/jb'; then
    echo "  WARN: hardcoded /var/jb strings found"
    strings "$DYLIB_PATH" | grep '/var/jb' | head -5 | sed 's/^/    /'
else
    echo "  OK: no /var/jb strings"
fi

rm -rf "$TMP"
echo "=== done ==="
```

Запускаете: `./check-compat.sh packages/com.example.firsthook_0.0.1_iphoneos-arm.deb`. Если видите WARN — есть ошибки.

Этот скрипт — часть вашей личной CI/CD. Запускайте перед каждым релизом.

## 12. Rootless vs RootHide — должен ли твик знать разницу?

Хороший вопрос: нужно ли вашему коду вообще знать, на какой установке он работает?

**В 99% случаев — нет**. `libroot_get_jbroot_prefix()` возвращает правильное значение независимо от типа установки, и ваш код работает одинаково.

**В 1% случаев** — редкие сценарии:

- Твик хочет жить **только** на RootHide и упасть на rootless.
- Твик хочет применить дополнительные анти-detect меры **только** на RootHide.

Для этого — проверяем, что префикс jbroot начинается не с `/var/jb`:

```c
const char *jbroot = libroot_get_jbroot_prefix();
BOOL isRootHide = (strncmp(jbroot, "/var/jb", 7) != 0);
```

Это **эвристика**, не contract. В будущем RootHide может выдавать `/var/jb/` же в каких-то режимах. Поэтому для продакшн-кода лучше вообще не делать этого различения.

## 13. Что проверить при переносе старого твика на RootHide

Если у вас есть твик, работавший на старом Dopamine (rootless), и вы хотите портировать на RootHide:

| Шаг | Что делать |
|---|---|
| 1 | Поменять `THEOS_PACKAGE_SCHEME = rootless` на `roothide` |
| 2 | Добавить `*_LIBRARIES += root` |
| 3 | В `control` добавить зависимость `libroot-dopamine` |
| 4 | Найти все `/var/jb` в коде → заменить на `JBROOT_PATH` или `libroot_get_jbroot_prefix` |
| 5 | Проверить все `fopen`/`stat`/`access` на абсолютные пути — то же |
| 6 | Если твик использовал `access("/var/jb", F_OK)` — заменить на `jbclient_dopamine_is_jailbroken` или вообще убрать |
| 7 | Запустить автотест совместимости (раздел 11) |
| 8 | Протестировать на RootHide-устройстве |

Обычно весь порт занимает полдня — если код не содержал более серьёзного хардкода.

## 14. Что подтверждено

- **Подтверждено проектом**: `THEOS_PACKAGE_SCHEME=roothide` — обязателен для RootHide; `libroot-dopamine` версия 1.0.1 в `DOBootstrapper.m:21`.
- **Подтверждено практикой**: примеры хуков в `BaseBin/rootlesshooks/lsd.x` и `BaseBin/roothidehooks/src/*` — все используют `JBROOT_PATH`/`libroot_get_jbroot_prefix`.
- **Подтверждено Theos**: `*_LIBRARIES = root` — стандартный способ линковки.
- **Требует проверки на устройстве**: что ваш конкретный пакет корректно распаковывается в randomized jbroot.
- **Не подтверждено**: гарантии стабильности `/basebin` симлинка в будущих RootHide-версиях.

## 15. Выводы

Главный тезис: **совместимость с RootHide — это не отдельная работа, это дисциплина использовать `libroot` вместо хардкода**. Если делаете это с первого дня — ничего дополнительного не нужно, твик работает всюду.

Три якорных правила:

1. `THEOS_PACKAGE_SCHEME = roothide` в Makefile.
2. `MyTweak_LIBRARIES = root` в Makefile; `Depends: libroot-dopamine` в control.
3. `JBROOT_PATH(...)` или `libroot_get_jbroot_prefix()` вместо `"/var/jb/..."`.

Соблюдайте — и о «а на RootHide не работает» забудете навсегда.

## 16. Что дальше

- Сборка, инспекция, упаковка — глава IV.10.
- Полный учебный проект №1 — глава IV.11.
- Шпаргалка путей — `cheatsheets/08-paths-table.md`.
- Шпаргалка libroot API — `cheatsheets/05-libroot-api.md`.
