# Шпаргалка 7. jbserver домены

Из `BaseBin/libjailbreak/src/jbserver_domains.h`. Таблица «домен → ID → доступен → actions».

## Общая таблица

| Домен | ID | Доступ | Назначение |
|---|---|---|---|
| `JBS_DOMAIN_SYSTEMWIDE` | 1 | Все процессы | Общие операции для твиков |
| `JBS_DOMAIN_PLATFORM` | 2 | CS_PLATFORMIZED / platform-app / Dopamine | Операции платформы (запись settings) |
| `JBS_DOMAIN_WATCHDOG` | 3 | Только `watchdogd` | Userspace panic handling |
| `JBS_DOMAIN_ROOT` | 4 | Только root-процессы | Kernel primitives |
| `JBS_DOMAIN_ROOTHIDE` | ? | RootHide-specific | Randomized jbroot ops |
| `JBS_DOMAIN_DOPAMINE` | ? | Только Dopamine.app | App-specific actions |

## Domain 1: SYSTEMWIDE (ID = 1)

Доступен из любого процесса, где установлен `systemhook` (т. е. практически везде).

| Action | Константа | Client API |
|---|---|---|
| 1 | `JBS_SYSTEMWIDE_GET_JBROOT` | `jbclient_get_jbroot()` |
| 2 | `JBS_SYSTEMWIDE_GET_BOOT_UUID` | `jbclient_get_boot_uuid()` |
| 3 | `JBS_SYSTEMWIDE_TRUST_FILE` | `jbclient_trust_file(...)` / `jbclient_trust_file_by_path(...)` |
| 4 | `JBS_SYSTEMWIDE_PROCESS_CHECKIN` | `jbclient_process_checkin(...)` |
| 5 | `JBS_SYSTEMWIDE_FORK_FIX` | `jbclient_fork_fix(...)` |
| 6 | `JBS_SYSTEMWIDE_CS_REVALIDATE` | `jbclient_cs_revalidate()` |
| 7 | `JBS_SYSTEMWIDE_JBSETTINGS_GET` | `jbclient_jbsettings_get(...)` / `_bool/_uint64/_double` |
| 8 | `JBS_SYSTEMWIDE_PERSONA_FIX` | `jbclient_persona_fix(...)` |

## Domain 2: PLATFORM (ID = 2)

Требует одно из:

- Процесс запущен с entitlement `com.apple.private.security.container-required=false` + `platform-application=true`.
- Процесс помечен CS_PLATFORMIZED в codesign flags.
- Процесс — сама `Dopamine.app`.

| Action | Константа | Client API |
|---|---|---|
| 1 | `JBS_PLATFORM_SET_PROCESS_DEBUGGED` | `jbclient_platform_set_process_debugged(pid, flag)` |
| 2 | `JBS_PLATFORM_STAGE_JAILBREAK_UPDATE` | `jbclient_platform_stage_jailbreak_update(tar)` |
| 3 | `JBS_PLATFORM_JBSETTINGS_SET` | `jbclient_platform_jbsettings_set(key, value)` + `_bool/_uint64/_double` |
| 4 | `JBS_PLATFORM_SET_SYSTEMWIDE_DOMAIN_ENABLED` | нет публичной обёртки (закомментировано в header) |

## Domain 3: WATCHDOG (ID = 3)

Только для `watchdogd` (отдельный компонент basebin).

| Action | Константа | Client API |
|---|---|---|
| 1 | `JBS_WATCHDOG_INTERCEPT_USERSPACE_PANIC` | `jbclient_watchdog_intercept_userspace_panic(msg)` |
| 2 | `JBS_WATCHDOG_GET_LAST_USERSPACE_PANIC` | `jbclient_watchdog_get_last_userspace_panic(&out)` |

## Domain 4: ROOT (ID = 4)

Доступен только процессам, работающим от root. Это:

- `jailbreakd`.
- Специализированные helper'ы.
- **Не** обычные приложения, даже после твика.

| Action | Client API |
|---|---|
| get_physrw | `jbclient_root_get_physrw(bool singlePTE, uint64_t *asidPtr)` |
| sign_thread | `jbclient_root_sign_thread(thread_port)` |
| get_sysinfo | `jbclient_root_get_sysinfo(&out)` |
| steal_ucred | `jbclient_root_steal_ucred(ucred, &origOut)` |
| set_mac_label | `jbclient_root_set_mac_label(slot, label, &origOut)` |
| trustcache_info | `jbclient_root_trustcache_info(&out)` |

Эти функции делают прямые kernel-операции. Автор твика к ним не ходит.

## Domain 5: ROOTHIDE

Специфика RootHide. Детали — в `BaseBin/launchdhook/src/jbserver/jbdomain_roothide.c`.

Операции обычно связаны с path translation между randomized jbroot и rootfs.

## Domain 6: DOPAMINE

Только для `Dopamine.app`. Детали — в `jbdomain_dopamine.c`.

| Action (концептуально) | Назначение |
|---|---|
| `is_jailbroken` | Проверка «запущен ли jailbreak на устройстве» |
| App-level diagnostics | Чтение состояния для UI Dopamine |

## Проверка доступа

Как jbserver решает, пустить ли клиент:

1. Получить `audit_token_t` из mach-сообщения клиента.
2. Из audit token извлечь `pid`, `uid`, codesign flags.
3. По домену проверить:
   - SYSTEMWIDE — всегда yes.
   - PLATFORM — проверить CS_PLATFORMIZED или entitlement.
   - WATCHDOG — проверить, что это watchdogd (по пути бинарника).
   - ROOT — `uid == 0`.
   - ROOTHIDE — специфические правила.
   - DOPAMINE — проверить bundle id = `com.opa334.Dopamine` (или аналог).
4. Выполнить action.

Это обеспечивает, что `jbclient_root_*` функции, вызванные из обычного процесса, молча отвергаются (возвращают error).

## Пример: безопасный вызов из твика

```c
#include <libjailbreak/jbclient_xpc.h>

bool is_jailbreak_tweak_enabled(void) {
    // SYSTEMWIDE — доступно всем:
    bool enabled = jbclient_jbsettings_get_bool("tweaks-enabled");
    return enabled;
}
```

## Пример: что НЕ работает из обычного твика

```c
// ПЛОХО — не из PLATFORM-процесса:
int r = jbclient_platform_jbsettings_set_bool("tweaks-enabled", false);
// r != 0 — permission denied. jbserver отверг.
```

## См. также

- `BaseBin/libjailbreak/src/jbserver_domains.h` — полный файл.
- Шпаргалка 6 — Public functions.
- Глава III.2 — libjailbreak.
- Глава III.4 — launchdhook (где живёт серверная часть).
