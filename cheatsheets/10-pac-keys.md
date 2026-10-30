# Шпаргалка 10. PAC-ключи и когда они нужны автору твика

Pointer Authentication Code (PAC) — ARMv8.3-A функция. Для автора твика на **A13** (arm64, без PAC в userspace) — почти всегда прозрачно. Для **A14+** (arm64e) — иногда нужно знать.

Эта шпаргалка — референс.

## Четыре ключа PAC

| Ключ | Для чего |
|---|---|
| **IA** | Code-pointers, variant A (return addresses при `blraa`, непрямые вызовы) |
| **IB** | Code-pointers, variant B (функции, vtable, …) |
| **DA** | Data-pointers, variant A (данные в heap) |
| **DB** | Data-pointers, variant B (мета-данные, часто для метаданных objc) |

Два variant'а (A/B) — это **разные ключи**: указатель, подписанный ключом IA, не валидируется ключом IB. Это даёт layered protection.

## Modifier (salt)

Помимо ключа, каждая подпись имеет **64-битный modifier** (salt). Один и тот же указатель с разными modifier даёт разную подпись.

Типичные modifier'ы:

- `0` — generic.
- Адрес хранения указателя (`(uintptr_t)&ptr`) — классический Apple pattern.
- Хеш типа — для type diversity.
- Произвольное число — для custom-signed pointers.

Пример: Apple подписывает return address с modifier = `(stack address of return slot) | (0x6DD5 << 48)` (гипотетический salt).

## ptrauth_* intrinsics в Clang

Для arm64e кода доступны:

```c
#include <ptrauth.h>

// Подписать указатель
void *signed_ptr = ptrauth_sign_unauthenticated(
    unsigned_ptr,
    ptrauth_key_function_pointer,    // IB для функций обычно
    modifier
);

// Снять подпись (аутентифицировать)
void *unsigned_ptr = ptrauth_auth_data(
    signed_ptr,
    ptrauth_key_process_independent_data,
    modifier
);

// Переподписать (resign) с новыми ключом/modifier
void *resigned = ptrauth_auth_and_resign(
    old_signed,
    ptrauth_key_process_independent_data, old_modifier,
    ptrauth_key_function_pointer, new_modifier
);

// Просто снять подпись (не аутентифицируя — unsafe но полезно для диагностики)
void *stripped = ptrauth_strip(ptr, ptrauth_key_asia);
```

Ключи:

- `ptrauth_key_function_pointer` → IB (на arm64e обычно).
- `ptrauth_key_process_independent_code` → IA.
- `ptrauth_key_process_independent_data` → DA.
- `ptrauth_key_asia` / `asib` / `asda` / `asdb` — для `_strip`.

## A13 (arm64, не arm64e) — что это значит

На A13 с iOS 26.0/26.0.1 Dopamine работает в arm64 slice. В userspace **PAC не применяется** к указателям. Это значит:

1. **Ваш твик** компилируется с `ARCHS = arm64 arm64e`, но реально загружается arm64 slice. PAC-инструкции в этом slice — NOP.
2. **Хуки** — обычный swizzle / fishhook работает без ptrauth_* intrinsics.
3. **Указатели функций** в вашем коде — просто адреса, без подписи.

Для A13 **90% автору твика про PAC не нужно знать**.

## Когда PAC всё-таки касается A13

Один случай: когда Dopamine сам в своих компонентах читает/пишет указатели dyld, подписанные Apple для arm64e. Пример из `BaseBin/systemhook/src/main.c:66–84` (функция `dyld_hook_routine`):

```c
int dyld_hook_routine(void **dyld, int idx, void *hook, void **orig, uint16_t pacSalt)
{
    if (!dyld) return -1;

    uint64_t dyldPacDiversifier = ((uint64_t)dyld & ~(0xFFFFull << 48)) | (0x63FAull << 48);
    void **dyldFuncPtrs = ptrauth_auth_data(*dyld,
        ptrauth_key_process_independent_data,
        dyldPacDiversifier);
    if (!dyldFuncPtrs) return -1;

    if (vm_protect(mach_task_self_, (mach_vm_address_t)&dyldFuncPtrs[idx],
                   sizeof(void *), false,
                   VM_PROT_READ | VM_PROT_WRITE) == 0) {
        uint64_t location = (uint64_t)&dyldFuncPtrs[idx];
        uint64_t pacDiversifier = (location & ~(0xFFFFull << 48)) |
                                   ((uint64_t)pacSalt << 48);

        *orig = ptrauth_auth_and_resign(dyldFuncPtrs[idx],
            ptrauth_key_process_independent_code, pacDiversifier,
            ptrauth_key_function_pointer, 0);
        dyldFuncPtrs[idx] = ptrauth_auth_and_resign(hook,
            ptrauth_key_function_pointer, 0,
            ptrauth_key_process_independent_code, pacDiversifier);
        vm_protect(mach_task_self_, (mach_vm_address_t)&dyldFuncPtrs[idx],
                   sizeof(void *), false, VM_PROT_READ);
        return 0;
    }

    return -1;
}
```

Разбор:

- Dopamine ставит хук на dyld-функцию (например, `dlsym`).
- Указатель на функцию в dyld-table подписан process-independent code key с особым diversifier.
- Для подмены: прочитать (аутентифицировать), переподписать на свой hook с тем же diversifier.
- `pacSalt` — специальный salt, который использовал Apple для подписи конкретной функции.

Это **внутренний** код Dopamine. Автор твика его не повторяет; но полезно понимать, почему вокруг хуков dyld встречается этот ptrauth-паттерн.

## A14+ — что меняется (справочно)

На arm64e (A14+) userspace указатели **подписаны**:

- `IMP` ObjC — подписан.
- Return address на stack — подписан.
- Function pointers в `__DATA` — подписаны.
- vtables C++ — подписаны.

Это означает, что твик на arm64e:

- При сборке с `arm64e` slice компилятор автоматически вставляет `ptrauth_*` там, где нужно.
- При подмене `IMP` вручную (через `method_setImplementation`) — работает прозрачно (API сам обрабатывает PAC).
- При работе с C function pointers — обычно тоже прозрачно (компилятор делает всё за вас).
- При ручной записи в executable память (патч через vm_write) — нужны ptrauth для нового значения.

Но **для нашей целевой среды (A13 + iOS 26)** всё это не актуально — iOS 26 на arm64e (A14+) Dopamine не декларирует.

## Как узнать, в каком slice запущен ваш твик

```c
#include <sys/utsname.h>
#include <stdio.h>

#if __arm64e__
    #warning "Compiled for arm64e"
#else
    #warning "Compiled for arm64 (no PAC in userspace)"
#endif

// Runtime (на арматуре A13 — всегда arm64):
#if __arm64e__
    printf("Running arm64e slice\n");
#else
    printf("Running arm64 slice\n");
#endif
```

Для **одной** fat dylib, запущенной на разных чипах:

- На A13 → загружается arm64 slice → `__arm64e__` не определён.
- На A14+ → загружается arm64e slice → `__arm64e__` определён.

Это даёт возможность писать условный код.

## Таблица «нужно ли думать о PAC»

| Задача | A13 (arm64) | A14+ (arm64e) |
|---|---|---|
| `%hook` ObjC метод | нет | нет (прозрачно) |
| `%hookf` C-функция | нет | нет (ellekit делает) |
| Подмена `IMP` через `method_setImplementation` | нет | нет (API handles) |
| Прямой `vm_write` executable code | нет | нужны ptrauth для указателей |
| Чтение function pointers из `__DATA` | нет | нужны ptrauth для auth |
| ARM assembler с `BR`/`BLR` | нет (обычные) | нужны `BRAA`/`BRAB` варианты |
| ObjC runtime API | нет | нет |

## Инструменты для анализа PAC

```bash
otool -h MyTweak.dylib | grep -i subtype
# CPU_SUBTYPE_ARM64E = PAC code
# CPU_SUBTYPE_ARM64 = не PAC

otool -tV MyTweak.dylib -arch arm64e | grep 'paciasp\|autiasp'
# paciasp — подписать x30 (lr) при входе в функцию
# autiasp — аутентифицировать x30 на выходе
```

## См. также

- Deep-dive 03 — PAC в деталях.
- Deep-dive 04 — PPL и A12/A13.
- Глава I.4 — arm64 vs arm64e.
- Пример в коде: `BaseBin/systemhook/src/main.c:66–84` (`dyld_hook_routine`).

Этой шпаргалкой закрывается подсекция `cheatsheets/`. Все 10 шпаргалок написаны.
