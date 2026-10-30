# Приложение B. Справочник файлов и API проекта

Это карта проекта roothide/Dopamine 3.0.12 на уровне «что где лежит» и «какой API предоставляет». Служит быстрым поиском при разработке.

Разделён на три части:

1. **Файловая карта**: что в каком каталоге.
2. **Публичный API для твиков**: функции, которые разработчик сторонних твиков может легитимно использовать.
3. **Внутренние API**: функции, которые есть, но не предназначены для сторонних твиков.

Все пути — относительно корня репозитория `/home/user/Dopamine-roothide/` или аналога.

---

## 1. Файловая карта

### 1.1. Корень

| Путь | Назначение |
|---|---|
| `Makefile` | Главная сборочная цель (`make all`, `make update`, `make update-basebin`) |
| `BUILD.md` | Инструкции по сборке на GitHub Actions |
| `README.md` | Краткое описание и ссылки |
| `LICENSE.md` | Лицензия |
| `Dopamine.md` | Текст, выводимый пользователю |
| `.gitmodules` | Субмодули (ChOma, XPF, kfd, opainject, litehook) |
| `.github/workflows/` | GitHub Actions pipelines |
| `.pictures/` | Иконки/картинки UI |
| `sign_and_install.sh` | Утилита для dev-итерации |
| `sort_localizations.sh` | Служебный скрипт |

### 1.2. `Application/Dopamine/`

| Путь | Назначение |
|---|---|
| `main.m` | Точка входа приложения; содержит TrollStore-режим `delete-bootstrap` |
| `Info.plist` | Metadata приложения |
| `Dopamine.entitlements` | Entitlements |
| `Dopamine.xcodeproj/` | Xcode проект |
| `App Delegates/` | UI AppDelegate |
| `UI/` | Экраны пользователя |
| `UI/DOMainViewController.m` | Главный экран с кнопкой Jailbreak |
| `UI/DOUIManager.m` | Singleton UI |
| `UI/Settings/*` | Экраны настроек в Dopamine.app |
| `Jailbreak/DOEnvironmentManager.{h,m}` | Carrier среды. Строка 254 — поддержка версий |
| `Jailbreak/DOExploit.{h,m}` | Выбор эксплойта по CPU/версии |
| `Jailbreak/DOExploitManager.{h,m}` | Менеджер эксплойтов |
| `Jailbreak/DOBootstrapper.{h,m}` + `+zstd` | Разворачивание basebin |
| `Jailbreak/DOJailbreaker.{h,m}` | Главный flow jailbreak |
| `Jailbreak/DOPreferenceManager.{h,m}` | Настройки Dopamine.app |
| `Jailbreak/clock_alarm.c` | Вспомогательное время |
| `Exploits/` | Все эксплойты (DarkSword, ClearSword, kfd, momentarius, Titan, …) |
| `Headers/` | Переиспользуемые headers |
| `Resources/` | Resources (иконки, строки) |
| `Extensions/` | Расширения приложения |
| `*.lproj/` | Локализации (en, ru, de, …) |

### 1.3. `BaseBin/` — ключевые компоненты

| Путь | Назначение |
|---|---|
| `Makefile` | Сборка `basebin.tar` |
| `_external/basebin/.version` | Версия («3.0.12») |
| `_external/basebin/.version-roothide` | Версия RootHide-слоя («28») |
| `_external/basebin/prebuilt/` | Готовые бинарники (например, launchctl) |
| `_external/modules/litehook/` | Submodule litehook |
| `_external/include/` | Включаемые headers |
| `_external/frameworks/` | Пустые/служебные frameworks |
| `ChOma/` | Submodule для Mach-O |
| `XPF/` | Submodule patchfinder |
| `MachOMerger/` | Утилита склейки dyld+dyldhook |
| `opainject/` | Submodule инжектора dylib |
| `libjailbreak/src/` | Главная библиотека, см. 1.4 |
| `libroothider/src/` | RootHide-слой, см. 1.5 |
| `systemhook/src/` | Hook в каждом процессе, см. 1.6 |
| `launchdhook/src/` | Hook в launchd, см. 1.7 |
| `dyldhook/src/` | Hook в dyld, см. 1.8 |
| `hookd/src/` | Демон записи в чужой процесс, см. 1.9 |
| `jailbreakd/src/` | Привилегированный координатор, см. 1.10 |
| `rootlesshooks/*.x` | Hooks системных процессов (rootless) |
| `roothidehooks/src/*.x` | Hooks системных процессов (roothide) |
| `bootstrapper/src/main.m` | Пустая заглушка (будущая логика) |
| `forkfix/` | Fix для fork() |
| `watchdoghook/` | Hook в watchdogd |
| `boomerang/` | Механизм восстановления упавших компонентов |
| `jbctl/` | CLI утилита |
| `idownloadd/` | Загрузчик обновлений |
| `libintl.8.dylib` | Prebuilt lib |

### 1.4. `BaseBin/libjailbreak/src/`

| Файл | Назначение |
|---|---|
| `libjailbreak.h` | Umbrella include |
| `jbclient_xpc.{h,c}` | XPC-клиент к jbserver — **публичный для твиков** |
| `jbclient_mach.{h,c}` | Mach-транспорт клиента |
| `jbclient_roothide.c` | RootHide-специфичные вызовы |
| `jbserver_domains.h` | Нумерация доменов (SYSTEMWIDE, PLATFORM, …) |
| `jbserver.{h,c}` | Внутренняя серверная часть |
| `jbserver_boomerang.{h,c}` | Boomerang integration |
| `info.{h,c}` | Информация о среде |
| `jbroot.{h,c}` | **Макросы JBROOT_PATH / ROOTFS_PATH** — публичные |
| `codesign.{h,c,m}` | Операции с подписью |
| `signatures.{h,c}` | Структуры подписи |
| `trustcache.{h,c}` | Trust cache primitives |
| `trustcache_fs.{h,c}` | FS-based trust cache |
| `trustcache_structs.h` | Структуры TC |
| `physrw.{h,c}` | Physical memory R/W (внутренний!) |
| `physrw_pte.{h,c}` | PTE-level operations |
| `pte.h` | PTE константы |
| `pvh.h` | PVH константы |
| `kcall_Fugu14.{h,c,S}` | Fugu14-style kcall |
| `kcall_arm64.{h,c}` | Generic arm64 kcall |
| `inline_svc.{h,c,S}` | Прямые syscalls |
| `machine_info.h` | CPU constants |
| `kernel.{h,c}` | Kernel R/W |
| `primitives.{h,c}` | High-level primitives |
| `primitives_IOSurface.{h,m}` | IOSurface-based primitives |
| `primitives_external.h` | External primitives interface |
| `translation.{h,c}` | Rosetta/translation |
| `stock_fixes.{h,c}` | «Фиксы для стандартных багов» |
| `hookd.{h,c}` | **API к hookd** (глава III.7) |
| `txm.{h,c}` | TXM (arm64e only) |
| `basebin_gen.{h,m}` | Служебное |
| `carboncopy.{h,m}` | Служебное |
| `display.{h,m}` | Boot logo rendering |
| `util.{h,c,m}` | Утилиты |
| `main.c` | Entry (standalone build) |

### 1.5. `BaseBin/libroothider/src/`

| Файл | Назначение |
|---|---|
| `libroothider.h` | Umbrella |
| `common.{h,m}` | Утилиты (getAppBundlePathFromSpawnPath, …) |
| `log.{h,c}` | **JBLog* макросы** — публичные для твиков |
| `crashreporter.{h,m}` | Crash reporter для internal компонентов |
| `blacklist.{cpp,m}` | Blacklist процессов |
| `dyld_patch.m` | Внутренние операции над dyld |
| `exec_patch.{h,m}`, `exec_trace.m` | Перехват exec* |
| `jailbreakd.{h,c}` | Client к jailbreakd демону |
| `launchd.m` | Specialized launchd hooks |
| `mach_exc.defs`, `mach_exc.h` | Mach exception |
| `ptrace.h` | Ptrace константы |
| `recdhash.m` | Recovery cdhash |
| `signatures.m` | Расширение libjailbreak/signatures |
| `unsandbox.h`, `unsandbox1.m`, `unsandbox2.m` | Внутренняя sandbox-операция |
| `xpc_hook.m`, `xpc_private.h` | XPC-хуки для внутренних компонентов |
| `xpf.m` | XPF интерфейс |

### 1.6. `BaseBin/systemhook/src/`

| Файл | Назначение |
|---|---|
| `main.c` | Главный файл; csops/NECP/dlsym/TweakLoader hooks |
| `common/` | Общие headers (`common.h`, `hookd_external.h`, `private.h`, `inline.h`) |
| `roothider.h`, `roothider_main.c`, `roothider_common.c` | RootHide-специфичные хуки |
| `roothider_inline.x` | Inline-x (compiles separately, no stdlib deps) |
| `upload.sh` | Dev-итерационный скрипт |

### 1.7. `BaseBin/launchdhook/src/`

| Файл | Назначение |
|---|---|
| `main.m` | Главный entry |
| `spawn_hook.{h,c}` | **Hook `__posix_spawn`** |
| `xpc_hook.{h,c}` | XPC-инфраструктура |
| `daemon_hook.{h,m}` | Hook загрузки демонов |
| `ipc_hook.{h,c}` | IPC-хуки |
| `jetsam_hook.{h,c}` | Защита launchd от jetsam |
| `crashreporter.{h,m}` | Crash reporter |
| `boomerang.{h,c}` | Boomerang client |
| `update.{h,m}` | Update API |
| `hookd_provider.{h,c}` | Hookd integration |
| `asl.h` | ASL hack |
| `roothider.m` | RootHide specifics |
| `jbserver/jbserver_global.{h,c}` | Серверная часть jbserver (глобальный) |
| `jbserver/jbserver_local.{h,c}` | Серверная часть (локальная) |
| `jbserver/jbserver_mach.c` | Mach transport |
| `jbserver/jbdomain_*.c` | Обработчики доменов (systemwide, platform, root, roothide, watchdog, dopamine) |
| `jbserver/jbsettings.{h,c}` | Settings backend |

### 1.8. `BaseBin/dyldhook/src/`

| Файл | Назначение |
|---|---|
| `main.c`, `main.S` | Главный entry (встраивается в dyld) |
| `dyld.h` | Internal structures |
| `dyld_jbinfo.h` | `struct dyld_jbinfo` |
| `fakelib_redirect.c` | `/usr/lib → fakelib` редирект (сейчас `#if 0`) |
| `lv_bypass.c` | **Library validation bypass** через `fcntl` hook |
| `spinlock_fix.c` | Fix для spinlock-deadlock |
| `machomerger_hook.h` | HOOK/ORIG макросы |
| `reimpl.c` | Reimplementation `memcpy`/`strlen`/… для dyld-контекста |
| `roothider.S`, `roothider.c` | RootHide specifics |
| `generated/` | Сгенерированные headers |

### 1.9. `BaseBin/hookd/src/`

| Файл | Назначение |
|---|---|
| `main.c` | Весь демон: `apply_hook`, `apply_fixup`, `server_loop` |

### 1.10. `BaseBin/jailbreakd/src/`

| Файл | Назначение |
|---|---|
| `main.m` | Entry point, инициализация primitives |
| `server.m` | Серверная часть: `jailbreakd_received_message` + обработчики |

### 1.11. `BaseBin/rootlesshooks/` и `BaseBin/roothidehooks/src/`

| Файл | Процесс | Назначение |
|---|---|---|
| `rootlesshooks/main.x` | all | Dispatcher по имени процесса |
| `rootlesshooks/SpringBoard.x` | SpringBoard | Snapshot path redirect + fcntl patch |
| `rootlesshooks/cfprefsd.x` | cfprefsd | `_CFPrefsGetPathForTriplet` redirect |
| `rootlesshooks/installd.x` | installd | Закомментировано — bootloop risk |
| `rootlesshooks/lsd.x` | lsd | Inbox URL redirect + uicache |
| `roothidehooks/src/main.x` | all | RootHide dispatcher |
| `roothidehooks/src/common.{h,m}` | — | Утилиты |
| `roothidehooks/src/springboard.x` | SpringBoard | RootHide-specifics |
| `roothidehooks/src/cfprefsd.x` | cfprefsd | RootHide path |
| `roothidehooks/src/lsd.x` | lsd | URL scheme hiding |
| `roothidehooks/src/runningboardd.x` | runningboardd | r^x path fix |
| `roothidehooks/src/pathhook.x` | all | `_CFCopyHomeDirURLForUser` redirect |
| `roothidehooks/src/palera1n.x` | — | palera1n compatibility |

### 1.12. `Packages/`

| Путь | Назначение |
|---|---|
| `Makefile` | Собирает libroot, libkrw-provider, basebin-link |
| `libroot/control` | Debian package info |
| `libroot/Makefile` | Сборка `libroot.dylib` |
| `libroot/src/paths.{h,c}` | **3 функции libroot API — публичные** |
| `libkrw-provider/` | libkrw провайдер |
| `basebin-link/` | Пакет симлинка `/basebin` |
| `additional/launchctl_*.deb` | Prebuilt launchctl |

### 1.13. `Standalone/` и прочие

| Путь | Назначение |
|---|---|
| `Standalone/` | Standalone-сборка Dopamine (без App) |
| `Standalone/Corellium/` | Varianta для Corellium virtualization |

---

## 2. Публичный API для сторонних твиков

Эти API подходят для стороннего твика без оговорок.

### 2.1. libroot (3 функции)

Включение: `#include <libroot.h>`; линковка `-lroot`; зависимость `libroot-dopamine` в control.

```c
const char *libroot_get_root_prefix(void);
```
- Возвращает: `""` (пустая строка).
- Назначение: префикс настоящей rootfs. На jailbroken устройстве — не переадресуется, поэтому пусто.
- Собственность: статическая строка, не освобождать.

```c
const char *libroot_get_jbroot_prefix(void);
```
- Возвращает: префикс jbroot. На rootless — `/var/jb`. На RootHide — `/var/<rand>/<rand>/.jbroot`.
- Назначение: правильный способ узнать корень jailbreak-окружения.
- Собственность: статическая, не освобождать.

```c
const char *libroot_get_boot_uuid(void);
```
- Возвращает: UUID текущего cold-boot сеанса.
- Назначение: идентификация «переживания» операций через userspace reboot.
- Собственность: статическая.

### 2.2. jbroot.h макросы (header-only)

Включение: `#include <libjailbreak/jbroot.h>` (из `.include/libjailbreak/jbroot.h`).

```c
JBROOT_PATH(path)
```
- Принимает: `const char *` или `NSString *`.
- Возвращает: тот же тип, с префиксом jbroot.
- Работает: inline, без allocations в heap (использует alloca).
- Требует: `get_jbroot()` реализован (в systemhook — есть; в твике — опционально через libjailbreak).

```c
ROOTFS_PATH(path)
```
- Аналогично, для rootfs-view.

```c
JBROOT_PATH_NSSTRING(path)
JBROOT_PATH_CSTRING(path)
```
- Явные варианты для NSString / C-string.

### 2.3. jbclient_xpc.h — publикный клиент к jbserver

Включение: `#include <libjailbreak/jbclient_xpc.h>`; линковка `-ljailbreak` (или через re-export libroothider).

#### 2.3.1. Информация

```c
char *jbclient_get_jbroot(void);
```
- Возвращает: malloc'нутую строку с путём jbroot.
- **Освободить**: `free()` после использования.

```c
char *jbclient_get_boot_uuid(void);
```
- Аналогично.

#### 2.3.2. Trust cache

```c
int jbclient_trust_file(int fd, struct siginfo *siginfo, bool attach);
int jbclient_trust_file_by_path(const char *path);
```
- Назначение: добавить cdhash файла в dynamic trust cache.
- Возвращает: 0 при успехе, ненулевое при ошибке.
- Применение: когда твик кладёт на диск доп. `.dylib` и хочет её `dlopen`-ить.

#### 2.3.3. Process checkin

```c
int jbclient_process_checkin(char **rootPathOut, char **bootUUIDOut,
                             char **sandboxExtensionsOut, bool *fullyDebuggedOut,
                             bool *forceCSAdhocOut);
```
- Назначение: регистрация процесса в jbserver.
- Автор твика обычно **не вызывает** — делает `systemhook`.

#### 2.3.4. Codesign

```c
int jbclient_cs_revalidate(void);
```
- Назначение: переустановить `CS_VALID` в текущем процессе.
- Применение: вызывается `systemhook` при NECP syscalls (глава III.5). Автору твика обычно не нужно.

#### 2.3.5. Fork fix и persona fix

```c
int jbclient_fork_fix(uint64_t childPid);
int jbclient_persona_fix(int childPid, uid_t overwriteUid, gid_t overwriteGid);
```
- Назначение: корректное обслуживание дочернего процесса.
- Автор твика обычно не вызывает.

#### 2.3.6. jbsettings

```c
int jbclient_jbsettings_get(const char *key, xpc_object_t *valueOut);
bool jbclient_jbsettings_get_bool(const char *key);
uint64_t jbclient_jbsettings_get_uint64(const char *key);
double jbclient_jbsettings_get_double(const char *key);
```
- Назначение: чтение глобальных настроек jailbreak.
- Применение: твик уважает «tweaks enabled», «safe mode» и т. п.

```c
int jbclient_platform_jbsettings_set(const char *key, xpc_object_t value);
int jbclient_platform_jbsettings_set_bool(const char *key, bool boolValue);
// + _uint64, _double
```
- Назначение: запись настроек.
- Доступно: только из CS_PLATFORMIZED процессов или с entitlement `platform-application`. Обычный твик не получит.

#### 2.3.7. Debug flag

```c
int jbclient_platform_set_process_debugged(uint64_t pid, bool fullyDebugged);
```
- Назначение: установить флаг CS_DEBUGGED у процесса (для debugserver).
- Доступ: platform-only.

#### 2.3.8. Update

```c
int jbclient_platform_stage_jailbreak_update(const char *updateTar);
```
- Назначение: подкинуть новый `basebin.tar` на обновление.
- Доступ: platform-only.

#### 2.3.9. Watchdog

```c
int jbclient_watchdog_intercept_userspace_panic(const char *panicMessage);
int jbclient_watchdog_get_last_userspace_panic(char **panicMessage);
```
- Назначение: обработка userspace panic.
- Доступ: только `watchdogd`.

### 2.4. libroothider/log.h — JBLog* макросы

Включение: `#include <libroothider/log.h>`.

```c
JBLogDebug(fmt, ...)
JBLogError(fmt, ...)
JBLog(fmt, ...)
```
- Назначение: диагностический лог с префиксом jailbreak.
- Применение: отладка твика в режиме «включите JBLOG_LEVEL=DEBUG».

### 2.5. hookd API

Включение: `#include <libjailbreak/hookd.h>`.

```c
kern_return_t hookd_hook(mach_port_t taskPort, uint64_t address,
                         uint8_t *data, size_t dataSize);
```
- Назначение: записать `data` по адресу `address` в процесс с таск-портом `taskPort`.
- Применение: редкие случаи записи в чужой процесс (глава III.7).

```c
kern_return_t hookd_vm_protect(mach_port_t taskPort, vm_address_t address,
                               vm_size_t size, bool set_maximum, vm_prot_t new_protection);
```
- Назначение: `vm_protect` в чужом процессе.

```c
int hookd_send_requests(task_port_t taskPort,
                        struct hookd_hook *hooks, int hooksCount, int *hookResultsOut,
                        struct hookd_encoded_fixup *fixups, int fixupsCount, int *fixupResultsOut);
```
- Назначение: пакетная отправка нескольких hooks + fixups в одном сообщении.

---

## 3. Внутренние API — не для сторонних твиков

Эти функции есть, но их прямое использование не предусмотрено и может сломаться между версиями.

### 3.1. Kernel RW (libjailbreak/kernel.h)

```c
uint64_t kread64(uint64_t kaddr);
void kwrite64(uint64_t kaddr, uint64_t value);
int kread_virtual(uint64_t kaddr, void *buf, size_t size);
int kwrite_virtual(uint64_t kaddr, const void *buf, size_t size);
```

Доступ: только root-процессы в домене `JBS_DOMAIN_ROOT`.

### 3.2. Physical memory (libjailbreak/physrw.h, physrw_pte.h)

```c
int physwrite(uint64_t paddr, const void *data, size_t size);
int physread(uint64_t paddr, void *data, size_t size);
// + physrw_pte variants
```

Доступ: только внутренние компоненты Dopamine.

### 3.3. Kernel call (libjailbreak/kcall_*.h)

```c
uint64_t kcall(uint64_t func_addr, ...);
```

Доступ: только внутренние.

### 3.4. jbserver_root domain

```c
int jbclient_root_get_physrw(bool singlePTE, uint64_t *singlePTEAsidPtr);
int jbclient_root_sign_thread(mach_port_t threadPort);
int jbclient_root_get_sysinfo(xpc_object_t *sysInfoOut);
int jbclient_root_steal_ucred(uint64_t ucredToSteal, uint64_t *orgUcred);
int jbclient_root_set_mac_label(uint64_t slot, uint64_t label, uint64_t *orgLabel);
int jbclient_root_trustcache_info(xpc_object_t *infoOut);
```

Доступ: только root-процессы.

### 3.5. Inline SVC primitives (libjailbreak/inline_svc.h)

Используется для прямых syscall мимо хуков. Внутренняя магия.

---

## 4. Таблица jbserver_domains

Из `BaseBin/libjailbreak/src/jbserver_domains.h`:

| Домен | ID | Доступен | Основные actions |
|---|---|---|---|
| `JBS_DOMAIN_SYSTEMWIDE` | 1 | все процессы | `GET_JBROOT`, `GET_BOOT_UUID`, `TRUST_FILE`, `PROCESS_CHECKIN`, `FORK_FIX`, `CS_REVALIDATE`, `JBSETTINGS_GET`, `PERSONA_FIX` |
| `JBS_DOMAIN_PLATFORM` | 2 | CS_PLATFORMIZED / `platform-application` / Dopamine сам | `SET_PROCESS_DEBUGGED`, `STAGE_JAILBREAK_UPDATE`, `JBSETTINGS_SET`, `SET_SYSTEMWIDE_DOMAIN_ENABLED` |
| `JBS_DOMAIN_WATCHDOG` | 3 | только `watchdogd` | `INTERCEPT_USERSPACE_PANIC`, `GET_LAST_USERSPACE_PANIC` |
| `JBS_DOMAIN_ROOT` | 4 | только root-процессы | kernel primitives, trustcache info |
| `JBS_DOMAIN_ROOTHIDE` | ? | roothide-specifics | path translation, hiding helpers |

---

## 5. Утилиты устройства

| Утилита | Путь | Назначение |
|---|---|---|
| `jbctl` | `<jbroot>/basebin/jbctl` | Управление: `update basebin`, `update tipa`, `trustcache ...` |
| `uicache` | `<jbroot>/usr/bin/uicache` | Регистрация .app в SpringBoard |
| `launchctl` | `<jbroot>/bin/launchctl` | Управление launchd services |
| `dpkg` | `<jbroot>/usr/bin/dpkg` | Debian package manager |
| `ldid` | `<jbroot>/usr/bin/ldid` | Ad-hoc подпись |
| `opainject` | `<jbroot>/basebin/opainject` | Инжекция в живой процесс |
| `MachOMerger` | `<jbroot>/basebin/MachOMerger` | Склейка Mach-O |

---

## 6. Пути файловой системы

| Назначение | Путь на устройстве (rootless/RootHide) |
|---|---|
| basebin компоненты | `<jbroot>/basebin/*` |
| TweakLoader | `<jbroot>/usr/lib/TweakLoader.dylib` |
| libroot.dylib | `<jbroot>/usr/lib/libroot.dylib` |
| libellekit.dylib | `<jbroot>/usr/lib/libellekit.dylib` |
| Твики | `<jbroot>/Library/MobileSubstrate/DynamicLibraries/*.dylib` + `*.plist` |
| PreferenceBundles | `<jbroot>/Library/PreferenceBundles/*.bundle` |
| PreferenceLoader entries | `<jbroot>/Library/PreferenceLoader/Preferences/*.plist` |
| LaunchDaemons | `<jbroot>/Library/LaunchDaemons/*.plist` |
| Jailbreak-apps | `<jbroot>/Applications/*.app` |
| Preferences для не-apple | `<jbroot>/var/mobile/Library/Preferences/*.plist` |
| basebin.tar staging (на устройстве при обновлении) | `/var/mobile/Documents/basebin.tar` |
| CrashReporter | `/var/mobile/Library/Logs/CrashReporter/` + `<jbroot>/private/var/mobile/Library/Logs/CrashReporter/` |

---

## 7. Что подтверждено

Всё в этом приложении проверено по актуальному репозиторию:

- Файловая карта — `ls -R` и `find`.
- Публичные API — по header файлам `BaseBin/libjailbreak/src/*.h`.
- Внутренние API — по сопутствующим `.c`/`.m` файлам.
- jbserver_domains — `BaseBin/libjailbreak/src/jbserver_domains.h`.

При изменениях в репозитории — используйте это приложение как ориентир, а не source of truth; верифицируйте текущим `grep`/`find`.

---

## 8. Использование

**Задача «нужна функция X»** → ищите в разделе 2 (публичный API).

**Задача «что в этом файле?»** → ищите в разделе 1.

**Задача «куда писать?»** → раздел 6.

**Задача «можно ли вызвать Y из твика?»** → если Y в разделе 2 — да; если в разделе 3 — нет.
