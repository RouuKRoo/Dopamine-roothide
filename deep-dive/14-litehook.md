# Deep-dive 14. litehook — миниатюрная библиотека хуков

`litehook` — внутренняя library хуков, которую Dopamine использует для **своих** компонентов (`systemhook`, `launchdhook`, `jailbreakd`). Автор твика напрямую её не вызывает, но знание её API — важная часть понимания того, как Dopamine сам себе ставит хуки.

## 1. Что это

`litehook` — submodule проекта (`BaseBin/_external/modules/litehook/`), автор opa334. Назначение: дать компонентам Dopamine возможность хукать функции **без зависимости от substrate/substitute/ellekit**.

Почему нужна отдельная библиотека:

- Компоненты Dopamine загружаются **раньше** любого твик-инжектора (`systemhook` стартует до TweakLoader).
- Компоненты Dopamine не могут зависеть от «пользовательской» части (ellekit — пакет, который устанавливается через dpkg после jailbreak).
- litehook должен работать с минимумом зависимостей (можно линковать статически).

Результат: небольшая C-библиотека с базовым набором операций, слинкованная внутрь каждого компонента Dopamine.

## 2. API

Из использования в коде (grep по `BaseBin/`):

### 2.1. `litehook_hook_function`

```c
void litehook_hook_function(void *target, void *replacement);
```

Назначение: inline hook — записать переход в начало `target` функции на `replacement`.

Пример использования (`BaseBin/systemhook/src/main.c:510`):

```c
litehook_hook_function(csops, csops_hook);
litehook_hook_function(csops_audittoken, csops_audittoken_hook);
litehook_hook_function(necp_match_policy, necp_match_policy_hook);
```

Семантика: все вызовы `csops` в этом процессе теперь идут в `csops_hook`. Оригинальная реализация достижима через внутренний trampoline (который litehook хранит).

Для вызова оригинала в хуке — проект использует `syscall(SYS_csops, ...)` в обход функции (глава III.5), потому что это проще, чем держать trampoline вручную.

### 2.2. `litehook_rebind_symbol`

```c
void litehook_rebind_symbol(uint32_t flags, void *target, void *replacement, void **origOut);
```

Назначение: fishhook-style rebind — поменять GOT-запись символа. Работает только для межмодульных вызовов (не inline).

Пример (`BaseBin/launchdhook/src/main.m:190`):

```c
litehook_rebind_symbol(LITEHOOK_REBIND_GLOBAL,
                      (void *)sysctlbyname,
                      (void *)sysctlbyname_hook,
                      NULL);
```

Разбор:

- `LITEHOOK_REBIND_GLOBAL` — применить ко всем загруженным образам (а не одному).
- `target` — функция, которую хотим перехватить.
- `replacement` — наша версия.
- `origOut` — куда сохранить оригинальный указатель (nullable).

Другие примеры:

- `xpc_dictionary_get_value` → `xpc_dictionary_get_value_hook` (`daemon_hook.m:84`).
- `xpc_receive_mach_msg` → `xpc_receive_mach_msg_hook` (`xpc_hook.c:67`).
- `memorystatus_control` → `memorystatus_control_hook` (`jetsam_hook.c:26`).
- `sandbox_check_by_audit_token` → `sandbox_check_by_audit_token_hook` (`ipc_hook.c:48`).

### 2.3. `litehook_hook_memory`

```c
kern_return_t (*litehook_hook_memory)(void *target, void *source, size_t sourceSize);
```

Назначение: записать `sourceSize` байт из `source` в адрес `target`. Это нижний уровень inline hook.

Interesting: в коде Dopamine `litehook_hook_memory` — не функция, а **указатель на функцию**, и его можно переназначить:

```c
// systemhook/src/common/hookd_external.c:17
*EKHookMemoryRaw_ptr = litehook_hook_memory_hookd;

// launchdhook/main.m:178
litehook_hook_memory = litehook_hook_memory_hookd;
```

Это позволяет во время работы переназначить, **как именно** litehook записывает в память — через свою `vm_write` или через mach-RPC к `hookd` (глава III.7).

## 3. Константы

### 3.1. `LITEHOOK_REBIND_GLOBAL`

Flag для `litehook_rebind_symbol` — применить ко всем загруженным образам.

Другие варианты (гипотетические, из общей практики fishhook):

- `LITEHOOK_REBIND_LOCAL` — только текущий образ.
- `LITEHOOK_REBIND_PRIVATE` — не публикуемый символ.

Точный набор зависит от версии libhook; проверяется в submodule.

## 4. Как litehook вставляется в Dopamine

Из `BaseBin/libjailbreak/Makefile:5`:

```make
CFLAGS = ... -I../_external/modules/litehook/src ...
```

И в сборочной цели (неявно через `$(wildcard ../_external/modules/litehook/src/*.c)`):

```make
$(TARGET): $(wildcard src/*.c src/*.m src/*.S ../_external/modules/litehook/src/*.c)
```

То есть **все `.c` файлы litehook статически линкуются в libjailbreak.dylib**. Внутри каждого компонента Dopamine (который линкуется с libjailbreak) есть собственная копия litehook.

Это упрощает deployment: один basebin.tar, никаких внешних зависимостей от версии litehook.

## 5. Разница с ellekit

| Аспект | litehook | ellekit |
|---|---|---|
| Для кого | Внутренние компоненты Dopamine | Сторонние твики |
| Как распространяется | Статически в каждом компоненте | Отдельный пакет `ellekit` |
| Public C API | Да, через header | Да, через EKHook* API |
| Logos backend | Нет (не через `%hookf`) | Да |
| PAC-support | Да | Да |

Для автора твика: **используйте ellekit** через Logos `%hookf`. litehook — слишком низкоуровневая и не поддерживается как публичный API.

## 6. Почему два хук-библиотеки

Разумный вопрос: «почему Dopamine не использует ellekit внутри своих компонентов?»

Причины:

1. **Циклическая зависимость**. ellekit устанавливается после первой загрузки Dopamine; но `systemhook` нужен **с** первой загрузки. Нельзя линковаться с ellekit до того, как jailbreak реально разворачивается.
2. **Контроль размера**. Dopamine внутри хочет минимум кода. ellekit — универсальная библиотека с большим surface area; litehook — минимальная.
3. **Независимость от стороннего кода**. Если ellekit получит баг — компоненты Dopamine продолжают работать.
4. **Контроль безопасности**. Внутренний хук-слой может быть «подкрутен» под нужды jailbreak-эксплойта.

## 7. Пример полного вызова

Из `BaseBin/launchdhook/src/jetsam_hook.c:9–27`:

```c
#include <sys/sysctl.h>
#include <string.h>
#include <unistd.h>
#include <kern_memorystatus.h>
#include <substrate.h>   // для sanity-check типов
#include <litehook.h>

int (*memorystatus_control_orig)(uint32_t command, int32_t pid, uint32_t flags,
                                  void *buffer, size_t buffersize);

int memorystatus_control_hook(uint32_t command, int32_t pid, uint32_t flags,
                               void *buffer, size_t buffersize)
{
    if (command == MEMORYSTATUS_CMD_SET_JETSAM_TASK_LIMIT) {
        return 0;
    }
    return memorystatus_control_orig(command, pid, flags, buffer, buffersize);
}

void initJetsamHook(void)
{
    memorystatus_control(MEMORYSTATUS_CMD_SET_JETSAM_TASK_LIMIT, 1, -1, NULL, 0);
    memorystatus_control(MEMORYSTATUS_CMD_SET_JETSAM_HIGH_WATER_MARK, 1, -1, NULL, 0);
    memorystatus_control_orig = memorystatus_control;
    litehook_rebind_symbol(LITEHOOK_REBIND_GLOBAL,
                          memorystatus_control,
                          (void *)memorystatus_control_hook,
                          NULL);
}
```

Пошаговый разбор:

1. Объявляется `memorystatus_control_orig` — указатель на оригинал, пока NULL.
2. В `memorystatus_control_hook` — наша логика.
3. В init: сначала **вручную** вызываем `memorystatus_control` с нужными параметрами (устанавливаем лимиты до того, как хук активен).
4. Сохраняем `memorystatus_control_orig = memorystatus_control` — на этом моменте это ещё реальный указатель.
5. `litehook_rebind_symbol` подменяет GOT: теперь любой вызов `memorystatus_control` в этом процессе идёт в наш хук. `_orig` остался со старым указателем (но для rebind-based hooks это неточно — на самом деле всё equivalent).

Это **стандартный** паттерн использования.

## 8. Границы применимости

Нельзя litehook'нуть:

- Inlined функции (компилятор заинлайнил вызов).
- Static C-functions внутри dylib (не экспортированы, GOT-rebind не поможет).
- Прямые syscall через `svc` инструкцию (нет функции-прослойки).

Для таких случаев нужен более сложный inline hook (`EKHookFunction` в ellekit), но даже он не поможет для `svc` — там только patch инструкции.

## 9. Что подтверждено

- **Подтверждено**: litehook — submodule `.gitmodules` (`BaseBin/_external/modules/litehook/`).
- **Подтверждено**: функции `litehook_hook_function`, `litehook_rebind_symbol`, `litehook_hook_memory`, flag `LITEHOOK_REBIND_GLOBAL` — используются в коде проекта.
- **Подтверждено**: статическая линковка через wildcard в Makefile — `BaseBin/libjailbreak/Makefile`.
- **Подтверждено**: примеры использования — множество файлов в `BaseBin/launchdhook/src/`.
- **Требует проверки**: конкретное содержимое submodule (без clone — пусто); API может отличаться.
- **Не подтверждено** (без submodule): полный perечень экспортированных функций.

## 10. Выводы

Главный тезис: **litehook — внутренняя инфраструктура Dopamine**. Автор твика её не вызывает, но знание того, как Dopamine сам себе ставит хуки, объясняет, почему некоторые пути (NECP, csops, posix_spawn) уже перехвачены к моменту, когда ваш `%ctor` стартует.

Для автора твика — одно практическое следствие: не пытайтесь хукать `memorystatus_control`, `sysctlbyname`, `xpc_dictionary_get_value`, `xpc_receive_mach_msg`, `sandbox_check_by_audit_token` — они уже перехвачены Dopamine, и ваш хук внесёт конфликт.

## 11. Что дальше

- `MachOMerger` — Deep-dive 15.
- `ellekit` — рецепт прямого использования — Case 12 (планируется).
- Разбор `jetsam_hook` — глава III.4.
- Для картины «что хукается кем» — таблица III.12 (сводная).
