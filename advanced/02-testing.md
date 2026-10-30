# Продвинуто 2. Тестирование твика (unit-like)

Твик тестировать трудно: он выполняется в чужом процессе, зависит от iOS runtime, инициализируется через TweakLoader. Эта глава — паттерны, которые делают часть кода твика unit-testable на хосте без устройства.

## 1. Главный приём: отделить логику от хуков

Структура:

```
MyTweak/
├── Tweak.x           — только hooks, тонкий слой; вся логика — отсюда делегирована
├── Core/
│   ├── Logic.h       — pure header
│   ├── Logic.m       — pure logic, никаких UIKit/system зависимостей
│   └── LogicTests/
│       └── Tests.m   — unit тесты через XCTest
├── Makefile          — для твика
└── Tests.xcodeproj   — для XCTest (запускается на macOS/симулятор)
```

Пример.

### 1.1. Core/Logic.h

```objc
#import <Foundation/Foundation.h>

@interface MyTweakLogic : NSObject

// Чистая функция: принимает входные данные, возвращает решение.
+ (BOOL)shouldInterceptURL:(NSURL *)url
                userMode:(NSInteger)mode;

// Форматирует log-строку из contextual данных
+ (NSString *)formatLogWithPath:(NSString *)path
                        exists:(BOOL)exists
                      duration:(NSTimeInterval)ms;

@end
```

### 1.2. Core/Logic.m

```objc
#import "Logic.h"

@implementation MyTweakLogic

+ (BOOL)shouldInterceptURL:(NSURL *)url userMode:(NSInteger)mode {
    if (!url) return NO;
    if (mode == 0) return NO;  // off
    if ([url.scheme isEqualToString:@"blocked"]) return YES;
    if (mode == 2) return YES; // aggressive — всё перехватываем
    return NO;
}

+ (NSString *)formatLogWithPath:(NSString *)path exists:(BOOL)exists duration:(NSTimeInterval)ms {
    return [NSString stringWithFormat:@"[%.1fms] %@ %@",
            ms, exists ? @"OK" : @"MISS", path];
}

@end
```

### 1.3. Core/LogicTests/Tests.m — для macOS/simulator

```objc
#import <XCTest/XCTest.h>
#import "Logic.h"

@interface MyTweakLogicTests : XCTestCase
@end

@implementation MyTweakLogicTests

- (void)testURL_shouldInterceptBlockedScheme {
    NSURL *url = [NSURL URLWithString:@"blocked://foo"];
    XCTAssertTrue([MyTweakLogic shouldInterceptURL:url userMode:1]);
}

- (void)testURL_offModeNeverIntercepts {
    NSURL *url = [NSURL URLWithString:@"blocked://foo"];
    XCTAssertFalse([MyTweakLogic shouldInterceptURL:url userMode:0]);
}

- (void)testURL_aggressiveInterceptsEverything {
    NSURL *url = [NSURL URLWithString:@"https://example.com"];
    XCTAssertTrue([MyTweakLogic shouldInterceptURL:url userMode:2]);
}

- (void)testURL_nilReturnsFalse {
    XCTAssertFalse([MyTweakLogic shouldInterceptURL:nil userMode:1]);
}

- (void)testFormat {
    NSString *s = [MyTweakLogic formatLogWithPath:@"/tmp/x"
                                              exists:YES
                                            duration:12.5];
    XCTAssertEqualObjects(s, @"[12.5ms] OK /tmp/x");
}

@end
```

### 1.4. Tweak.x — тонкая обёртка

```objc
#import "Logic.h"

%hook UIApplication

- (void)openURL:(NSURL *)url options:(NSDictionary *)options
         completionHandler:(void (^)(BOOL))completion
{
    NSInteger mode = /* из preferences */ 1;
    if ([MyTweakLogic shouldInterceptURL:url userMode:mode]) {
        if (completion) completion(NO);
        return;
    }
    %orig(url, options, completion);
}

%end
```

Теперь `MyTweakLogic` — pure и тестируется через XCTest. Хук — тонкий; его логика проверена косвенно через тесты.

## 2. Запуск XCTest на хосте

На macOS:

```bash
cd Tests.xcodeproj
xcodebuild test -scheme MyTweakLogicTests -destination 'platform=macOS'
```

Или через simulator:

```bash
xcodebuild test -scheme MyTweakLogicTests \
    -destination 'platform=iOS Simulator,name=iPhone 15'
```

Результаты — pass/fail для каждого теста.

## 3. Что НЕЛЬЗЯ тестировать таким способом

- **Хуки** сами (замена IMP) — требует запуска в target-процессе.
- **UI**-эффекты — требуют iOS runtime.
- **Thread safety** — трудно воспроизвести race conditions в unit тестах; нужна интеграционная проверка.
- **Производительность** — только на реальной железке.

Для них — **integration tests**: собрать .deb, установить на реальное устройство, запустить сценарий, собрать логи, сравнить.

## 4. CI-friendly workflow

```yaml
# .github/workflows/test.yml
name: tests
on: [push]
jobs:
  unit:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run XCTest
        run: |
          xcodebuild test \
            -project Tests.xcodeproj \
            -scheme MyTweakLogicTests \
            -destination 'platform=macOS' \
            CODE_SIGNING_ALLOWED=NO
```

Запускается на каждый push. Если тесты падают — pipeline зелёный не станет.

## 5. Dependency injection для изолируемого тестирования

Если логика ссылается на что-то (например, читает `CFPreferences`), сделайте DI:

```objc
@interface MyTweakLogic : NSObject
+ (instancetype)sharedInstance;
@property (strong) id<MyPrefsProvider> prefs;
@end

@protocol MyPrefsProvider <NSObject>
- (BOOL)boolForKey:(NSString *)key;
@end

// В production — CFPreferences-based:
@interface CFPrefsProvider : NSObject <MyPrefsProvider>
@end

// В тестах — mock:
@interface MockPrefsProvider : NSObject <MyPrefsProvider>
@property (strong) NSDictionary *values;
@end
```

В тесте:

```objc
- (void)testLogic_disabledModeDoesntIntercept {
    MockPrefsProvider *mock = [MockPrefsProvider new];
    mock.values = @{@"enabled": @NO};
    MyTweakLogic.sharedInstance.prefs = mock;

    XCTAssertFalse([MyTweakLogic.sharedInstance shouldDoThing]);
}
```

Это **чистая** классическая DI; работает отлично в твиках.

## 6. Поведенческое тестирование через лог

Для хуков напрямую — соберите `.deb`, установите на устройство, запустите сценарий, зафиксируйте лог, сравните с expected.

```bash
# install-and-run.sh
#!/usr/bin/env bash
set -e
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@device:/tmp/
ssh mobile@device "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

# Запустить сценарий (например, UI автоматизация через xcode-build simctl, не для jailbroken device)
# или вручную

# Собрать лог за последние 30 секунд
ssh mobile@device "log collect --last 30s --output /tmp/log.logarchive"
scp mobile@device:/tmp/log.logarchive .

# Распаковать и grep
log show --predicate 'subsystem == "com.example.mytweak"' --archive log.logarchive > actual.log

diff expected.log actual.log || { echo "FAIL"; exit 1; }
echo "PASS"
```

Это **integration test**. Запускается не на каждый commit (slow), а на pre-release.

## 7. Таблица «что тестирую где»

| Что | Где | Скорость |
|---|---|---|
| Pure logic (функции без UIKit/system) | XCTest на macOS/sim | Быстро (ms) |
| Хуки (замена IMP) | Integration на устройстве | Медленно (секунды-минуты) |
| UI изменения | Integration + manual visual check | Самое медленное |
| Thread safety | Integration с нагрузкой | Медленно, flaky |
| Производительность | Instruments на устройстве | Специализированно |
| Compatibility 26.0 vs 26.0.1 | Два устройства | Разовый при релизе |

## 8. Автоматизация через скрипты

Для integration тестов:

- **expected.log** — git-committed, обновляется осознанно.
- **actual.log** — на каждый run.
- `diff` — быстрая проверка.

Для unit тестов:

- XCTest — встроенная система.
- Fastlane — если проект сложный.

## 9. Что подтверждено

- **Подтверждено Apple docs**: XCTest как framework.
- **Подтверждено практикой**: DI pattern — стандарт для testable ObjC кода.
- **Требует проверки**: конкретный workflow — зависит от вашей инфраструктуры.
- **Не подтверждено**: универсальный framework для «unit testing твиков на устройстве» — такого стандарта нет.

## 10. Выводы

Главный тезис: **разделите «logic» и «hooks»**. Логика — чистая, тестируется XCTest. Хуки — тонкая прослойка, тестируется на устройстве.

70% багов можно поймать на этапе `logic` unit tests, экономя часы отладки на устройстве.

## 11. См. также

- Advanced 3 — CI/CD.
- Advanced 9 — static analysis.
- Глава IV.7 — threads/ARC/lifecycle.
