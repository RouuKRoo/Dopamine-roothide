# План расширения книги «Dopamine RootHide на iOS 26.0–26.0.1 (A13): разработка патчей и твиков»

Расширение дополняет основную книгу (Часть I–III) и состоит из **65 дополнительных глав**, распределённых по пяти секциям-каталогам:

- `deep-dive/` — глубокое погружение в подсистемы Dopamine RootHide (15 глав)
- `cookbook/` — рецепты разработки твиков и патчей (15 глав)
- `cheatsheets/` — краткие справочники для ежедневной работы (10 глав)
- `ios26-changes/` — что именно изменилось в iOS 26.0 и 26.0.1 (10 глав)
- `advanced/` — продвинутые темы: PPL, PAC, обход новых митигейшенов (15 глав)

Все главы ориентированы на A12/A13 (PPL-устройства, единственные поддерживаемые iOS 26.0–26.0.1 ветки Dopamine RootHide, подтверждено `Application/Dopamine/Jailbreak/DOEnvironmentManager.m:254` и `Application/Dopamine/Exploits/{DarkSword,ClearSword}/Info.plist`).

---

## Секция 1. `deep-dive/` — глубокое погружение (15 глав)

| № | Файл | Заголовок |
|---|---|---|
| D01 | `deep-dive/01-doenvironmentmanager.md` | DOEnvironmentManager: определение устройства, версии и поддержки |
| D02 | `deep-dive/02-darksword-exploit.md` | DarkSword: устройство эксплойта и его роль на A13/iOS 26.0–26.0.1 |
| D03 | `deep-dive/03-clearsword-exploit.md` | ClearSword: PPL-bypass и kread/kwrite примитивы |
| D04 | `deep-dive/04-fugu17-kernel-primitives.md` | fugu17: примитивы работы с ядром поверх PPL |
| D05 | `deep-dive/05-bootstrap-and-jbroot.md` | bootstrap, jbroot и симлинк-топология rootless/roothide |
| D06 | `deep-dive/06-dopamine-launchd-tinypatch.md` | tinypatch и ранние хуки launchd |
| D07 | `deep-dive/07-systemhook-injection.md` | systemhook: механика инъекции в процессы |
| D08 | `deep-dive/08-ellekit-substitute.md` | ellekit vs substitute: цепочка хуков на iOS 26 |
| D09 | `deep-dive/09-fakelib-and-dyld.md` | fakelib, DYLD_INSERT_LIBRARIES и обход SIP-подобных проверок |
| D10 | `deep-dive/10-trustcache-and-amfi.md` | TrustCache и AMFI на iOS 26: что всё ещё работает |
| D11 | `deep-dive/11-roothide-path-virtualization.md` | roothide: виртуализация путей и `/var/jb` против обнаружения |
| D12 | `deep-dive/12-sandbox-and-platform-apps.md` | Sandbox, platform-apps и container-паттерны |
| D13 | `deep-dive/13-codesigning-flow.md` | Поток подписи кода: ldid, fastPathSign, entitlements |
| D14 | `deep-dive/14-ipc-xpc-launchd.md` | IPC, XPC и launchd-domains для твиков |
| D15 | `deep-dive/15-userspace-reboot.md` | Userspace reboot: как переустанавливается jailbreak-state |

## Секция 2. `cookbook/` — рецепты (15 глав)

| № | Файл | Заголовок |
|---|---|---|
| C01 | `cookbook/01-first-tweak-theos.md` | Первый твик на Theos под iOS 26 roothide |
| C02 | `cookbook/02-logos-hooks-basics.md` | Logos: %hook, %orig, %c — рабочие идиомы |
| C03 | `cookbook/03-objc-runtime-hooks.md` | Хуки Objective-C runtime без Logos |
| C04 | `cookbook/04-c-function-hooks-ellekit.md` | Хуки C-функций через ellekit (MSHookFunction-замена) |
| C05 | `cookbook/05-swift-hooks.md` | Хуки Swift-символов: приёмы и ограничения |
| C06 | `cookbook/06-springboard-tweak.md` | Твик для SpringBoard: инициализация и safety-nets |
| C07 | `cookbook/07-preferences-bundle.md` | Preferences-bundle для твика (PSSpecifiers, roothide) |
| C08 | `cookbook/08-launchdaemon-plist.md` | LaunchDaemon от jailbreak-процесса |
| C09 | `cookbook/09-url-scheme-handler.md` | URL-scheme handler поверх twik'а |
| C10 | `cookbook/10-background-work.md` | Фоновые задачи и BackgroundTasks на jailbroken-устройстве |
| C11 | `cookbook/11-filter-plist-targets.md` | Filter.plist: таргетинг процессов и bundle-id |
| C12 | `cookbook/12-shared-cache-symbols.md` | Поиск символов в dyld shared cache iOS 26 |
| C13 | `cookbook/13-codesign-and-ldid.md` | Подпись бинарей ldid, entitlements, fastPathSign |
| C14 | `cookbook/14-packaging-deb.md` | Упаковка .deb под roothide, control-поля |
| C15 | `cookbook/15-debugging-on-device.md` | Отладка твика на устройстве: logs, lldb, tweak-crashes |

## Секция 3. `cheatsheets/` — справочники (10 глав)

| № | Файл | Заголовок |
|---|---|---|
| S01 | `cheatsheets/01-paths-roothide.md` | Пути roothide: jbroot, /var/jb, симлинки |
| S02 | `cheatsheets/02-entitlements-table.md` | Таблица часто используемых entitlements на iOS 26 |
| S03 | `cheatsheets/03-ellekit-api.md` | API ellekit: EKHook*, EKReplace* |
| S04 | `cheatsheets/04-logos-directives.md` | Шпаргалка по директивам Logos |
| S05 | `cheatsheets/05-theos-makefiles.md` | Theos Makefiles для roothide-твиков |
| S06 | `cheatsheets/06-command-reference.md` | Команды: dopamine, roothelper, jbctl, rebuildapps |
| S07 | `cheatsheets/07-common-classes.md` | Часто-хукаемые классы SpringBoard/UIKit |
| S08 | `cheatsheets/08-plist-keys.md` | Ключи Info.plist и Tweak.plist |
| S09 | `cheatsheets/09-arm64e-ptrauth.md` | arm64e: PAC-ключи, ptrauth_* макросы |
| S10 | `cheatsheets/10-error-codes.md` | Коды ошибок Dopamine и как их читать |

## Секция 4. `ios26-changes/` — изменения iOS 26 (10 глав)

| № | Файл | Заголовок |
|---|---|---|
| I01 | `ios26-changes/01-overview-26-0-vs-26-0-1.md` | Обзор: 26.0 vs 26.0.1 — что реально разное |
| I02 | `ios26-changes/02-amfi-tightening.md` | Усиление AMFI на iOS 26 |
| I03 | `ios26-changes/03-sandbox-changes.md` | Изменения Sandbox-профилей |
| I04 | `ios26-changes/04-launchd-changes.md` | launchd и bootstrap на iOS 26 |
| I05 | `ios26-changes/05-dyld-shared-cache.md` | dyld shared cache: layout и символы |
| I06 | `ios26-changes/06-springboard-internals.md` | SpringBoard internals, затрагиваемые твиками |
| I07 | `ios26-changes/07-webkit-changes.md` | WebKit/WebContent изменения, важные для хуков |
| I08 | `ios26-changes/08-ppl-and-sptm.md` | PPL/SPTM на A13 в iOS 26 |
| I09 | `ios26-changes/09-kernel-mitigations.md` | Новые kernel-митигейшены, влияющие на эксплойт |
| I10 | `ios26-changes/10-userspace-mitigations.md` | Userspace-митигейшены: PAC, BTI, hardened runtime |

## Секция 5. `advanced/` — продвинутые темы (15 глав)

| № | Файл | Заголовок |
|---|---|---|
| A01 | `advanced/01-ppl-internals-a13.md` | PPL на A13: модель угроз и bypass-поверхности |
| A02 | `advanced/02-pac-strategies.md` | PAC: стратегии подделки и легитимной подписи указателей |
| A03 | `advanced/03-kernel-rw-primitives.md` | Построение stable kread/kwrite из начальных примитивов |
| A04 | `advanced/04-tfp0-replacement.md` | Замена task_for_pid(0): kcall/physread на iOS 26 |
| A05 | `advanced/05-trustcache-injection.md` | Инъекция в dynamic trustcache на iOS 26 |
| A06 | `advanced/06-amfi-bypass-vectors.md` | Вектора обхода AMFI поверх PPL |
| A07 | `advanced/07-codesigning-bypass.md` | Обход codesigning без модификации AMFI |
| A08 | `advanced/08-sandbox-escape-patterns.md` | Паттерны выхода из sandbox для твиков system-процессов |
| A09 | `advanced/09-launchd-hooks.md` | Хуки launchd и ранний старт твиков |
| A10 | `advanced/10-xpc-interception.md` | Перехват XPC для observability и патчей |
| A11 | `advanced/11-jit-and-ptrace.md` | JIT, ptrace, debugserver на roothide |
| A12 | `advanced/12-anti-detection.md` | Анти-детект: как твик скрывает jailbreak |
| A13 | `advanced/13-persistence.md` | Персистентность после userspace reboot |
| A14 | `advanced/14-crash-recovery.md` | Восстановление после краха tweak loader'а |
| A15 | `advanced/15-contributing-upstream.md` | Как вкладываться в roothide/Dopamine upstream |

---

**Итого:** 15 + 15 + 10 + 10 + 15 = **65 глав**.

Все главы пишутся по правилам из задачи: только русский, факты через `путь/файл:строка`, явные разделы «подтверждено исходниками / документацией / требует проверки на устройстве / не подтверждено», отсутствие выдуманных private API, PAC-salts, offsets, селекторов. Для различий 26.0 vs 26.0.1 — ссылка или явная пометка «различия не выявлены».
