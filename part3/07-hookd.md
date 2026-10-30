# Глава III.7. BaseBin/hookd — внешние хуки через mach

`hookd` — отдельный привилегированный демон, который позволяет **писать в чужую память** через mach-порты. Это тот самый API, через который Frida и аналогичные инструменты применяют хуки на процессы, в которые они сами не могут инжектироваться. В этой главе — его архитектура и API.

## 1. Назначение

Главная проблема: твик, живущий в процессе A, не может напрямую писать в код процесса B — у него нет `task_for_pid` на процесс B (iOS его не даёт обычным процессам). Нужен **посредник с правами**.

Этот посредник — `hookd`:

- Отдельный маленький процесс, запускаемый launchd из `BaseBin/hookd/`.
- Имеет entitlements: `task_for_pid-allow`, `platform-application`, `com.apple.private.cs.debugger`.
- Слушает mach-сообщения; клиенты (включая Frida) отправляют ему запросы «применить хук в процессе X по адресу Y с данными Z».
- Применяет через `vm_protect` + `vm_write`.

Это чистая и локальная реализация «хука в чужой процесс» без необходимости клиенту иметь ядерные примитивы.

## 2. Entitlements демона

Из `BaseBin/hookd/entitlements.plist`:

```xml
<key>platform-application</key><true/>
<key>com.apple.private.security.container-required</key><false/>
<key>com.apple.private.cs.debugger</key><true/>
<key>task_for_pid-allow</key><true/>
```

Разбор:

- `platform-application` — делает `hookd` аналогом системного приложения с расширенными правами.
- `container-required=false` — не нужен sandbox-контейнер.
- `cs.debugger` — право «отладочного процесса» (может читать чужую память).
- `task_for_pid-allow` — право получать task-port для `task_for_pid()`.

Эти entitlements подписаны и попадают в trust cache через `BaseBin/libjailbreak/src/codesign.m` + `trustcache*`.

## 3. API со стороны клиента

Из `BaseBin/libjailbreak/src/hookd.h`:

```c
struct hookd_encoded_hook {
    vm_address_t address;
    vm_size_t dataSize;
    uint8_t data[];
};

struct hookd_encoded_fixup {
    vm_address_t address;
    vm_size_t size;
    bool set_maximum;
    vm_prot_t prot;
};

struct hookd_hook {
    uint64_t address;
    uint8_t *data;
    size_t dataSize;
};

int hookd_send_requests(task_port_t taskPort,
                        struct hookd_hook *hooks, int hooksCount, int *hookResultsOut,
                        struct hookd_encoded_fixup *fixups, int fixupsCount, int *fixupResultsOut);
kern_return_t hookd_vm_protect(mach_port_t taskPort, vm_address_t address,
                               vm_size_t size, bool set_maximum, vm_prot_t new_protection);
kern_return_t hookd_hook(mach_port_t taskPort, uint64_t address,
                         uint8_t *data, size_t dataSize);
```

Три уровня:

- **`hookd_hook`** — простая обёртка «запиши в адрес A в процессе T данные D». Один вызов — один хук.
- **`hookd_vm_protect`** — только `vm_protect` над адресом в чужом процессе.
- **`hookd_send_requests`** — пакетная версия: несколько hooks + несколько fixups в одном сообщении. Эффективнее, когда нужно установить сразу много инструкций.

## 4. Протокол сообщения

```c
struct hookd_mach_msg {
    mach_msg_header_t hdr;
    pid_t clientPid;
    task_port_t taskPortInClient;
    uint64_t hooksStartOff;
    uint64_t fixupsStartOff;
    uint8_t data[];
};
```

Клиент собирает:

1. Заголовок mach-сообщения.
2. PID процесса, в который применить хуки.
3. task-port-in-client — опционально, если клиент сам хочет указать task-port (`-1` для default).
4. Два offset'а в `data[]`: где лежат hooks и где fixups.
5. Полезная нагрузка: сериализованные `hookd_encoded_hook` и `hookd_encoded_fixup`.

Максимальный размер — `HOOKD_MSG_MAX_SIZE = 4096` байт. Это лимит нагрузки на одно mach-сообщение; если нужно больше — несколько запросов.

## 5. Серверная сторона — что делает hookd

Главный цикл в `BaseBin/hookd/src/main.c:85–180` (функция `server_loop`):

1. Ждать mach-сообщение.
2. Проверить audit trailer: кто вызвал.
3. Проверить, что `callerPid == 1` (клиент — launchd/launchdhook; это защищает от прямых обращений от неавторизованных процессов).
4. `task_for_pid(mach_task_self(), msg->clientPid, &clientTaskPort)` — получить task-port клиентского процесса.
5. Из клиентского task-port извлечь (через `mach_port_extract_right`) task-port целевого процесса — тот, в который применяется хук.
6. Для каждого encoded hook:
   - `vm_protect(taskPort, vmaddr, size, false, VM_PROT_READ | VM_PROT_WRITE | VM_PROT_COPY)` — временно сделать страницы RW.
   - `vm_write(taskPort, vmaddr, data, size)` — записать данные.
   - `vm_protect(taskPort, vmaddr, size, false, VM_PROT_READ | VM_PROT_EXECUTE)` — вернуть RX.
7. Для каждого fixup:
   - `vm_protect(taskPort, address, size, set_maximum, prot)`.
8. Собрать результаты и отправить ответ.

Это **ровно** то, что любой пользовательский код сделал бы, если бы имел task_for_pid-права — но вместо того, чтобы раздавать эти права всему userspace, Dopamine делает один маленький демон-посредник.

Красивая деталь — коммит e3f3cb5 «hookd: Preserve write and protection errors» (упомянуто в истории проекта). Смысл: раньше `apply_hook` маскировал ошибки `vm_write`/`vm_protect`, теперь — честно возвращает их. Для автора твика это полезно: когда вызов `hookd_hook` возвращает ненулевое значение, можно диагностировать, что именно пошло не так.

## 6. Проверка `callerPid != 1`

Код в `main.c:140–143`:

```c
#if !HOOKD_SKIP_CALLER_VERIFICATION
    if (callerPid != 1) goto deny;
#endif
```

**Единственный легитимный клиент — launchd** (PID 1). Это критическая деталь безопасности:

- Прямое обращение к `hookd` от обычного процесса — отбрасывается.
- Клиент должен проходить через launchd (а конкретно — через launchdhook, который держит порт `hookd`).
- Это означает, что `hookd_send_msg` в libjailbreak на деле вызывается через `jbclient_mach_hookd_send_msg` (см. `BaseBin/libjailbreak/src/hookd.c:9`) — она сначала идёт в launchd, launchd пересылает в `hookd`.

Для автора твика это даёт ясное понимание: нельзя обойти цепочку, нельзя подключиться к `hookd` напрямую. Все вызовы идут через `libjailbreak`.

## 7. Как hookd используется в системе

Главный клиент — **Frida для jailbroken-устройств** (через `frida-server` или подобные инструменты). Коммит в истории проекта упоминает «Expose a memory hook API for Frida» — это ровно про этот путь.

Другие сценарии для автора твика (относительно редкие):

1. Инструментировать процесс, который по каким-то причинам не получает `systemhook` (находится в blacklist, или `TweakLoader` не грузится). Тогда — твой helper-процесс через `hookd_hook` может применить inline-хук в target'е.
2. Применить хук **не из `%ctor`**, а намного позже, по условию — пример: fingerprint приложения, при обнаружении — применить patch.

## 8. Пример использования из C-кода

Полный пример применения хука (концептуальный, собирается с `-ljailbreak`):

```c
#include <libjailbreak/hookd.h>
#include <mach/mach.h>

int apply_patch_in_remote_process(pid_t targetPid, uint64_t address,
                                  uint8_t *patch_bytes, size_t patch_size)
{
    task_port_t taskPort = MACH_PORT_NULL;
    kern_return_t kr = task_for_pid(mach_task_self(), targetPid, &taskPort);
    if (kr != KERN_SUCCESS) return -1;

    kr = hookd_hook(taskPort, address, patch_bytes, patch_size);

    mach_port_deallocate(mach_task_self(), taskPort);
    return (kr == KERN_SUCCESS) ? 0 : -2;
}
```

Важные детали:

- `task_for_pid` в вашем клиентском процессе должен **существовать** — это значит, что ваш клиент должен быть либо platformised, либо иметь соответствующий entitlement. Для обычного `.dylib`-твика это обычно не так.
- Поэтому реальный путь: твой твик через `jbclient_mach_hookd_send_msg` (в libjailbreak) обращается к `hookd` через mach-port, и там уже сервер делает `task_for_pid` сам (он на это entitled).

## 9. Таблица «что hookd даёт автору твика»

| Задача | Как |
|---|---|
| Применить inline-патч в процессе, куда твик не загружается | `hookd_hook(taskPort, addr, bytes, size)` |
| Поменять защиту страницы в чужом процессе | `hookd_vm_protect(...)` |
| Пакетно применить множество патчей | `hookd_send_requests(...)` |
| Применить хук не из `%ctor`, а по событию | Тот же `hookd_hook` в любой момент |

Для **большинства** твиков этот API не нужен — Logos `%hook` + ellekit в своём процессе покрывают всё. `hookd` нужен в редких инструментальных случаях.

## 10. Что подтверждено

- **Подтверждено исходниками**: entitlements демона `hookd` — `BaseBin/hookd/entitlements.plist`; `server_loop`, `apply_hook`, `apply_fixup` — `BaseBin/hookd/src/main.c`; API client-side — `BaseBin/libjailbreak/src/hookd.h` и `hookd.c`; проверка `callerPid != 1` — `main.c:140–143`.
- **Подтверждено историей**: коммит «hookd: Preserve write and protection errors» (хэш e3f3cb5 в git log) — улучшение обработки ошибок.
- **Требует проверки на устройстве**: что ваш twik имеет возможность вызвать `jbclient_mach_hookd_send_msg` (это нужна линковка с `libjailbreak` и запуск в процессе, которому доступно XPC к jbserver).
- **Не подтверждено**: все ли клиенты по умолчанию прошедшей проверки — зависит от версии Dopamine и настройки `HOOKD_SKIP_CALLER_VERIFICATION`.

## 11. Выводы

Главный тезис: `hookd` — это **минимально-привилегированный мост для записи в чужую память**. Он решает проблему, которую обычному userspace решить невозможно (нет `task_for_pid`-прав), но делает это аккуратно: один вход через launchd, строгая проверка клиента, фиксированный размер сообщения, честная обработка ошибок.

Для автора твика в большинстве случаев этого демона не касаться. Но если задача выходит за рамки «hook в моём собственном процессе» (например, инструментировать системный процесс, в который `systemhook` не инжектится) — `hookd_hook` даёт правильный путь.

## 12. Что дальше

- `jailbreakd` — глава III.8 (демон-координатор, у которого ещё более широкие полномочия).
- Для деталей архитектуры эксплойтов и как hookd туда вписывается — Deep-dive о trustcache.
- Для кейса «инструментировать процесс, куда не загружается systemhook» — Case в разделе `cases/` (планируется).
