# Продвинуто 9. Статический анализ своего твика

Статический анализ — проверка кода без его выполнения. Для твиков это особенно полезно: баги в твиках сложнее воспроизвести, чем в обычных приложениях (нужен jailbreak, нужный процесс, нужное состояние).

## 1. Инструменты

| Инструмент | Что проверяет | Для чего в твике |
|---|---|---|
| **Clang Static Analyzer** (`scan-build`) | nil deref, memory leaks, undefined behavior | Главный инструмент |
| **clang-tidy** | code style, modernization, bug patterns | Качество кода |
| **oclint** | ObjC-specific issues | ObjC code |
| **Infer** (Facebook) | NIL deref, race conditions | Серьёзные bugs |
| **llvm-cov** | code coverage | Для tests |
| **Sanitizers** (ASan, TSan, UBSan) | runtime detection | На XCTest, не на tweak |

## 2. Clang Static Analyzer (`scan-build`)

Самый мощный инструмент. Входит в Xcode command-line tools.

### 2.1. Для Theos tweak

```bash
cd MyTweak
scan-build --use-cc=$(xcrun -f clang) make clean package THEOS_PACKAGE_SCHEME=roothide

# scan-build сохранит отчёт в /tmp/scan-build-<id>/
# Открыть HTML отчёт:
scan-view /tmp/scan-build-<id>/
```

### 2.2. Что ищет

- **Nil pointer dereference**: `[nil-object method]` возвращает nil/0, но `*nilPtr = x` — краш.
- **Memory leaks**: `malloc` без `free`.
- **Dead code**: условия, которые всегда true/false.
- **Uninitialized variables**: использование до assignment.
- **API misuse**: неправильные параметры к Apple API.

### 2.3. Пример warning

```
Tweak.x:42:5: warning: Null pointer argument in call to string length function
    [NSString stringWithUTF8String:path];
    ^
```

На строке 42, если `path` может быть NULL — scan-build предупреждает.

## 3. clang-tidy

```bash
# Проверить один файл:
clang-tidy -checks='*' Tweak.x -- \
    -isysroot $(xcrun --sdk iphoneos --show-sdk-path) \
    -I$THEOS/include \
    -fobjc-arc
```

### 3.1. Checks groups

- `bugprone-*` — bug patterns (e.g., `bugprone-use-after-move`).
- `modernize-*` — update to modern C++/ObjC idioms.
- `performance-*` — unnecessary copies, etc.
- `readability-*` — code style.

Для твика фокус на `bugprone-*` и `clang-analyzer-*`.

### 3.2. Пример

```bash
clang-tidy -checks='bugprone-*,clang-analyzer-*' Tweak.x -- ...
```

Выдаст список issues с file:line.

## 4. Интеграция в Makefile

```make
# Статический анализ как отдельная цель:
analyze:
	scan-build --use-cc=$(xcrun -f clang) \
		--status-bugs \
		make -B package \
		THEOS_PACKAGE_SCHEME=roothide \
		DEBUG=1

tidy:
	clang-tidy -checks='bugprone-*,clang-analyzer-*,-clang-analyzer-security.insecureAPI.strcpy' \
		Tweak.x \
		-- \
		-isysroot $$(xcrun --sdk iphoneos --show-sdk-path) \
		-I$$THEOS/include \
		-fobjc-arc
```

Запуск:

```bash
make analyze
make tidy
```

`--status-bugs` в scan-build — exit status 1 если найдены bugs. Полезно для CI.

## 5. Сanitizers (только для XCTest, не для твика)

Твик не может использовать ASan/TSan — требуют Apple runtime, не работают с ellekit/mobilesubstrate. Но ваш **unit test** (Advanced 2) может:

В Tests.xcodeproj:

```
Scheme → Diagnostics → check:
  [x] Address Sanitizer
  [x] Thread Sanitizer
  [x] Undefined Behavior Sanitizer
```

Запуск тестов — ASan/TSan/UBSan ловят memory issues в вашей **logic** library. Твик тестируется косвенно через эту library.

## 6. CI integration

```yaml
# .github/workflows/analyze.yml
name: Static Analysis

on: [push, pull_request]

jobs:
  analyze:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Theos
        run: |
          git clone --recursive https://github.com/roothide/Theos.git ~/theos
          echo "THEOS=$HOME/theos" >> $GITHUB_ENV
          brew install ldid dpkg

      - name: Run scan-build
        run: |
          scan-build --use-cc=$(xcrun -f clang) \
            --status-bugs \
            --html-title "MyTweak Analysis" \
            -o /tmp/scan-build \
            make -B package \
            THEOS_PACKAGE_SCHEME=roothide \
            DEBUG=1
        continue-on-error: true

      - name: Upload scan-build report
        uses: actions/upload-artifact@v3
        with:
          name: scan-build-report
          path: /tmp/scan-build
          retention-days: 30
```

Отчёт сохраняется как artifact — download и смотрите HTML.

## 7. Типовые warnings и как их фиксить

### 7.1. "Potential null dereference"

```c
NSString *result = someFunction();
[result length];  // warning: result may be nil
```

Фикс:

```c
NSString *result = someFunction();
if (result) {
    [result length];
}
// или:
NSUInteger len = result.length;  // nil-safe в ObjC
```

### 7.2. "Memory leak"

```c
CFStringRef s = CFStringCreateWithCString(NULL, "x", kCFStringEncodingUTF8);
// забыли CFRelease(s)
```

Фикс:

```c
CFStringRef s = CFStringCreateWithCString(NULL, "x", kCFStringEncodingUTF8);
// ... use s ...
CFRelease(s);
```

### 7.3. "Dead code"

```c
int r = someFunction();
if (r != 0) return;
if (r == 0) {         // dead: уже вернулись выше
    doStuff();
}
```

Фикс:

```c
int r = someFunction();
if (r != 0) return;
doStuff();
```

### 7.4. "Uninitialized variable"

```c
int result;
if (condition) {
    result = 10;
}
return result;  // если condition == NO — undefined
```

Фикс:

```c
int result = 0;
if (condition) {
    result = 10;
}
return result;
```

## 8. Отключение false positives

Если warning — заведомо false positive:

```c
// clang-format off
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wincompatible-pointer-types"
    some_code_here;
#pragma clang diagnostic pop
// clang-format on
```

Для clang-tidy:

```c
// NOLINT(bugprone-check-name)
int x = should_be_ok();
```

Но **всегда** добавляйте комментарий «почему это OK».

## 9. Chain: linter → static analysis → unit tests

Полный pipeline качества:

1. **`clang-format`** — автоматическое форматирование (не catch bugs, но consistency).
2. **`clang-tidy`** — style + базовые bug patterns.
3. **`scan-build`** — deep static analysis.
4. **XCTest** — unit tests.
5. **Integration test on device** — финал.

Если 1-4 пройдены — вероятность бага в прод уменьшается в разы.

## 10. Таблица «что ловит каждый инструмент»

| Проблема | clang-format | clang-tidy | scan-build | XCTest + ASan |
|---|---|---|---|---|
| Форматирование | ✓ | — | — | — |
| Style issues | — | ✓ | — | — |
| Nil dereference (статически) | — | ✓ | ✓ | — |
| Memory leak (статически) | — | partially | ✓ | — |
| Memory leak (runtime) | — | — | — | ✓ |
| Use-after-free | — | partially | ✓ | ✓ |
| Race condition | — | — | — | ✓ (TSan) |
| Logic bugs | — | — | — | ✓ (tests) |

## 11. Что подтверждено

- **Подтверждено Apple docs**: `scan-build`, `clang-tidy` входят в Xcode CLI tools.
- **Подтверждено практикой**: `scan-build` работает с Makefile-based projects (включая Theos).
- **Подтверждено**: ASan/TSan не работают в твиках (конфликт с ellekit/dyld runtime hooks).
- **Требует проверки**: конкретная версия clang-tidy — API может отличаться.

## 12. Выводы

Главный тезис: **статический анализ дёшев** — минуты в CI, часы отладки сэкономлены. Для твика особенно, т. к. runtime baggage трудно воспроизвести.

Минимум: `scan-build` + `clang-tidy` в CI. Максимум: + ASan/TSan для unit tests.

## 13. См. также

- Advanced 2 — testing.
- Advanced 3 — CI/CD.
- Advanced 10 — релиз (где эти проверки — ворота в репозиторий).
