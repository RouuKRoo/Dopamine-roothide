# Глава III.3. BaseBin/libroothider — RootHide-слой

Эта глава — о том, что отличает RootHide-ветку Dopamine от «просто rootless»: случайный префикс jbroot, проактивное скрытие маркеров jailbreak, корректные перезапуски процессов через exec_patch, обработка форка. Если `libjailbreak` даёт инфраструктуру, то `libroothider` — политику её применения на устройстве, где детектирование jailbreak со стороны приложений — реальная проблема.

## 1. Назначение

`libroothider.dylib` — вторая по важности библиотека `BaseBin/`, собирается после `libjailbreak`. Зонтичный заголовок — `BaseBin/libroothider/src/libroothider.h`:

```c
#ifndef ROOTHIDER_H
#define ROOTHIDER_H

#include "log.h"
#include "common.h"
#include "ptrace.h"
#include "mach_exc.h"
#include "exec_patch.h"
#include "jailbreakd.h"
#include "xpc_private.h"
#include "crashreporter.h"

#endif
```

На момент сборки `libjailbreak` **re-exports** `libroothider` (см. `BaseBin/libjailbreak/Makefile:9`: `LDFLAGS += -Wl,-reexport-lroothider`). Это значит, что компонент, слинкованный с `libjailbreak`, автоматически получает и `libroothider` без дополнительной `-l`.

Для автора стороннего твика важно: `libroothider` — это **в основном внутренняя инфраструктура Dopamine**. Твик её напрямую не использует (кроме отдельных функций логирования или crash reporting, если нужно). Но её работа создаёт среду, в которой твик живёт — поэтому её нужно понимать.

## 2. Карта модулей

| Файл | Назначение |
|---|---|
| `blacklist.{cpp,m}` | Чёрные списки процессов, которым **не** инъектится твики (чтобы не светить jailbreak) |
| `common.{h,m}` | Утилиты — получение bundle id по пути, определение приложений |
| `crashreporter.{h,m}` | Перехват крешей компонентов jailbreak |
| `dyld_patch.m` | Внутренние операции над dyld-пространством; `show_dyld_regions` для отладки |
| `exec_patch.{h,m}`, `exec_trace.m` | Перехват exec* для корректного запуска процессов с правильной среды |
| `jailbreakd.{c,h}` | Клиент-часть к `jailbreakd` (не путать с одноимённым демоном) |
| `launchd.m` | Специализированный хук для сервиса launchd |
| `libroothider.h` | Umbrella |
| `log.{c,h}` | Лог-API (используется и в сторонних твиках опционально) |
| `mach_exc.defs`, `mach_exc.h` | Mach exception messages |
| `ptrace.h` | Структуры/константы для ptrace-операций |
| `recdhash.m` | Вычисление/проверка cdhash для recovery |
| `signatures.m` | Структуры подписи, расширение `libjailbreak/signatures` |
| `unsandbox.h`, `unsandbox1.m`, `unsandbox2.m` | **Внутренние** операции расширения sandbox |
| `xpc_hook.m`, `xpc_private.h` | XPC-хуки для внутренних компонентов |
| `xpf.m` | Интерфейс к XPF (eXtensible Patchfinder) |

## 3. Главный сюжет: случайный jbroot

`libroothider` помогает «спрятать» jailbreak-инфраструктуру за **случайным путём**: вместо хардкодного `/var/jb` используется что-то вроде `/var/<rand1>/<rand2>/.jbroot`. Это настоящий путь на диске, но с точки зрения публичного `/basebin` симлинка он выглядит иначе.

Зачем:

- `/var/jb` — палево. Приложение, детектирующее jailbreak, может сделать `stat("/var/jb/")` и получить ответ «да, это jailbroken». RootHide убирает эту возможность.
- Приложения, честно обращающиеся через libroot (`libroot_get_jbroot_prefix()`), работают прозрачно. Приложения, хардкодящие путь — ломаются (это и есть «палево детектора» в обратную сторону).

Для автора твика: **никогда не хардкодь `/var/jb` в своём твике**. Используй `JBROOT_PATH(...)` макрос из `libjailbreak/jbroot.h` или `libroot_get_jbroot_prefix()` из `libroot.h`. Детальный разбор — глава III.10.

## 4. Blacklist — где НЕ работают твики

Файл `BaseBin/libroothider/src/blacklist.m` — осознанное решение Dopamine: некоторые процессы **не получают** `DYLD_INSERT_LIBRARIES=.../systemhook.dylib`. Это обычно означает «процессы, где инъекция ломает систему» или «процессы, которые должны оставаться «чистыми» для анти-cheat / анти-tamper проверок».

Как это работает:

```objc
#define APP_PATH_PREFIX "/private/var/containers/Bundle/Application/"
#define NULL_UUID "00000000-0000-0000-0000-000000000000"

NSString *getAppBundlePathFromSpawnPath(const char *path)
{
    if (!path) return nil;
    char abspath[PATH_MAX];
    if (!realpath(path, abspath)) return nil;

    if (strncmp(abspath, APP_PATH_PREFIX, sizeof(APP_PATH_PREFIX) - 1) != 0)
        return nil;
    ...
}
```

Логика (упрощённо):

1. Получить путь к бинарнику запускаемого процесса.
2. Выяснить, принадлежит ли он установленному приложению.
3. По нескольким правилам (списки, settings) решить — инъектировать `systemhook.dylib` или нет.

Для автора твика это даёт понимание: если ваш твик «не загружается в такое-то приложение», проверьте — не попало ли приложение в blacklist. Это легитимная причина; настройки обычно управляются через `jbsettings`.

## 5. exec_patch — зачем перехватывается exec

Файл `BaseBin/libroothider/src/exec_patch.m` реализует перехват семейства `exec*` syscalls (`execve`, `posix_spawn` и т. д.). Задача:

- Проверить, что запускаемый бинарник имеет корректную подпись (или добавить её в trust cache).
- Подложить `DYLD_INSERT_LIBRARIES=.../systemhook.dylib` в среду.
- Пустить реальный exec.

Это основная причина, почему **каждый** новый процесс на jailbroken-устройстве получает `systemhook`. Без `libroothider` этой логики twikов в приложениях бы не было.

## 6. Исправление fork

Файл `BaseBin/libroothider/src/launchd.m` плюс функции в `libjailbreak` обеспечивают, что когда процесс делает `fork()`, дочерний получает всю ту же инструментовку. Проблема: `fork()` на iOS — редкая операция, но некоторые системные компоненты (`xpcproxy`, вспомогательные helper'ы) её делают. Без `fork_fix` дочерний процесс оказался бы без `jbclient_cs_revalidate` и NECP-хуков — и его сеть перестала бы работать.

API: `jbclient_fork_fix(childPid)` из `libjailbreak`. Автор твика это не вызывает — за него делает система.

## 7. XPC-хуки

Файл `BaseBin/libroothider/src/xpc_hook.m` — внутренние хуки на XPC-API, которые позволяют проактивно работать с mach-портами. Применяется, например, когда нужно корректно обработать «jbserver»-соединения в неожиданных местах.

Это внутренняя инфраструктура — автор стороннего твика с ней не работает. Но знать, что часть XPC-функций «подкручена», полезно: ваши хуки на `xpc_connection_send_message` сработают **после** хуков `libroothider`, и аргументы, которые вы увидите, могут отличаться от «сырых».

## 8. Логирование — то, что доступно твикам

Один из немногих компонентов `libroothider`, который реально имеет смысл использовать в стороннем твике — лог-API `BaseBin/libroothider/src/log.h`. Макросы вида `JBLogDebug(fmt, ...)` пишут в системный лог с префиксом «jailbreak», удобно для диагностики.

Пример:

```c
#include <libroothider/log.h>

JBLogDebug("MyTweak: hooked view=%p controller=%@", view, controller);
```

Требует линковки с `-lroothider` или через re-export из `libjailbreak`.

## 9. Crash reporter

Файл `BaseBin/libroothider/src/crashreporter.m` реализует перехват крешей внутренних jailbreak-компонентов — чтобы они не терялись в системном `.ips` потоке. Это чисто внутренняя функция; автор стороннего твика сюда не лезет.

Но полезно знать: если ваш твик вызывает UB в хуке на критический селектор, креш может быть **перехвачен** crash reporter'ом Dopamine и записан отдельно. Проверить — есть ли ваш `.ips` в стандартных путях `/private/var/mobile/Library/Logs/CrashReporter/` + jailbreak-пути (через `jbroot`).

## 10. Таблица «что доступно твикам»

| Функция/подсистема | Статус |
|---|---|
| `JBLog*` макросы | **Доступны** |
| `log.h` API | **Доступны** |
| `common.m` утилиты (getAppBundlePathFromSpawnPath, …) | Доступны, но не публичны |
| `blacklist.m` функции | Внутренние, не для сторонних твиков |
| `exec_patch` | Внутренние |
| `dyld_patch`, `show_dyld_regions` | Внутренние (но пригодится для отладки, если включить `JBLOG_LEVEL=DEBUG`) |
| `xpc_hook`, `xpc_private` | Внутренние |
| `jailbreakd.c` client functions | Косвенно, через `jbclient_*` |

## 11. Что подтверждено

- **Подтверждено исходниками**: все перечисленные файлы существуют в `BaseBin/libroothider/src/`. Umbrella-заголовок — `libroothider.h`.
- **Подтверждено Makefile**: `libjailbreak` re-exports `libroothider` (`-Wl,-reexport-lroothider` в `BaseBin/libjailbreak/Makefile`).
- **Подтверждено логикой**: прямое грепание `blacklist`, `exec_patch`, `jailbreakd` показывает блочную структуру.
- **Требует проверки на устройстве**: конкретные правила blacklist для iOS 26 (меняются с версиями Dopamine).
- **Не подтверждено**: гарантии стабильности `JBLog*` API между версиями; API-contract для сторонних твиков.

## 12. Выводы

Главный тезис: `libroothider` — это **инфраструктура «невидимости» и правильности** jailbreak на jailbroken-устройстве. Автор стороннего твика с ней напрямую взаимодействует редко (в основном через `JBLog*` для диагностики). Но её работа создаёт среду, в которой твик имеет jbroot-путь через libroot, инструментовку через systemhook в каждом процессе, корректное наследование при fork, и защиту URL-схем на уровне `lsd`.

Если ваш твик начинает вести себя странно (не загружается в конкретное приложение, пропадает после `fork()`, видит разные пути в разных процессах) — проверьте `libroothider`-слой: вероятно, одно из его правил (blacklist, exec_patch, path translation) объясняет происходящее.

## 13. Что дальше

- Разбор `launchdhook`, где живёт серверная часть jbserver — глава III.4.
- Разбор `systemhook`, главного клиента `libroothider`-инфраструктуры — глава III.5.
- Разбор `libroot` (минимальный API путей) — глава III.10.
- Для деталей blacklist — глава III.9 (`rootless/roothide` hooks).
