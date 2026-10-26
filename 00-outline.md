# План книги «Разработка патчей и твиков для iOS 26.0–26.0.1 на A13»

База: Dopamine RootHide 3.0.12, ветка `claude/serene-knuth-h5qs09`, HEAD `24ea745`.

Структура: 7 частей, 50 глав, 5 приложений. Для каждой главы указан относительный путь внутри `scratchpad/book/`.

## Часть I. Среда и фундамент

| № | Заголовок | Файл |
|---|-----------|------|
| 01 | Целевая среда: iOS 26.0–26.0.1 на A13, границы применимости | `part1/01-target-environment.md` |
| 02 | Архитектура A13: Lightning/Thunder, PAC, PPL | `part1/02-a13-architecture.md` |
| 03 | Чем roothide отличается от rootless: модель путей | `part1/03-roothide-vs-rootless.md` |
| 04 | Обзор репозитория Dopamine 3.0.12: карта каталогов | `part1/04-repo-map.md` |
| 05 | Инструментарий разработчика (Theos, ldid, `nm`, `otool`, `jtool2`) | `part1/05-toolchain.md` |
| 06 | Сборка и подпись твика: минимальный пример | `part1/06-build-and-sign.md` |
| 07 | Установка пакета на устройство через Sileo / TrollStore | `part1/07-install-deliver.md` |

## Часть II. Цепочка джейлбрейка Dopamine

| № | Заголовок | Файл |
|---|-----------|------|
| 08 | Общая схема: от UI до kernel R/W | `part2/08-chain-overview.md` |
| 09 | DOEnvironmentManager и выбор эксплойта по CPU/iOS | `part2/09-environment-manager.md` |
| 10 | Эксплойт ClearSword: диапазон, требования, вход | `part2/10-clearsword.md` |
| 11 | Эксплойт DarkSword как запасной путь | `part2/11-darksword.md` |
| 12 | Postexploit momentarius: A13-ветка | `part2/12-momentarius-a13.md` |
| 13 | Patchfinder и байт-паттерны вместо offsets | `part2/13-patchfinder.md` |
| 14 | BaseBin и jailbreakd: что живёт в userspace | `part2/14-basebin-jailbreakd.md` |

## Часть III. Trust cache и подпись кода

| № | Заголовок | Файл |
|---|-----------|------|
| 15 | AMFI и trust cache в iOS 26: что известно из кода | `part3/15-amfi-tc.md` |
| 16 | Динамический trust cache: добавление CDHash | `part3/16-dynamic-tc.md` |
| 17 | fakelib / fakedyld и обход проверок подписи | `part3/17-fakelib.md` |
| 18 | ldid: подпись twink-бинарников для устройства | `part3/18-ldid-signing.md` |
| 19 | CoreTrust bypass в линейке Dopamine 3 | `part3/19-coretrust.md` |

## Часть IV. Внедрение кода в процессы

| № | Заголовок | Файл |
|---|-----------|------|
| 20 | Два механизма: `DYLD_INSERT_LIBRARIES` и rootlesshook | `part4/20-injection-modes.md` |
| 21 | Жизненный цикл процесса: от launchd до main() | `part4/21-process-lifecycle.md` |
| 22 | rootlesshook: редирект путей и подмена загружаемых библиотек | `part4/22-rootlesshook.md` |
| 23 | Фильтры твика: какие процессы цеплять | `part4/23-filters.md` |
| 24 | Отладка внедрения: лог-линии, Console.app, oslog | `part4/24-debug-injection.md` |

## Часть V. Написание твика

| № | Заголовок | Файл |
|---|-----------|------|
| 25 | Objective-C runtime: method swizzling на iOS 26 | `part5/25-swizzling.md` |
| 26 | Logos: синтаксис, `%hook`, `%orig`, `%new` | `part5/26-logos.md` |
| 27 | Хуки на C-функциях: `MSHookFunction` / fishhook | `part5/27-cfn-hooks.md` |
| 28 | Хуки на Swift-символах: ограничения ABI | `part5/28-swift-hooks.md` |
| 29 | Mach-сообщения и XPC: перехват IPC | `part5/29-xpc.md` |
| 30 | Работа с SpringBoard: безопасность изменений UI | `part5/30-springboard.md` |
| 31 | Пример твика: модификация статус-бара | `part5/31-example-statusbar.md` |
| 32 | Пример твика: кастомный жест в приложении | `part5/32-example-gesture.md` |
| 33 | Пример твика: системное уведомление без API | `part5/33-example-notification.md` |

## Часть VI. Патчи системных компонентов

| № | Заголовок | Файл |
|---|-----------|------|
| 34 | Патч бинарного файла в системе: план действий | `part6/34-binary-patch-plan.md` |
| 35 | Создание патча через fakelib: пример для MobileGestalt | `part6/35-fakelib-mobilegestalt.md` |
| 36 | Патч plist-файлов, не ломая SIP-подобные проверки | `part6/36-plist-patches.md` |
| 37 | Патчи демонов (daemons): launchd-плейсы в roothide | `part6/37-daemon-patches.md` |
| 38 | Безопасный undo: откат патча без ребута | `part6/38-safe-undo.md` |
| 39 | Патчи для кастомизации WebKit (в рамках дозволенного) | `part6/39-webkit-patches.md` |

## Часть VII. Качество, стабильность, развитие

| № | Заголовок | Файл |
|---|-----------|------|
| 40 | Таксономия крашей (crashes) iOS 26 и чтение `.ips` | `part7/40-crashes.md` |
| 41 | Отладка твика на устройстве: lldb через usbmux | `part7/41-lldb-usb.md` |
| 42 | Телеметрия твика: сбор диагностики без PII | `part7/42-telemetry.md` |
| 43 | Пакетирование: .deb, control-файл, зависимости | `part7/43-packaging.md` |
| 44 | Публикация в репозитории Sileo / Zebra | `part7/44-repos.md` |
| 45 | Совместимость с rootless-твиками: прослойка roothide | `part7/45-rootless-compat.md` |
| 46 | Производительность: цена хука и как её снизить | `part7/46-performance.md` |
| 47 | Безопасность: как не превратить твик в руткит | `part7/47-security.md` |
| 48 | Этика и правовые аспекты публикации твиков | `part7/48-legal.md` |
| 49 | Что меняется в iOS 26.0.1 по сравнению с 26.0 | `part7/49-26.0-vs-26.0.1.md` |
| 50 | Куда двигаться: iOS 26.1 и далее | `part7/50-future.md` |

## Приложения

| № | Заголовок | Файл |
|---|-----------|------|
| A | Карта исходников Dopamine RootHide 3.0.12 с построчными ссылками | `appendix/A-source-map.md` |
| B | Список CPU-семейств Apple и соответствие моделям | `appendix/B-cpu-families.md` |
| C | Шаблоны Theos для roothide-твика | `appendix/C-theos-templates.md` |
| D | Таблица поддерживаемых билдов iOS 26 (из Info.plist эксплойтов) | `appendix/D-ios-builds.md` |
| E | Глоссарий терминов: PPL, SPTM, PAC, CDHash, trust cache, patchfinder | `appendix/E-glossary.md` |
