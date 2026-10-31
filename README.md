# Dopamine RootHide книга — snapshot ветки

Эта ветка (`claude/book-snapshot`) содержит snapshot русскоязычной технической книги
**«Разработка патчей и твиков для iOS 26.0–26.0.1 на A13: практическое руководство по Dopamine RootHide»**.

- Целевой jailbreak: Dopamine RootHide 3.0.12 (fork `opa334/Dopamine` + roothide).
- Целевая платформа: iPhone 11 (A13) на iOS **26.0** и **26.0.1**.
- Поддерживаемые устройства/версии подтверждены исходниками: `Application/Dopamine/UI/Environment Updater/DOEnvironmentManager.m:254` и `BaseBin/Resources/DarkSword.app/Info.plist`, `ClearSword.app/Info.plist`, `momentarius_a13.app/Info.plist`, `Titan.app/Info.plist` (A12/A13, PPL).
- Классификация достоверности везде: **подтверждено исходниками / документацией / требует проверки на устройстве / не подтверждено**.
- Исходный репозиторий (`/home/user/Dopamine-roothide`) **не изменялся**; устанавливать что-либо на устройство в рамках подготовки книги **не предполагалось и не делалось**.

Клонировать:

```
git clone -b claude/book-snapshot https://github.com/RouuKRoo/Dopamine-roothide.git book
cd book
```

Начать чтение с `00-verification.md` → `00-outline.md` → `part1/01-patch-hook-tweak.md`.

## Как читать книгу

- Главы I–VII — основной учебник; читать по порядку.
- `appendix/` — четыре приложения-справочника (A–D).
- `deep-dive/` — углублённые разборы инфраструктуры Dopamine.
- `cases/` — пятнадцать практических кейс-стади с полным кодом.
- `cookbook/` — двадцать коротких рецептов.
- `cheatsheets/` — десять шпаргалок, читать по ссылке из основного текста.
- `ios26-changes/` — что именно поменялось в iOS 26, обзор для автора твика.
- `advanced/` — продвинутые темы (CI/CD, LLDB, собственный инжектор и т. п.).
- `daemons/` — атлас системных демонов iOS 26: что это и что хукать.
- `framework-atlas/` — атлас фреймворков: что имеет смысл хукать, что — нет.
- `re-walkthroughs/` — пошаговые RE-упражнения (от «найти селектор» до «diff 26.0 vs 26.0.1»).

Каждая глава содержит явный раздел «Что подтверждено / требует проверки» в конце.

## Планы томов и верификация

- `00-verification.md` — верификация целевой среды (где подтверждено, что A13 + iOS 26.0–26.0.1 поддерживается).
- `00-outline.md`, `00-extended-plan.md`, `00-vol3-plan.md` — исходные планы томов.

## Полное оглавление

### Часть I. Среда разработки патчей и твиков под iOS 26

- `part1/01-patch-hook-tweak.md` — Глава I.1. Патч, runtime-хук, твик и binary patch — что чем отличается
- `part1/02-processes-services-extensions.md` — Глава I.2. Процессы, службы, приложения и расширения в iOS
- `part1/03-objc-c-swift-boundaries.md` — Глава I.3. Objective-C runtime, C-функции, Swift — границы перехвата
- `part1/04-macho-dyld-pac-arm64-vs-arm64e.md` — Глава I.4. Mach-O, dyld, символы, ABI, arm64 vs arm64e, PAC
- `part1/05-what-jailbreak-can-and-cannot.md` — Глава I.5. Что jailbreak даёт и что остаётся недоступным

### Часть II. Атлас подсистем iOS 26 с точки зрения автора твика

- `part2/01-springboard.md` — Глава II.1. SpringBoard: домашний экран, иконки, папки
- `part2/02-lockscreen.md` — Глава II.2. Экран блокировки и аутентификация
- `part2/03-notifications.md` — Глава II.3. Уведомления
- `part2/04-controlcenter.md` — Глава II.4. Пункт управления
- `part2/05-app-lifecycle-switcher.md` — Глава II.5. Переключатель приложений и жизненный цикл процессов
- `part2/06-uikit-swiftui-scenes.md` — Глава II.6. Окна, сцены, UIKit, SwiftUI, ввод и жесты
- `part2/07-settings-preferences.md` — Глава II.7. Настройки и preferences
- `part2/08-fs-containers.md` — Глава II.8. Файловая система, контейнеры и доступ к ресурсам
- `part2/09-networking-background.md` — Глава II.9. Сеть и фоновые задачи
- `part2/10-media-haptics.md` — Глава II.10. Мультимедиа, камера, звук и тактильная отдача
- `part2/11-power-battery-thermal.md` — Глава II.11. Питание, батарея, тепловое состояние
- `part2/12-install-extensions.md` — Глава II.12. Установка приложений, расширения, межслужебное взаимодействие
- `part2/13-accessibility.md` — Глава II.13. Доступность
- `part2/14-xpc-system-services.md` — Глава II.14. Системные процессы и XPC-службы

### Часть III. Что именно даёт Dopamine RootHide — компонент за компонентом

- `part3/01-dopamine-app.md` — Глава III.1. Application/Dopamine — приложение-установщик
- `part3/02-libjailbreak.md` — Глава III.2. BaseBin/libjailbreak — ядро userland-API
- `part3/03-libroothider.md` — Глава III.3. BaseBin/libroothider — RootHide-слой
- `part3/04-launchdhook.md` — Глава III.4. BaseBin/launchdhook — вход в userland
- `part3/05-systemhook.md` — Глава III.5. BaseBin/systemhook — hook каждого процесса
- `part3/06-dyldhook.md` — Глава III.6. BaseBin/dyldhook — патч dyld на уровне процесса
- `part3/07-hookd.md` — Глава III.7. BaseBin/hookd — внешние хуки через mach
- `part3/08-jailbreakd.md` — Глава III.8. BaseBin/jailbreakd — демон-координатор
- `part3/09-rootless-and-roothide-hooks.md` — Глава III.9. rootlesshooks и roothidehooks — хуки системных процессов
- `part3/10-libroot-paths.md` — Глава III.10. Packages/libroot — публичный API путей
- `part3/11-basebin-build-bootstrap.md` — Глава III.11. Как собирается basebin и как разворачивается bootstrap
- `part3/12-task-component-table.md` — Глава III.12. Таблица «задача → компонент iOS → компонент jailbreak → файл → способ изменения → проверка»

### Часть IV. Как писать свой твик — пошаговый курс

- `part4/01-dev-env.md` — Глава IV.1. Воспроизводимая среда разработки (Theos + RootHide)
- `part4/02-minimal-project.md` — Глава IV.2. Минимальный Theos-проект: Makefile, control, filter plist
- `part4/03-process-selection.md` — Глава IV.3. Выбор процесса и фильтры инъекции
- `part4/04-first-objc-hook.md` — Глава IV.4. Первый Objective-C hook
- `part4/05-c-function-hook.md` — Глава IV.5. Перехват C-функции
- `part4/06-orig-recursion-state.md` — Глава IV.6. Вызов оригинала, рекурсия, разделяемое состояние
- `part4/07-threads-arc-lifecycle.md` — Глава IV.7. Потоки, ARC, жизненный цикл
- `part4/08-settings-resources.md` — Глава IV.8. Настройки твика и ресурсы
- `part4/09-roothide-compat.md` — Глава IV.9. Совместимость с RootHide
- `part4/10-build-inspect-pack.md` — Глава IV.10. Сборка, инспекция и упаковка
- `part4/11-tutorial-own-app.md` — Глава IV.11. Учебный проект №1: твик для собственного тестового приложения
- `part4/12-tutorial-springboard.md` — Глава IV.12. Учебный проект №2: безопасный SpringBoard-твик
- `part4/13-tutorial-system-subsystem.md` — Глава IV.13. Учебный проект №3: твик для системной подсистемы с риском

### Часть V. Патч-инженерия: как делать правки, а не только hook

- `part5/01-patch-jailbreak-sources.md` — Глава V.1. Патчи исходников самого jailbreak
- `part5/02-patch-existing-tweak.md` — Глава V.2. Исправления существующего твика
- `part5/03-runtime-patches.md` — Глава V.3. Runtime-патчи поведения приложения
- `part5/04-macho-metadata-patches.md` — Глава V.4. Изменения Mach-O и метаданных
- `part5/05-patch-matrix-26-0-vs-26-0-1.md` — Глава V.5. Контроль качества патча: пути, dylib, init, фильтры, настройки, совместимость 26.0 ↔ 26.0.1

### Часть VI. Исследовательская методика

- `part6/01-research-algorithm.md` — Глава VI.1. Воспроизводимый алгоритм «от наблюдения до работающего хука»
- `part6/02-static-vs-device.md` — Глава VI.2. Статический анализ vs наблюдения на устройстве

### Часть VII. Диагностика и совместимость с 26.0 ↔ 26.0.1

- `part7/01-tweak-wont-load.md` — Глава VII.1. Твик не загружается / dylib загружается, но hook не срабатывает
- `part7/02-crash-reports-logging.md` — Глава VII.2. Crash reports и полезное логирование
- `part7/03-26-0-vs-26-0-1-matrix.md` — Глава VII.3. Матрица отличий iOS 26.0 vs 26.0.1 для авторов твиков

### Приложения

- `appendix/A-glossary.md` — Приложение A. Словарь терминов
- `appendix/B-file-api-reference.md` — Приложение B. Справочник файлов и API проекта
- `appendix/C-verified-unverified.md` — Приложение C. Проверенные и непроверенные возможности
- `appendix/D-bibliography.md` — Приложение D. Библиография первичных источников

### Deep-dive — углублённые разборы инфраструктуры Dopamine

- `deep-dive/01-macho-detailed.md` — Mach-O подробно: header, load commands, segments, LC_*
- `deep-dive/02-dyld-chained-fixups.md` — dyld 4+ и chained fixups
- `deep-dive/03-pac.md` — Pointer Authentication (PAC)
- `deep-dive/04-ppl-and-a12-a13.md` — PPL (Page Protection Layer) и почему iOS 26 на A13 оказалась доступной
- `deep-dive/05-trustcache.md` — Trust cache: как Dopamine «подписывает» твики
- `deep-dive/06-codesign.md` — codesign.m: вспомогательные операции над подписью
- `deep-dive/07-physrw.md` — physrw и physrw_pte — чтение/запись физической памяти
- `deep-dive/08-inline-svc.md` — Inline SVC primitives
- `deep-dive/09-kcall.md` — kcall_Fugu14 vs kcall_arm64
- `deep-dive/10-exploits-overview.md` — Эксплойты Dopamine 3 для A13 + iOS 26: архитектурный обзор
- `deep-dive/11-xpf.md` — XPF (eXtensible Patchfinder)
- `deep-dive/12-choma.md` — ChOma — Mach-O библиотека, которую использует Dopamine
- `deep-dive/13-opainject.md` — opainject — тонкий инжектор dylib
- `deep-dive/14-litehook.md` — litehook — миниатюрная библиотека хуков
- `deep-dive/15-machomerger.md` — MachOMerger — склейка Mach-O для внедрения dyldhook

### Кейс-стади — пятнадцать проектов с полным кодом

- `cases/01-full-logger-tweak.md` — Кейс 1. Полноценный file-system tracer как переносимый твик
- `cases/02-pref-ui-end-to-end.md` — Кейс 2. Твик + собственная PreferenceBundle (Settings → твик)
- `cases/03-reboot-persistent-launcher.md` — Кейс 3. Автозапускатор своей службы после userspace reboot
- `cases/04-two-process-ipc.md` — Кейс 4. Два процесса, общающихся через XPC
- `cases/05-swizzle-with-group.md` — Кейс 5. Группировка хуков через `%group` и условное включение
- `cases/06-filesystem-redirect.md` — Кейс 6. Redirect файловых путей в своём приложении
- `cases/07-crash-safe-tweak-skeleton.md` — Кейс 7. Устойчивый к падениям скелет твика
- `cases/08-fishhook-selfdefense.md` — Кейс 8. Защита от fishhook других твиков
- `cases/09-swift-interop.md` — Кейс 9. Твик для Swift-приложения: что реально хукать
- `cases/10-cfprefsd-aware-prefs.md` — Кейс 10. Корректное чтение/запись prefs (cfprefsd + roothide)
- `cases/11-background-tracker.md` — Кейс 11. Трекер работы приложений в фоне
- `cases/12-ellekit-alt.md` — Кейс 12. Использование ellekit как рантайм-хукер напрямую
- `cases/13-dylib-only-no-logos.md` — Кейс 13. Твик без Logos — только C/ObjC + ellekit API
- `cases/14-trustcache-inject-helper.md` — Кейс 14. Собственный helper, вносящий cdhash в trustcache
- `cases/15-uninstall-safely.md` — Кейс 15. Безопасный uninstall твика без доступа к устройству

### Cookbook — двадцать коротких рецептов с полным кодом

- `cookbook/01-log-file-ops.md` — Рецепт 1. Логирование файловых операций в своём приложении
- `cookbook/02-urlsession-datatask.md` — Рецепт 2. Логирование URLSession-запросов в своём приложении
- `cookbook/03-pasteboard.md` — Рецепт 3. Перехват доступа к pasteboard
- `cookbook/04-controlcenter-module.md` — Рецепт 4. Собственный модуль Пункта управления
- `cookbook/05-springboard-icon-replace.md` — Рецепт 5. Замена иконки системного приложения в SpringBoard
- `cookbook/06-keychain-logging.md` — Рецепт 6. Логирование Keychain доступа внутри своего приложения
- `cookbook/07-url-scheme-hook.md` — Рецепт 7. Перехват `openURL:options:completionHandler:`
- `cookbook/08-la-context-hook.md` — Рецепт 8. Перехват LAContext (биометрическая проверка)
- `cookbook/09-settings-bundle.md` — Рецепт 9. PreferenceBundle с libroot
- `cookbook/10-screenshot-restrictions.md` — Рецепт 10. Снятие ограничения скриншота в своём приложении
- `cookbook/11-dlsym-logging.md` — Рецепт 11. Логирование всех `dlsym` в своём процессе
- `cookbook/12-load-init-trace.md` — Рецепт 12. Трейс `+load` и `-init` методов
- `cookbook/13-user-agent.md` — Рецепт 13. Переопределение User-Agent URLSession
- `cookbook/14-nsdate-override.md` — Рецепт 14. Подмена `NSDate` для теста
- `cookbook/15-location-requests.md` — Рецепт 15. Логирование запросов локации в своём приложении
- `cookbook/16-photo-picker.md` — Рецепт 16. Логирование `PHPickerViewController`
- `cookbook/17-toast.md` — Рецепт 17. Добавить тост в любой `UIViewController`
- `cookbook/18-sandbox-denials.md` — Рецепт 18. Фиксация sandbox-отказов в своём приложении
- `cookbook/19-xpc-service-client.md` — Рецепт 19. Клиент к XPC-сервису приложения
- `cookbook/20-fakeroot-paths.md` — Рецепт 20. Корректное обращение к файлам в rootless + RootHide

### Шпаргалки

- `cheatsheets/01-logos-syntax.md` — Шпаргалка 1. Синтаксис Logos
- `cheatsheets/02-theos-makefile.md` — Шпаргалка 2. Переменные Theos Makefile
- `cheatsheets/03-control-file.md` — Шпаргалка 3. control-файл Debian-пакета
- `cheatsheets/04-filter-plist.md` — Шпаргалка 4. Filter plist — все варианты
- `cheatsheets/05-libroot-api.md` — Шпаргалка 5. libroot API — все экспортированные функции
- `cheatsheets/06-libjailbreak-public.md` — Шпаргалка 6. Публичные функции libjailbreak для сторонних твиков
- `cheatsheets/07-jbserver-domains.md` — Шпаргалка 7. jbserver домены
- `cheatsheets/08-paths-table.md` — Шпаргалка 8. Таблица путей rootless vs RootHide
- `cheatsheets/09-analysis-commands.md` — Шпаргалка 9. Инспекционные команды для твиков и пакетов
- `cheatsheets/10-pac-keys.md` — Шпаргалка 10. PAC-ключи и когда они нужны автору твика

### iOS 26 — что именно поменялось

- `ios26-changes/01-uikit.md` — Изменения UIKit в iOS 26 с точки зрения автора твика
- `ios26-changes/02-swiftui.md` — SwiftUI в iOS 26
- `ios26-changes/03-springboard.md` — SpringBoard и Home Screen в iOS 26 (по открытым источникам)
- `ios26-changes/04-preferences.md` — Settings.app в iOS 26
- `ios26-changes/05-backgroundtasks.md` — BackgroundTasks и жизненный цикл в iOS 26
- `ios26-changes/06-notifications.md` — Уведомления в iOS 26
- `ios26-changes/07-shortcuts-intents.md` — Shortcuts и Intents в iOS 26
- `ios26-changes/08-privacy-security.md` — Privacy и Security в iOS 26
- `ios26-changes/09-networking.md` — Networking в iOS 26
- `ios26-changes/10-26-0-vs-26-0-1-delta.md` — Дельта 26.0 → 26.0.1 детально

### Продвинутые темы

- `advanced/01-multi-tweak-coord.md` — Продвинуто 1. Координация нескольких твиков
- `advanced/02-testing.md` — Продвинуто 2. Тестирование твика (unit-like)
- `advanced/03-cicd.md` — Продвинуто 3. CI/CD для твика
- `advanced/04-custom-injector.md` — Продвинуто 4. Пишем свой dylib-инжектор
- `advanced/05-jbserver-client.md` — Продвинуто 5. Собственный jbserver-клиент
- `advanced/06-package-formats.md` — Продвинуто 6. Форматы пакетов: `.deb` vs `.tipa` vs `.ipa`
- `advanced/07-trollstore-vs-dopamine.md` — Продвинуто 7. TrollStore vs Dopamine — для автора твика
- `advanced/08-lldb-debug.md` — Продвинуто 8. Отладка твика через LLDB
- `advanced/09-static-analysis.md` — Продвинуто 9. Статический анализ своего твика
- `advanced/10-release.md` — Продвинуто 10. Релиз твика: подписание, версионирование, host-репозитории

### Атлас системных демонов iOS 26

- `daemons/01-springboard.md` — SpringBoard
- `daemons/02-backboardd.md` — backboardd
- `daemons/03-cfprefsd.md` — cfprefsd
- `daemons/04-installd.md` — installd
- `daemons/05-lsd.md` — lsd (Launch Services)
- `daemons/06-runningboardd.md` — runningboardd
- `daemons/07-mediaserverd.md` — mediaserverd
- `daemons/08-apsd.md` — apsd (Apple Push Service)
- `daemons/09-nehelper.md` — nehelper (Network Extension helper)
- `daemons/10-trustd.md` — trustd (TLS trust evaluation)

### Атлас фреймворков

- `framework-atlas/01-uikit.md` — UIKit — что хукать, что — нет
- `framework-atlas/02-foundation.md` — Foundation
- `framework-atlas/03-corefoundation.md` — CoreFoundation
- `framework-atlas/04-security.md` — Security framework
- `framework-atlas/05-localauth.md` — LocalAuthentication
- `framework-atlas/06-usernotifications.md` — UserNotifications
- `framework-atlas/07-backgroundtasks.md` — BackgroundTasks
- `framework-atlas/08-network-urlsession.md` — Network.framework и URLSession
- `framework-atlas/09-photokit.md` — PhotoKit
- `framework-atlas/10-corelocation-coremotion.md` — CoreLocation и CoreMotion

### RE-walkthroughs — пошаговые упражнения по reverse engineering

- `re-walkthroughs/01-find-selector.md` — RE 1. Как найти частный селектор в SpringBoard iOS 26
- `re-walkthroughs/02-find-c-function.md` — RE 2. Как найти неэкспортированную C-функцию в фреймворке
- `re-walkthroughs/03-decrypt-ivars.md` — RE 3. Как изучить внутренние структуры (ivars) класса
- `re-walkthroughs/04-dyld-cache.md` — RE 4. Работа с dyld shared cache
- `re-walkthroughs/05-fs-diff.md` — RE 5. Как сравнить файловую систему 26.0 и 26.0.1

## Карта «Задача → глава»

| Что хотите сделать | Куда идти |
| --- | --- |
| Понять, что такое патч/хук/твик | I.1 |
| Выбрать процесс и фильтр инъекции | I.2, IV.3, шпаргалка 4 |
| Написать первый ObjC-хук | IV.1 → IV.4, cookbook 1–2 |
| Хукнуть C-функцию | IV.5, cookbook 11, RE 2 |
| Понять roothide и правильно читать файлы | III.3, III.10, шпаргалка 5, cookbook 20 |
| Правильно работать с prefs | II.7, кейс 10, cookbook 9 |
| Собрать `.deb` и установить (на устройстве) | IV.1, IV.10, advanced 6 |
| Разобраться с trust cache и cdhash | deep-dive 05, кейс 14 |
| Понять, что меняется в iOS 26 vs 18 | ios26-changes/*, VII.3 |
| Диагностировать «твик не грузится» | VII.1, VII.2 |
| Разобрать Dopamine сам по себе | III.1–III.12, deep-dive/* |
| Настроить CI/CD, LLDB, релиз | advanced 3, 8, 10 |
| Хукать SpringBoard | II.1, daemons/01, framework-atlas/01, RE 1, кейс 12 |

## Статус достоверности

Каждая глава помечена одной из четырёх меток в конце:

1. **Подтверждено исходниками** — путь `file:line` в репозитории Dopamine-roothide или сопутствующих.
2. **Подтверждено документацией** — Apple-документация, open-source dyld, iPhone Wiki, рецензируемые публикации.
3. **Требует проверки на устройстве** — факт правдоподобный, но автор книги физически не прогонял на A13 + iOS 26.
4. **Не подтверждено** — отмечено явно; читателю предлагается проверить и, если уместно, прислать правку.

См. `appendix/C-verified-unverified.md` для сводной матрицы.

## Credits

Книга написана как исследование открытых исходников Dopamine RootHide и официальной документации Apple/open-source dyld.
Все утверждения о коде проверяются по `file:line` в репозитории `rouukroo/dopamine-roothide` (а также smartlinked форках `opa334/Dopamine`, `roothide/*`, `apple-oss-distributions/dyld`).
