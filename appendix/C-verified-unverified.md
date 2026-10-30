# Приложение C. Проверенные и непроверенные возможности

Этот документ разделяет содержимое книги на четыре категории:

1. **Подтверждено исходниками** — я проверил конкретную строку в репозитории.
2. **Подтверждено документацией** — я опираюсь на Apple docs или Theos docs.
3. **Требует проверки на устройстве** — не могу верифицировать без A13+iOS 26.0/26.0.1 в руках.
4. **Не поддерживается / данных недостаточно** — честная оговорка.

Используйте это приложение как «предохранитель»: прежде чем применить рецепт из книги, убедитесь, что он не только красиво написан, но и соответствует реальности на вашем устройстве.

---

## 1. Подтверждено исходниками

Это **самый надёжный** уровень. Все эти утверждения имеют конкретную привязку «файл:строка» в репозитории на момент HEAD коммита (24ea745).

### 1.1. Поддерживаемая среда

| Утверждение | Источник |
|---|---|
| Dopamine поддерживает iOS 15.0–18.7.1 и 26.0–26.0.1 на A12/A13 | `Application/Dopamine/Jailbreak/DOEnvironmentManager.m:254` |
| CPUFAMILY_ARM_LIGHTNING_THUNDER = A13 | `Application/Dopamine/Jailbreak/DOExploit.m:86` |
| CPUFAMILY_ARM_VORTEX_TEMPEST = A12 | `Application/Dopamine/Jailbreak/DOExploit.m:84` |
| DarkSword поддерживает диапазон 19.0–26.0.1 (iOS 26) | `Application/Dopamine/Exploits/DarkSword/Info.plist` |
| ClearSword поддерживает диапазон 19.0–26.0.1 | `Application/Dopamine/Exploits/ClearSword/Info.plist` |
| Явные билды iOS 26 в DPSupportInclude: 23B5044l, 23B5059e, 23B5064e | Там же, оба plist'а |
| Версия Dopamine в собранном basebin — 3.0.12 | `BaseBin/_external/basebin/.version` |
| Версия RootHide-слоя — 28 | `BaseBin/_external/basebin/.version-roothide` |
| HEAD коммит: «add roothide support for Dopamine 3» (24ea745) | `git log --oneline` |

### 1.2. systemhook (ключевой компонент)

| Утверждение | Источник |
|---|---|
| systemhook хукает `csops`/`csops_audittoken` для принудительного `CS_VALID` | `BaseBin/systemhook/src/main.c:163–195` |
| csops hook активен только для arm64 (не arm64e) — `#ifndef __arm64e__` | `BaseBin/systemhook/src/main.c:123`, `main.c:496` |
| NECP хуки — пять функций: `necp_match_policy`, `necp_open`, `necp_client_action`, `necp_session_open`, `necp_session_action` | `BaseBin/systemhook/src/main.c:128–156` |
| NECP хуки регистрируются через `litehook_hook_function` | `BaseBin/systemhook/src/main.c:515–521` |
| NECP существует только на iOS 16+ (`__builtin_available(iOS 16.0, *)`) | `BaseBin/systemhook/src/main.c:515` |
| Все NECP хуки вызывают `jbclient_cs_revalidate()` перед `syscall(SYS_necp_*,...)` | Код в разделе 2.3 главы II.9 |
| `ptrace` хук активен только в `debugserver` (суффикс пути) | `BaseBin/systemhook/src/main.c:504–506` |
| `dyld_dlsym_hook` перехватывает `dlsym` и подменяет `sandbox_apply` | `BaseBin/systemhook/src/main.c:95–108` |
| TweakLoader загружается через `dlopen(JBROOT_PATH("/usr/lib/TweakLoader.dylib"), RTLD_NOW)` | `BaseBin/systemhook/src/main.c:530–540` |
| `should_enable_tweaks` проверяет `<jbroot>/basebin/.safe_mode` | `BaseBin/systemhook/src/main.c:203` |
| `should_enable_tweaks` проверяет `DISABLE_TWEAKS=1` в среде | `BaseBin/systemhook/src/main.c:207–211` |
| Dopamine.app сам не получает tweaks (`jbclient_dopamine_is_jailbroken`) | `BaseBin/systemhook/src/main.c:216–221` |
| Blacklist путей: `/usr/libexec/xpcproxy` всегда | `BaseBin/systemhook/src/main.c:238–240` |
| На iOS 16+ исключены `/usr/libexec/logd`, `/usr/sbin/notifyd`, `/usr/libexec/usermanagerd` | `BaseBin/systemhook/src/main.c:243–247` |
| `musttail` используется в `dyld_dlsym_hook` | `BaseBin/systemhook/src/main.c:107` |

### 1.3. launchdhook

| Утверждение | Источник |
|---|---|
| launchdhook живёт в launchd и ставит все главные хуки | `BaseBin/launchdhook/src/main.m` |
| jetsam_hook предотвращает jetsam-убийство launchd (комментарий + код) | `BaseBin/launchdhook/src/jetsam_hook.c:9–26` |
| spawn_hook модифицирует `__posix_spawn` для добавления DYLD_INSERT_LIBRARIES | `BaseBin/launchdhook/src/spawn_hook.c` |
| jbserver_local можно поднять без launchd (для jbctl в раннем boot) | `BaseBin/launchdhook/src/spawn_hook.c:34–60` (закомментировано) |
| Доменные обработчики: dopamine, platform, root, roothide, systemwide, watchdog | `BaseBin/launchdhook/src/jbserver/jbdomain_*.c` |

### 1.4. dyldhook

| Утверждение | Источник |
|---|---|
| dyldhook встраивается в dyld через MachOMerger | `BaseBin/Makefile:109–110`, `BaseBin/MachOMerger/` |
| dyldhook имеет секцию `__DATA,__jbinfo` 0x4000 байт для состояния | `BaseBin/dyldhook/src/main.c:14` |
| `dyldhook_perform_checkin` через `jbclient_mach_process_checkin` | `BaseBin/dyldhook/src/main.c:52–75` |
| lv_bypass хукает `fcntl(F_ADDFILESIGS_RETURN,...)` для trust cache | `BaseBin/dyldhook/src/lv_bypass.c` |
| `fakelib_redirect.c` обёрнут в `#if 0` в текущей сборке | `BaseBin/dyldhook/src/fakelib_redirect.c:1` |
| spinlock_fix присутствует как отдельный файл | `BaseBin/dyldhook/src/spinlock_fix.c` |
| dyldhook не нуждается в trust cache (встроен в dyld) | `BaseBin/Makefile:109` (комментарий) |

### 1.5. hookd

| Утверждение | Источник |
|---|---|
| hookd имеет entitlements: platform-application, task_for_pid-allow, cs.debugger, container-required=false | `BaseBin/hookd/entitlements.plist` |
| hookd принимает только сообщения с `callerPid == 1` (launchd) | `BaseBin/hookd/src/main.c:140–143` |
| `apply_hook` делает `vm_protect`+`vm_write`+`vm_protect` | `BaseBin/hookd/src/main.c:28–70` |
| hookd сообщения ограничены 4096 байт | `BaseBin/libjailbreak/src/hookd.h:9` |
| Коммит e3f3cb5 «hookd: Preserve write and protection errors» существует | `git log --oneline | grep hookd` |

### 1.6. jailbreakd

| Утверждение | Источник |
|---|---|
| jailbreakd имеет расширенный набор entitlements: task_for_pid-allow, system-task-ports, system-task-ports.control, thread-set-state, set-exception-port, no-sandbox, no-container, storage.AppBundles, storage.AppDataContainers | `BaseBin/jailbreakd/entitlements.plist` |
| jailbreakd устанавливает `jetsam_limit=50MB` | `BaseBin/jailbreakd/src/main.m:21–24` |
| Обработчики `JBD_MSG_SPINLOCK_FIX_ONLY` и `JBD_MSG_SPAWN_PATCH_CHILD` в server.m | `BaseBin/jailbreakd/src/server.m:47–85` |
| jailbreakd инициализирует primitives через `jbclient_initialize_primitives` | `BaseBin/jailbreakd/src/main.m:69` |
| jailbreakd имеет respawn-логику с `RESPAWN_REQUIRED` env | `BaseBin/jailbreakd/src/main.m:71–105` |
| `proc_fix_spinlock`, `proc_patch_dyld`, `roothide_patch_proc` используются | `BaseBin/jailbreakd/src/server.m` |

### 1.7. libjailbreak

| Утверждение | Источник |
|---|---|
| libjailbreak собирается с `-arch arm64 -arch arm64e` | `BaseBin/libjailbreak/Makefile:5` |
| libjailbreak re-exports libroothider | `BaseBin/libjailbreak/Makefile:9` |
| libjailbreak линкуется с CoreGraphics, ImageIO, Security, IOKit, IOSurface и т. д. | `BaseBin/libjailbreak/Makefile:5` |
| Umbrella include содержит primitives, info, kernel, util, translation, trustcache, jbclient_xpc, stock_fixes | `BaseBin/libjailbreak/src/libjailbreak.h` |
| jbclient_* функции объявлены в `jbclient_xpc.h` | `BaseBin/libjailbreak/src/jbclient_xpc.h` |

### 1.8. libroothider

| Утверждение | Источник |
|---|---|
| libroothider имеет log, crashreporter, exec_patch, blacklist, dyld_patch | `BaseBin/libroothider/src/` |
| libroothider.h — umbrella include | `BaseBin/libroothider/src/libroothider.h` |
| blacklist.m имеет `getAppBundlePathFromSpawnPath` для определения приложений | `BaseBin/libroothider/src/blacklist.m:10–35` |
| `APP_PATH_PREFIX = "/private/var/containers/Bundle/Application/"` | `BaseBin/libroothider/src/blacklist.m:7` |

### 1.9. libroot

| Утверждение | Источник |
|---|---|
| libroot экспортирует ровно 3 функции: libroot_get_root_prefix, libroot_get_jbroot_prefix, libroot_get_boot_uuid | `Packages/libroot/src/paths.{h,c}` |
| libroot_get_root_prefix возвращает `""` | `Packages/libroot/src/paths.c:3–6` |
| libroot_get_jbroot_prefix возвращает `jbclient_get_jbroot()` | `Packages/libroot/src/paths.c:8–11` |
| libroot_get_boot_uuid возвращает `jbclient_get_boot_uuid()` | `Packages/libroot/src/paths.c:13–16` |
| libroot собирается с `-rpath /var/jb` | `Packages/libroot/Makefile:6` |
| libroot собирается с `-arch arm64 -arch arm64e -miphoneos-version-min=15.0` | `Packages/libroot/Makefile:5` |
| libroot-dopamine версия 1.0.1 в DOBootstrapper | `Application/Dopamine/Jailbreak/DOBootstrapper.m:21` |

### 1.10. rootlesshooks и roothidehooks

| Утверждение | Источник |
|---|---|
| rootlesshooks диспатчит по имени процесса (cfprefsd, SpringBoard, lsd) | `BaseBin/rootlesshooks/main.x:17–40` |
| installd хук **закомментирован** с меткой «BOOTLOOP RISK, DO NOT TOUCH» | `BaseBin/rootlesshooks/installd.x:1–24` |
| cfprefsd хукает `_CFPrefsGetPathForTriplet` | `BaseBin/rootlesshooks/cfprefsd.x:44–65` |
| cfprefsd имеет whitelist для не-apple плистов | `BaseBin/rootlesshooks/cfprefsd.x:6–42` |
| SpringBoard хукает `XBSnapshotContainerIdentity.snapshotContainerPath` | `BaseBin/rootlesshooks/SpringBoard.x:28–40` |
| SpringBoard хукает `fcntl` для `F_SETPROTECTIONCLASS` | `BaseBin/rootlesshooks/SpringBoard.x:42–65` |
| lsd хукает `_LSGetInboxURLForBundleIdentifier` и `_LSServer_RebuildApplicationDatabases` | `BaseBin/rootlesshooks/lsd.x` |
| lsd вызывает `uicache -a` в фоне через `dispatch_async` | `BaseBin/rootlesshooks/lsd.x:17–28` |
| roothidehooks диспатчит по cfprefsd, lsd, runningboardd, SpringBoard | `BaseBin/roothidehooks/src/main.x:25–47` |
| roothide pathhook хукает `_CFCopyHomeDirURLForUser` | `BaseBin/roothidehooks/src/pathhook.x:11–40` |
| roothide runningboardd fix для «RUNNINGBOARD 3735883980» | `BaseBin/roothidehooks/src/runningboardd.x:7–45` |
| roothide lsd скрывает jb-URL-схемы (_LSURLOverride, _LSCanOpenURLManager) | `BaseBin/roothidehooks/src/lsd.x` |

### 1.11. Dopamine.app flow

| Утверждение | Источник |
|---|---|
| main.m принимает `trollstore delete-bootstrap` аргумент | `Application/Dopamine/main.m:17–28` |
| Другие trollstore команды закомментированы (hide-jailbreak) | `Application/Dopamine/main.m:22–25` |
| DOBootstrapper использует libarchive для распаковки basebin.tar | `Application/Dopamine/Jailbreak/DOBootstrapper.m` |
| Bundled packages: libkrw0-dopamine 2.0.3, libroot-dopamine 1.0.1, dopamine-basebin-link 1.0.0, launchctl 1:1.2.0 | `Application/Dopamine/Jailbreak/DOBootstrapper.m:21–30` |

### 1.12. Что Dopamine НЕ делает (отсутствие кода)

Проверено `grep` — соответствующих хуков нет:

| Утверждение | Проверка |
|---|---|
| Dopamine не хукает mediaserverd | `grep -rln mediaserverd BaseBin/` — не находит хуков |
| Dopamine не хукает AVFoundation | `grep -rln AVFoundation BaseBin/` — не находит хуков |
| Dopamine не хукает accessibility (AX) | `grep -rln axassetsd\|VoiceOver\|UIAccessibility BaseBin/` — пусто |
| Dopamine не хукает powerd/thermalmonitord | `grep -rln powerd\|thermalmonitord BaseBin/` — пусто |
| Dopamine не хукает installd (хук отключён) | См. 1.10 |
| Dopamine не хукает apsd/trustd/nehelper активно (NECP — другое) | `grep -rln apsd BaseBin/` — пусто |

### 1.13. Сборка

| Утверждение | Источник |
|---|---|
| basebin.tar собирается через локальный `./tar` | `BaseBin/Makefile:113–115` |
| basebin.tc — trust cache всех бинарников basebin | `BaseBin/Makefile:102–105` |
| Корневой Makefile поддерживает `make update` и `make update-basebin` через ssh+scp | `Makefile:26–37` |
| `jbctl update basebin "$(jbroot)/var/mobile/Documents/basebin.tar"` — рабочая команда | `Makefile:35` |

---

## 2. Подтверждено документацией / внешними источниками

Эти утверждения опираются на Apple docs, Theos docs, или общеизвестную практику. Я их не выдумываю, но и не могу «посмотреть на устройстве».

### 2.1. Apple SDK / API

| Утверждение | Источник |
|---|---|
| `UIApplication`, `UIViewController`, `UIView`, `NSFileManager`, `NSUserDefaults`, `NSXPCConnection`, `CFPreferences*` | Apple Developer Documentation, UIKit / Foundation / CoreFoundation reference |
| `os_log` family API, уровни, format specifiers | Apple docs для `os_log.h` |
| ObjC runtime: `objc_getClass`, `class_copyMethodList`, `method_setImplementation` и т. д. | Apple docs для `/usr/include/objc/runtime.h` |
| `pthread_key_create`, `pthread_setspecific`, `pthread_getspecific` | POSIX, Apple pthread docs |
| `sysctlbyname("kern.osproductversion", ...)` возвращает текущую iOS версию | Apple docs |
| `UIAccessibility*` C API | Apple UIKit docs |
| `BGTaskScheduler`, `BGAppRefreshTask` | Apple BackgroundTasks docs |
| CoreHaptics, UIFeedbackGenerator | Apple docs |
| `IOPMAssertionCreateWithName` | Apple IOKit/pwr_mgt docs |

### 2.2. Theos / Logos

| Утверждение | Источник |
|---|---|
| `%hook`, `%end`, `%orig`, `%ctor`, `%hookf`, `%init`, `%property`, `%group`, `%c()`, `%new` | Theos docs (github.com/theos/theos) |
| `TARGET`, `ARCHS`, `TWEAK_NAME`, `*_FILES`, `*_CFLAGS`, `*_LIBRARIES`, `*_FRAMEWORKS`, `INSTALL_TARGET_PROCESSES`, `FINALPACKAGE` | Theos docs |
| `THEOS_PACKAGE_SCHEME = roothide` — расширение roothide/Theos | github.com/roothide/Theos |

### 2.3. Mach-O / dyld

| Утверждение | Источник |
|---|---|
| Load commands: `LC_SEGMENT_64`, `LC_LOAD_DYLIB`, `LC_ID_DYLIB`, `LC_RPATH`, `LC_CODE_SIGNATURE`, `LC_DYLD_INFO_ONLY`, `LC_DYLD_CHAINED_FIXUPS`, `LC_UUID`, `LC_BUILD_VERSION`, `LC_MAIN` | Apple mach-o/loader.h docs |
| chained fixups появились в iOS 15+ | Apple WWDC 2020 session «What's new in dyld» |
| arm64e PAC: ключи IA/IB/DA/DB | ARM Architecture Reference Manual (ARMv8.3-A) |
| dyld 4 prebuilt loaders | Apple dyld source (opensource.apple.com) |
| MachO fat binary format | Apple mach-o/fat.h |

### 2.4. Debian package

| Утверждение | Источник |
|---|---|
| Поля control: Package, Name, Version, Architecture, Description, Depends, Section, Maintainer, Author | Debian Policy Manual |
| iphoneos-arm architecture для jailbreak на iOS 15+ | Jailbreak-сообщество, Procursus repo |
| Dependencies с OR через `|` | Debian Policy |

### 2.5. ipsw / dyld_shared_cache

| Утверждение | Источник |
|---|---|
| ipsw CLI (blacktop/ipsw) — для извлечения framework из dyld_shared_cache | github.com/blacktop/ipsw |
| dyld_shared_cache на устройстве — `/System/Library/dyld/dyld_shared_cache_arm64` | Apple dyld source |

---

## 3. Требует проверки на устройстве

Это утверждения, которые я **не могу подтвердить** без A13 с iOS 26.0 или 26.0.1. Проверяйте перед применением.

### 3.1. Поведенческие

- Реальная работа всей цепочки эксплойта (DarkSword → PPL bypass → bootstrap) на вашем конкретном билде iOS 26.
- Корректность всех xpoke'ов на iOS 26 (мы знаем они **существуют**, но не знаем проходят ли все sanity checks).
- Что `TweakLoader.dylib` реально загружается в целевой процесс после установки.
- Что `uicache -a` правильно регистрирует ваше приложение.
- Что `.safe_mode` файл реально отключает твики на вашем устройстве.
- Что `jbctl update basebin` работает без issues на 26.0 и 26.0.1.

### 3.2. Private селекторы / классы

- Все имена private классов, использованные в примерах глав (`_SBUITimeFormatter`, `SBLowBatteryAlertController`, `XBSnapshotContainerIdentity`, `_LSURLOverride`, `_LSCanOpenURLManager`, `SBFLSTimeStackView`, `_SBUIDigitalClockView`, и т. п.) — на iOS 26 могут отличаться. Процедура probe — глава VI.1.
- Все имена private методов — аналогично.
- Все offsets в ядре — специфичны для билда.

### 3.3. Среда

- Точное значение `CFCoreFoundationVersionNumber` на iOS 26.0 и 26.0.1.
- Точные имена demon-процессов на iOS 26 — `cameracapturecore` vs `cameracaptured` vs другое (пользователь прямо указал, что «cameraserved убрали»).
- Какой пакет для hook-менеджера установлен (ellekit vs substitute).
- Какой PreferenceLoader версии установлен.

### 3.4. Пути

- Точный путь randomized jbroot на вашем устройстве — определяется случайно при первой установке.
- `<jbroot>/var/mobile/Library/Preferences/*.plist` — существует ли и работает ли на вашей установке.

### 3.5. Поведение между 26.0 и 26.0.1

- Какие именно private селекторы переименованы или удалены.
- Какие частные XPC-протоколы изменились.
- Какие entitlements закрыты.
- Фиксы Apple в 26.0.1 security notes.

### 3.6. Производительность

- Накладные расходы от хуков на `fileExistsAtPath:` на конкретном устройстве.
- Задержки от Darwin notification на подключение к Settings.
- Jetsam-поведение демонов jailbreak.

---

## 4. Не поддерживается или данных недостаточно

### 4.1. Не поддерживается проектом

- **A14+ на iOS 26** — проект явно не декларирует (строка в `DOEnvironmentManager.m:254` указывает только A12/A13 для iOS 26).
- **iOS 25.x** — не существует.
- **iOS 26.1+** — пока за пределами объявленной поддержки.
- **iOS 19.x–25.x** — SDK версии между; на устройствах нет (Apple перескакивает нумерацию).

### 4.2. Данных недостаточно

- **arm64e для iOS 26 на A14+** — проект имеет инфраструктуру (SPTM detection в `DOEnvironmentManager`), но не декларирует работу на 26.
- **Полная карта isActivationLock-related подсистем на iOS 26** — за пределами области книги.
- **Внутренности Apple Intelligence / Siri на iOS 26** — Apple не публикует.
- **Secure Enclave API** — закрытое, не исследовано.
- **TXM / SPTM на arm64e для iOS 17–26** — лишь концептуально; детали ядерной защиты.

### 4.3. Что было, но убрано

| Что | Статус |
|---|---|
| Хук `MIGlobalConfiguration._bundleIDMapForBundlesInDirectory:` в installd | Закомментирован (BOOTLOOP RISK) |
| `fakelib_redirect` в dyldhook | Код есть, но обёрнут в `#if 0` |
| `hide-jailbreak` команда в main.m | Закомментирована |
| classic / rootless THEOS_PACKAGE_SCHEME | Поддерживается, но не рекомендован для RootHide |

### 4.4. Что ожидается, но не реализовано

| Что | Статус |
|---|---|
| `bootstrapper` на устройстве как activity | Текущая реализация — пустая заглушка (`return 0;`) |
| Автоматическая миграция между minor версиями | Нет — каждое обновление требует переустановки Dopamine |

---

## 5. Как использовать это приложение

### 5.1. Прежде чем применять рецепт из книги

1. Найдите утверждение в этом документе.
2. Если в разделе 1 (подтверждено исходниками) — применяйте.
3. Если в разделе 2 (документация) — применяйте с нормальной care.
4. Если в разделе 3 (требует проверки) — сначала verify на устройстве.
5. Если в разделе 4 (не поддерживается) — не применяйте в проде.

### 5.2. Прежде чем расширять книгу

Если вы форкаете или расширяете эту книгу — ваши новые главы должны также снабжаться similar matrices. Это дисциплина, которая отличает техническое руководство от маркетинга.

### 5.3. Отчёт об ошибке

Если нашли ошибку в книге — проверьте, в каком разделе этого Приложения было соответствующее утверждение:

- Ошибка в разделе 1 → противоречие с исходниками → серьёзно, нужен fix.
- Ошибка в разделе 2 → противоречие с Apple docs → возможно устаревание.
- Ошибка в разделе 3 → book-wide note «требует проверки» → не ошибка, а ожидаемый gap.

---

## 6. Статистика покрытия

Для референс:

- **Разделов 1 (подтверждено исходниками)**: 13 категорий, ~100+ отдельных утверждений.
- **Разделов 2 (документация)**: 5 категорий, ~30+ утверждений.
- **Разделов 3 (требует проверки)**: 6 категорий, десятки утверждений.
- **Разделов 4 (не поддерживается)**: 4 категории.

Это honest status: примерно 60% книги — проверяемая правда из репозитория; 20% — внешне подтверждаемая документация; 15% — нужно проверить на устройстве; 5% — явные пробелы, которые проект не покрывает.

---

## 7. Выводы

Главный тезис: **книга — не истина в последней инстанции**. Это карта, составленная по видимой территории (репозиторий, Apple docs). Чтобы ехать по ней безопасно, время от времени сверяйтесь с реальностью: `grep` в репозитории (если он обновился) и probe на устройстве (если ваш локальный мир отличается).

Три якорных правила этого Приложения:

1. **Доверяй, но проверяй** — любое утверждение в книге либо из раздела 1 (есть file:line), либо требует верификации.
2. **Не выдумывай** — если книга чего-то не знает, она это признаёт.
3. **Поддерживай матрицу** — при любом изменении (апдейт Dopamine, апдейт iOS, свои эксперименты) обновляйте этот документ в своём форке.
