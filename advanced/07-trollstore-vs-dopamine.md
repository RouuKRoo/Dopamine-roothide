# Продвинуто 7. TrollStore vs Dopamine — для автора твика

Две разные экосистемы для «нестандартной» разработки под iOS. Иногда конкурируют, иногда — дополняют. В этой главе — границы и когда что использовать.

## 1. Короткая разница

**TrollStore** — permasigning инструмент. Устанавливает `.ipa` **без** 7-дневного срока, используя баг в `CoreTrust`. Приложение получает все entitlements, которые ему прописаны (включая `platform-application`, etc.).

- **НЕ** jailbreak.
- **НЕ** требует эксплойта ядра.
- **НЕ** даёт tweak injection.
- **ДАЁТ** permasigned приложения с expansive entitlements.

**Dopamine** — полный jailbreak. Эксплойт ядра + userland инъекция + trust cache.

- **ДАЁТ** всё: kernel RW, system hooks, tweak injection, custom dylib loading.
- **ТРЕБУЕТ** periodic re-activation после cold reboot.

## 2. Совместимость

TrollStore работает на:

- iOS 14.0 – 16.6.1 (CoreTrust bug).
- iOS 17.0 (с ограничениями).

Dopamine работает на:

- iOS 15.0–18.7.1 на всех чипах arm64/arm64e.
- **iOS 26.0 и 26.0.1** на A12/A13 (наша целевая среда).

На iOS 26 **TrollStore недоступен** (как минимум публично на момент книги). Для этой версии — только Dopamine. Это важный факт: автор твика на iOS 26 **не выбирает** между ними, выбирает только Dopamine.

Но Dopamine сам распространяется через TrollStore на старых iOS, а на 26 — через другие механизмы (разового permasigning или подобное). Детали — в документации проекта.

## 3. Что может TrollStore

Если доступен (на старых iOS):

- Permasign любого `.ipa` → приложение работает всегда.
- Прописать entitlements (включая private) в бинарнике.
- Запустить CLI утилиты (кoторые внутри `.ipa` wrapper).

Что **не** может:

- Инжектировать dylib в чужой процесс.
- Хукать методы системных процессов.
- Писать в kernel memory.

## 4. Что может Dopamine

Всё вышеперечисленное **плюс**:

- Инжекция `systemhook.dylib` в каждый процесс.
- Загрузка твиков через TweakLoader.
- Trust cache management.
- Kernel R/W через XNU-эксплойт.
- Modification of system behaviour (SpringBoard, cfprefsd, lsd).

## 5. Как выбрать для автора

| Цель | Выбор |
|---|---|
| Hook UIKit метод в приложении | **Dopamine** (нужен tweak injection) |
| Запустить CLI утилиту с entitlements | **TrollStore** (если доступен) или Dopamine helper |
| Permasigned stand-alone приложение | **TrollStore** (если доступен) |
| Полный jailbreak | **Dopamine** |
| Приложение + tweak объединённо | **Dopamine** (включает .app через .deb с `_INSTALL_PATH = /Applications`) |

Для iOS 26 на A13 — **всегда Dopamine**.

## 6. Dopamine устанавливается через TrollStore

Исторически:

- Пользователь ставит TrollStore (на поддерживаемую iOS).
- Через TrollStore ставит Dopamine.tipa.
- Запускает Dopamine, нажимает Jailbreak.
- Dopamine разворачивает basebin, активирует jailbreak.

На iOS 26 (где TrollStore не работает) — альтернативные пути установки Dopamine. Детали в проекте roothide/Dopamine.

## 7. TrollStore apps vs Dopamine apps

Приложение, установленное через TrollStore:

- Permasigned — работает всегда.
- Может иметь `platform-application` и другие расширенные entitlements.
- **Не** получает `DYLD_INSERT_LIBRARIES=.../systemhook.dylib` автоматически.
- Если jailbreak не активен — tweaks в нём не грузятся.

Приложение, установленное через Dopamine (.deb с .app):

- Устанавливается только после jailbreak.
- Получает `systemhook` при каждом запуске.
- Загружает все tweaks по filter plist.
- Исчезает (функционально), если jailbreak не активен.

Для production — автор обычно хочет и то, и другое. Приложение-companion — через TrollStore (чтобы работало без JB), tweaks — через Dopamine.

## 8. Entitlements

TrollStore может прописать **любые** entitlements в подписи. Включая private Apple ones. Это даёт возможность:

- Создавать tool-утилиты с `task_for_pid-allow`, `com.apple.private.cs.debugger` (без jailbreak!).
- Собирать helper-утилиты для авто-установки Dopamine.

Dopamine использует trust cache для своих собственных entitled binary (`hookd`, `jailbreakd`, глава III.7, III.8). Пользовательские приложения получают entitlements через подпись + trust cache.

## 9. Тестирование совместимости

Если ваш твик должен работать «везде где можно» — тест-матрица:

| Среда | Твик | Companion app |
|---|---|---|
| iOS 15–17 + TrollStore (без JB) | — не загружается | работает (permasigned) |
| iOS 15–17 + Dopamine + TrollStore | работает | работает |
| iOS 26.0/26.0.1 + Dopamine | работает | работает (но установка через .deb с .app) |
| iOS без ничего | — | работает временно (7 дней) |

## 10. Общее с trust cache

И TrollStore, и Dopamine используют trust cache механизм:

- Static trust cache — зашит в firmware; содержит cdhash Apple binary.
- Dynamic trust cache — загружается при runtime (TrollStore при installation; Dopamine при каждом jailbreak).

Для автора приложения/твика:

- TrollStore добавит cdhash вашего `.ipa` в dynamic tc при install.
- Dopamine добавит cdhash вашего `.dylib` при dpkg -i (через хуки в dpkg).

Это даёт AMFI-approve для не-Apple-signed бинарников.

## 11. Распространённые заблуждения

| Заблуждение | Реальность |
|---|---|
| «TrollStore = light jailbreak» | Нет, не JB. Нет kernel RW. Нет tweak injection. |
| «Dopamine работает без TrollStore» | На iOS 26 — да (другие механизмы install). На старых — через TrollStore. |
| «Твик можно установить через TrollStore» | Нет. Нужен jailbreak для TweakLoader. |
| «TrollStore повреждает iOS» | Нет. Он эксплуатирует CoreTrust bug для подписи, не меняет systеm. |
| «Dopamine устанавливает твики» | Нет. Dopamine даёт jailbreak; tweaks устанавливаются Sileo/Zebra через dpkg. |

## 12. Диагностика «не загружается»

Если ваш пользователь жалуется:

1. Проверить — jailbreak активен? (`ls /basebin` на устройстве через SSH).
2. Если нет — пользователь должен запустить Dopamine.app → Jailbreak.
3. После активации — твик загружается при следующем запуске target приложения.
4. Если TrollStore app:
   - Permasigned, работает.
   - Но твиков **в нём** нет без jailbreak.

## 13. Что подтверждено

- **Подтверждено**: TrollStore — community project, использует CoreTrust bug (публичное.
- **Подтверждено**: Dopamine 3 поддерживает iOS 26.0/26.0.1 на A12/A13 (глава I.5).
- **Подтверждено**: `main.m` Dopamine.app имеет TrollStore-режим (`trollstore delete-bootstrap`).
- **Требует проверки**: конкретный путь установки Dopamine на iOS 26 — зависит от доступности TrollStore или других методов на конкретном билде.

## 14. Выводы

Главный тезис: **TrollStore и Dopamine — не конкуренты**. TrollStore — permasigning (без JB). Dopamine — полный JB. Автор твика на iOS 26 — **Dopamine**. Если нужно permasigned companion app — ищет пути (TrollStore если доступен, иначе — специфические механизмы).

## 15. См. также

- Advanced 6 — package formats.
- Глава I.5 — что jailbreak даёт.
- Глава III.1 — Dopamine.app.
- Deep-dive 05 — trust cache.
