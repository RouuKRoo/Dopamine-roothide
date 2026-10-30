# Глава III.2. BaseBin/libjailbreak — ядро userland-API

`libjailbreak.dylib` — главная библиотека jailbreak-экосистемы. Она собирается первой и линкуется почти ко всему остальному в `BaseBin/`. Это и внутренний API (для `launchdhook`, `systemhook`, `jailbreakd`), и — ограниченно — публичный API, который могут использовать сторонние твики. В этой главе покажем, что в ней есть, как она устроена, и что из этого полезно именно автору твика.

## 1. Назначение

`BaseBin/libjailbreak/src/libjailbreak.h` — главный зонтичный заголовок:

```c
#include "primitives.h"
#include "info.h"
#include "kernel.h"
#include "util.h"
#include "translation.h"
#include "trustcache.h"
#include "jbclient_xpc.h"
#include "stock_fixes.h"

#include <libroothider/libroothider.h>

int jbclient_initialize_primitives_internal(bool physrwPTE);
int jbclient_initialize_primitives(void);
```

Два главных «профиля» использования:

- **Внутренний** — компоненты Dopamine линкуются ко всей библиотеке и используют и клиент-API (`jbclient_*`), и kernel-primitives (`kernel.*`, `primitives.*`, `trustcache.*`, `physrw.*`).
- **Внешний (для твиков)** — используется только клиент-часть (`jbclient_xpc.h`). Все серверные/root-операции делегируются через XPC к `launchdhook`/`jailbreakd`.

Ключевое правило: **сторонний твик никогда не должен включать внутренние заголовки `physrw.h`, `kernel.h`, `kcall_*.h`, `pte.h`, `pvh.h`, `txm.h`**. Они дают примитивы, которые требуют реального доступа к kernel task — а у твика его нет. Правильный путь — вызвать `jbclient_*` функцию, которая проделает нужную операцию через XPC-сервер.

## 2. Карта модулей

Таблица «что где лежит»:

| Файл | Назначение | Доступно твикам? |
|---|---|---|
| `jbclient_xpc.{c,h}` | XPC-клиент к jbserver | **Да, публично** |
| `jbclient_mach.{c,h}` | Mach-клиент (альтернативный транспорт) | Внутренне |
| `jbclient_roothide.c` | roothide-специфичные вызовы | Внутренне (но вызывается `libroot`) |
| `jbserver_*.{c,h}` | Серверная часть; реализация доменов | Внутренне |
| `jbserver_domains.h` | Нумерация доменов | **Да, для чтения** |
| `info.{c,h}` | Информация о среде (jbroot, boot_uuid, cpu) | Да, частично |
| `jbroot.{c,h}` | Макросы JBROOT_PATH/ROOTFS_PATH | **Да, публично** |
| `codesign.{c,h,m}` | Операции с подписью | Внутренне |
| `trustcache*.{c,h}`, `trustcache_structs.h` | Trust cache | Внутренне |
| `physrw*.{c,h}` | Физическая память | **Никогда** в твиках |
| `kcall_*.{c,S,h}` | Вызовы ядра | **Никогда** в твиках |
| `kernel.{c,h}` | Внутр. kernel R/W | **Никогда** в твиках |
| `inline_svc.{c,S,h}` | Прямые syscalls в обход своих хуков | Внутренне |
| `pte.h`, `pvh.h`, `txm.*` | PTE, PVH, TXM | **Никогда** |
| `signatures.{c,h}` | Структуры подписи для jbclient | Да, косвенно |
| `stock_fixes.{c,h}` | Исправления «стандартных» багов | Да, автоматически |
| `translation.{c,h}` | Rosetta/translation | Специализированное |
| `hookd.{c,h}` | API к `hookd` | **Да, публично** (с осторожностью) |
| `util.{c,h,m}`, `display.{c,h,m}` | Утилиты | Разное |
| `machine_info.h` | CPU constants | Да (чтение) |
| `basebin_gen.{h,m}`, `carboncopy.{h,m}` | Внутренняя инфраструктура | Нет |
| `primitives*.{c,h,m}` | Высокоуровневые примитивы (IOSurface и т. п.) | Внутренне |

## 3. jbclient — публичный клиент-API

Главный поверхностный API для внешних твиков — `BaseBin/libjailbreak/src/jbclient_xpc.h`. Разбор ключевых функций:

### 3.1. Информация о среде

```c
char *jbclient_get_jbroot(void);
char *jbclient_get_boot_uuid(void);
```

- `jbclient_get_jbroot()` возвращает текущий префикс jbroot (например, `/var/<rand1>/<rand2>/.jbroot` для roothide, или `/var/jb` для rootless). Это **правильный** способ узнать корневой путь. Выделенная строка принадлежит клиенту — освободить `free()` после использования.
- `jbclient_get_boot_uuid()` возвращает UUID текущего cold-boot сеанса. Полезно для идентификации «переживание» каких-то операций через userspace reboot.

Эти две функции — те, которые на самом деле вызывает `libroot_get_jbroot_prefix()` в `Packages/libroot/src/paths.c`.

### 3.2. Trust cache и подпись

```c
int jbclient_trust_file(int fd, struct siginfo *siginfo, bool attach);
int jbclient_trust_file_by_path(const char *path);
```

- `jbclient_trust_file_by_path` — для файла по пути, Dopamine посчитает его cdhash и добавит в dynamic trust cache. Применение в твиках: если твой твик кладёт на диск дополнительный `.dylib` и хочет его загрузить потом через `dlopen`.
- `jbclient_trust_file` — та же логика, но для открытого дескриптора + известной `siginfo`. Полезно, если ты уже разобрал подпись сам.

### 3.3. Процесс и подпись

```c
int jbclient_process_checkin(char **rootPathOut, char **bootUUIDOut,
                             char **sandboxExtensionsOut, bool *fullyDebuggedOut,
                             bool *forceCSAdhocOut);
int jbclient_cs_revalidate(void);
int jbclient_fork_fix(uint64_t childPid);
int jbclient_persona_fix(int childPid, uid_t overwriteUid, gid_t overwriteGid);
```

- `jbclient_process_checkin` — вызывается при старте процесса (на уровне `systemhook`). Автор твика сам не вызывает, но знание о нём важно: он получает jbroot-путь, boot UUID, список sandbox extensions.
- `jbclient_cs_revalidate` — тот самый, который используется в NECP-хуках (см. главу II.9). Публичен, но автор стороннего твика обычно его не вызывает — за него делает `systemhook`.
- `jbclient_fork_fix` — для случаев, когда родитель форкнулся и дочерний процесс оказывается без правильной инструментовки. Относится к `forkfix.dylib`.
- `jbclient_persona_fix` — исправление UID/GID после создания нового процесса.

### 3.4. Настройки jailbreak

```c
int jbclient_jbsettings_get(const char *key, xpc_object_t *valueOut);
bool jbclient_jbsettings_get_bool(const char *key);
uint64_t jbclient_jbsettings_get_uint64(const char *key);
double jbclient_jbsettings_get_double(const char *key);
```

Чтение глобальных настроек jailbreak (например, «включены ли tweaks», «включён ли TrollStore mode»). Это полезно, если твой твик хочет уважать настройки пользователя.

```c
int jbclient_platform_jbsettings_set(const char *key, xpc_object_t value);
// + bool/uint64/double варианты
```

Запись — доступна только процессам с CS_PLATFORMIZED или с entitlement `platform-application`. Обычный твик это получит крайне редко, не считай эту часть доступной.

### 3.5. Внутренние (root-only, недоступные твикам)

```c
int jbclient_platform_set_process_debugged(uint64_t pid, bool fullyDebugged);
int jbclient_root_get_physrw(bool singlePTE, uint64_t *singlePTEAsidPtr);
int jbclient_root_sign_thread(mach_port_t threadPort);
int jbclient_root_get_sysinfo(xpc_object_t *sysInfoOut);
int jbclient_root_steal_ucred(uint64_t ucredToSteal, uint64_t *orgUcred);
int jbclient_root_set_mac_label(uint64_t slot, uint64_t label, uint64_t *orgLabel);
int jbclient_root_trustcache_info(xpc_object_t *infoOut);
```

Эти функции достижимы только из процессов, зарегистрированных в домене `JBS_DOMAIN_ROOT` (это `jailbreakd` и его клиенты). Автор стороннего твика их не вызывает.

## 4. Структура jbserver — домены

`BaseBin/libjailbreak/src/jbserver_domains.h` описывает «что видно из каких процессов»:

| Домен | Константа | Кому доступен |
|---|---|---|
| 1 — Systemwide | `JBS_DOMAIN_SYSTEMWIDE` | Все процессы |
| 2 — Platform | `JBS_DOMAIN_PLATFORM` | С CS_PLATFORMIZED, `platform-application` или сам Dopamine |
| 3 — Watchdog | `JBS_DOMAIN_WATCHDOG` | Только `watchdogd` |
| 4 — Root | `JBS_DOMAIN_ROOT` | Только root-процессы |
| ... | ... | ... |

Полная таблица — в шпаргалке `cheatsheets/07-jbserver-domains.md` (когда будет написана), но автору твика в 95% случаев нужны операции из `JBS_DOMAIN_SYSTEMWIDE`:

- `JBS_SYSTEMWIDE_GET_JBROOT` → `jbclient_get_jbroot`
- `JBS_SYSTEMWIDE_GET_BOOT_UUID` → `jbclient_get_boot_uuid`
- `JBS_SYSTEMWIDE_TRUST_FILE` → `jbclient_trust_file*`
- `JBS_SYSTEMWIDE_CS_REVALIDATE` → `jbclient_cs_revalidate`
- `JBS_SYSTEMWIDE_JBSETTINGS_GET` → `jbclient_jbsettings_get*`

## 5. Как твик физически подключает libjailbreak

Minimum viable Makefile:

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
THEOS_PACKAGE_SCHEME = roothide

INSTALL_TARGET_PROCESSES = SpringBoard

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = MyTweak
MyTweak_FILES = Tweak.xm
MyTweak_CFLAGS = -fobjc-arc
MyTweak_LIBRARIES = jailbreak    # <- ключевой момент

include $(THEOS_MAKE_PATH)/tweak.mk
```

Для подключения `libroot` (чаще — предпочтительный вариант):

```make
MyTweak_LIBRARIES = root
```

`libroot` — это тонкая обёртка над `jbclient_get_jbroot()`, которая не требует линковать всю `libjailbreak` (но её тоже можно, если нужны более низкоуровневые jbclient-функции).

В исходнике твик вызывает:

```c
#include <libjailbreak/jbclient_xpc.h>

NSString *jbRoot() {
    char *path = jbclient_get_jbroot();  // не забыть free() потом
    NSString *s = [NSString stringWithUTF8String:path];
    free(path);
    return s;
}
```

## 6. jbroot.h — макросы, которые работают без линковки

Самое красивое изобретение: `BaseBin/libjailbreak/src/jbroot.h` — заголовок **header-only**, который работает с макросами `JBROOT_PATH(...)` и `ROOTFS_PATH(...)`. Требует только определить `get_jbroot()` в своём коде. Удобно для твиков, которые не хотят зависеть от jbclient на этапе компиляции.

Полное использование с libroot:

```c
#include <libroot.h>

const char *path = JBROOT_PATH("/usr/share/mytweak/data.json");
// path теперь указывает на /var/<rand>/.jbroot/usr/share/mytweak/data.json
```

Для Objective-C есть `JBROOT_PATH_NSSTRING(...)` вариант.

## 7. info.h — сведения о среде

Внутри `BaseBin/libjailbreak/src/info.h` живут функции типа:

- `is_jailbreak_platformised()` — проверить, этот ли процесс platformised.
- Сведения о CPU family — через `machine_info.h`.

Для автора твика это удобно, если нужно сделать условный код «на A13 работаем так, на A14 — иначе» — хотя на практике для твика достаточно `sysctlbyname("hw.cpufamily", ...)` напрямую.

## 8. Что подтверждено

- **Подтверждено исходниками**: все перечисленные функции `jbclient_*` описаны в `BaseBin/libjailbreak/src/jbclient_xpc.h`. Домены — в `jbserver_domains.h`. Макросы `JBROOT_PATH/ROOTFS_PATH` — в `jbroot.h`.
- **Подтверждено Makefile**: library собирается с `-arch arm64 -arch arm64e` (поддерживает оба slice).
- **Требует проверки на устройстве**: конкретное поведение `jbclient_fork_fix` на iOS 26 — архитектура могла измениться.
- **Не подтверждено**: гарантии стабильности API между версиями Dopamine 3.0.x — проект не публикует semver-контракта.

## 9. Выводы

Главный тезис: **для стороннего твика `libjailbreak` — это `jbclient_*` + jbroot-макросы, и ничего больше**. Все, что ниже (физическая память, trust cache, kernel RW) — внутренние компоненты, стоящие за XPC-мостом jbserver. Твик, который зовёт `jbclient_get_jbroot` или `jbclient_trust_file_by_path`, работает корректно и живёт между версиями. Твик, который пытается подключить `physrw.h`, упадёт при первой сборке.

Таблица «что хочу → что звать» для автора:

| Хочу | Звать |
|---|---|
| Префикс jbroot | `jbclient_get_jbroot()` или макрос `JBROOT_PATH("...")` |
| Boot UUID | `jbclient_get_boot_uuid()` |
| Добавить cdhash своего `.dylib` в trust cache | `jbclient_trust_file_by_path("...")` |
| Прочитать настройку Dopamine | `jbclient_jbsettings_get_bool/uint64/double` |
| Применить любой root-primitive напрямую | **Нельзя** — через XPC через `jbclient_*` или вообще никак |

## 10. Что дальше

- Для `libroothider` (roothide-слоя над libjailbreak) — глава III.3.
- Для `launchdhook` (где живёт серверная часть jbserver) — глава III.4.
- Для `systemhook` (главного потребителя `jbclient_*`) — глава III.5.
- Для `trustcache` в деталях — Deep-dive 05.
- Для шпаргалки по доменам — `cheatsheets/07-jbserver-domains.md`.
