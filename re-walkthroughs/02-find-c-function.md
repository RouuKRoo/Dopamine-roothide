# RE 2. Как найти неэкспортированную C-функцию в фреймворке

Пошаговый workflow: находим внутреннюю (не публичную) функцию, которую нужно хукнуть.

## 1. Задача

В CoreFoundation есть внутренняя функция `__CFPreferencesCacheFlush` (гипотетическое имя). Нужно её хукнуть.

Публичного заголовка — нет. Как достать адрес и повесить хук.

## 2. Шаг 1: nm — список символов

```bash
# Извлечь framework (как в RE 1):
ipsw dyld extract dyld_shared_cache_arm64 \
    --image "/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation" \
    -o /tmp/cf/

# Все external symbols:
nm -gU /tmp/cf/CoreFoundation | head -30

# Поиск по patterns:
nm -gU /tmp/cf/CoreFoundation | grep -i "prefs\|cache"
# ...
# 0x1234567 T __CFPreferencesCacheFlush
# ...
```

`T` — text (code). `U` — undefined (внешняя). Мы хотим `T`.

## 3. Шаг 2: nm на все (включая локальные)

Если `nm -gU` не показал — функция **не экспортирована** external:

```bash
nm -a /tmp/cf/CoreFoundation | grep "preferencescacheflush"
# Может показать local символы — те, которые доступны только внутри .dylib
```

## 4. Шаг 3: strings для подсказок

Если имя функции не очевидно:

```bash
strings /tmp/cf/CoreFoundation | grep -i "prefs" | head
```

Иногда — имена функций попадают в error-logging strings:

```
__CFPreferencesCacheFlush: locked prefs
```

Это подсказка на существующее имя.

## 5. Шаг 4: Hopper / Ghidra / IDA

Полноценный disassembler дает:

- Function names.
- Cross-references.
- Pseudo-code decompilation.

Для hopper:

```bash
hopper /tmp/cf/CoreFoundation
# В интерактивном UI:
# - Procedures tab показывает все функции.
# - Поиск по имени / substring.
# - Для найденной функции — Decompile для pseudo-C.
```

Это даёт понимание, что функция делает и как её использовать.

## 6. Шаг 5: Dynamic probe на устройстве через dlsym

```c
#include <dlfcn.h>

%ctor {
    // RTLD_DEFAULT ищет во всех загруженных:
    void *fn = dlsym(RTLD_DEFAULT, "__CFPreferencesCacheFlush");
    os_log(gLog, "function at: %p", fn);
}
```

Если `dlsym` возвращает NULL — функция **не экспортирована** (symbol stripped).

## 7. Шаг 6: Если dlsym возвращает NULL — через MSFindSymbol

Classic substrate utility (ellekit should provide equivalent):

```objc
#import <substrate.h>

%ctor {
    MSImageRef image = MSGetImageByName("/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation");
    if (image) {
        void *fn = MSFindSymbol(image, "___CFPreferencesCacheFlush");
        os_log(gLog, "found at: %p", fn);
    }
}
```

`MSFindSymbol` работает с символами, не экспортированными publicly, если они в symbol table (не strip'ed).

## 8. Шаг 7: Hook через fishhook / ellekit

Получив адрес:

```c
static void (*orig_fn)(void);

static void my_fn(void) {
    os_log(gLog, "__CFPreferencesCacheFlush called");
    orig_fn();
}

%ctor {
    MSImageRef image = MSGetImageByName("/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation");
    void *fn_addr = MSFindSymbol(image, "___CFPreferencesCacheFlush");

    if (fn_addr) {
        MSHookFunction(fn_addr, (void *)my_fn, (void **)&orig_fn);
        // Или через ellekit:
        // EKHookFunction(fn_addr, (void *)my_fn, (void **)&orig_fn);
    }
}
```

Готово — hook установлен на внутреннюю функцию.

## 9. Нюансы

### 9.1. Static functions

Функции с `static` в C — компилируются **без** symbol entry. Их адрес нельзя получить через nm.

Единственный путь — pattern-matching в disassembly (найти pattern bytes, scan memory).

Это **сложно**. Для автора твика в 99% не нужно.

### 9.2. Inlined functions

Компилятор может inline функцию во все callers. Тогда функция «не существует» в бинарнике — только copies в callers.

Nothing to hook. Нужно хукать callers.

### 9.3. Symbol stripping

Apple strip'ит некоторые символы в release. Если нашли в SDK header, но `nm` в device-binary не показывает — strip'нуто. Используйте pattern-based подход (Hopper/Ghidra signatures).

## 10. Что подтверждено

- **Подтверждено Apple docs**: `nm`, `dlsym`.
- **Подтверждено substrate docs**: `MSFindSymbol`, `MSHookFunction`.
- **Требует проверки**: что ваша target функция экспортирована — через `nm -gU` на framework.

## 11. См. также

- Глава IV.5 — перехват C-функций.
- RE 1 — поиск селектора.
- Глава VI.1 — алгоритм исследования.
- Атлас CoreFoundation.
