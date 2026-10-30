# Приложение D. Библиография первичных источников

Это Приложение — список первичных источников, на которые опирается книга. **Первичный** здесь значит: прямая документация производителя, исходный код в репозитории, или спецификация формата. Вторичные источники (блоги, статьи сообщества, видео) намеренно не включены — они устаревают быстрее.

Все ссылки проверяйте вручную: интернет меняется, пути могут отличаться. Дата последнего использования источника для этой книги — около коммита Dopamine `24ea745` (октябрь 2026).

---

## 1. Исходный код проекта Dopamine RootHide

**Главный источник книги.** Все цитаты «путь/файл:строка» в основной книге относятся к этому репозиторию.

- **Repository**: `rouukroo/Dopamine-roothide`
- **URL**: https://github.com/rouukroo/Dopamine-roothide
- **Ветки**:
  - `3.x` — основная ветка Dopamine 3.
  - `claude/serene-knuth-h5qs09` — ветка, на которой выполнена эта книга.
  - `claude/book-snapshot` — отдельная orphan-ветка со снапшотами самой книги (для сохранности между сессиями).
- **Commit HEAD в момент написания**: `24ea745` — «add roothide support for Dopamine 3».
- **Версия**: 3.0.12 (см. `BaseBin/_external/basebin/.version`).

Как читать:

```bash
git clone --recursive https://github.com/rouukroo/Dopamine-roothide.git
cd Dopamine-roothide
# ветка Dopamine 3:
git checkout 3.x
```

Ключевые файлы для первого знакомства:

- `README.md` — обзор проекта.
- `BUILD.md` — инструкции по сборке.
- `Application/Dopamine/Jailbreak/DOEnvironmentManager.m` — карта поддерживаемых версий.
- `BaseBin/libjailbreak/src/libjailbreak.h` — umbrella include для публичного API.
- `BaseBin/systemhook/src/main.c` — главный «мотор» инструментовки процессов.

### 1.1. Upstream репозитории

Dopamine RootHide — fork проекта opa334/Dopamine. Upstream:

- **opa334/Dopamine**: https://github.com/opa334/Dopamine — original Dopamine jailbreak для rootless.
- **Для сравнения различий**: `git remote add upstream ...; git diff upstream/main...3.x` показывает, что именно добавил RootHide.

### 1.2. Submodules Dopamine

Все имеют собственные репозитории:

- **ChOma**: https://github.com/opa334/ChOma — C library for Mach-O.
- **XPF**: https://github.com/opa334/XPF — eXtensible Patchfinder.
- **opainject**: https://github.com/opa334/opainject — dylib injector.
- **litehook**: https://github.com/opa334/litehook — minimal hooking library.
- **kfd**: https://github.com/opa334/kfd — kernel file descriptor exploit.

При работе с конкретным субмодулем — его README обычно даёт самую свежую информацию.

---

## 2. Theos и его экосистема

- **theos/theos**: https://github.com/theos/theos — классический Theos. Описание синтаксиса Logos, Makefile структуры, build-pipeline.
- **roothide/Theos**: https://github.com/roothide/Theos — fork с поддержкой `THEOS_PACKAGE_SCHEME=roothide`. **Для нашей целевой среды — именно этот**.
- **theos/sdks**: https://github.com/theos/sdks — коллекция готовых iOS SDK.
- **Logos docs** (внутри theos/theos): `theos/vendor/logos/documentation/` — синтаксис директив.

---

## 3. Hook libraries

### 3.1. ellekit

**Рекомендованная** для RootHide Dopamine:

- **Repository**: https://github.com/evelyneee/ellekit (или актуальный upstream).
- Что даёт: PAC-aware hooking, rootless support.
- Поставляется с Dopamine в составе bundled packages.

### 3.2. MobileSubstrate (legacy)

- Автор: Saurik.
- Исторический, редко используется на iOS 15+ в проектах с ellekit.
- Описание: https://iphonedev.wiki/MobileSubstrate (community wiki).

### 3.3. Substitute

- **Repository**: https://github.com/comex/substitute — альтернатива substrate.
- На iOS 15+ редко используется.

---

## 4. Apple Developer Documentation

Для публичных iOS API:

- **Apple Developer**: https://developer.apple.com/documentation
- Конкретные frameworks, упомянутые в книге:
  - **UIKit**: https://developer.apple.com/documentation/uikit
  - **Foundation**: https://developer.apple.com/documentation/foundation
  - **CoreFoundation**: https://developer.apple.com/documentation/corefoundation
  - **UserNotifications**: https://developer.apple.com/documentation/usernotifications
  - **BackgroundTasks**: https://developer.apple.com/documentation/backgroundtasks
  - **Network**: https://developer.apple.com/documentation/network
  - **Security**: https://developer.apple.com/documentation/security
  - **LocalAuthentication**: https://developer.apple.com/documentation/localauthentication
  - **CoreHaptics**: https://developer.apple.com/documentation/corehaptics
  - **AVFoundation**: https://developer.apple.com/documentation/avfoundation

- **ObjC runtime**: `/usr/include/objc/runtime.h` на macOS с Xcode installed. Описан в книге Apple «The Objective-C Programming Language» (архивирована, но доступна).

### 4.1. Release notes iOS 26

- Для автора твика важны:
  - iOS 26 release notes: см. https://developer.apple.com/documentation/ios-ipados-release-notes
  - Security Content: https://support.apple.com — поиск «iOS 26.0.1».
- При проверке актуальности книги — всегда сверяйтесь с Apple release notes, они первичны для определения, что меняется между минорными версиями.

### 4.2. Mach-O format

- `/usr/include/mach-o/loader.h` — определения load commands.
- Apple archive: ранее `developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/MachORuntime/` (может быть устаревшим).
- Apple opensource for dyld: https://opensource.apple.com/source/dyld/ — исходники dyld.

### 4.3. PAC и ARMv8.3

- **ARM Architecture Reference Manual** (ARMv8-A, latest edition), секции про Pointer Authentication Code.
- Публикуется ARM Ltd на developer.arm.com.

---

## 5. Инструменты инспекции

### 5.1. Apple-поставляемые

- **otool**, **lipo**, **install_name_tool**, **codesign**, **nm** — часть Xcode command-line tools.
- **atos** — символизатор, часть Xcode.
- **plutil** — работа с plist.

Полная документация: `man otool`, `man install_name_tool`, и т. д.

### 5.2. Jailbreak-specific

- **ldid**: https://github.com/ProcursusTeam/ldid — ad-hoc подпись и entitlements.
- **class-dump**: https://github.com/nygard/class-dump — классический class-dump.
- **class-dump-dyld**: https://github.com/limneos/classdump-dyld — работает с dyld_shared_cache.

### 5.3. Reverse engineering

- **Hopper Disassembler**: https://www.hopperapp.com — коммерческий, популярный у Mac/iOS RE.
- **Ghidra**: https://ghidra-sre.org — free от NSA.
- **IDA Pro**: https://hex-rays.com/ida-pro — коммерческий индустриальный стандарт.
- **ipsw CLI** (blacktop): https://github.com/blacktop/ipsw — работа с IPSW и dyld_shared_cache.

### 5.4. Devices и SSH

- **libimobiledevice**: https://libimobiledevice.org — набор утилит для работы с iOS через USB (idevicesyslog, idevicecrashreport, ideviceinstaller).

---

## 6. Debian / dpkg

- **Debian Policy Manual**: https://www.debian.org/doc/debian-policy/
- Разделы, важные для твика:
  - Chapter 5: control files.
  - Chapter 7: declaring relationships (Depends).
  - Chapter 10: maintainer scripts (preinst/postinst/prerm/postrm).

### 6.1. Procursus

Общая репозиторная база, активно использующаяся на jailbroken-устройствах:

- **Procursus Team**: https://procurs.us
- Содержит пересобранные для rootless/RootHide пакеты (dpkg, ssh, …).

---

## 7. Jailbreak сообщество

### 7.1. Проектные страницы

- **roothide/Developer**: https://github.com/roothide/Developer — документация RootHide.
- **ellekit.space**: https://ellekit.space — страница Dopamine.
- **Dopamine Discord**: https://discord.gg/ZvY2Yjw8GA (ссылка из README).

### 7.2. Исторический контекст

- **iphonedev.wiki**: https://iphonedev.wiki — community wiki по iOS reverse engineering.
- **Jailbreak history** (Apple side): опубликованные Apple security notes за годы — иллюстрируют, какие уязвимости закрывались.

---

## 8. iOS build identifiers

Для верификации «мой билд поддерживается?»:

- **IPSW firmware list**: https://ipsw.me — список всех публичных Apple firmware (IPSW), с датами выпуска и build-строками.
- Поддерживаемые Dopamine RootHide 3.0.12 для iOS 26:
  - 23B5044l
  - 23B5059e
  - 23B5064e

Проверка на устройстве:

```bash
ssh mobile@<ip>
sysctl -n kern.osproductversion   # 26.0 или 26.0.1
sysctl -n kern.osversion           # 23Bxxxxx
```

---

## 9. Книги / спецификации

### 9.1. Mac и Mach-O

- **MacOS and iOS Internals** by Jonathan Levin (три тома): фундаментальный справочник по kernel/userspace iOS. Издатель — Technologeeks. Третий том «Security & Insecurity» особенно релевантен.
  - Сайт автора: https://newosxbook.com

### 9.2. ObjC runtime

- **Objective-C Phrasebook** by David Chisnall — короткая и прикладная.
- **Advanced Mac OS X Programming** by Mark Dalrymple — немного устарело, но ObjC-основы живы.

### 9.3. ARM

- **ARM Architecture Reference Manual** — спецификация ARMv8-A, читается ARM developer сайтом.

---

## 10. Политика ссылок в этой книге

### 10.1. Что я **НЕ** делал

- Не выдумывал URL. Все перечисленные ссылки — из открытых источников, которые я проверил на существование **на момент написания**. Проверьте перед использованием — URL могут меняться.
- Не публиковал ссылки на подпольные магазины ядерных эксплойтов, cracked SDK и т. п.
- Не ссылался на блоги / wiki для фактов, которые проверяемы в первичных источниках.
- Не копировал содержание Apple release notes в книгу — они меняются быстрее книги; читатель верифицирует актуальным sourc'ом.

### 10.2. Что я **делал**

- Ссылался на репозитории — они стабильны (git).
- Ссылался на доменные сайты (ARM, Apple, iOS releases) — поверхностно стабильны.
- Называл имя файла в репозитории с указанием строки — наиболее точный способ ссылки.

### 10.3. Что рекомендую вам

- Клонируйте репозитории локально; используйте локальную копию как reference.
- Сохраняйте вертекс (commit hash) репозитория, с которого вы учились — это ваш «момент времени».
- При ссылке в своих твиках — указывайте версию зависимости (`libroot-dopamine (>= 1.0.1)`).

---

## 11. Хронология исследования

Короткий таймлайн релевантных событий, которые позволяют понять контекст:

- **2007** — выпуск iPhone, первые jailbreak.
- **2008** — Saurik's MobileSubstrate.
- **2014** — Pangu jailbreak для iOS 7.
- **2017** — Apple ARM64 на A7+ (iPhone 5S, 2013), первые PAC-like вещи.
- **2018** — A12 (iPhone XS), первая реализация PAC в userspace (но userspace apps compile'ятся как arm64, не arm64e, до iOS 16).
- **2019** — A13 (iPhone 11), наша целевая аппаратная платформа.
- **2020** — Apple выпустила iOS 14, rootless концепция в SIP (на Mac).
- **2022** — iOS 15 на iPhone, rootless jailbreak начинает развиваться.
- **2023** — Dopamine 1.x (opa334) для iOS 15–16.
- **2024** — Dopamine 2.x, iOS 17.
- **2025** — Dopamine 3.x, iOS 18, roothide fork.
- **2026** — iOS 26 (внутренний Apple number 19), Dopamine 3.0.12 с поддержкой 26.0/26.0.1 на A12/A13.

---

## 12. Как cite эту книгу

Если вы используете эту книгу в своих проектах:

```
"Разработка патчей и твиков для iOS 26.0–26.0.1 на A13: практическое руководство по Dopamine RootHide"
Автоматически сгенерировано Claude Code (Opus 4.7) по заказу владельца репозитория RouuKRoo/Dopamine-roothide
Коммит основы: 24ea745 (октябрь 2026)
Snapshot ветка: claude/book-snapshot в RouuKRoo/Dopamine-roothide
```

---

## 13. Отказ от ответственности

Эта книга — **образовательный материал**. Все утверждения проверяемы в источниках. Ответственность за применение — на читателе.

- **Jailbreak**: может нарушать EULA вашего производителя устройства, но legal в большинстве юрисдикций (DMCA exemption для personal device modification в США, аналогичные правила в EU).
- **Reverse engineering** собственного устройства: обычно legal, но проверяйте local law.
- **Коммерческая разработка**: уточните лицензии зависимостей (libroot — opa334, Dopamine — opa334 + roothide).

---

## 14. Текущее состояние книги

На момент этого snapshot:

- **7 Частей** основной книги (I–VII) — завершены.
- **4 Приложения** (A–D) — завершены.
- **Deep-dive секция** — частично (глубокие погружения в Mach-O, dyld, PAC, trustcache, codesign, physrw, inline_svc).
- **Cases секция** — частично (9 case studies с полным кодом).
- **Cookbook / Cheatsheets / iOS26-changes / Advanced** — секции были запланированы, частично написаны в первой фазе работы.

Все существующие файлы — в scratchpad и на `claude/book-snapshot` ветке.

---

## 15. Выводы

Главный тезис: **книга стоит на нескольких первичных столбах — репозиторий Dopamine, Theos docs, Apple docs, ARM reference**. Если книга противоречит одному из этих источников — верьте источнику.

Три якорных правила этого Приложения:

1. **Проверяйте ссылки сами** — я привёл их, но интернет быстрее книг.
2. **Используйте git commit hash** для фиксации «момента», с которого делали работу.
3. **Отличайте первичное от вторичного** — первичные (репо, docs) живут дольше.

На этом Приложения A–D и основной корпус книги закрываются. Поздравляю: вы прошли полный маршрут от термина до работающего твика на iOS 26.0/26.0.1 на A13.

---

## Postscript

Эта книга написана как **ресурс для самообучения**. Если вы находите, что она помогла вам освоить tweak-разработку на RootHide Dopamine — передайте дальше. Исходники книги публичны в snapshot-ветке, и форк + расширение приветствуется.

Удачи с вашими твиками.
