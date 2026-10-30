# Продвинуто 4. Пишем свой dylib-инжектор

Для **понимания**, как работают системы вроде `DYLD_INSERT_LIBRARIES` и `opainject`. Для production — используйте стандартные механизмы (TweakLoader, opainject). Эта глава — учебная.

## 1. Два пути инъекции

**Launch-time** (вместе со стартом процесса):

- Через `DYLD_INSERT_LIBRARIES` env variable.
- dyld загружает dylib до `main()`.
- Самый простой и стабильный.

**Runtime** (в живой процесс):

- Через `task_for_pid` + mach + создание треда.
- Это то, что делает `opainject` (Deep-dive 13).

Разберём оба.

## 2. Launch-time: DYLD_INSERT_LIBRARIES

Самый простой случай — вы запускаете процесс сами:

```bash
DYLD_INSERT_LIBRARIES=/path/to/your.dylib /path/to/binary
```

dyld при запуске читает env, видит эту переменную, грузит вашу dylib **до** любого другого кода процесса.

Ваш `__attribute__((constructor))` в этой dylib выполнится первым.

Пример минимальной "инжектируемой" dylib:

```c
// inject-demo.c
#include <stdio.h>
#include <mach-o/dyld.h>

__attribute__((constructor))
static void init(void) {
    const char *name = _dyld_get_image_name(_dyld_image_count() - 1);
    fprintf(stderr, "[inject-demo] loaded; last image = %s\n", name);
}
```

Собрать:

```bash
clang -arch arm64 -dynamiclib \
    -isysroot $(xcrun --sdk iphoneos --show-sdk-path) \
    -miphoneos-version-min=15.0 \
    -o inject-demo.dylib \
    inject-demo.c
ldid -Sadhoc inject-demo.dylib
```

Запуск на устройстве:

```bash
DYLD_INSERT_LIBRARIES=/tmp/inject-demo.dylib /usr/bin/some-cli-tool
```

Работает, если:

1. Процесс не platform binary со строгим codesign.
2. dylib подписан (ad-hoc) и в trust cache (на jailbreak — через `jbclient_trust_file_by_path`).
3. Target процесс не отбрасывает injection (hardened runtime, platform=1 с restricted entitlements).

Это **то же самое**, что делает `launchdhook` → `spawn_hook.c` для `systemhook` (глава III.4): добавляет `DYLD_INSERT_LIBRARIES=.../systemhook.dylib` в env всех процессов.

## 3. Runtime: инъекция в живой процесс

Концепт:

1. Получить task-port target-процесса (`task_for_pid`).
2. Аллокировать небольшой блок памяти внутри target (`mach_vm_allocate`).
3. Записать туда shellcode, который делает `dlopen` вашей dylib.
4. Создать поток в target с pc на нашем shellcode.
5. Поток выполняет `dlopen`, затем `thread_terminate`.
6. Вернуть результат.

Это **далеко** не легко. Детали:

### 3.1. Требования

- **Entitlements**: `task_for_pid-allow`, `com.apple.private.cs.debugger`. Без них `task_for_pid` возвращает `KERN_FAILURE`.
- **Trust cache**: ваш инжектор и target dylib должны быть доверены.
- **PAC**: на arm64e нужны правильные подписи указателей в thread_state.
- **Dyld compatibility**: нужно знать адрес `dlopen` в target через `_dyld_get_image_header_containing_address` или `nlist`.

Пример (концептуальный):

```c
// inject-into-pid.c
#include <stdio.h>
#include <mach/mach.h>
#include <mach/mach_vm.h>
#include <mach-o/dyld.h>
#include <dlfcn.h>

int inject(pid_t pid, const char *dylib_path) {
    task_t task;
    kern_return_t kr = task_for_pid(mach_task_self(), pid, &task);
    if (kr != KERN_SUCCESS) {
        fprintf(stderr, "task_for_pid failed: %d\n", kr);
        return -1;
    }

    // 1. Аллокировать память в target
    mach_vm_address_t remote_memory = 0;
    kr = mach_vm_allocate(task, &remote_memory, 0x1000, VM_FLAGS_ANYWHERE);
    if (kr != KERN_SUCCESS) { return -2; }

    // 2. Записать путь к dylib
    size_t path_len = strlen(dylib_path) + 1;
    kr = mach_vm_write(task, remote_memory, (vm_offset_t)dylib_path, (mach_msg_type_number_t)path_len);
    if (kr != KERN_SUCCESS) { return -3; }

    // 3. Shellcode: вызов dlopen (упрощённо)
    // В реальности — ARM64 инструкции:
    //   adr x0, path       — адрес пути
    //   mov w1, #0x101     — RTLD_NOW
    //   movz x16, lo16(dlopen_addr), lsl 0
    //   movk x16, hi16(dlopen_addr), lsl 16
    //   ...
    //   blr x16
    //   svc #0x80 (SYS_thread_terminate)
    // (детали — в opa334/opainject sources)

    // 4. Создать thread с PC на shellcode
    //    thread_create_running_from_state(task, ARM_THREAD_STATE64, &state, &thread)
    //    где state.__pc = remote_shellcode_address

    // Это упрощённый псевдокод. Полная реализация — ~300 строк.

    return 0;
}
```

Для production — используйте **opainject** (Deep-dive 13). Он сделает всё правильно, с PAC и error handling.

## 4. Hybrid-подход: spawn + env

Если хотите собственный «launchdhook-lite» для своего приложения:

```c
#include <spawn.h>

int main(int argc, char *argv[]) {
    // Запустить child процесс c нашей инжектированной dylib
    pid_t pid;
    const char *path = "/path/to/target";

    char *env[] = {
        "DYLD_INSERT_LIBRARIES=/path/to/your.dylib",
        NULL
    };

    posix_spawnattr_t attr;
    posix_spawnattr_init(&attr);
    posix_spawn(&pid, path, NULL, &attr, argv, env);
    posix_spawnattr_destroy(&attr);

    return 0;
}
```

Это **в userland**, без привилегий. Работает для процессов, которые вы запускаете сами (не Apple-signed с restricted entitlements).

## 5. Чем это отличается от TweakLoader

TweakLoader:

- Инжектируется в **каждый** инструментуемый процесс через `systemhook` (глава III.5).
- Загружает **все** `.dylib` из `/var/jb/Library/MobileSubstrate/DynamicLibraries/`, применяя filter plists.
- Централизованно.

Ваш собственный инжектор:

- Инжектирует **одну** dylib в **один** процесс.
- Вручную выбирается, что и куда.
- Для специализированных случаев.

Для 99% задач — TweakLoader подходит, свой инжектор не нужен.

## 6. Что подтверждено

- **Подтверждено Apple docs**: `DYLD_INSERT_LIBRARIES`, `task_for_pid`, `mach_vm_*`, `thread_create_running`, `posix_spawn`.
- **Подтверждено Dopamine исходниками**: `BaseBin/launchdhook/src/spawn_hook.c` — реальный пример launch-time injection.
- **Подтверждено Dopamine**: `BaseBin/opainject/` — runtime injector.
- **Требует проверки**: конкретные PAC detail на arm64e — зависят от версии iOS.

## 7. Выводы

Главный тезис: **инжекция dylib концептуально проста** — env variable для launch-time, mach manipulation для runtime. **Детали** — сложны, особенно PAC. Для production — opainject. Для понимания — пишите свой учебный.

## 8. См. также

- Deep-dive 13 — opainject.
- Глава III.4 — launchdhook с spawn_hook.
- Глава III.5 — systemhook (главный потребитель DYLD_INSERT_LIBRARIES).
