# Deep-dive 13. opainject — тонкий инжектор dylib

`opainject` — одна из двух dylib-инжекторов в составе Dopamine. В отличие от `DYLD_INSERT_LIBRARIES` (который работает при запуске процесса), `opainject` инжектит в **уже работающий** процесс. В этой главе — его роль и ограничения.

## 1. Что это

`opainject` — отдельный бинарник (`BaseBin/opainject/opainject`), собираемый как subproject Dopamine. Исходники — в репозитории `opa334/opainject` (submodule `.gitmodules`).

Назначение: взять PID работающего процесса + путь к `.dylib` и загрузить эту `.dylib` в адресное пространство целевого процесса.

Технически делает это через mach ports: `task_for_pid` → `mach_vm_allocate` в чужом процессе → копирование загрузчика → запуск нового треда → `dlopen` там → выход треда.

## 2. Два пути инъекции

| Путь | Когда |
|---|---|
| `DYLD_INSERT_LIBRARIES` | При запуске процесса, через env. На Dopamine делает `launchdhook` в `spawn_hook` |
| `opainject` | Процесс уже запущен, env менять нельзя |

Для нашего случая (Dopamine инъектит `systemhook` во все процессы) — используется первый путь. Второй применяется в редких специфических сценариях.

## 3. Где opainject используется в проекте

```bash
grep -rln "opainject\|OPAINJECT" BaseBin --include="*.c" --include="*.m" --include="Makefile"
```

Находки:

- `BaseBin/Makefile:33–35` — цель сборки.
- `BaseBin/libroothider/src/common.m` — используется для специализированной инъекции.
- `BaseBin/libroothider/src/jailbreakd.c` — client-side helper.

## 4. API

Прямого C-API у opainject нет — это CLI бинарник:

```
opainject <pid> <path-to-dylib>
```

Пример:

```bash
opainject 1234 /var/jb/Library/MyTweak.dylib
```

Процесс 1234 получит загруженную dylib в своё адресное пространство. Её `__attribute__((constructor))` выполнится.

## 5. Требования

Для успешной работы opainject:

1. **Entitlements** — требует `task_for_pid-allow`, `platform-application`. Поставляется с ad-hoc подписью + trust cache.
2. **jbroot доступен** — opainject сам линкуется с `libroot` и правильно резолвит пути.
3. **Target валидно подписан** в trust cache — чтобы `.dylib` могла пройти AMFI load.
4. **Target процесс не в blacklist** — некоторые системные процессы отвергают инъекцию.

## 6. Отличие от DYLD_INSERT

Таблица для автора твика:

| Аспект | DYLD_INSERT_LIBRARIES | opainject |
|---|---|---|
| Когда срабатывает | до `main()` целевого процесса | в любой момент |
| Требует перезапуск target | да | нет |
| Устанавливается через | env в `posix_spawn` | mach-операция |
| Доступен всем процессам | да (через `systemhook`) | только привилегированным |
| Виден в crash reports | как «loaded via DYLD_INSERT» | как «dlopen'd at runtime» |

## 7. Применение в собственных сценариях

Автор твика **обычно не вызывает** opainject напрямую. Полезен в следующих случаях:

1. **Debug workflow** — инжектировать diagnostic dylib в живой процесс, не ломая его state перезапуском.
2. **Инъекция в процесс, стартовавший до jailbreak** — редко, но бывает.
3. **Тестирование сборки** — быстрая проверка без переустановки `.deb`.

Пример dev-итерации:

```bash
# Собрали твик:
make clean package THEOS_PACKAGE_SCHEME=roothide

# Скопировали на устройство:
scp .theos/obj/MyTweak.dylib mobile@<ip>:/tmp/

# Инжектировали в живой процесс (вместо respring):
ssh mobile@<ip>
pid=$(pgrep -f TestbedApp)
sudo opainject $pid /tmp/MyTweak.dylib
```

**Внимание**: без `jbclient_trust_file_by_path("/tmp/MyTweak.dylib")` перед опайнджектом dyld откажется грузить. Для dev-итераций удобнее сделать твик частью установленного `.deb`.

## 8. Что внутри (архитектурно)

Общие шаги реального opainject (детали — в репозитории opa334/opainject):

1. `task_for_pid(mach_task_self(), pid, &task_port)` — получить task port целевого.
2. Аллокировать memory внутри целевого через `mach_vm_allocate`.
3. Сконструировать небольшой «загрузчик»: ARM64 код, вызывающий `dlopen(path, RTLD_NOW)`.
4. `mach_vm_write` — записать этот код + путь к dylib + stack.
5. `thread_create_running` — создать новый тред с pc на нашем коде.
6. Тред выполняет `dlopen`, затем `thread_terminate`.
7. Проверить успешность через `dlopen_from` return value.

Это тот же механизм, что и в hookd, но для специфической задачи (dlopen вместо произвольной записи).

## 9. Ограничения

- **PAC**: на arm64e процессах нужно корректно обрабатывать PAC-подписанные указатели в thread state. На arm64 (A13) — проще.
- **Codesign**: dylib должна быть в trust cache; opainject не делает это за вас.
- **Sandbox**: если target в strict sandbox, его `dlopen` может быть ограничен.
- **Debugging**: если target уже под отладкой — конфликт thread_create.

## 10. Что подтверждено

- **Подтверждено**: opainject — submodule `.gitmodules`, собирается как часть `basebin.tar` (`BaseBin/Makefile:33–35`).
- **Подтверждено**: ссылки на opainject из `libroothider` — `common.m`, `jailbreakd.c`.
- **Требует проверки**: конкретное API командной строки — может варьироваться между версиями, см. README репозитория opa334/opainject.
- **Не подтверждено** (без clone submodule): конкретный исходный код; возможно есть расширения.

## 11. Выводы

Главный тезис: **opainject — инструмент edge-case'ов**. В обычном twic-flow Dopamine полагается на `DYLD_INSERT_LIBRARIES` через `systemhook`. opainject применяется, когда нужно инжектировать в живой процесс.

Для автора твика в 99% случаев этот инструмент не нужен. В 1% — используется через SSH-команду, не через C-API.

## 12. Что дальше

- `litehook` — Deep-dive 14 (минимальная библиотека хуков).
- `MachOMerger` — Deep-dive 15 (склейка dyld+dyldhook).
- `hookd` — глава III.7 (другой инжектор, через mach).
- Для dev workflow — глава IV.10.
