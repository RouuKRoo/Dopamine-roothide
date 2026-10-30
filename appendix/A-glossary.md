# Приложение A. Словарь терминов

Алфавитный глоссарий. Для каждого термина — краткое определение (1–3 предложения) и ссылка на главу, где он подробно разбирается. Используйте как быстрый справочник при чтении основной книги.

Термины приводятся в русском написании (где используется) и латинском оригинале.

---

## А

**AMFI (Apple Mobile File Integrity)**. Подсистема iOS kernel, которая проверяет codesign каждого бинарника и dylib перед загрузкой. На jailbroken устройстве её проверка «валидно» выставляется через trust cache. См. главу I.5 и Deep-dive о trustcache.

**ARC (Automatic Reference Counting)**. Механизм компилятора Clang, автоматически вставляющий `retain`/`release`/`autorelease` в Objective-C код. Включается `-fobjc-arc` в CFLAGS. В твиках — всегда включайте. См. главу IV.7.

**ARM64 vs arm64e**. Две инструкционные архитектуры 64-бит ARM. arm64 — без PAC (A12 и A13 в userspace); arm64e — с PAC (A14+). Dopamine для A13+iOS 26 собирается с обоими slices, но реально работает в arm64. См. главы I.4 и Deep-dive о PAC.

**Associated object**. Способ в ObjC runtime прикрепить дополнительные данные к существующему объекту без изменения его класса, через `objc_setAssociatedObject`. Основа паттерна `%property` в Logos. См. главу IV.6.

---

## B

**basebin**. «Миниатюрный userland Dopamine» — набор dylib и бинарников, которые разворачиваются в `<jbroot>/basebin/` при первом jailbreak. Собирается в один `basebin.tar`. См. главу III.11.

**BB (BBServer)**. BulletinBoard — private framework, отвечающий за хранение и доставку уведомлений в iOS. См. главу II.3.

**Blacklist (в контексте Dopamine)**. Список процессов, в которые `systemhook.dylib` не инъектируется, определяемый в `BaseBin/libroothider/src/blacklist.m`. Нужен для стабильности системы. См. главу III.3.

**Bundle id**. Reverse-DNS идентификатор приложения или расширения iOS. Пример: `com.apple.springboard`, `com.example.testbed`. См. главу IV.3.

---

## C

**cdhash**. 20-байтовый SHA-1 hash code directory каждой подписи iOS бинарника. Единица, по которой AMFI проверяет «валидно ли». На Dopamine cdhash каждой dylib добавляется в trust cache. См. Deep-dive о trustcache.

**cfprefsd**. Системный демон iOS, обслуживающий `CFPreferences` API (чтение/запись .plist настроек). На Dopamine хукнут для перенаправления путей в rootless (`rootlesshooks/cfprefsd.x`). См. главы II.7, III.9.

**ChOma**. C-библиотека для работы с Mach-O, submodule проекта (`BaseBin/ChOma/`). Используется Dopamine для чтения/модификации бинарников. Автор твика может использовать для сложных операций. См. Deep-dive 12.

**class-dump / class-dump-dyld**. Утилиты для reverse engineering Objective-C: выгружают интерфейсы классов из Mach-O или dyld shared cache. См. главы VI.1, VI.2.

**codesign**. Apple-утилита для подписания бинарников. На macOS — часть Xcode. Для jailbreak-разработки обычно используется `ldid` как альтернатива. См. главу IV.10.

**CoreFoundation**. Основная C-библиотека iOS/macOS, низкоуровневый фундамент над `Foundation`. Содержит `CFPreferences`, `CFBundle`, `CFString`, `CFRunLoop` и др. См. главу IV.8.

**CS_VALID**. Codesign flag, означающий «подпись процесса валидна». В iOS 16+ единственное место в ядре, проверяющее `CS_VALID`, — NECP. На arm64 (A13) после записи в executable pages флаг снимается, что требует хуков в `systemhook`. См. главы II.9, III.5.

---

## D

**Darwin notification**. Нотификация системного уровня, доставляемая через `notifyd` между процессами. Используется для связи «Settings.app ↔ твик». API: `notify_post` / `notify_register_dispatch`. См. главу IV.8.

**DEB (Debian package)**. Формат пакета, в который упаковывается твик. Структура: control + файлы. На Dopamine устанавливается через `dpkg -i` или Sileo/Zebra. См. главу IV.10.

**debugserver**. Утилита для отладки через LLDB. На iOS — Apple-поставляемая, но без valid подписи; Dopamine ставит специальный `ptrace_hook` для совместимости с ней (`BaseBin/systemhook/src/main.c:504`). См. главу III.5.

**Dopamine**. Rootless + rootHide jailbreak для iOS 15.0–18.7.1 и 26.0–26.0.1 на A12/A13. Проект forked от opa334/Dopamine. См. главу I.5 и III.1.

**dpkg**. Debian package manager, используемый на Dopamine для установки/удаления `.deb` пакетов. См. главу IV.10.

**dyld**. Dynamic linker iOS, запускаемый ядром при старте процесса. Загружает исполнимый + все dylib. На Dopamine модифицирован через `dyldhook`. См. главы I.4, III.6, Deep-dive 02.

**dyldhook**. Специальная dylib Dopamine, встраиваемая в сам dyld через `MachOMerger`. Делает library validation bypass и path redirect. См. главу III.6.

**dyld_shared_cache**. Единый кэш всех iOS frameworks в одном файле `/System/Library/dyld/dyld_shared_cache_arm64`. Можно извлечь отдельный framework через `ipsw` CLI. См. главу VI.2, Deep-dive 02.

**DYLD_INSERT_LIBRARIES**. Переменная среды, которая заставляет dyld загрузить дополнительную dylib до main(). Dopamine использует её для инъекции `systemhook.dylib` в каждый процесс. См. главы I.1, III.4, III.5.

---

## E

**ellekit**. Современная библиотека хуков для jailbreak (альтернатива substrate/substitute). PAC-aware, активно поддерживается, рекомендована для Dopamine. См. главы I.3, IV.1, IV.5.

**entitlement**. Строка в XML Info.plist бинарника, дающая ему особые права (например, `platform-application`, `com.apple.private.cs.debugger`). Задаётся при подписи через `ldid -Sentitlements.xml`. См. главы III.7, III.8, V.4.

---

## F

**fakelib**. Концепция в `dyldhook`: редирект `dlopen` вызовов к `/usr/lib/*` в `<jbroot>/basebin/.fakelib/*`. На момент текущего Dopamine 3.0.12 выключено (`#if 0`), но инфраструктура присутствует. См. главу III.6.

**fat binary**. Mach-O с несколькими slices (архитектурами) в одном файле. Для твиков — обычно arm64 + arm64e. Создаётся через `lipo -create` или автоматически Theos при `ARCHS = arm64 arm64e`. См. главу V.4.

**filter plist**. Файл `<TweakName>.plist` рядом с `<TweakName>.dylib`, задающий условия загрузки твика в процесс (Bundles / Classes / Executables / CoreFoundationVersion). См. главу IV.3.

**FINALPACKAGE**. Переменная сборки Theos (`make package FINALPACKAGE=1`). Включает `-O2` оптимизацию и strip debug info для production. См. главу IV.10.

**fishhook**. Библиотека для перехвата C-функций через подмену указателя в GOT (indirect symbol binding table). Работает только для межмодульных вызовов. См. главу IV.5.

**forkfix**. Dopamine-dylib, обеспечивающая корректную инъекцию в дочерние процессы после `fork()`. См. главу III.3 (упоминание), глава III.11.

---

## G

**GOT (Global Offset Table)**. Таблица в Mach-O, содержащая указатели на внешние символы. Fishhook перехватывает функции, подменяя записи в GOT. См. главы I.4, IV.5.

---

## H

**haptics**. Тактильная отдача. API: `CoreHaptics`, `UIFeedbackGenerator`. Реализована через клиент к `backboardd`. См. главу II.10.

**hookd**. Dopamine-демон (`BaseBin/hookd/`), предоставляющий mach-API для записи в чужую память (через `vm_protect`+`vm_write`). Используется Frida и аналогичными. См. главу III.7.

---

## I

**IMP**. `typedef id (*IMP)(id, SEL, ...)` — указатель на реализацию ObjC метода. `method_setImplementation` подменяет IMP; это основа ObjC swizzling. См. главу IV.6.

**install_name**. `LC_ID_DYLIB` в Mach-O — «имя, под которым меня знает dyld». Для твика — путь установки (`/var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.dylib`). См. главу IV.10.

**install_name_tool**. Apple-утилита для правок Mach-O load commands: `-change`, `-id`, `-add_rpath`, `-delete_rpath`. См. главу V.4.

**IPSW**. Формат файла iOS firmware — zip-архив, содержащий dyld shared cache и образы разделов. См. главу VI.2.

**IPS file**. Формат crash report iOS 15+ (JSON). Содержит stack trace, loaded images, exception info. См. главу VII.2.

**Isa-swizzling**. Техника подмены `isa`-указателя одного экземпляра ObjC объекта на subclass, созданный динамически. Используется для перехвата методов только одного объекта, а не всего класса. См. главу V.3.

---

## J

**jailbreakd**. Dopamine-демон (`BaseBin/jailbreakd/`), координирующий привилегированные операции (spawn_patch, spinlock_fix, root primitives). См. главу III.8.

**jbclient_***. Прекс функций в `libjailbreak`, предоставляющих клиент-API к jbserver. Главные: `jbclient_get_jbroot`, `jbclient_trust_file_by_path`, `jbclient_cs_revalidate`. См. главу III.2.

**jbctl**. CLI-утилита Dopamine на устройстве для управления: `jbctl update basebin`, `jbctl update tipa`, `jbctl trustcache show`. См. главы III.11, V.1.

**jbroot**. Корневой путь jailbreak-окружения. На rootless — `/var/jb/`; на RootHide — `/var/<rand1>/<rand2>/.jbroot/`. Получается через `libroot_get_jbroot_prefix()` или `jbclient_get_jbroot()`. См. главы III.10, IV.9.

**JBROOT_PATH / JBROOT_PATH_NSSTRING**. Макросы из `BaseBin/libjailbreak/src/jbroot.h`, конкатенирующие префикс jbroot и путь. Header-only, не требует линковки. См. главу III.10.

**jbserver**. XPC/mach-сервер внутри Dopamine, доступный процессам через `jbclient_*` функции. Серверная часть живёт в `launchdhook`. См. главы III.2, III.4.

**jbserver_domains**. Набор доменов jbserver: `SYSTEMWIDE`, `PLATFORM`, `WATCHDOG`, `ROOT`, `ROOTHIDE`. Разные привилегии. См. `BaseBin/libjailbreak/src/jbserver_domains.h`.

**Jetsam**. Подсистема iOS, убивающая процессы при нехватке памяти. Dopamine в `launchdhook/jetsam_hook.c` защищает launchd от jetsam. См. главу III.4.

---

## K

**kernel RW primitives**. Способ читать/писать произвольные адреса ядра из userspace. Dopamine получает их через эксплойт (DarkSword/ClearSword на A13+iOS 26). См. Deep-dive о exploits.

---

## L

**launchd**. PID 1 на iOS, запускаемый ядром при старте userland. Запускает все остальные процессы. На Dopamine в него инжектируется `launchdhook.dylib`. См. главы I.2, III.4.

**launchdhook**. Dopamine-dylib, живущая в launchd. Делает spawn hook, запускает jbserver, защищает от jetsam. См. главу III.4.

**LC_LOAD_DYLIB**. Load command Mach-O, объявляющий зависимость от другой dylib. См. главы I.4, V.4.

**LC_RPATH**. Load command Mach-O, добавляющий путь поиска для `@rpath/*.dylib` зависимостей. См. главы IV.10, V.4.

**ldid**. CLI-утилита для ad-hoc подписания iOS бинарников, альтернатива `codesign`. Главный инструмент jailbreak-подписи. См. главу V.4.

**libarchive**. C-библиотека для работы с tar/zip/cpio. Используется в `DOBootstrapper.m` для распаковки `basebin.tar`. См. главу III.11.

**libellekit**. Собранная dylib ellekit, устанавливается через пакет `ellekit`. См. главу IV.5.

**libjailbreak**. Главная библиотека Dopamine (`BaseBin/libjailbreak/`). Содержит `jbclient_*` API, `jbroot.h` макросы, trust cache primitives, kernel RW primitives. См. главу III.2.

**libroot**. Публичный API путей для твиков (`Packages/libroot/`). Три функции: `libroot_get_jbroot_prefix`, `libroot_get_root_prefix`, `libroot_get_boot_uuid`. См. главу III.10.

**libroothider**. RootHide-слой поверх libjailbreak (`BaseBin/libroothider/`). Blacklist, exec_patch, path translation. См. главу III.3.

**lipo**. Apple-утилита для fat binary: `-thin`, `-create`, `-archs`, `-extract`. См. главу V.4.

**litehook**. Миниатюрная библиотека хуков, submodule Dopamine (`BaseBin/_external/modules/litehook/`). Используется внутри Dopamine (не публично). См. главу III.5.

**Logos**. Препроцессор Theos, превращающий `.x`/`.xm` файлы в чистый `.m`/`.mm`. Директивы: `%hook`, `%end`, `%ctor`, `%orig`, `%hookf`, `%property`, `%init`, `%group`. См. главу IV.4 и шпаргалку `cheatsheets/01-logos-syntax.md`.

**lsd (Launch Services daemon)**. Системный демон, ведущий реестр установленных приложений и расширений. Dopamine хукает `_LSGetInboxURLForBundleIdentifier` и `_LSServer_RebuildApplicationDatabases`. См. главу II.12, III.9.

---

## M

**Mach-O**. Формат бинарников macOS/iOS. Структура: header + load commands + segments. См. главу I.4, Deep-dive 01.

**MachOMerger**. Утилита Dopamine (`BaseBin/MachOMerger/`), склеивающая dyldhook с настоящим dyld. См. главу III.6, Deep-dive 15.

**mediaserverd**. Системный демон, обслуживающий мультимедиа. Dopamine **намеренно не хукает**. См. главу II.10.

**method_setImplementation**. ObjC runtime API, подменяющий IMP у метода. Основа Logos `%hook`. См. главу IV.6.

**MIGlobalConfiguration**. Private класс в MobileInstallation framework. Dopamine когда-то пытался хукать, но получил bootloop; оставлено закомментированным (`BaseBin/rootlesshooks/installd.x`). См. главу II.12.

**MobileSubstrate**. Классическая (legacy) библиотека хуков от Saurik. Сейчас заменена ellekit. См. главу IV.1.

**musttail**. `__attribute__((musttail))` — заставляет компилятор делать tail-call. Используется в `systemhook/main.c` для сохранения правильного `__builtin_return_address`. См. главу IV.5.

---

## N

**NECP (Network Extension Control Policy)**. Kernel-подсистема iOS, проверяющая CS_VALID. Dopamine хукает все пять NECP-syscalls в `systemhook` (только на arm64). См. главы II.9, III.5.

**NSBundle**. Класс для доступа к ресурсам бандла. `[NSBundle mainBundle]` в хуке — бандл **хост-приложения**, не твика. Используйте `bundleWithPath:` с `JBROOT_PATH`. См. главу IV.8.

**NSClassFromString**. Безопасный способ получить `Class` в рантайме. Возвращает `Nil` если класс не найден. См. главу V.2.

---

## O

**ObjC runtime**. C-API для динамического доступа к Objective-C: `objc_getClass`, `class_copyMethodList`, `method_setImplementation` и др. См. главы I.3, VI.1.

**opainject**. Утилита Dopamine (`BaseBin/opainject/`) для инъекции dylib в живой процесс (альтернатива DYLD_INSERT). См. Deep-dive 13.

**os_log**. Современная система логирования iOS. API: `os_log_create`, `os_log`, `os_log_debug`, `os_log_error`. Thread-safe, не делает file I/O. См. главу VII.2.

**otool**. Apple-утилита для инспекции Mach-O: `-h` (header), `-l` (load commands), `-L` (dependencies), `-tV` (disassembly), `-D` (install_name). См. главу IV.10.

---

## P

**PAC (Pointer Authentication)**. ARMv8.3-A функция для подписывания указателей. Используется в arm64e (A14+); в arm64 (A13) не задействован. См. главу I.4, Deep-dive 03.

**platform-application**. Entitlement, дающий бинарнику права системного приложения. Присутствует у `hookd`, `jailbreakd` и других внутренних компонентов Dopamine. См. главы III.7, III.8.

**PlistBuddy / plutil**. CLI для чтения/записи .plist файлов. `plutil -p file.plist` показывает содержимое. См. главу IV.10.

**posix_spawn**. POSIX API для создания нового процесса (замена fork+exec). На iOS launchd вызывает его для всех новых процессов; `launchdhook` перехватывает для добавления DYLD_INSERT. См. главу III.4.

**PPL (Page Protection Layer)**. Hardware/firmware-level защита страниц памяти ядра на Apple SoC. Dopamine обходит её на A12/A13 через PPL bypass в эксплойтах (momentarius_a13, Titan). См. Deep-dive 04.

**PreferenceBundle**. `.bundle` с `PSListController subclass`, загружаемый Settings.app для отображения страницы настроек твика. См. главы II.7, IV.8.

**PreferenceLoader**. Утилита/dylib, загружающая PreferenceBundles в Settings.app. Пакет `preferenceloader`. См. главу IV.8.

**ptrace**. POSIX API для отладки процессов. На iOS с ограничениями; `debugserver` требует специального `ptrace_hook` в Dopamine. См. главу III.5.

---

## R

**respring**. Перезапуск SpringBoard без перезагрузки устройства (`killall -9 SpringBoard`). Нужен после установки твиков, задевающих SpringBoard. См. главу IV.10.

**rootless**. Концепция jailbreak без модификации rootfs: все jailbreak-данные живут в `/var/jb/`. Dopamine поддерживает. См. главу I.5.

**RootHide**. Расширение rootless: случайный префикс jbroot для сокрытия от детекции. Dopamine 3 поддерживает. См. главы I.5, III.3.

**rpath**. См. LC_RPATH.

---

## S

**safe_mode**. Файл-маркер `<jbroot>/basebin/.safe_mode`. При его наличии `systemhook` не загружает TweakLoader (`BaseBin/systemhook/src/main.c:203`), что отключает все твики. См. главу III.5.

**selector**. `SEL` в Objective-C — уникальный идентификатор метода. `@selector(doStuff:)` на уровне компиляции, `sel_registerName("doStuff:")` в runtime. См. главы I.3, VI.1.

**Sileo / Zebra**. GUI-менеджеры пакетов для jailbreak, аналоги Cydia. Устанавливают `.deb` пакеты. См. главу IV.10.

**SPTM/TXM (Secure Page Table Monitor / Trusted eXecution Monitor)**. Защитные подсистемы на новых iPhone (A14+). Для A13 не применимы. См. главу I.5.

**SpringBoard**. Процесс iOS, отвечающий за домашний экран и системный UI. Hooking SpringBoard — рискованно (может уйти в respring loop). См. главу II.1.

**substitute**. Jailbreak hook-библиотека (альтернатива mobilesubstrate), от Comex. Сейчас редко используется. См. главу IV.1.

**swizzling**. Техника подмены метода в ObjC через `method_exchangeImplementations` или `method_setImplementation`. Основа `%hook`. См. главы I.3, IV.6.

**systemhook**. Dopamine-dylib (`BaseBin/systemhook/`), инжектируемая через DYLD_INSERT_LIBRARIES в каждый процесс. Делает csops/NECP hooks и загружает TweakLoader. См. главу III.5.

---

## T

**task_for_pid**. Mach API, возвращающий task port процесса по его PID. Требует entitlement `task_for_pid-allow`. См. главу III.7.

**Theos**. Набор Makefile-правил + препроцессор для разработки iOS твиков. Для RootHide — fork `roothide/Theos` с `THEOS_PACKAGE_SCHEME=roothide`. См. главу IV.1.

**THEOS_PACKAGE_SCHEME**. Переменная сборки Theos: `rootless` / `roothide` / классическая. Главный флаг для RootHide-совместимости. См. главы IV.1, IV.9.

**tipa**. Формат пакета Dopamine (`.tipa`), который принимает `jbctl update tipa`. Альтернатива `.deb`. См. главу IV.10.

**TrollStore**. Утилита для установки `.ipa` с ad-hoc подписью (permasigning). Не требует jailbreak. Совместим с Dopamine. См. Advanced 7 (планируется).

**trustcache**. Таблица cdhash в ядре AMFI, воспринимаемая как «валидные подписи». Dopamine добавляет туда cdhash всех своих компонентов + компонентов твиков. См. главу III.2, Deep-dive 05.

**TweakLoader**. Dylib (устанавливается с ellekit/substitute), которую `systemhook` `dlopen`-ит. Читает filter plists всех установленных твиков и загружает подходящие. См. главу III.5.

---

## U

**uicache**. Утилита (`<jbroot>/usr/bin/uicache`), регистрирующая установленные .app в SpringBoard. Dopamine вызывает `uicache -a` после ребилда LS database (`BaseBin/rootlesshooks/lsd.x`). См. главы II.12, III.9.

**usernotificationsd**. Системный демон доставки уведомлений (UNUserNotifications). Dopamine не хукает. См. главу II.3.

---

## V

**vm_protect / vm_write**. Mach API для управления защитой и записи памяти. Используются в `hookd` для записи в чужой процесс. См. главу III.7.

---

## X

**XPC**. Межпроцессный транспорт iOS/macOS. Поверх mach ports, с сериализацией через `xpc_object_t` или `NSXPCConnection`. См. главу II.14.

**XPF (eXtensible Patchfinder)**. Submodule Dopamine (`BaseBin/XPF/`) для поиска offsets в ядре. Нужен для кросс-версионной работы эксплойтов. См. Deep-dive 11.

---

## Z

(нет терминов)

---

## Служебные сокращения

- **A13** — Apple A13 Bionic chip (iPhone 11, 11 Pro, 11 Pro Max, SE 2nd gen).
- **ABI** — Application Binary Interface.
- **ARM64** — 64-bit ARM instruction set.
- **CF** — CoreFoundation.
- **DRM** — Digital Rights Management.
- **GOT** — Global Offset Table.
- **IPC** — Inter-Process Communication.
- **IPSW** — iPhone/iPad Software (firmware).
- **KPI** — Kernel Programming Interface.
- **LC** — Load Command (в Mach-O).
- **NFC** — Near-Field Communication.
- **PID** — Process ID.
- **PLT** — Procedure Linkage Table.
- **PPL** — Page Protection Layer.
- **SE** — Secure Enclave.
- **SPI** — System Private Interface.
- **TCC** — Transparency, Consent, and Control (privacy framework).
- **TLS** — Transport Layer Security.
- **UI** — User Interface.

---

## Как использовать этот глоссарий

- При чтении основной книги — при встрече незнакомого термина загляните сюда.
- При разработке — используйте как справочник имён файлов/API Dopamine.
- При write-up своих твиков — одна-две ссылки на этот глоссарий избавят читателя от попутного исследования.

Все ссылки на главы ведут в основную книгу. Пропавшие или устаревшие термины просим обновлять в своих форках.
