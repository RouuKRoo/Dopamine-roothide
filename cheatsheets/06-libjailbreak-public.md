# Шпаргалка 6. Публичные функции libjailbreak для сторонних твиков

Из `BaseBin/libjailbreak/src/jbclient_xpc.h`. Таблица — какие функции безопасно вызывать из обычного твика, какие — нет.

## Включение

```c
#include <libjailbreak/jbclient_xpc.h>
```

В Makefile — опционально:

```make
MyTweak_LIBRARIES = jailbreak
# или через re-export libroot:
MyTweak_LIBRARIES = root
```

(`libjailbreak` re-exports `libroothider`, что подхватывается через `-lroot`.)

## Публичные для твика (JBS_DOMAIN_SYSTEMWIDE)

Домен 1 — доступен всем процессам.

### Получение информации

```c
char *jbclient_get_jbroot(void);
```
- Возвращает malloc'нутую строку. **Освободить `free()`**.
- Проще: используйте `libroot_get_jbroot_prefix()` (возвращает static).

```c
char *jbclient_get_boot_uuid(void);
```
- Аналогично. `free()`.
- Альтернатива: `libroot_get_boot_uuid()`.

### Trust cache

```c
int jbclient_trust_file_by_path(const char *path);
```
- Добавляет cdhash файла в dynamic trust cache.
- Возвращает: 0 при успехе.
- Применение: твик кладёт на диск доп. `.dylib` и хочет её `dlopen`-уть.

```c
int jbclient_trust_file(int fd, struct siginfo *siginfo, bool attach);
```
- Та же логика, но для открытого fd + готового `siginfo`.
- Чаще нужен внутренним компонентам Dopamine.

### Codesign revalidation

```c
int jbclient_cs_revalidate(void);
```
- Переустанавливает `CS_VALID` в текущем процессе.
- Автор твика **обычно не вызывает** — `systemhook` делает это сам в NECP хуках.

### Fork / persona

```c
int jbclient_fork_fix(uint64_t childPid);
int jbclient_persona_fix(int childPid, uid_t overwriteUid, gid_t overwriteGid);
```
- Фикс дочернего процесса после `fork()`.
- Автор твика не вызывает — делает Dopamine.

### Settings (чтение)

```c
int jbclient_jbsettings_get(const char *key, xpc_object_t *valueOut);
bool jbclient_jbsettings_get_bool(const char *key);
uint64_t jbclient_jbsettings_get_uint64(const char *key);
double jbclient_jbsettings_get_double(const char *key);
```
- Чтение глобальных jailbreak-настроек.
- Применение: твик уважает «tweaks enabled», safe mode и т. п.

Пример:

```c
bool tweaksEnabled = jbclient_jbsettings_get_bool("tweaks-enabled");
if (!tweaksEnabled) return;
```

### Checkin

```c
int jbclient_process_checkin(char **rootPathOut, char **bootUUIDOut,
                             char **sandboxExtensionsOut, bool *fullyDebuggedOut,
                             bool *forceCSAdhocOut);
```
- Автор твика **не вызывает** — делает `systemhook` на старте каждого процесса.

## Только для платформенных процессов (JBS_DOMAIN_PLATFORM)

Требуют CS_PLATFORMIZED или `platform-application` entitlement. Обычный твик не получит доступ.

```c
int jbclient_platform_set_process_debugged(uint64_t pid, bool fullyDebugged);
int jbclient_platform_stage_jailbreak_update(const char *updateTar);
int jbclient_platform_jbsettings_set(const char *key, xpc_object_t value);
int jbclient_platform_jbsettings_set_bool(const char *key, bool boolValue);
int jbclient_platform_jbsettings_set_uint64(const char *key, uint64_t uint64Value);
int jbclient_platform_jbsettings_set_double(const char *key, double doubleValue);
```

## Только для watchdogd (JBS_DOMAIN_WATCHDOG)

```c
int jbclient_watchdog_intercept_userspace_panic(const char *panicMessage);
int jbclient_watchdog_get_last_userspace_panic(char **panicMessage);
```

## Только для root-процессов (JBS_DOMAIN_ROOT)

Автор твика **не вызывает**:

```c
int jbclient_root_get_physrw(bool singlePTE, uint64_t *singlePTEAsidPtr);
int jbclient_root_sign_thread(mach_port_t threadPort);
int jbclient_root_get_sysinfo(xpc_object_t *sysInfoOut);
int jbclient_root_steal_ucred(uint64_t ucredToSteal, uint64_t *orgUcred);
int jbclient_root_set_mac_label(uint64_t slot, uint64_t label, uint64_t *orgLabel);
int jbclient_root_trustcache_info(xpc_object_t *infoOut);
```

## Низкоуровневые — не используйте

Эти — внутренние API Dopamine. Доступ есть, но вызов из твика сломает что-то:

| API | Файл | Почему не трогать |
|---|---|---|
| `kread64 / kwrite64` | `kernel.h` | kernel RW — только root |
| `physread / physwrite` | `physrw.h` | физическая память |
| `kcall(...)` | `kcall_*.h` | вызов kernel-функций |
| `trustcache_insert_*` | `trustcache.h` | напрямую — только root |
| `signatures_*` | `signatures.h` | внутренняя логика подписей |

Правильный путь: вызывайте `jbclient_*` (SYSTEMWIDE домен), не прямые kernel функции.

## jbroot.h макросы

(Из главы III.10 и шпаргалки 5.)

```c
#include <libjailbreak/jbroot.h>

JBROOT_PATH(path)           // с префиксом jbroot
ROOTFS_PATH(path)            // с префиксом rootfs
JBROOT_PATH_CSTRING(path)    // явный C-string
JBROOT_PATH_NSSTRING(path)   // явный NSString
```

Header-only, не требует линковки.

## Таблица «ваша задача → функция»

| Хочу... | Функция |
|---|---|
| Путь jbroot | `libroot_get_jbroot_prefix()` или `JBROOT_PATH(...)` |
| Boot UUID | `libroot_get_boot_uuid()` или `jbclient_get_boot_uuid()` |
| Добавить cdhash своей dylib | `jbclient_trust_file_by_path("/path/to/my.dylib")` |
| Прочитать настройку jailbreak | `jbclient_jbsettings_get_bool("key")` |
| Записать настройку jailbreak | нельзя (platform-only) |
| Вызвать функцию ядра | нельзя (root-only) |
| Писать в память другого процесса | через `hookd_hook(...)` (глава III.7) |

## Пример: подготовить свою dylib для `dlopen`

Если твик положил `/var/jb/Library/MyTweak/plugin.dylib` и хочет её загрузить:

```c
#include <libjailbreak/jbclient_xpc.h>
#include <libjailbreak/jbroot.h>
#include <dlfcn.h>

static void *load_my_plugin(void) {
    const char *path = JBROOT_PATH("/Library/MyTweak/plugin.dylib");

    // 1. Добавить в trust cache
    int tc = jbclient_trust_file_by_path(path);
    if (tc != 0) {
        NSLog(@"trust failed: %d", tc);
        return NULL;
    }

    // 2. Загрузить
    void *handle = dlopen(path, RTLD_NOW);
    if (!handle) {
        NSLog(@"dlopen failed: %s", dlerror());
        return NULL;
    }

    return handle;
}
```

Без `jbclient_trust_file_by_path` dyld откажется грузить `.dylib` (нет cdhash в trust cache → AMFI reject).

## См. также

- Глава III.2 — libjailbreak в деталях.
- Шпаргалка 5 — libroot API.
- Шпаргалка 7 — jbserver_domains.
- Глава III.11 — bootstrap process.
