# Шпаргалка 5. libroot API — все экспортированные функции

`libroot.dylib` — публичная библиотека путей для сторонних твиков. Всего **три** функции + два header-only макроса.

## Включение

```c
#include <libroot.h>
```

В Makefile:

```make
MyTweak_LIBRARIES = root
```

В control:

```
Depends: ..., libroot-dopamine
```

## Три функции

### `libroot_get_jbroot_prefix`

```c
const char *libroot_get_jbroot_prefix(void);
```

| | |
|---|---|
| Возвращает | Префикс jbroot (например, `/var/jb` или `/var/<rand1>/<rand2>/.jbroot`) |
| Собственность | Статическая строка; **не освобождайте** |
| Nullable | Нет (всегда возвращает что-то; на не-jailbreak может вернуть пустое) |
| Thread-safe | Да |

Применение:

```c
const char *jbroot = libroot_get_jbroot_prefix();
char path[PATH_MAX];
snprintf(path, sizeof(path), "%s/Library/Example/data.json", jbroot);
```

### `libroot_get_root_prefix`

```c
const char *libroot_get_root_prefix(void);
```

| | |
|---|---|
| Возвращает | `""` (пустая строка на jailbroken устройстве) |
| Собственность | Статическая |

Применение: редко; используется для explicit документирования, что путь ведёт к rootfs.

### `libroot_get_boot_uuid`

```c
const char *libroot_get_boot_uuid(void);
```

| | |
|---|---|
| Возвращает | UUID текущего cold-boot сеанса |
| Собственность | Статическая |

Применение: отслеживание, пережила ли операция userspace reboot.

```c
const char *bootUUID = libroot_get_boot_uuid();
// Сохранить в файл; при следующем запуске сравнить — если тот же — тот же boot
```

## Макросы (header-only)

Из `BaseBin/libjailbreak/src/jbroot.h`. Не требуют линковки с libjailbreak:

### `JBROOT_PATH(path)`

```c
JBROOT_PATH(path)
```

- Принимает: `const char *` или `NSString *`.
- Возвращает: соответствующий тип с префиксом jbroot.
- Внутри: `strlcpy` + `strlcat` с alloca-буфером (zero-heap).

Пример для C:

```c
#include <libjailbreak/jbroot.h>
FILE *f = fopen(JBROOT_PATH("/Library/MyData/config.plist"), "r");
```

Пример для ObjC:

```c
NSString *path = JBROOT_PATH_NSSTRING(@"/Library/MyData/config.plist");
NSData *d = [NSData dataWithContentsOfFile:path];
```

### `ROOTFS_PATH(path)`

Аналог для rootfs-view. На jailbroken устройстве обычно эквивалентен исходному пути.

### `JBROOT_PATH_CSTRING(path)` и `JBROOT_PATH_NSSTRING(path)`

Явные варианты, если тип аргумента неясен.

## Полный пример использования

```objc
#import <libroot.h>
#import <libjailbreak/jbroot.h>

%ctor {
    // Способ 1: через API функцию
    const char *jbroot = libroot_get_jbroot_prefix();
    NSLog(@"jbroot = %s", jbroot);

    // Способ 2: через макрос
    const char *resourcePath = JBROOT_PATH("/Library/Application Support/MyTweak/");
    NSLog(@"resources = %s", resourcePath);

    // Способ 3: ObjC
    NSString *dataPath = JBROOT_PATH_NSSTRING(@"/Library/MyData/");
    NSBundle *bundle = [NSBundle bundleWithPath:dataPath];
}
```

## Соответствие путей

Таблица «что передать → что получите на разных установках»:

| Передали | rootless (простой) | RootHide |
|---|---|---|
| `"/Library/Example"` | `/var/jb/Library/Example` | `/var/<rand>/<rand>/.jbroot/Library/Example` |
| `"/usr/lib/libmy.dylib"` | `/var/jb/usr/lib/libmy.dylib` | `/var/<rand>/<rand>/.jbroot/usr/lib/libmy.dylib` |
| `"/basebin/jbctl"` | `/var/jb/basebin/jbctl` | `/var/<rand>/<rand>/.jbroot/basebin/jbctl` (через `/basebin` symlink) |

## Антипаттерны

**НЕ ДЕЛАЙТЕ**:

```c
// ПЛОХО:
NSString *path = @"/var/jb/Library/MyData/data.plist";

// ПЛОХО:
char buf[PATH_MAX];
sprintf(buf, "/var/jb%s", "/Library/MyData/data.plist");

// ПЛОХО:
const char *jbroot = getenv("JBROOT");  // такой переменной нет
```

**ПРАВИЛЬНО**:

```c
// ХОРОШО (ObjC):
NSString *path = JBROOT_PATH_NSSTRING(@"/Library/MyData/data.plist");

// ХОРОШО (C):
const char *path = JBROOT_PATH("/Library/MyData/data.plist");

// ХОРОШО (через функцию):
const char *jbroot = libroot_get_jbroot_prefix();
char path[PATH_MAX];
snprintf(path, sizeof(path), "%s/Library/MyData/data.plist", jbroot);
```

## Диагностика «пути не работают»

Чеклист:

```
[ ] Makefile содержит MyTweak_LIBRARIES = root
[ ] control содержит libroot-dopamine в Depends
[ ] #include <libroot.h> в коде
[ ] Используется JBROOT_PATH или libroot_get_jbroot_prefix
[ ] НЕТ хардкода /var/jb в коде
[ ] strings .dylib | grep /var/jb — пусто
[ ] otool -L .dylib содержит @rpath/libroot.dylib
```

## См. также

- Глава III.10 — libroot в деталях (полный исходный код).
- Глава IV.9 — RootHide-совместимость.
- Шпаргалка 8 — таблица путей rootless vs RootHide.
