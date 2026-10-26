# Кейс 7. Устойчивый к падениям скелет твика

Этот кейс отвечает на вопрос, который в прикладной литературе про твикописание обычно обходят стороной: «что сделать так, чтобы ошибка в моём коде не превращала весь целевой процесс в SIGABRT». На iOS 26.0 / 26.0.1 (A12/A13, PPL; подтверждено `Application/Dopamine/Jailbreak/DOEnvironmentManager.m:254`) цена падения SpringBoard'а, `backboardd` или даже обычного приложения в RootHide выше, чем в rootful-джейлбрейке 15.x: во-первых, перезапуск SpringBoard тащит за собой перенастройку substrate-прослойки через `jbctl`-инфраструктуру (см. `BaseBin/jbctl/src/*`), во-вторых, Apple в 26.x активнее опирается на `ReportCrash` для телеметрии через `symptomsd`, и серия подряд упавших процессов имеет шанс оказаться в diagnostic buffer, который в редких случаях синхронизируется в iCloud Analytics (наблюдение по iOS 26 SDK Diagnostics; требует проверки на устройстве). Третья причина более приземлённая: твик, который валит таргет, в сообществе получает плохие отзывы раньше, чем вы успеете понять, почему он валит.

Задача кейса — собрать скелет, пригодный как стартовый `Tweak.xm` для почти любого твика под RootHide. Скелет состоит из трёх независимых слоёв защиты: (1) явные проверки `nil`/`NULL` до каждого вызова `%orig` и перед обращением к ivars, (2) ObjC-level `@try/@catch` вокруг «горячих» селекторов и глобальный `NSSetUncaughtExceptionHandler`, (3) signal handler на C для диагностики SIGSEGV/SIGBUS/SIGILL/SIGTRAP без того, чтобы процесс завершился до того, как мы запишем контекст. При этом слои не маскируют ошибку — они диагностируют её, пишут отчёт в контейнер приложения и, если возможно, безопасно продолжают работу (или контролируемо завершают процесс так, чтобы Apple-овский `ReportCrash` всё равно получил корректный диагноз). Образцом для signal handler служит реализация из самого Dopamine: `BaseBin/libroothider/src/crashreporter.m:506-611`.

---

## 1. Что такое «падение» твика и зачем ловить его в себе

Падение твика — это одно из трёх событий в адресном пространстве целевого процесса:

1. ObjC-исключение, долетевшее до `NSUncaughtExceptionHandler` (например, `[NSNull length]`, `[nil mutableArrayValueForKey:]`, `KVC`-рассинхрон после `%hook`-а setter'а).
2. Сигнал ядра: SIGSEGV на разыменовании висячего указателя, SIGBUS на невыровненной записи в MMIO-подобную память, SIGILL на попытке выполнить PAC-обнулённый указатель (на A13 это основной способ, которым PPL-подпись не пускает нас внутрь системного фреймворка), SIGTRAP на `__builtin_trap()` в системных ассертах, SIGABRT на `abort()` из рантайма.
3. Контролируемый `exit`/`abort` из нашего собственного кода, например из assert'а в C-утилите, которую мы линкуем в твик.

Второе — наиболее болезненное: на arm64e попытка вызвать `IMP`-указатель, который Logos получил ошибочно (например, через `class_getInstanceMethod` на уже выгруженном классе), приводит не к красивому «unrecognized selector», а к мгновенному SIGILL из-за несовпадения PAC-штампа. Поэтому полагаться только на ObjC-уровень `@try/@catch` недостаточно: половина проблем приходит из C-уровня (включая Swift-функции, которые при развёртывании C++-эквивалентов опираются на `__builtin_trap`).

Чтобы увидеть, насколько разнообразны источники падений в твике на iOS 26, полезна следующая классификация с привязкой к механизму доставки:

| Источник | Как превращается в сигнал/исключение | Какой слой перехватит |
|---|---|---|
| `[NSNull length]` | `objc_exception_throw` → `NSInvalidArgumentException` | `@try/@catch` или `NSUncaughtExceptionHandler` |
| `nil->ivar` разыменование | SIGSEGV на адрес ~0x0..0x20 | signal handler |
| Устаревший `IMP` после `class_replaceMethod` | SIGILL из-за PAC-mismatch | signal handler |
| `assert(0)` в системном фреймворке | SIGABRT через `abort` | signal handler |
| Swift `nil!` force-unwrap | SIGILL через `__builtin_trap` | signal handler |
| OOM / jetsam | внешний SIGKILL | не перехватывается |
| Watchdog таймаут SpringBoard'а | внешний SIGKILL | не перехватывается |
| TCC-деним на файл | `EACCES` в syscall, не сигнал | ваш собственный `errno`-check |

Из восьми категорий шесть перехватываются одним из трёх слоёв, две — принципиально нет. Это уточняет границы «устойчивого скелета»: он защищает от того, что протекает через ваш код в userspace, но не защищает от решений ядра («жить ли этому процессу») и внешних watchdog'ов.

С другой стороны, подавить все три источника не получится и не нужно: цель скелета — локализовать и задокументировать падение, не выдавая процессу «обратного билета» в неконсистентное состояние. Образец того же подхода в Dopamine: `crashreporter_catch_objc` (BaseBin/libroothider/src/crashreporter.m:358) пишет отчёт и завершает процесс — но делает это после того, как состояние зафиксировано.

Классификация падений важна ещё и потому, что реакция ОС на них разная. ObjC-исключение, если его никто не поймал, превращается в вызов `objc_terminate` → `std::terminate` → `abort`, то есть в итоге всё равно SIGABRT. Но между моментом «исключение было выпущено» и моментом «пришёл сигнал» есть окно в несколько десятков наносекунд, в котором `NSUncaughtExceptionHandler` работает в полноценном Objective-C рантайме: доступны все классы, выделение памяти, запись в файл через `NSFileManager`. Это окно и есть ценность второго слоя. С сигналом же такого окна нет: мы попадаем в handler уже под строгими правилами async-signal-safety.

Третий источник — наш собственный `abort()` или `__builtin_trap()` — технически предсказуем и мог бы быть обработан нашим же signal handler'ом, но важно помнить: `abort()` в POSIX сначала разблокирует SIGABRT (на случай, если он был заблокирован через `sigprocmask`), затем шлёт сигнал через `raise(SIGABRT)`, затем, если управление вернётся, восстанавливает `SIG_DFL` и шлёт SIGABRT повторно. Это значит, что наш обработчик получит сигнал, запишет отчёт, и после возврата (помните — `SA_RESETHAND` у нас активен) процесс всё равно умрёт. Это ожидаемое поведение и то, что мы хотим.

> ПОДТВЕРЖДЕНО ИСХОДНИКАМИ: Dopamine регистрирует и ObjC-handler, и `sigaction` для 8 сигналов — `SIGILL, SIGTRAP, SIGABRT, SIGEMT, SIGFPE, SIGBUS, SIGSEGV, SIGSYS` (BaseBin/libroothider/src/crashreporter.m:586-597).
> ТРЕБУЕТ ПРОВЕРКИ НА УСТРОЙСТВЕ: что `ReportCrash` получает те же данные после нашего self-report'а; теоретически `SA_RESETHAND` в Dopamine (`crashreporter.m:608`) снимает наш обработчик после первого срабатывания и даёт Apple обработать сигнал ещё раз.

---

## 2. Три слоя защиты — обзор

Прежде чем переходить к коду, важно понять, зачем нужны все три слоя одновременно и как они пересекаются. Односложно: nil-проверки защищают от известных сценариев, `@try/@catch` ловит «неизвестное известное» (ObjC-исключения из рантайма), signal handler — «неизвестное неизвестное» (повреждение памяти, PAC, пропущенный тип).

```mermaid
flowchart LR
    A[Входящий %orig] --> B{nil / NULL<br/>проверки}
    B -- fail --> R1[Возврат безопасного значения]
    B -- ok --> C[@try]
    C --> D[%orig<br/>или ваша логика]
    D -- NSException --> E[@catch:<br/>лог + fallback]
    D -- POSIX signal --> F[signal_handler]
    F --> G[Запись отчёта]
    G --> H[SA_RESETHAND:<br/>повторная доставка<br/>штатному ReportCrash]
    C -- ok --> I[Нормальный возврат]
    E --> I
```

Каждому слою соответствует своя зона ответственности:

| Слой | Что ловит | Чем реализован | Чем НЕ является |
|---|---|---|---|
| nil/NULL guards | предсказуемые деградации (отсутствующий класс, пустой словарь, NULL из `dlsym`) | обычные `if`-ветки на входе хука | не ловит повреждение памяти |
| `@try/@catch` + `NSSetUncaughtExceptionHandler` | ObjC-исключения, `NSInvalidArgumentException`, KVC-пропажи | `objc_exception_throw`-инфраструктура | не ловит сигналы, не ловит Swift-ошибки без ObjC-моста |
| `sigaction` | SIGSEGV/SIGBUS/SIGILL/SIGTRAP/SIGABRT/SIGFPE | C-функция + `backtrace()` | не умеет безопасно восстанавливать выполнение; только диагностика |

Из таблицы следует главный архитектурный принцип: **слои не заменяют друг друга и ничего не маскируют**. Они формируют приоритизированный пайплайн диагностики; при этом «восстановление» (continue execution) допустимо только на самом верхнем слое — то есть после `@catch` или после проваленного `nil`-guard'а. На уровне signal handler «восстановление» запрещено: единственный безопасный выход — записать отчёт и дать штатному механизму доставить сигнал повторно.

Полезно также понимать, что между слоями есть «мёртвая зона»: ошибка в C-коде, которую нельзя превратить в сигнал. Пример — утечка `malloc`, которая постепенно съедает память и приводит к `jetsam`-киллу процесса ядром. Такое событие сигнала не породит; ядро просто пошлёт процессу `SIGKILL`, который не ловится принципиально. Диагностировать этот сценарий можно только снаружи: через `log stream --predicate 'eventMessage CONTAINS "jetsam"'`. Этот паттерн выходит за границы скелета и упоминается здесь, чтобы читатель не ожидал от скелета того, чего он архитектурно не способен дать: защиту от OOM / jetsam / внешних принудительных киллов.

Ещё одна зона, граничащая с «мёртвой» — SIGPIPE и SIGCHLD. Их мы сознательно НЕ ловим: SIGPIPE в твике почти всегда означает закрытое соединение, обрабатываемое штатным `errno=EPIPE`, а SIGCHLD прилетает только если мы форкаем, что в твике для iOS практически не делается. Оставить их на `SIG_IGN` или `SIG_DFL` — верный выбор.

---

## 3. Nil/NULL-проверки до %orig: формальный контракт

Половина падений твиков случается на пустом месте: `%hook` вызывает `%orig` с аргументом, который на `nil` вызовет NSInvalidArgumentException внутри нехукнутой реализации. Обычная ошибка — считать, что Logos сам «пробрасывает nil как no-op». Он не пробрасывает: `%orig` разворачивается в прямой вызов оригинального `IMP` с теми же аргументами, что пришли в хук.

Поэтому в любом хуке действуют четыре правила:

1. **Проверяй self.** Если ваш `%hook` стоит на экземплярном методе и вы вызываете `%orig` после модификации, убедитесь, что `self` жив (`isa` не 0). На практике это проявляется редко, но в хуках `-dealloc` критично: вызов `%orig` обязателен и обязателен ровно один раз.
2. **Проверяй аргументы по контракту метода, даже если в документации метода написано `nonnull`.** В реальности Apple-овские внутренние вызывающие сайты регулярно нарушают собственный nonnull, особенно в iOS-26-перестроенных путях (например, `UIScene` жизненный цикл). Нулевой аргумент — норма, SIGSEGV из-за него — ваша вина.
3. **Проверяй возвращаемое значение `%orig`.** Если верхний код ожидает nonnull, а оригинал вернул nil (например, из-за того, что системный класс в iOS 26 реорганизован), ваш твик не должен добивать процесс.
4. **Проверяй результаты `dlsym`, `objc_getClass`, `class_getInstanceMethod`.** Это самое больное место для twik'ов, переносимых между iOS 18 и iOS 26: класс на 26-й может называться иначе, селектора могут быть сдвинуты. См. также кейс 5 (`%group`) про условную инициализацию.

Формальный контракт в коде выглядит так — показана заготовка «безопасной обёртки» для метода, который принимает три аргумента и может вернуть `nil`:

```objc
// Контракт: self != nil, arg1 — путь (nonnull по документации, но может быть nil в iOS26),
// arg2 — опциональный, arg3 — не используется, но прокидывается в %orig.
- (NSArray *)fs_lookupPath:(NSString *)arg1
                  withHint:(NSDictionary *)arg2
                     flags:(NSUInteger)arg3
{
    // (1) self-check: защищаемся от «повторного deallocation»
    if (!self || !object_getClass(self)) {
        return @[];
    }

    // (2) argument-check: контракт нарушается — не идём в %orig,
    // возвращаем «пустое, но корректное» значение.
    if (!arg1 || ![arg1 isKindOfClass:[NSString class]]) {
        FSTweakLog(@"fs_lookupPath: nil/invalid arg1, returning empty");
        return @[];
    }

    NSArray *result = nil;
    @try {
        result = %orig(arg1, arg2, arg3);
    } @catch (NSException *e) {
        FSTweakLog(@"fs_lookupPath: NSException caught: %@", e);
        return @[];
    }

    // (3) result-check: оригинал имеет право вернуть nil
    if (![result isKindOfClass:[NSArray class]]) {
        return @[];
    }

    return result;
}
```

Четыре принципа видны в коде построчно. Проверка `object_getClass(self)` — это дешёвый способ отличить живой объект от «пустой коробки» после ARC-освобождения: сам `self` при dangling-ссылке может быть ненулевым, но `isa` будет затёрт. `isKindOfClass:` после `%orig` защищает от ситуации, когда Apple между 26.0 и 26.0.1 поменяла возвращаемый тип (прецедент — не доказан, но iOS 17.x знал такие изменения; статус: не подтверждено на 26.x).

> ПОДТВЕРЖДЕНО ДОКУМЕНТАЦИЕЙ: `object_getClass` — публичная ObjC-рантайм-функция, доступна с iOS 2.0; её безопасно звать на любом адресе-указателе (включая освобождённый), пока адрес отображён в текущую VM-карту; в противном случае будет SIGSEGV, который перехватит наш signal handler.

Стоит отметить тонкость, которая вылезает в хуках свойств (`@property`). Если вы хукаете setter `-setFoo:`, nil-guard должен различать два сценария: «вызывающий намеренно хочет установить nil» (валидный случай для `strong`-свойств с `nil`-семантикой) и «вызывающий получил nil из-за своей ошибки». В первом случае вы должны вызвать `%orig(nil)`; во втором — вернуть без вызова. Разграничить их статически нельзя; эмпирический подход — следовать документированному контракту свойства из заголовочников SDK. Если контракт говорит `_Nullable`, пропускаем nil в `%orig`. Если `_Nonnull`, nil означает ошибку и мы её гасим без вызова `%orig`.

Отдельный класс nil-ошибок — "KVC-рассинхрон". Пример: ваш `%hook` на setter'е положил вместо `NSString` объект типа `NSNumber`. Следующий getter вернёт `NSNumber`, но вызывающий код через `-length` вызовет `NSInvalidArgumentException`. Это не падение *вашего* кода, но падение *из-за* вашего кода. Защита — проверять в setter'е класс, а не только nil: `if (![value isKindOfClass:[NSString class]]) return;`. Для скелета мы это иллюстрируем в примере с `%orig` возвращающим `NSArray` — проверка `isKindOfClass:` защищает именно от этой проблемы.

---

## 4. ObjC-exception слой: @try/@catch и глобальный handler

ObjC-исключения в iOS уже много лет являются «неидиоматическим» механизмом: Apple настоятельно рекомендует возвращать `NSError`-объекты. Тем не менее они до сих пор щедро летают из Cocoa (`NSInvalidArgumentException` из `-performSelector:`, `NSRangeException` из `-objectAtIndex:`, `NSInternalInconsistencyException` из KVO). В твике, который перехватывает чужие классы, пропустить такое исключение — почти гарантированно обрушить процесс. Два инструмента:

1. **Локальный `@try/@catch`** вокруг каждого вызова `%orig` в хуке «горячей» или «небезопасной» точки — например, обработка UIKit-ивентов, сетевых колбэков, dispatch-queue обработчиков.
2. **Глобальный `NSSetUncaughtExceptionHandler`** как последняя линия для исключений, которые пришли не из нашего кода, но пойманы мы будем первыми, если установили handler.

Второй механизм — тот же, что использует сам Dopamine в `BaseBin/libroothider/src/crashreporter.m:493`. Разница между этими двумя инструментами больше, чем кажется: локальный `@try/@catch` выполняется в полном рантайме Objective-C с живой стековой рамкой вызова, что позволяет корректно выполнять fallback (возврат значения по умолчанию, повторная попытка с другими аргументами и т.д.). Глобальный handler вызывается уже после того, как рантайм решил «пора умирать»: стек в этот момент разворачивается через `_Unwind_Backtrace`, многие C++-деструкторы уже отработали, и ваш handler работает поверх «частично разобранного» стека. Поэтому локальный `@try/@catch` рекомендуется ставить на всех горячих хуках, а не полагаться только на global handler.

```objc
defaultNSExceptionHandler = NSGetUncaughtExceptionHandler();
NSSetUncaughtExceptionHandler(crashreporter_catch_objc);
```

Важный паттерн — **цепочка**: мы запоминаем предыдущий handler через `NSGetUncaughtExceptionHandler()` и после своей диагностики зовём его. Если так не делать, твик перехватит исключения другого твика, установленного раньше, и тот не узнает о падении.

Полный глобальный handler для нашего скелета выглядит так:

```objc
#import <Foundation/Foundation.h>

static NSUncaughtExceptionHandler *gPrevExceptionHandler = NULL;

static void FSTweakUncaughtExceptionHandler(NSException *exception)
{
    @autoreleasepool {
        NSString *report =
            [NSString stringWithFormat:
             @"[FSTweak] uncaught %@: %@\nuserInfo: %@\nstack:\n%@",
             exception.name, exception.reason,
             exception.userInfo, exception.callStackSymbols];
        FSTweakWriteReport(@"objc", report);
    }
    // Передаём дальше по цепочке — наш твик не единственный на устройстве.
    if (gPrevExceptionHandler) {
        gPrevExceptionHandler(exception);
    }
}

%ctor {
    gPrevExceptionHandler = NSGetUncaughtExceptionHandler();
    NSSetUncaughtExceptionHandler(FSTweakUncaughtExceptionHandler);
}
```

Разбор. `@autoreleasepool` обязателен: мы находимся на выходе из рантайма, когда штатные пулы могут быть уже дренированы. `callStackSymbols` работает только для исключений, выброшенных через `objc_exception_throw` (не для `NSException`, собранных вручную без `raise`); для скелета это достаточно: UIKit/Foundation выбрасывают именно через `raise`. Функция `FSTweakWriteReport` — наша собственная асинхронно-небезопасная запись в файл; в контексте `NSUncaughtExceptionHandler` она допустима, так как мы ещё в рантайме Objective-C, а не внутри сигнала (это важное различие — см. раздел 5).

> ПОДТВЕРЖДЕНО ИСХОДНИКАМИ: симметричный pause/resume pattern Dopamine (`crashreporter_pause`/`crashreporter_resume` на `crashreporter.m:451-502`) сохраняет и восстанавливает именно `defaultNSExceptionHandler`, что подтверждает идиому «цепочки».

Отдельное предупреждение по Swift. Swift-ошибки не являются `NSException` и в `NSUncaughtExceptionHandler` не попадут. Если ваш твик вызывает Swift-функции через ObjC-мост и там случится `fatalError`, вы получите SIGTRAP, который перехватит signal handler из раздела 5. Для Swift-слоя защиты нужен `do/catch` на уровне Swift и защитный wrapper на границе моста — но это выходит за рамки скелета.

Ещё одна тонкость — `objc_setUncaughtExceptionHandler` (C-уровневая версия `NSSetUncaughtExceptionHandler`). Эти два API делятся на `objc` и `NS` варианты не случайно: ObjC-уровневый handler ставится раньше в процедуре terminate и видит исключение в момент, когда рантайм ещё не позвал `[NSException raise]`. Для твика достаточно `NS`-версии: Foundation на iOS 26 вызывается первой подавляющим большинством UIKit-путей. Но если ваш таргет — демон без Foundation (встречается у минимальных сервисов), ставьте `objc_setUncaughtExceptionHandler`. Для скелета мы остаёмся на `NS`.

---

## 5. Signal handler: заимствованная из Dopamine техника

Этот слой — последний рубеж. Его реализация в Dopamine (BaseBin/libroothider/src/crashreporter.m:506-584) — лучший из открытых образцов на A13, потому что она аккуратно обрабатывает PAC-штампованные указатели в контексте регистров, что отличает A12/A13 от A11 и выше (A17 Pro уже имеет другой набор state-flags, но для скелета это неважно). Воспроизводим ключевую идею, адаптируя под твик.

Ограничения signal handler, которые нужно принять раз и навсегда:

- Внутри `sigaction`-обработчика безопасен только набор async-signal-safe функций: `write`, `_exit`, `signal`, `sigaction`, `backtrace` (на Darwin — да, async-signal-safe по документации), часть `pthread`-опросных. **Нельзя**: `malloc`, `printf`, `fopen`, Objective-C runtime, NSFileHandle, dispatch_async.
- Поэтому мы не создаём отчёт «на лету» через `NSString` — мы используем preallocated буфер и `write(fd, …)` в заранее открытый файловый дескриптор.
- Используем `SA_SIGINFO` (чтобы получить `siginfo_t` и `ucontext_t`) и `SA_RESETHAND` (чтобы после первого срабатывания штатный `ReportCrash` всё-таки получил сигнал). Это ровно та комбинация, что в Dopamine: `crashreporter.m:608`.

```c
#include <signal.h>
#include <execinfo.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <sys/ucontext.h>
#include <pthread.h>

#define SIG_REPORT_PATH "/var/mobile/Documents/FSTweak/last_signal.txt"

static int gSignalFd = -1;

static void fs_sig_write_str(const char *s) {
    if (gSignalFd < 0 || !s) return;
    size_t n = strlen(s);
    (void)write(gSignalFd, s, n);
}

static void fs_sig_write_hex(uint64_t v) {
    if (gSignalFd < 0) return;
    char buf[20]; buf[0] = '0'; buf[1] = 'x';
    const char *hex = "0123456789abcdef";
    for (int i = 0; i < 16; i++) {
        buf[2 + i] = hex[(v >> (60 - i*4)) & 0xf];
    }
    buf[18] = '\n'; buf[19] = 0;
    (void)write(gSignalFd, buf, 19);
}

static void fs_signal_handler(int signo, siginfo_t *info, void *ctx)
{
    // (A) Открываем лог один раз, в async-signal-safe манере.
    if (gSignalFd < 0) {
        gSignalFd = open(SIG_REPORT_PATH,
                         O_WRONLY | O_CREAT | O_APPEND, 0644);
    }

    // (B) Пишем «шапку».
    fs_sig_write_str("---- FSTweak signal ----\n");
    fs_sig_write_str(strsignal(signo));
    fs_sig_write_str("\n");

    // (C) Регистры с учётом PAC.
    ucontext_t *uc = (ucontext_t *)ctx;
    if (uc && uc->uc_mcontext) {
        fs_sig_write_str("pc=");
        fs_sig_write_hex((uint64_t)__darwin_arm_thread_state64_get_pc(uc->uc_mcontext->__ss));
        fs_sig_write_str("lr=");
        fs_sig_write_hex((uint64_t)__darwin_arm_thread_state64_get_lr(uc->uc_mcontext->__ss));
        fs_sig_write_str("sp=");
        fs_sig_write_hex((uint64_t)__darwin_arm_thread_state64_get_sp(uc->uc_mcontext->__ss));
    }

    // (D) backtrace, если успеем — на Darwin он async-signal-safe.
    void *frames[32];
    int n = backtrace(frames, 32);
    fs_sig_write_str("frames:\n");
    for (int i = 0; i < n; i++) {
        fs_sig_write_hex((uint64_t)frames[i]);
    }

    // (E) SA_RESETHAND уже снял нас. Возвращаемся — ядро повторно
    //     доставит сигнал штатному обработчику. ReportCrash получит
    //     нормальный crash log.
}

void fs_install_signal_handler(void) {
    static const int sigs[] = {
        SIGILL, SIGTRAP, SIGABRT, SIGEMT,
        SIGFPE, SIGBUS, SIGSEGV, SIGSYS
    };
    for (unsigned i = 0; i < sizeof(sigs)/sizeof(sigs[0]); i++) {
        struct sigaction act = {0};
        act.sa_flags = SA_SIGINFO | SA_RESETHAND;
        act.sa_sigaction = fs_signal_handler;
        sigaction(sigs[i], &act, NULL);
    }
}
```

Разбор по блокам. (A) `open` открывается один раз и сохраняется в статической переменной, чтобы не вызывать malloc внутри обработчика при повторном сигнале. (B) `strsignal` в Darwin возвращает указатель на статическую строку в libc, что безопасно в сигнале (не выделяет память). (C) мы читаем регистры через `__darwin_arm_thread_state64_get_pc/lr/sp` — те же accessor'ы, что использует Dopamine (`crashreporter.m:547-553`); на arm64e они автоматически стрипают PAC-штамп, что критично для осмысленности pc/lr. (D) `backtrace` на Darwin разрешён в async-signal context начиная с macOS 10.5 и iOS 2 (это документированное исключение из общего правила), но результат — только адреса, без символов (`backtrace_symbols` вызывать нельзя, он лезет в dyld-таблицы через malloc). (E) мы не зовём `exit`/`abort` сами: `SA_RESETHAND` снял наш обработчик, и при возврате из `fs_signal_handler` ядро повторно доставит сигнал — теперь уже дефолтному обработчику, что приведёт к стандартному Apple-овскому crash log. Это и есть «не маскировать».

Путь `SIG_REPORT_PATH` намеренно показан как `/var/mobile/Documents/FSTweak/last_signal.txt`. В реальном твике внутри приложения-таргета более грамотно писать в `~/Documents` целевого приложения (см. кейс 1), чтобы отчёт был изолирован и не конфликтовал с системными TCC-правилами. Для демонов (`backboardd` и др.) запись должна идти в `/var/jb/<rand>/Library/Logs/FSTweak/…` через `JBROOT_PATH` (ПОДТВЕРЖДЕНО ИСХОДНИКАМИ: `BaseBin/libjailbreak/src/jbroot.h:38-40`). Жёстко прошитый путь в скелете — учебный приём; в коммерческом твике разрешите его через `NSHomeDirectory()`.

> ПОДТВЕРЖДЕНО ИСХОДНИКАМИ: набор ловимых сигналов в Dopamine (`crashreporter.m:586-597`) совпадает с нашим; `SA_SIGINFO|SA_RESETHAND` комбинация — `crashreporter.m:608`.
> ТРЕБУЕТ ПРОВЕРКИ НА УСТРОЙСТВЕ: что `backtrace()` действительно не выделяет память на iOS 26 — заголовки это декларируют, но наблюдений за новым runtime мало. Fallback — не звать `backtrace` вовсе, писать только pc/lr/fp.

Отдельного обсуждения заслуживает способ, которым мы читаем регистры. На arm64e `__darwin_arm_thread_state64_get_pc` и `__darwin_arm_thread_state64_get_lr` — это не макро-ссылки на `__ss.__pc`/`__ss.__lr`, а инлайн-функции в `<mach/arm/_structs.h>`, которые внутри дёргают `ptrauth_strip(pc, ptrauth_key_asia)` и `ptrauth_strip(lr, ptrauth_key_asib)`. Благодаря этому мы в своём отчёте видим канонический адрес без PAC-штампа — таким же, каким его увидит Apple-овский `ReportCrash`. Это и есть причина, почему Dopamine в `crashreporter.m:547-553` использует именно accessor-функции, а не прямое чтение полей. Прямое чтение дало бы «мусорный» high-18 бит и запутало бы последующий matching адресов с символами.

Второй тонкий момент — поле `__opaque_flags` (на A13 — часть `__pad`-пространства). В `crashreporter.m:538-544` Dopamine проверяет макрос `__arm64e__` и работает с этим полем, выставляя `__DARWIN_ARM_THREAD_STATE64_FLAGS_NO_PTRAUTH`. Для нашего скелета это не нужно (мы не возвращаем изменённый контекст), но знать о наличии таких флагов полезно: они говорят рантайму, что мы собираемся восстановить состояние без PAC-штампов. Без установки флага возврат из signal handler'а с модифицированным pc/lr приведёт к SIGILL на следующей инструкции. Поскольку мы НЕ модифицируем контекст, флаги оставляем как есть.

Отдельно: `backtrace_symbols` НЕЛЬЗЯ звать внутри handler'а, но её вполне можно вызвать в не-signal-контексте, если `gSignalFd` оставить открытым и сам отчёт сохранить. Расширенная версия handler'а может писать «сырые» адреса в один файл, а при старте процесса (в `%ctor`) парсить файл и разворачивать адреса в символы через `backtrace_symbols`, сохраняя человеко-читаемый отчёт рядом. Для скелета это не включено, но хорошо иллюстрирует принцип «только собрать данные, обработка — после».

---

## 6. Готовый скелет Tweak.xm

Ниже — собранный скелет, который можно взять за основу под любой твик. Он содержит все три слоя и один «пример-хук» на UIViewController, чтобы было понятно, как встраивать свою логику.

```objc
// Tweak.xm — SafeTweak skeleton
// Для iOS 26.0 / 26.0.1 (A12/A13, PPL).
// Dopamine RootHide: подтверждение целевой среды —
// Application/Dopamine/Jailbreak/DOEnvironmentManager.m:254.

#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import <UIKit/UIKit.h>
#import <signal.h>
#import <execinfo.h>
#import <unistd.h>
#import <fcntl.h>
#import <sys/ucontext.h>

#pragma mark - Логирование

static NSString *FSTweakReportDir(void) {
    NSArray *paths = NSSearchPathForDirectoriesInDomains(
        NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *base = paths.firstObject ?: @"/tmp";
    NSString *dir = [base stringByAppendingPathComponent:@"FSTweak"];
    [[NSFileManager defaultManager] createDirectoryAtPath:dir
                              withIntermediateDirectories:YES
                                               attributes:nil error:nil];
    return dir;
}

static void FSTweakWriteReport(NSString *kind, NSString *body) {
    @autoreleasepool {
        NSString *path = [FSTweakReportDir()
            stringByAppendingPathComponent:
             [NSString stringWithFormat:@"%@-%ld.txt",
              kind, (long)[NSDate date].timeIntervalSince1970]];
        [body writeToFile:path atomically:YES
                 encoding:NSUTF8StringEncoding error:nil];
    }
}

#define FSTweakLog(fmt, ...) \
    NSLog(@"[FSTweak] " fmt, ##__VA_ARGS__)

#pragma mark - Слой 2: ObjC uncaught handler

static NSUncaughtExceptionHandler *gPrevExceptionHandler = NULL;

static void FSTweakUncaughtExceptionHandler(NSException *exception) {
    @autoreleasepool {
        NSString *report = [NSString stringWithFormat:
            @"name: %@\nreason: %@\nuserInfo: %@\nsymbols:\n%@",
            exception.name, exception.reason,
            exception.userInfo,
            [exception.callStackSymbols
                componentsJoinedByString:@"\n"]];
        FSTweakWriteReport(@"objc", report);
    }
    if (gPrevExceptionHandler) gPrevExceptionHandler(exception);
}

#pragma mark - Слой 3: signal handler

static int gSignalFd = -1;

static void fs_sig_write(const char *s) {
    if (gSignalFd < 0 || !s) return;
    (void)write(gSignalFd, s, strlen(s));
}

static void fs_sig_hex(uint64_t v) {
    if (gSignalFd < 0) return;
    static const char *hex = "0123456789abcdef";
    char buf[20] = {'0','x'};
    for (int i = 0; i < 16; i++)
        buf[2+i] = hex[(v >> (60 - i*4)) & 0xf];
    buf[18] = '\n';
    (void)write(gSignalFd, buf, 19);
}

static void fs_signal_handler(int signo, siginfo_t *info, void *ctx) {
    if (gSignalFd < 0) {
        gSignalFd = open("/tmp/FSTweak-sig.txt",
                         O_WRONLY|O_CREAT|O_APPEND, 0644);
    }
    fs_sig_write("---- FSTweak signal ----\n");
    fs_sig_write(strsignal(signo));
    fs_sig_write("\n");

    ucontext_t *uc = (ucontext_t *)ctx;
    if (uc && uc->uc_mcontext) {
        fs_sig_write("pc="); fs_sig_hex(
            (uint64_t)__darwin_arm_thread_state64_get_pc(uc->uc_mcontext->__ss));
        fs_sig_write("lr="); fs_sig_hex(
            (uint64_t)__darwin_arm_thread_state64_get_lr(uc->uc_mcontext->__ss));
        fs_sig_write("sp="); fs_sig_hex(
            (uint64_t)__darwin_arm_thread_state64_get_sp(uc->uc_mcontext->__ss));
    }

    void *frames[24];
    int n = backtrace(frames, 24);
    fs_sig_write("bt:\n");
    for (int i = 0; i < n; i++) fs_sig_hex((uint64_t)frames[i]);
    // return — SA_RESETHAND вернёт сигнал системному обработчику.
}

static void fs_install_signal_handlers(void) {
    static const int sigs[] = {
        SIGILL, SIGTRAP, SIGABRT, SIGEMT,
        SIGFPE, SIGBUS, SIGSEGV, SIGSYS
    };
    for (unsigned i = 0; i < sizeof(sigs)/sizeof(sigs[0]); i++) {
        struct sigaction act = {0};
        act.sa_flags = SA_SIGINFO | SA_RESETHAND;
        act.sa_sigaction = fs_signal_handler;
        sigaction(sigs[i], &act, NULL);
    }
}

#pragma mark - Слой 1: пример безопасного хука

%hook UIViewController

- (void)viewDidAppear:(BOOL)animated {
    // self-check
    if (!self || !object_getClass(self)) {
        return;
    }

    @try {
        %orig;
    } @catch (NSException *e) {
        FSTweakLog(@"viewDidAppear raised: %@", e);
        FSTweakWriteReport(@"objc-hook", e.description ?: @"(no desc)");
        return;
    }

    // Пример безопасной собственной логики ПОСЛЕ оригинала.
    @try {
        NSString *cls = NSStringFromClass(object_getClass(self));
        if (cls.length) {
            FSTweakLog(@"did appear: %@", cls);
        }
    } @catch (NSException *e2) {
        FSTweakLog(@"post-orig block failed: %@", e2);
    }
}

%end

#pragma mark - Загрузка

%ctor {
    @autoreleasepool {
        // Инициализируем слои в правильном порядке.
        // 1) signal — чтобы ловить падения уже в фазе хуков.
        fs_install_signal_handlers();
        // 2) ObjC uncaught — после того как Foundation точно поднят.
        gPrevExceptionHandler = NSGetUncaughtExceptionHandler();
        NSSetUncaughtExceptionHandler(FSTweakUncaughtExceptionHandler);
        // 3) Теперь регистрируем хуки.
        %init;

        FSTweakLog(@"FSTweak loaded in %@", NSProcessInfo
                   .processInfo.processName);
    }
}
```

Разбор по разделам. **Логирование** вынесено в две функции: `FSTweakLog` — быстрый syslog через `NSLog`, пригодный только в не-signal контексте; `FSTweakWriteReport` — запись в `~/Documents/FSTweak/`, которая опирается на `NSSearchPathForDirectoriesInDomains`, что в контексте приложения резолвит в его собственный sandbox. **Слой 2** разворачивает классический pattern из Dopamine. **Слой 3** — упрощённая, но полная версия signal-handler'а. **Слой 1** показан на `UIViewController -viewDidAppear:` только как иллюстрация — ту же обёртку нужно повторить на каждом своём хуке.

Отдельно обратите внимание на макрос `FSTweakLog`: он использует конкатенацию строковых литералов на этапе препроцессинга (`@"[FSTweak] " fmt`), что даёт нулевой runtime-overhead по сравнению с `NSString stringWithFormat:`. В production-твике полезно обернуть этот макрос в условную компиляцию `#if DEBUG`, чтобы в release-билде `NSLog` не загромождал системный журнал. Dopamine придерживается похожего подхода (BaseBin/libjailbreak/src/jbroot.h демонстрирует макрос-first подход).

Второй важный аспект — `createDirectoryAtPath:withIntermediateDirectories:` вызывается на каждом обращении к `FSTweakReportDir()`. Это дёшево (один `stat` + `mkdir` при его отсутствии), но для hot-path логов стоит кэшировать результат. Для скелета — не оптимизируем, читабельность важнее.

Ключевая деталь — порядок инициализации в `%ctor`. Signal handler ставится первым, потому что `%init` сам внутри зовёт `MSHookMessageEx`, и ошибка в этой фазе должна быть диагностирована. ObjC uncaught — вторым, после того как Foundation инициализирована (на твик-инъекции она уже инициализирована всегда, так что это скорее формальность). `%init` — последним.

> ПОДТВЕРЖДЕНО ИСХОДНИКАМИ: порядок «signal → objc → работа» воспроизводит порядок из Dopamine: сначала `sigaction` в цикле (`crashreporter.m:606-610`), затем `NSSetUncaughtExceptionHandler` через `crashreporter_resume` (`crashreporter.m:493`).

---

## 6a. Почему именно такой порядок в %ctor и чем он отличается от «наивного»

Многие твики ставят `%init` первой строкой `%ctor`, а защиту — после. На тестовом стенде это работает, в production — нет. Проблема в том, что `%init` вызывает `MSHookMessageEx` в цикле по всем декларациям `%hook`, и каждый такой вызов внутри себя читает Objective-C class list через `objc_copyClassList` / `class_copyMethodList`. На iOS 26.0 shared cache перестроен так, что часть классов живёт в lazy-инициализируемых разделах `__objc_const`, и первое к ним обращение триггерит realization. Если какой-то класс в этом списке оказался «битым» (например, forward-declared, но без тела в текущем образе), сам realization может SIGSEGV'нуть — и SIGSEGV прилетит ещё до того, как `%init` даст вам шанс поставить обработчик. По открытым наблюдениям такое поведение фиксировалось в редких комбинациях твиков в 18.x; на 26.x воспроизведено не было, но дефолт «ставить sigaction раньше %init» дешёвый и правильный (ТРЕБУЕТ ПРОВЕРКИ НА УСТРОЙСТВЕ).

Из этого следует и второе правило: `fs_install_signal_handlers` обязана быть ИДЕМПОТЕНТНОЙ. Если ваш `%ctor` по какой-то причине вызовется дважды (это редко, но возможно, когда один `.dylib` реэкспортирует другой через `-reexport_library`), повторный `sigaction` затрёт обработчик, оставшийся от первого вызова. В показанном коде идемпотентность обеспечивается тем, что `gSignalFd = -1` — глобал, не локаль, и `open` зовётся лениво только при фактическом сигнале, а сам цикл `sigaction` переустановит ровно наш обработчик (не чужой — поведение корректное). Если бы мы использовали, например, `signal(3)` вместо `sigaction(2)`, повторный вызов мог бы сбросить флаги `SA_SIGINFO|SA_RESETHAND` на дефолтные, что сломало бы PAC-корректное чтение регистров.

---

## 6b. Взаимодействие с Dopamine-овским crashreporter'ом

Отдельно разберём ситуацию, когда наш твик загружается в процесс, где уже активен crashreporter самого Dopamine (это норма для всех процессов, в которые `launchdhook` или `libroothider` установили свои обработчики через `crashreporter_start`, см. `BaseBin/libroothider/src/crashreporter.m:599-620`). У Dopamine свой `sigaction` на тех же 8 сигналах с теми же флагами `SA_SIGINFO|SA_RESETHAND`. Как уживутся два обработчика?

С `sigaction` правило простое: кто позже установил, тот и получит сигнал. Если наш `%ctor` отрабатывает после `libroothider`-инициализации (а это обычный случай для dylib-инъекции в приложение: `libroothider` подтягивается через systemhook ещё в `__dyld_start`, задолго до наших `%ctor`), наш обработчик встанет поверх Dopamine-овского, и мы получим сигнал первыми. При флаге `SA_RESETHAND` после возврата из нас ядро снимет наш обработчик и перепошлёт сигнал — но следующий получатель будет уже дефолтным Apple-овским, не Dopamine'овским, потому что Dopamine свой обработчик уже «уступил» нам, когда мы перезаписали `sigaction`-слот.

Это важно: наш отчёт появится, но Dopamine-овский отчёт — нет. Если задача — сохранить оба, нужно сохранить `oldact` при каждом `sigaction` и в теле нашего handler'а после своей диагностики честно вызвать `oldact.sa_sigaction(signo, info, ctx)`. Приведём «правильный» вариант установки:

```c
static struct sigaction gOldActs[8];
static int gOldActCount = 0;

static void fs_install_signal_handlers_chained(void) {
    static const int sigs[] = {
        SIGILL, SIGTRAP, SIGABRT, SIGEMT,
        SIGFPE, SIGBUS, SIGSEGV, SIGSYS
    };
    for (unsigned i = 0; i < sizeof(sigs)/sizeof(sigs[0]); i++) {
        struct sigaction act = {0};
        act.sa_flags = SA_SIGINFO;              // без SA_RESETHAND —
                                                // цепочкой управляем сами
        act.sa_sigaction = fs_signal_handler;
        sigaction(sigs[i], &act, &gOldActs[gOldActCount++]);
    }
}
```

Для цепочечного вызова tail-of-handler выглядит так:

```c
static void fs_signal_handler_chained(int signo, siginfo_t *info, void *ctx) {
    // ... наш отчёт как раньше ...

    // Цепочка: ищем oldact именно для signo.
    static const int sigs[] = {
        SIGILL, SIGTRAP, SIGABRT, SIGEMT,
        SIGFPE, SIGBUS, SIGSEGV, SIGSYS
    };
    for (unsigned i = 0; i < sizeof(sigs)/sizeof(sigs[0]); i++) {
        if (sigs[i] == signo) {
            struct sigaction *o = &gOldActs[i];
            if (o->sa_flags & SA_SIGINFO) {
                if (o->sa_sigaction) o->sa_sigaction(signo, info, ctx);
            } else {
                if (o->sa_handler && o->sa_handler != SIG_DFL
                                 && o->sa_handler != SIG_IGN) {
                    o->sa_handler(signo);
                }
            }
            return;
        }
    }
}
```

В этой версии мы отказались от `SA_RESETHAND` и сами передаём управление предыдущему обработчику. Это даёт лучшую совместимость: и наш отчёт появится, и Dopamine'овский (BaseBin/libroothider/src/crashreporter.m), а дефолтный Apple-овский `ReportCrash` будет вызван уже из Dopamine-овского обработчика по его собственной логике. Платой за это — усложнение кода и необходимость следить, чтобы наш handler сам не наделал ошибок, которые породят рекурсивный сигнал. Для подавляющего большинства твиков такой уровень оправдан.

> ПОДТВЕРЖДЕНО ИСХОДНИКАМИ: Dopamine-овский `crashreporter_start` (crashreporter.m:599) ставит `SA_SIGINFO|SA_RESETHAND`; после его срабатывания ядро действительно переустанавливает дефолт, что подтверждает возможность перехвата сигнала «поверх» Dopamine с сохранением цепочки.
> НЕ ПОДТВЕРЖДЕНО: порядок инъекции libroothider относительно substrate-овских dylib на iOS 26.0.1. Эмпирически libroothider раньше, но стенда для доказательства у автора нет.

---

## 6c. Swift и границы ObjC-ObjС моста

Отдельная история — твики, которые по каким-то причинам имеют Swift-зависимость (например, подключают Swift-only фреймворк для парсинга JSON по типу `Codable`). Любой Swift-код, который вываливается в `fatalError`, `preconditionFailure`, или трапается через force-unwrap `nil!`, превращается в SIGILL. На A13 это не SIGTRAP, как иногда думают: трап-инструкция arm64 (`brk #1`) порождает SIGILL в userspace, если debugger не присоединён. Наш signal handler это ловит, но мы теряем сообщение Swift-рантайма (которое Swift обычно шлёт в `stderr` непосредственно перед трапом).

Правильная техника — иметь в коде маленький Swift-обёрточный слой, который превращает Swift-ошибки в `NSError` и возвращает их через ObjC. Например:

```swift
// Bridge.swift
@objc public final class FSBridge: NSObject {
    @objc public static func parseJSON(_ data: Data,
                                       error: NSErrorPointer) -> NSDictionary? {
        do {
            let obj = try JSONSerialization.jsonObject(with: data)
            return obj as? NSDictionary
        } catch let e as NSError {
            error?.pointee = e
            return nil
        }
    }
}
```

На границе моста Swift-ошибки превращаются в nil + NSError, и ваш ObjC-уровень обрабатывает их обычным `if (!result) { … }`. Без такого моста любая Swift-ошибка — SIGILL, диагностируемый, но не предотвратимый. Это второй аргумент в пользу: держите Swift в твике минимальным и всегда на границе `@objc` с явным возвратом `NSError`.

> ТРЕБУЕТ ПРОВЕРКИ: доступность Swift-stdlib во всех таргет-процессах. Для SpringBoard и большинства демонов стабильный Swift-runtime (`libswiftCore.dylib`) загружен, но для минимальных демонов типа `wifid` — не всегда. Безопаснее держать твик на чистом ObjC/C.

---

## 7. Что скелет НЕ закрывает

Прозрачность важнее иллюзии безопасности. Скелет не защищает от:

- **Повреждения heap до старта обработчика.** Если ваш твик успел затереть `malloc`-метаданные до того, как `fs_install_signal_handlers` завершилась, обработчик всё равно может упасть внутри `backtrace()`. Mitigation: ставить sigaction первой строкой `%ctor`, до любых `NSString`/`NSArray` вызовов.
- **Повторного срабатывания в самом handler'е.** `SA_RESETHAND` снимает нас после первого сигнала, но если во время `backtrace()` случится новый SIGSEGV, ядро доставит его дефолтному обработчику, не нам. Это ожидаемое поведение.
- **PAC-ошибок на возврате из `%orig`.** На arm64e возврат по неправильному LR-штампу — SIGILL. Мы это зафиксируем, но не предотвратим.
- **Падений других твиков.** `NSSetUncaughtExceptionHandler` — глобальный, но `sigaction` тоже глобальный. Если другой твик поставил свои обработчики ПОСЛЕ нас, наш pattern с `SA_RESETHAND` оставит у них «одноразовый» слот, что не критично, но не идеально. Чистое решение — не ставить `SA_RESETHAND` и зацеплять oldact, сохраняя его в массиве, затем звать oldact.sa_sigaction в конце нашего handler'а. Это полезное упражнение для v2 скелета.
- **Swift-fatalError.** Как обсуждалось в разделе 4, Swift завершается через `__builtin_trap` → SIGILL. Мы это ловим, но смысловой контекст (сообщение) теряется. Решение — ставить `#if canImport(Swift)`-shim на границе моста, где через `do/catch` ловим ошибку до того, как она превратится в trap.

Все эти ограничения — не повод отказаться от скелета, а повод не рекламировать его как «твик не может упасть». Правильная формулировка: «твик, если упадёт, оставит после себя воспроизводимый артефакт».

---

## 7a. Мульти-поточность и thread-local соображения

Signal handler на Darwin доставляется в тот поток, в котором произошло «нарушение». Это критически отличает его от `NSUncaughtExceptionHandler`, который в UIKit-приложениях на iOS всегда вызывается на основном потоке. Два следствия для скелета:

1. **`gSignalFd` должен выдерживать конкурентный доступ.** В показанном коде он читается без блокировок, и теоретически два потока одновременно упадут с разными сигналами. `write(2)` в POSIX гарантирует атомарность до `PIPE_BUF` (на Darwin — 512 байт), поэтому записи до этого размера не перемешаются, но `open` может быть вызван дважды, оставив утечку одного fd. Это несущественная проблема (процесс и так умрёт), но знать её нужно.
2. **`backtrace()` вызванный из вторичного потока вернёт стек этого потока.** Это именно то, что нам нужно — не стек main thread, а стек места падения.

Для ObjC-слоя всё иначе. `NSUncaughtExceptionHandler` вызывается на том потоке, который выпустил исключение. Если это worker-поток `NSOperationQueue`, после возврата из нашего handler'а runtime всё равно завершит процесс через `abort`. Поэтому fallback через `return` после записи отчёта в ObjC-handler невозможен — процесс всё равно умрёт. Это задокументированное поведение Foundation; «восстановление» на этом уровне не предусмотрено.

Отдельный вопрос — `dispatch_queue`. Если ваш `%hook` обёрнут в `dispatch_async`, падение внутри блока произойдёт на queue-thread'е, и `NSUncaughtExceptionHandler` действительно вызовется в том же потоке. Все утверждения выше для queue-thread'ов сохраняются; специфики для libdispatch здесь нет.

---

## 7b. Расширение: fallback-возврат через `siglongjmp`

Иногда допустимо попытаться продолжить выполнение после сигнала — например, если наш хук в десятой итерации цикла получает SIGSEGV, можно разумно продолжить без этого элемента. Техника — `sigsetjmp/siglongjmp` вокруг «подозрительной» секции плюс выход из signal handler через `siglongjmp`. Для скелета мы это НЕ закладываем, но покажем принцип, чтобы читатель мог оценить компромисс.

```c
#include <setjmp.h>
static sigjmp_buf gFsJmp;
static volatile sig_atomic_t gFsJmpActive = 0;

static void fs_signal_handler_longjmp(int signo, siginfo_t *info, void *ctx) {
    // Сначала короткая запись в лог (как раньше).
    fs_sig_write("longjmp recovery\n");
    if (gFsJmpActive) {
        gFsJmpActive = 0;
        siglongjmp(gFsJmp, signo);  // возврат в sigsetjmp
    }
    // Иначе — обычное поведение: ошибка не в защищённой секции.
}

// Использование в хуке:
BOOL recovered = NO;
int sig = sigsetjmp(gFsJmp, 1);
if (sig == 0) {
    gFsJmpActive = 1;
    // «подозрительная» секция
    gFsJmpActive = 0;
} else {
    recovered = YES;
    FSTweakLog(@"recovered from signal %d", sig);
}
```

Это работает, но имеет жёсткие ограничения: любое состояние, которое изменилось внутри защищённой секции (allocated memory, modified globals, open file handles), остаётся «полуфабрикатом». Для хука в UI-коде это обычно неприемлемо. Для хука в чистой numeric-функции без побочных эффектов — приемлемо и иногда полезно. В production-твике рекомендую не включать такой fallback без осознанного решения.

> НЕ ПОДТВЕРЖДЕНО: безопасность `siglongjmp` из `SA_SIGINFO`-handler'а на arm64e iOS 26 с PAC. Теоретически — safe (longjmp работает со своим собственным jmp_buf и не пытается восстановить PAC-штампованные LR), но наблюдений нет.

---

## 8. Проверка на устройстве

Проверить работоспособность скелета без устройства тяжело, но ряд шагов возможен и на хосте.

| Что проверяем | Как |
|---|---|
| Логика nil-guard'ов | Юнит-тесты на хосте для функций, вынесенных в отдельный `.c`/`.m` (без `%hook`) — компилируем `clang -c` и запускаем на macOS/Linux |
| Корректность вызова `objc_getClass` / `object_getClass` | На устройстве — через `log stream --predicate 'process == "<target>"'` и проверка, что в журнале нет записей о nil-returns |
| Фактическое срабатывание signal handler | На устройстве — спровоцировать SIGSEGV контролируемо: `volatile int *p = (int *)0x4; *p = 1;` в тестовом хуке; затем прочитать `/tmp/FSTweak-sig.txt` (или путь из вашего скелета) |
| `NSSetUncaughtExceptionHandler` | На устройстве — спровоцировать `[NSNull length]`; должна появиться запись в `~/Documents/FSTweak/objc-*.txt` |
| `SA_RESETHAND` поведение | На устройстве — проверить, что после нашего отчёта всё равно появляется стандартный crash log в `~/Library/Logs/CrashReporter/` (путь — требует проверки на iOS 26.0.1) |

Такой тест-план важно пройти целиком при первой адаптации скелета в собственный твик, потому что как минимум путь к системному crash log на iOS 26 мог измениться по сравнению с 18.x: direct observation на 26.0.1 рекомендуется (не подтверждено по открытым источникам, что путь остался тем же).

---

## 8a. Воспроизводимый minimal crash для тестирования

Отладка скелета сводится к намеренно спровоцированным крашам каждого из поддерживаемых типов. Полный матрикс тест-кейсов:

| Тип | Как спровоцировать | Что должно появиться |
|---|---|---|
| SIGSEGV | `*(volatile int *)0x4 = 1;` | запись в `/tmp/FSTweak-sig.txt` с `pc=` в вашей dylib |
| SIGBUS | `*(volatile uint64_t *)0x1 = 0;` (невыровненная запись на некоторых mapping'ах) | то же, `strsignal` = "Bus error" |
| SIGILL | `__builtin_trap();` | то же, `strsignal` = "Illegal instruction" |
| SIGABRT | `abort();` | то же, трейс покажет `__abort` в libsystem_c |
| SIGFPE | `volatile int d = 0; return 1/d;` на целочисленном делении (на arm64 — не трапается, вместо этого UB; реально SIGFPE редко случается) | возможно, не воспроизведётся |
| `NSException` | `@throw [NSException exceptionWithName:@"T" reason:@"test" userInfo:nil];` в хуке | запись в `~/Documents/FSTweak/objc-*.txt` |
| KVC-mismatch | `[[NSNull null] arrayForKey:@"x"];` | то же, но с `NSInvalidArgumentException` |
| Swift fatalError | вызов Swift-API, который гарантированно трапается | SIGILL с pc в `libswiftCore.dylib` |

Из этой матрицы видно, что покрытие signal handler'ом составляет 6 из 8 категорий — две нижние строки принципиально ObjC/Swift. Проверив все эти сценарии вручную (конечно, под отдельным «testing-only» билдом твика, не распространяемым пользователям), вы получаете уверенность, что скелет работает ровно так, как объявлено.

Проверка путей и ротации лога (которую мы в скелете не закладывали специально, чтобы не раздувать) требует отдельной утилиты в duty-cycle целевого приложения: при каждом старте процесса подчистить `~/Documents/FSTweak/`, если накопилось больше N файлов. Это — задача для v2 скелета.

Автоматизировать прогон можно через простой скрипт в `tests/` директории твика (не на устройстве, а при каждом билде). Пример:

```bash
#!/bin/bash
# tests/provoke-signals.sh
# Запускается на устройстве через SSH; собирает отчёты.

set -e
PID=$(pgrep -f "${TARGET_APP:-MyTarget}")
if [ -z "$PID" ]; then
    echo "target not running"
    exit 1
fi

# Отправляем SIGSEGV через kill (не ловится нашим обработчиком, но
# показывает, что процесс отреагировал на сигнал извне).
echo "sending SIGUSR1 (dummy) to pid=$PID"
kill -USR1 $PID

sleep 1

# Копируем отчёты.
scp -r mobile@$DEVICE_IP:/var/mobile/Containers/Data/Application/*/Documents/FSTweak/ ./reports/
ls -la ./reports/
```

Такой скрипт запускается на хосте, устройство должно быть доступно по SSH (`openssh` ставится через RootHide `jbctl setup`). Если scp недоступен, можно использовать `ideviceinstaller` + `idevicecrashreport` для ящика crash log'ов, но тогда вы смотрите на Apple-овский crash log, а не на свой собственный. Проверить обе выдачи — полезная перекрёстная сверка: наш `pc=` из signal handler должен совпасть с `Thread 0 crashed` в Apple-овском log'е.

---

## 9. 26.0 vs 26.0.1

Различий в поведении `sigaction`, `NSSetUncaughtExceptionHandler`, `backtrace()` между 26.0 и 26.0.1 по открытым источникам (release notes 26.0.1) не выявлено. Dopamine в коде не делает развилки по 26.0.1 для crash-reporter'а — тот же `BaseBin/libroothider/src/crashreporter.m` используется без условий на `kCFCoreFoundationVersionNumber`. Таким образом, скелет из этой главы безопасно работает на обеих версиях.

---

## 10. Миграция из rootful-твика

Если ваш твик был написан под rootful-джейлбрейк (iOS 15/16) и теперь переносится на RootHide-окружение iOS 26, стоит обратить внимание на четыре отличия, которые касаются именно crash-safety:

1. **Путь для логов.** В rootful-окружении было нормально писать в `/var/mobile/Library/Logs/MyTweak/`. В RootHide такой путь требует префикса `/var/jb/<rand>` для сервис-уровневых логов, либо остаётся в sandbox-овом `~/Documents` для приложений. Используйте `JBROOT_PATH` (BaseBin/libjailbreak/src/jbroot.h:38-40) для сервисов и `NSHomeDirectory()` для приложений.
2. **PAC-поведение.** На A12/A13 (iOS 15/16) PAC-штампы уже были, но PPL-ограничения были мягче. На iOS 26 попытка исполнить PAC-«украденный» указатель почти всегда даёт SIGILL. Это значит, что skeleton старого твика без signal handler'а на 26.x упадёт тише и без диагностики. Добавление signal handler'а — обязательно для совместимости.
3. **Dopamine-овский crashreporter.** В rootful-окружении не было общего crashreporter'а, поставленного джейлбрейком. В RootHide — есть (BaseBin/libroothider/src/crashreporter.m), и ваш обработчик с ним сосуществует (см. раздел 6b). Старый твик, не ожидающий этого, мог молча затереть Dopamine'овский обработчик и лишить пользователя системного crash log'а. Правильный паттерн — цепочечный вариант из раздела 6b.
4. **TCC в iOS 26.** Запись в произвольные `/var/mobile/Documents/*` на iOS 26 для некоторых процессов может блокироваться TCC или sandbox-ограничениями, в отличие от 15/16, где доступ был свободным. Для скелета это означает: не предполагайте, что `/tmp/FSTweak-sig.txt` всегда записываемый, оставляйте fallback-открытие без создания файла.

Миграция, таким образом, не косметическая: это переход от «молча падающего» твика к «воспроизводимо падающему». Расходы: порядка 50 строк кода, несколько килобайт `.dylib`-размера, пренебрежимое время выполнения в `%ctor` (несколько `sigaction` + один `NSSetUncaughtExceptionHandler` — суммарно десятки микросекунд).

---

## 11. Диагностика по следам

Что делать с отчётами, которые скелет пишет? Два подхода:

- **Локальный анализ.** Вы вручную забираете файлы из `~/Documents/FSTweak/` через `scp` или TrollStore-style file manager, читаете их и соотносите с версией твика. Этот подход прост, но требует, чтобы пользователь-тестировщик имел физический доступ к устройству.
- **Автоматизированный upload.** Твик сам отправляет новые отчёты на HTTP-эндпойнт при следующем запуске (не в момент краша!). Для этого нужен `NSURLSession`, фоновый upload-task, retry-очередь. В скелет НЕ встроено сознательно: сетевой код сам имеет шансы на падение, и его отсутствие делает скелет минимальным. Но этот слой — следующий логичный шаг для production-твика.

При автоматизированном upload'е обязательно учитывайте приватность: отчёт содержит backtrace с адресами, которые могут раскрыть ASLR-slide и помочь атакующему. Публикуйте такие данные только с ведома пользователя и желательно под отдельным «debug build»-билдом твика, не в stable-канале.

---

## Выводы

Устойчивость твика — это дисциплина трёх независимых слоёв: nil/NULL-проверки на входе и выходе каждого хука, `@try/@catch` на горячих путях с глобальным `NSSetUncaughtExceptionHandler` как последняя ObjC-линия, и signal handler как последний C-рубеж, настроенный с `SA_SIGINFO|SA_RESETHAND` и строго async-signal-safe-телом. Образец всех трёх идиом доступен прямо в репозитории Dopamine: `BaseBin/libroothider/src/crashreporter.m`, строки 358, 493, 506-611. Скелет из этой главы заимствует проверенные решения, но принципиально не маскирует падения: любой пойманный отказ сохраняется в sandbox-читаемом файле, и в случае сигнала процесс всё равно получает штатный `ReportCrash` для корректной Apple-овской телеметрии.

Главный архитектурный вывод: защита должна начинаться на самом верхнем слое (`nil`-проверки — дешево и предсказуемо) и опускаться только в ту глубину, которую вы готовы диагностировать. Signal handler — не кнопка «сделать всё хорошо», а «гарантировать, что я узнаю первым». Эта разница — между твиком, который падает молча на устройстве пользователя, и твиком, который присылает вам читаемый файл в `~/Documents/FSTweak/`.

Второй архитектурный вывод, менее очевидный: порядок инициализации в `%ctor` важнее, чем качество кода внутри слоёв. Signal handler должен встать первым, до любой логики, которая способна упасть. ObjC uncaught handler — вторым. `%init` — последним. Нарушение порядка превращает все три слоя в декорацию: падение в `%init` до установки обработчиков останется таким же молчаливым, каким было бы без них вовсе. Из того же порядка следует правило «идемпотентной установки»: ни один из слоёв не должен сломаться при повторной инициализации, потому что в сложных сценариях (например, при `respring` SpringBoard без выгрузки некоторых dylib) `%ctor` может быть вызван неожиданно рано или в неожиданном порядке относительно других твиков.

Третий вывод — про цепочечное взаимодействие. На устройстве с установленным Dopamine ваш твик — не единственный, кто ставит crash-обработчики: это уже делает сам `libroothider`. Правильный паттерн — сохранять и вызывать предыдущий обработчик (как показано в разделах 4 и 6b), а не замещать его. Это и этичное поведение в многоуровневой экосистеме, и способ не лишить пользователя системного crash log'а в `~/Library/Logs/CrashReporter/`.

## Что дальше

Следующий кейс (vol3-08) показывает, как переносить этот скелет в контекст демона (`backboardd`-инъекция для background-audio keep-alive): там часть идиом меняется — писать в `~/Documents` нельзя, нужны пути через `JBROOT_PATH` (`BaseBin/libjailbreak/src/jbroot.h:38-40`), а также `fs_install_signal_handlers` имеет смысл отложить до окончания load-ордера системного фреймворка (иначе наш обработчик перехватит сигналы, которые Apple-овский код ждёт сам). Отдельная глава «cheatsheets/signal-safety-for-tweaks.md» (Том III, расширение) соберёт полный список async-signal-safe функций Darwin для быстрой проверки без заглядывания в man.

Параллельно рекомендуется ознакомиться с кейсом 1 (базовая архитектура файл-логирования: тонкая Logos-прослойка плюс testable-C-ядро) и кейсом 5 (`%group`-условная инициализация): скелет из этой главы ортогонален обоим и склеивается с ними без изменений — slot для `%group` просто добавляется в тот же `%ctor` между `fs_install_signal_handlers()` и `%init`, слот для core-библиотеки живёт в отдельных `.c`-файлах под `core/` и покрывается юнит-тестами на хосте так, как это показано в кейсе 1. Таким образом, устойчивость — не «ещё одна функция» твика, а инвариант, выравниваемый по всем кейсам тома.

Для будущих глав III тома этот скелет используется как база: кейс 8 (background audio keep-alive в `backboardd`) наследует структуру `%ctor`-инициализации, кейс 9 (редактор APNs push в `apsd`) добавляет сюда цепочечный вариант обработчика сигналов, кейс 10 (StoreKit receipt replay) показывает, как «безопасно отключить» твик в детектированных окружениях (управляемое устройство, demo-mode) ровно через те же nil-guard'ы с ранним возвратом. Три главы атласа демонов (vol3-16, vol3-20, vol3-24) явно ссылаются на эту главу как на предпосылку: прежде чем лезть в `backboardd`, `apsd` или `amfid`, читатель должен уметь написать твик, который не положит свой целевой демон при первой же нулевой ссылке. Отдельная глава чит-листа `cheatsheets/signal-safety-for-tweaks.md` собирает async-signal-safe функции Darwin в табличной форме, чтобы по этой справке можно было быстро верифицировать свой signal handler, не читая `signal(3)` целиком.

Наконец, стоит зафиксировать меру применимости. Скелет рассчитан на твик, который инъектируется в приложение или в системный процесс через MobileSubstrate / libhooker в контексте Dopamine RootHide на iOS 26.0 / 26.0.1 (A12/A13). Для kernel-space-патчей, инструментов на базе `libkrw`, расширений ядра — он неприменим целиком (`sigaction` в ядерном контексте не существует), но принципы «локализовать и записать, не маскируя» переносятся и туда — в формате `panic_action` обёрток и explicit assertion logging. Это тема отдельного кейса; здесь мы останавливаемся на user-space, где у нас есть `sigaction`, `NSSetUncaughtExceptionHandler` и `backtrace()` как рабочий инструментарий.
