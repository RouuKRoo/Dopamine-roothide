# Pointer Authentication (PAC) — зачем, как, когда это важно для твика

Эта глава даёт автору твика ровно столько Pointer Authentication, сколько нужно, чтобы:
а) писать код, который не падает в SIGSEGV при первом же вызове хука на arm64e-процессе;
б) не писать PAC-код там, где он лишний (arm64-процесс на A13 — обычный пользовательский случай);
в) видеть разницу между «мой чип A13 аппаратно умеет PACIA» и «мой целевой бинарник физически содержит инструкции PACIA».
Внутри Dopamine roothide вся работа с PAC сосредоточена в нескольких местах: `BaseBin/systemhook/src/main.c:68-87`, `BaseBin/systemhook/src/roothider_main.c:433-452`, `BaseBin/libroothider/src/dyld_patch.m:82-84/725-756`. Эти фрагменты — именно те, которые делают «вот тебе `basebin.tar`, он работает и на A12/A13 (PAC-aware слайс arm64e), и на любом arm64-устройстве без PAC в юзерспейсе». Их мы и будем препарировать.

Все цифровые значения ключей, salt, дискриминаторов, которые упомянуты ниже, взяты из исходников Dopamine 3.0.12 roothide; все, что не удалось подтвердить через `grep` или `otool`, помечено отдельно. Для iOS 26.0 и 26.0.1 различия в самой логике PAC не выявлены — это одна и та же ARM-архитектура, одно и то же ядро семейства XNU и один и тот же набор ключей; различия возможны только в конкретных PAC-salt и смещениях символов в `dyld_shared_cache`, и о них в коде репозитория прямых указаний нет (кроме `0x839D`, `0x63FA`, `0xBF31`, `0xB1B6`, `0xD48C`, `0xD2A5` — см. ниже).

## 3.1. ARMv8.3-A Pointer Authentication: инструкции, ключи, соль

ARMv8.3-A вводит в архитектуру аппаратную подпись указателей. Идея на одном предложении: верхние, не используемые для адреса, биты 64-битного указателя заполняются подписью — усечённым (truncated) MAC от самого указателя, посчитанным по одному из четырёх секретных ключей, с добавлением «соли» (modifier). При любой попытке разыменовать такой указатель (вызов функции, чтение данных) выполнение проходит через AUT* инструкцию: она проверяет подпись и либо восстанавливает корректный указатель (если подпись сошлась), либо замещает его на заведомо невалидный, который гарантированно упадёт при первом же использовании. Любой атакующий, кто перезаписал указатель и не владеет ключом, не сможет подделать подпись: MAC использует криптографически стойкий QARMA.

Чтобы автор твика видел не абстрактную картинку, а конкретную схему памяти, ниже — то, как раскладывается 64-битный виртуальный адрес на arm64e-процессе iOS 26 на A13 (строго говоря, это конфигурация без TBI в userspace; TBI включается селективно):

```
63                              56 55                               48 47                                 0
+---------------------------------+----------------------------------+------------------------------------+
|          PAC[7..0]              |            PAC[14..8]            |              VA[47..0]             |
|   верхняя часть подписи         |   нижняя часть подписи           |   собственно адрес (48 бит)        |
+---------------------------------+----------------------------------+------------------------------------+
```

Длина `PAC`-поля — порядка 15 бит на 48-битной адресации с TBI=off (точная ширина зависит от конфигурации TCR, но на iOS в userspace устойчиво укладывается в этот размер). Это и есть «ширина замка»: любой атакующий, не знающий ключа, при однократной попытке угадать подпись имеет шанс один из ~32 тыс. — что при наличии любой проверки на корректность указателя (а AUT* проверяет каждый раз) делает brute-force практически бесполезным. Больше того, если AUT* не прошла, в современных ARMv8.3+ реализациях она не просто ставит невалидный бит, а меняет указатель на такой, у которого верхние биты специально выставлены в конфликтующий паттерн: при разыменовании это гарантированно даст fault в MMU, а не «тихо прочитает что-то не то» (в ARMv8.6-A этот режим называется FEAT_FPAC и даёт fault уже на самой AUT*).

Это критически важно для автора твика: **ошибиться в modifier на один бит нельзя, промежуточных состояний нет**. Либо подпись сошлась и `BRAA` аккуратно перешёл по адресу, либо не сошлась и процесс свалится ещё до того, как попадёт на целевой код. Отладка такой ошибки без устройства — ад: в лог вы увидите только `EXC_BAD_ACCESS` на адресе вида `0x5a4b000000104abc` (где `0x5a4b` — паттерн-сигнализатор о сломанной PAC). Поэтому единственный способ правильно подписывать — брать подписанный указатель, который сделал за вас dyld/компилятор, и переподписывать его через `ptrauth_auth_and_resign`.

### 3.1.1. Четыре ключа: IA, IB, DA, DB

ARMv8.3-A определяет четыре независимых PAC-ключа. Clang/LLVM называет их через свои intrinsic-константы: `ptrauth_key_asia`, `ptrauth_key_asib`, `ptrauth_key_asda`, `ptrauth_key_asdb`. Для платформы Apple эти константы псевдонимят через более читаемые имена, которые регулярно встречаются в исходниках Dopamine:

| Ключ | Clang-псевдоним Darwin | Назначение по ARM-семантике | Где встречается в коде Dopamine |
|---|---|---|---|
| IA | `ptrauth_key_process_independent_code` (`ptrauth_key_asia`) | Код, подпись не зависит от процесса (разделяемые адреса — shared cache) | `BaseBin/libroothider/src/dyld_patch.m:737,755`; `BaseBin/systemhook/src/main.c:80,81`; `BaseBin/systemhook/src/roothider_main.c:445,446` |
| IB | `ptrauth_key_process_dependent_code` (`ptrauth_key_asib`) | Код, подпись привязана к процессу — классический сценарий: возвращаемые адреса в стеке (`LR` на входе в функцию) | в коде Dopamine прямо не используется, но косвенно через `__darwin_arm_thread_state64_*` при `thread_state_t` |
| DA | `ptrauth_key_process_independent_data` (`ptrauth_key_asda`) | Данные, межпроцессный ключ. Используется, например, для подписи указателей внутри dyld4-таблицы API | `BaseBin/systemhook/src/main.c:73`; `BaseBin/systemhook/src/roothider_main.c:438` |
| DB | `ptrauth_key_process_dependent_data` (`ptrauth_key_asdb`) | Данные, подпись привязана к процессу | в коде Dopamine прямо не встречается (по `grep` — не нашёл) |

Отдельно есть ключ GA — только для general-purpose MAC через инструкцию `PACGA`. Он не используется для подписи указателей и в коде Dopamine, судя по `grep ptrauth_key_asg`, не фигурирует.

### 3.1.2. Инструкции PAC*/AUT*/XPAC*

Для каждого из четырёх указательных ключей ARMv8.3-A определяет симметричную пару инструкций — подписать и проверить:

| Операция | Код (IA) | Код (IB) | Данные (DA) | Данные (DB) |
|---|---|---|---|---|
| Подписать с modifier в регистре | `PACIA Xd,Xm` | `PACIB Xd,Xm` | `PACDA Xd,Xm` | `PACDB Xd,Xm` |
| Подписать с modifier = `SP` | `PACIASP` | `PACIBSP` | — | — |
| Подписать с modifier = 0 | `PACIAZ` | `PACIBZ` | `PACDZA` | `PACDZB` |
| Проверить и снять подпись | `AUTIA Xd,Xm` | `AUTIB Xd,Xm` | `AUTDA Xd,Xm` | `AUTDB Xd,Xm` |
| ...с modifier = `SP` | `AUTIASP` | `AUTIBSP` | — | — |
| ...с modifier = 0 | `AUTIAZ` | `AUTIBZ` | `AUTDZA` | `AUTDZB` |
| Снять подпись без проверки | `XPACI Xd` | `XPACI Xd` | `XPACD Xd` | `XPACD Xd` |

Плюс комбинированные инструкции, которые объединяют AUT*/BRANCH или AUT*/LDR в одну операцию (`BRAA`, `BRAAZ`, `BLRAA`, `BLRAAZ`, `LDRAA`, `LDRAB`) — именно их любит генерировать LLVM на arm64e, когда вы вызываете через подписанный указатель.

Для автора твика на A13 важно вот что: когда срез процесса — `arm64` (CPU_SUBTYPE_ARM64_ALL), ядро на этапе загрузки «не включает» эти инструкции в семантическом смысле. Физически на A13 (ARMv8.3-A) они всегда исполняются, но в arm64-ABI PAC-инструкции в бинарник не генерируются компилятором: там, где arm64e-слайс поставил бы `BRAA`, arm64-слайс ставит обычный `BLR`. Чтобы хукнуть такой `BLR`, PAC не нужен в принципе.

Когда срез процесса — `arm64e` (CPU_SUBTYPE_ARM64E), в его `__TEXT` реально есть `PACIA`/`BRAA`/`AUTIA` и т.д., и вставить свой хук «в лоб» уже нельзя — указатель, который вы положите на место целевого, рухнет на первом же `AUTIA`.

#### Какие комбинированные инструкции встречает автор твика

На arm64e-слайсе BaseBin'овских дылиб `otool -tV` на горячих точках хуков показывает преимущественно четыре комбинированных формы:

- `BRAA Xn, Xm` / `BRAAZ Xn` — ветвление по подписанному указателю, modifier в `Xm` (или 0 для `Z`-варианта), ключ IA. Это то, во что компилирует Clang вызов через подписанный `void (*)(void)`-указатель.
- `BLRAA Xn, Xm` / `BLRAAZ Xn` — то же, но `BL` (сохраняется адрес возврата в `LR`). Это рутина вызова виртуальной функции.
- `LDRAA Xt, [Xn, #imm]!` / `LDRAB Xt, [Xn, #imm]!` — загрузить по подписанному базовому указателю (`Xn` — указатель с подписью), с pre-indexed смещением. Применяется, например, при чтении полей структуры через подписанный `self`.
- `RETAA` / `RETAB` — возврат из функции с проверкой подписи LR, использующей ключ IB (это как раз «сохранённый LR на входе — подписать PACIBSP, при возврате — AUTIBSP»).

Для автора твика это означает, что подавляющее большинство PAC-кода, с которым он столкнётся, — операции над IA-подписями кодовых указателей (`BRAA`/`BLRAA`) и, в отдельных случаях, над DA-подписями указателей на структуры (`LDRAA`). PACIBSP/RETAB — забота компилятора; её вам обычно не трогать, кроме случая, когда вы меняете `arm_thread_state64_t` (см. `BaseBin/libroothider/src/dyld_patch.m:710-770` и `BaseBin/libroothider/src/crashreporter.m:240,536`).

### 3.1.3. QARMA, разрядность подписи и biased address

QARMA-64 — это PRF, который берёт 64-битный указатель и 64-битный modifier, и вместе с секретным ключом (хранится в системных регистрах `APIAKey_EL1` и т.п., недоступных пользователю) выдаёт короткий тег. Этим тегом заполняются «свободные верхние биты» виртуального адреса (на Apple A-серии в userspace — обычно биты 47..63, с учётом TBI и 48-битной адресации). Длина подписи — не 16 бит на старших, а меньше: биты 56..63 могут быть зарезервированы под TBI/селектор ядра/пользователя. Это важно, потому что любая «попытка освободить» верхние биты для своих нужд разрушает подпись и уводит процесс в SIGBUS/SEGV на следующей `AUT*`.

Для автора твика эта криптография — чёрный ящик, но из неё вытекает практический факт, который видно в коде: **снять PAC без ключа нельзя**, можно только либо иметь на вход уже подписанный пойнтер и попросить компилятор его «ресигнировать» (`ptrauth_auth_and_resign`), либо получить от того же процесса валидный подписанный указатель и переподписать на другой ключ/salt (та же `ptrauth_auth_and_resign`), либо получить где-то обход «из коробки» (например, dyld даёт вам `*gDyld` — там указатель на таблицу API подписан по известной паре ключ+salt, и вы его авторизуете) — но подделать подпись «с нуля» вы не можете. На A13 PAC в userspace Apple по умолчанию выключен, но в arm64e-дылде и в arm64e-коде, собранном Apple, он включён, и это накладывает ограничения на жизнь любого PAC-aware хука.

## 3.2. Modifier (salt): почему тот же указатель с разным modifier даёт разную подпись

Modifier (его же часто называют salt или discriminator) — это 64-битная величина, которая подмешивается в вход QARMA вместе с указателем. Два указателя с одинаковым значением, подписанные одним и тем же ключом, но с **разными modifier**, получают разные подписи и, как следствие, разные старшие биты. Это — главная защитная идея PAC поверх «просто MAC от указателя»: если у вас есть подписанный указатель-функция `foo` в слоте GOT, атакующий не может переподставить туда свой подписанный `bar` (пусть даже валидно подписанный тем же ключом) — модификатор у вашего слота привязан к адресу слота, а у `bar` modifier другой; подпись не сойдётся.

### 3.2.1. Address diversity и constant diversity

На практике Apple сочиняет modifier из двух частей:

- **address diversity** — старшие 48 бит modifier — это адрес места, где указатель хранится (т.е. `&slot`), либо часть этого адреса. Так подпись становится «местоспецифичной»: тот же `foo` в двух разных слотах подпишется по-разному.
- **constant diversity** — младшие 16 бит modifier (или другие биты, в зависимости от схемы) — это «салт», фиксированная 16-битная константа, вычисленная компилятором/генерируемая вручную. Для функций C++ она часто — хэш от имени символа; для struct-полей — хэш от имени поля.

В коде Dopamine обе части явно сконструированы руками. Смотрите `BaseBin/systemhook/src/main.c:68-87`:

```c
int dyld_hook_routine(void **dyld, int idx, void *hook, void **orig, uint16_t pacSalt)
{
    if (!dyld) return -1;

    uint64_t dyldPacDiversifier = ((uint64_t)dyld & ~(0xFFFFull << 48)) | (0x63FAull << 48);
    void **dyldFuncPtrs = ptrauth_auth_data(*dyld, ptrauth_key_process_independent_data, dyldPacDiversifier);
    if (!dyldFuncPtrs) return -1;

    if (vm_protect(mach_task_self_, (mach_vm_address_t)&dyldFuncPtrs[idx], sizeof(void *), false, VM_PROT_READ | VM_PROT_WRITE) == 0) {
        uint64_t location = (uint64_t)&dyldFuncPtrs[idx];
        uint64_t pacDiversifier = (location & ~(0xFFFFull << 48)) | ((uint64_t)pacSalt << 48);

        *orig = ptrauth_auth_and_resign(dyldFuncPtrs[idx], ptrauth_key_process_independent_code, pacDiversifier, ptrauth_key_function_pointer, 0);
        dyldFuncPtrs[idx] = ptrauth_auth_and_resign(hook, ptrauth_key_function_pointer, 0, ptrauth_key_process_independent_code, pacDiversifier);
        vm_protect(mach_task_self_, (mach_vm_address_t)&dyldFuncPtrs[idx], sizeof(void *), false, VM_PROT_READ);
        return 0;
    }

    return -1;
}
```

Разберём существенные строки.

1. `dyldPacDiversifier = ((uint64_t)dyld & ~(0xFFFFull << 48)) | (0x63FAull << 48)` — modifier для авторизации указателя `*dyld` (это указатель на таблицу функций dyld4). Нижние 48 бит — адрес `dyld` (то есть `&gDyld` в самой libdyld), верхние 16 бит — фиксированная константа `0x63FA`. Это и есть сочетание address diversity + constant diversity, но здесь оно жёстко прошито вручную, потому что `*gDyld` — это указатель на структуру, подпись которого ставит сам dyld при инициализации, и Apple-компилятор знает ровно те же два поля: адрес слота в libdyld и salt `0x63FA`.
2. `ptrauth_auth_data(*dyld, ptrauth_key_process_independent_data, dyldPacDiversifier)` — снимает подпись с ключа DA (`ptrauth_key_process_independent_data`) и возвращает «чистый» указатель `dyldFuncPtrs`. Если бы modifier сходился не по DA, а, скажем, по DB — вы бы уже здесь получили невалидный указатель и падение.
3. `location = (uint64_t)&dyldFuncPtrs[idx]` — адрес слота таблицы, в котором лежит подписанный указатель на функцию `dyld4::...`.
4. `pacDiversifier = (location & ~(0xFFFFull << 48)) | ((uint64_t)pacSalt << 48)` — для каждого слота таблицы собственный modifier: нижние 48 бит — адрес слота, верхние 16 бит — salt. Salt — свой у каждого API dyld4; в коде встречаются (`BaseBin/systemhook/src/main.c:466,470`, `BaseBin/systemhook/src/roothider_main.c:459-470`):

| Salt | Назначение | Файл |
|---|---|---|
| `0x839D` | `dyld_dlsym` | `BaseBin/systemhook/src/main.c:466,470` |
| `0xBF31` | `dyld_dlopen` | `BaseBin/systemhook/src/roothider_main.c:459,467` |
| `0xB1B6` | `dyld_dlopen_preflight` | `BaseBin/systemhook/src/roothider_main.c:460,468` |
| `0xD48C` | `dyld_dlopen_from` | `BaseBin/systemhook/src/roothider_main.c:461,469` |
| `0xD2A5` | `dyld_dlopen_audited` | `BaseBin/systemhook/src/roothider_main.c:462,470` |
| `0x63FA` | подпись самой таблицы `*gDyld`/`*gAPIs` (DA-ключ) | `BaseBin/systemhook/src/main.c:72`, `BaseBin/systemhook/src/roothider_main.c:437` |

**Непроверено:** эти salt-константы верны для билдов dyld4, с которыми встречается Dopamine 3.0.12 на iOS 15–18 и iOS 26.0–26.0.1. Прямого доказательства, что для сборки dyld из iOS 26.0.1 эти же salt сохранились, в коде репозитория нет (кроме того, что `BaseBin/dyldhook/Makefile:23-48` явно разводит сборку по трём ветвям `IOS=15/16/18`, и это намекает, что Apple могла и поменять). Для проверки нужно взять конкретный `/usr/lib/dyld` из iOS 26.0.1 и подсмотреть salt в месте, где `gDyld` инициализируется — через `otool -tV -arch arm64e /usr/lib/dyld` в районе `_main` dyld.

### 3.2.2. Зачем вообще перекладывать с одного ключа на другой

Обратите внимание на разные ключи в последних двух строках сниппета:

- `*orig = ptrauth_auth_and_resign(dyldFuncPtrs[idx], ptrauth_key_process_independent_code, pacDiversifier, ptrauth_key_function_pointer, 0);`
- `dyldFuncPtrs[idx] = ptrauth_auth_and_resign(hook, ptrauth_key_function_pointer, 0, ptrauth_key_process_independent_code, pacDiversifier);`

Клиентская сторона в systemhook хранит `*orig` под ключом `ptrauth_key_function_pointer` (`ptrauth_key_asia` по умолчанию для Apple-ABI; это IA с «пустым» modifier=0). Поэтому вызов `(*orig)(…)` в теле хука скомпилируется в обычный `BLRAA` с modifier=0 — без возни со специфическим pacDiversifier dyld. А в саму таблицу `*gDyld` кладётся «нормально подписанный» указатель на наш хук — с нужным modifier (адрес слота + salt) и под ключом `ptrauth_key_process_independent_code` (IA, DA-ключ не подошёл бы: в таблице указатели на код). Это каноническая «ресигнация» между двумя ABI: внешним (как мы сами привыкли вызывать) и внутренним (как dyld4 ожидает увидеть у себя в таблице).

### 3.2.3. Как найти salt для функции, которой нет в коде Dopamine

Вопрос «где же взять этот `0xBF31`, если я хочу хукнуть новую функцию dyld4, которой ещё нет в systemhook?» встаёт перед каждым, кто расширяет Dopamine. Прямого правила «берите N-й хэш от имени» нет; salt — это 16-битное значение, которое Apple-компилятор выставил при генерации `gDyld`-инициализатора. Единственный надёжный источник — реверс самого dyld.

Алгоритм добычи (не подтверждено в коде Dopamine, предлагается автором этой главы):

1. Извлечь `/usr/lib/dyld` из `dyld_shared_cache` устройства — на хост-Mac с помощью `dyld-shared-cache-util -extract`.
2. Открыть в Hopper/IDA, найти `__ZN5dyld45gDyldE` (или `__ZN5dyld45gAPIsE` на iOS 18.4+).
3. Найти место инициализации таблицы функций — вы увидите последовательность вроде:

   ```
   adrp  x8, _dyld_dlopen@PAGE
   add   x8, x8, _dyld_dlopen@PAGEOFF
   mov   x9, #0xBF31          ; salt
   movk  x9, #0xADDR, lsl #16 ; address diversity
   pacia x8, x9
   str   x8, [x0, #0x70]      ; <-- слот №14 (0x70/8=14) в таблице
   ```

4. Второй операнд `pacia` в `mov x9, #imm` — это и есть ваш salt. Третий — address diversity, но она автоматически получится из `(uint64_t)&dyldFuncPtrs[idx]` в вашем хуке — её вручную не копировать, компилятор dyld её поставил из адреса слота в своей таблице, а не из какого-то «особого» значения.

5. Индекс (14, 18, 97, 98 — для `/usr/lib/dyld` iOS <18.4; 13, 17, 92, 93 — для iOS 18.4+ и, по всей вероятности, iOS 26.0/26.0.1) берётся из `str [x0, #offset]/8`.

Важно: один и тот же salt, по-видимому, Apple сохраняет между мажорными версиями dyld4 (значения `0xBF31`, `0xB1B6`, `0xD48C`, `0xD2A5` в коде Dopamine не менялись при переходе с iOS 15 на iOS 18, судя по коду — там нет версионных if'ов для этих констант). Но «по-видимому» ≠ «подтверждено». Это именно тот пункт, который автору твика на iOS 26 стоит перепроверить на устройстве лично.

## 3.3. ptrauth_* intrinsics в Clang

Clang на arm64e выдаёт набор intrinsic'ов и макросов, которые ровно отражают инструкции PAC*/AUT* и позволяют писать код, не прибегая к inline-ассемблеру. Все они доступны при `#include <ptrauth.h>`. Для автора твика на A13 важны пять:

```c
// 1) Подписать "сырой" указатель, которому вы доверяете. На arm64 — no-op (возвращает p).
void *ptrauth_sign_unauthenticated(void *p, ptrauth_key key, ptrauth_extra_data_t data);

// 2) Проверить подпись и сразу переподписать на другой ключ/salt.
//    На arm64 — no-op.
void *ptrauth_auth_and_resign(void *p,
                              ptrauth_key old_key, ptrauth_extra_data_t old_data,
                              ptrauth_key new_key, ptrauth_extra_data_t new_data);

// 3) Авторизовать как data-указатель (XPAC/AUTD* логика).
void *ptrauth_auth_data(void *p, ptrauth_key key, ptrauth_extra_data_t data);

// 4) Срезать подпись без проверки (XPAC).
void *ptrauth_strip(void *p, ptrauth_key key);

// 5) То же, что и 1, но builtin-псевдоним; работает и на arm64, где всегда возвращает p.
void *__builtin_ptrauth_sign_unauthenticated(void *p, int key, uintptr_t data);
```

Ключи задаются литералами `ptrauth_key_asia`, `ptrauth_key_asib`, `ptrauth_key_asda`, `ptrauth_key_asdb`, или их Apple-псевдонимами, которые мы уже перечислили. Для автора твика важна одна тонкость: при сборке arm64-слайса все эти intrinsic'ы — просто no-op, они возвращают указатель как есть. Это означает:

- Писать общий (не #ifdef'ованный) код, который работает «и там, и там», можно: на arm64 вызовы выродятся в проброс параметра, на arm64e — в реальные `PAC*`/`AUT*` инструкции.
- Но **писать операции, которые имеют смысл только на arm64e**, под `#ifdef __arm64e__` всё равно обязательно, если результат на arm64 нарушает инвариант. Пример — `BaseBin/libroothider/src/dyld_patch.m:82-84`:

```objc
#ifdef __arm64e__
    getAMFI = (uint64_t)__builtin_ptrauth_sign_unauthenticated((void*)getAMFI, 0, 0);
#endif
```

Здесь на arm64 подписывать смысла нет (мы ничего не вызовем через PAC), а на arm64e без подписи код физически не исполнится — там, где это значение попадёт в `BRAA`, авторизация не пройдёт и мы получим SIGSEGV. Поэтому `#ifdef`.

Ещё один показательный случай — `BaseBin/libroothider/src/dyld_patch.m:725-756`, фрагмент, который чинит thread state на свежем запущенном процессе:

```c
arm_thread_state64_t strippedState = threadState; /* some process such as WebContent used a different pac key
 cause we can't auth-and-resign it with __darwin_arm_thread_state64_get_* in current processs (crash) , so just strip all first */
__darwin_arm_thread_state64_ptrauth_strip(strippedState);
...
#ifdef __arm64e__
    void* resignedPC = ptrauth_sign_unauthenticated((void*)strippedPC, ptrauth_key_process_independent_code, 0);
    __darwin_arm_thread_state64_set_pc_fptr(threadState, resignedPC);
```

Комментарий автора прямо называет грабли: `WebContent` подписывает регистры тред-стейта **другим PAC-ключом** (кастомная политика ядра для процессов с entitlement `com.apple.security.cs.allow-non-arm64e-code` или на web-rendering, точного пути я по grep не нашёл). Поэтому `__darwin_arm_thread_state64_get_pc` (который на arm64e под капотом делает `AUTIA`) падает — modifier у чужого процесса не сойдётся. Решение: сначала `_ptrauth_strip` (XPAC — выкусил подпись), работай с «голым» адресом, потом подписывай обратно с modifier=0 и ключом IA.

### 3.3.0. Что генерирует Clang для `ptrauth_auth_and_resign` на arm64e

Полезно один раз увидеть, во что компилятор превращает ваш `ptrauth_auth_and_resign`. Для вызова вида:

```c
void *x = ptrauth_auth_and_resign(p,
                                  ptrauth_key_process_independent_code, old_mod,
                                  ptrauth_key_function_pointer, 0);
```

на `-arch arm64e -O2` Clang сгенерирует последовательность, близкую к такой:

```asm
    ; x0 = p (подписанный под IA + old_mod)
    ; x1 = old_mod
    autia    x0, x1          ; снять подпись IA с modifier=old_mod → "голый" указатель в x0
                             ; (если не сошлось — x0 заражён pattern-ом, падение на следующем BLR/LDR)
    mov      x1, xzr         ; новый modifier = 0
    pacia    x0, x1          ; подписать IA с modifier=0 (т.е. ptrauth_key_function_pointer)
    ; x0 = результирующий подписанный указатель
```

На `-arch arm64` этот же код скомпилируется в тривиальное:

```asm
    ; x0 = p
    ; <ничего>
    ; x0 возвращается без изменений
```

То есть `ptrauth_auth_and_resign` на arm64 — чистый no-op. Это важно: автор твика может писать код «одной веткой» для обоих слайсов, и на arm64 он даёт правильное поведение «нет подписи — передай указатель как есть». Единственное, что нужно помнить: **результат на arm64 — не подписан**, и если потом этот указатель вы передадите в arm64e-API, тот арм64e-API применит AUT*, и упадёт. На практике это означает — либо весь путь использования `p` живёт в одном слайсе (вызов остаётся внутри), либо вы заранее проверяете `__arm64e__` и адаптируете код.

### 3.3.1. Разбор `dyld_hook_routine` построчно

Ещё раз посмотрим на вариант из `BaseBin/systemhook/src/main.c:68-87` в целом, уже с пониманием intrinsic'ов, и сверим с mermaid-схемой.

```mermaid
flowchart LR
    A[dyld: void **<br/>указатель на *gDyld] -->|ptrauth_auth_data<br/>DA, modifier=(dyld &~0xFFFF&lt;&lt;48) | 0x63FA&lt;&lt;48| B[dyldFuncPtrs: void **<br/>таблица API dyld4]
    B --> C[dyldFuncPtrs idx<br/>подписанный ptr на оригинал]
    C -->|ptrauth_auth_and_resign<br/>IA, loc|salt → IA, 0| D[*orig: вызов через ptrauth_key_function_pointer]
    E[hook: наш код,<br/>IA, 0] -->|ptrauth_auth_and_resign<br/>IA, 0 → IA, loc|salt| F[dyldFuncPtrs idx<br/>подписанный ptr на наш хук]
```

Схема читается так. На входе — указатель `dyld` на таблицу-указателей, которая сама подписана. Шаг 1 — авторизовать «через DA», получить «чистый» `dyldFuncPtrs`. Шаг 2 — для нужного индекса `idx` снять с кодового указателя подпись с «dyld-ABI modifier» и переподписать на «клиент-ABI modifier» (ключ IA, salt=0), сохранить это в `*orig`, чтобы наш хук мог вызвать оригинал через обычный Apple-ABI PAC-вызов. Шаг 3 — зеркально: взять указатель на наш `hook` (подписанный компилятором как `ptrauth_key_function_pointer`, т.е. IA, salt=0), переподписать с «dyld-ABI modifier» (IA, адрес слота в старших 48 + конкретный salt в младших 16) и положить в тот же слот. Теперь dyld, когда будет делать `BRAA` по `dyldFuncPtrs[idx]`, получит нашу функцию и не грохнется на авторизации.

Два `vm_protect` вокруг — потому что страница с таблицей dyld4 по умолчанию `__DATA_CONST` или R/O, и без перевода в RW мы не можем даже легально записать подписанный указатель. На A13 это важный шаг — на A12/A13 PPL ещё относительно «мягкий», но попытка писать в `R-X`-страницу без `vm_protect` сходу даст `KERN_PROTECTION_FAILURE`.

### 3.3.2. Все `ptrauth_key_*` псевдонимы на платформе Apple

Clang на Darwin подкидывает несколько слоёв синонимов, и в коде Dopamine встречаются сразу несколько. Для ясности — полная таблица:

| Канонический литерал | Apple-псевдоним (`<ptrauth.h>`) | ARMv8.3 ключ | Типовое применение |
|---|---|---|---|
| `ptrauth_key_asia` | `ptrauth_key_process_independent_code` | IA | Указатели на функции в shared cache и dyld4-таблице |
| `ptrauth_key_asib` | `ptrauth_key_process_dependent_code` | IB | Return-адреса на стеке (`PACIBSP`/`RETAB`), подпись C++-vtable |
| `ptrauth_key_asda` | `ptrauth_key_process_independent_data` | DA | Указатели на struct в разделяемой памяти (`*gDyld`) |
| `ptrauth_key_asdb` | `ptrauth_key_process_dependent_data` | DB | Указатели на данные, зависящие от процесса; isa-pointer Objective-C |
| `ptrauth_key_function_pointer` | — | IA (modifier=0 по умолчанию для Apple-ABI function ptr) | «Голый» C-указатель на функцию с ABI-salt=0 |
| `ptrauth_key_return_address` | — | IB | Служебный ключ для сохранения `LR`, автоматически используется компилятором |
| `ptrauth_key_cxx_vtable_pointer` | — | DA (на Apple-ABI) | isa-pointer в Objective-C объектах и vtable |

`ptrauth_key_function_pointer` — отдельная история. Это не отдельный аппаратный ключ, а именованная политика «использовать IA с modifier=0». Клиентский код во всём BaseBin хранит `*orig`-указатели именно так, чтобы вызов через них компилировался в тривиальный `BRAA Xn, XZR` без возни. Это и объясняет, почему `ptrauth_auth_and_resign` в `dyld_hook_routine` (`BaseBin/systemhook/src/main.c:80`) принимает в качестве «нового ключа» именно `ptrauth_key_function_pointer, 0`: мы сознательно превращаем «dyld-ABI подписанный» указатель в «простой клиент-ABI подписанный».

### 3.3.3. Почему `ptrauth_strip` опасен

Intrinsic `ptrauth_strip(p, key)` компилируется в `XPAC*` — инструкцию, которая **выкусывает подпись без проверки**. Это выглядит соблазнительно: «я не знаю modifier, но мне нужен голый адрес — возьму strip и всё». Опасность в том, что:

1. На arm64 `XPAC` отсутствует как инструкция, и intrinsic — просто no-op; значит, если ваш «голый адрес» вы потом попытаетесь передать в arm64e-API, подпись не поставится, и `AUTIA` на том конце сломается.
2. На arm64e strip действительно работает, но из него нельзя получить обратно подписанный указатель без modifier. Если вы stripped-ули и забыли оригинальный modifier, подписаться корректно уже не получится.

Пример корректного «strip-then-sign» для случая, когда modifier неизвестен — `BaseBin/libroothider/src/dyld_patch.m:721-737`:

```c
arm_thread_state64_t strippedState = threadState;
__darwin_arm_thread_state64_ptrauth_strip(strippedState);
uint64_t strippedPC = (uint64_t)__darwin_arm_thread_state64_get_pc(strippedState);
...
#ifdef __arm64e__
    void* resignedPC = ptrauth_sign_unauthenticated((void*)strippedPC, ptrauth_key_process_independent_code, 0);
    __darwin_arm_thread_state64_set_pc_fptr(threadState, resignedPC);
```

Разбор: берётся thread state (который на arm64e подписан ключом IA + modifier, зависящий от процесса), стрипается, выделяется голый PC, затем подписывается обратно уже под нашим контролем — IA + modifier=0. Это работает, потому что мы сами потом передадим этот подписанный PC обратно в `thread_set_state`, и ядро на arm64e-устройстве авторизует его с теми же IA + modifier=0 (modifier=0 — валидная политика для контролируемого ядром thread state входа).

## 3.4. Почему A13 — не arm64e: указатели в userspace не аутентифицируются

Это та часть, которая у многих новичков вызывает когнитивный диссонанс. Чип A13 — ARMv8.3-A, у него есть все `PAC*`/`AUT*` инструкции в кремнии. Чип A14+ — тоже ARMv8.3-A. Но «PAC в юзерспейсе» в системе включается не аппаратно, а политикой ядра и ABI, которые разворачивает dyld. На A14+ Apple выпускает `arm64e`-ABI «включённым по умолчанию» и ядро, которое его гарантирует: пользовательские процессы на A14+ собираются как arm64e и исполняются с PAC. На A13 (и A12) Apple оставила arm64e **в статусе ABI preview**: сам /usr/lib/dyld на устройстве — arm64e (чтобы корректно работать с arm64e-содержимым dyld_shared_cache), но пользовательские приложения из App Store поставляются как arm64, и PAC в них не включается.

Это утверждение в коде Dopamine напрямую подтверждается:

- `Application/Dopamine/Jailbreak/DOEnvironmentManager.m:252-254` — если устройство — arm64e и `hw.cpufamily` == `CPUFAMILY_ARM_VORTEX_TEMPEST` (A12) или `CPUFAMILY_ARM_LIGHTNING_THUNDER` (A13), выводится строка `@"iOS 15.0 - 18.7.1, 26.0 - 26.0.1 (A12/A13, PPL)"`. То есть именно эта пара чипов — отдельно поддерживаемый класс «arm64e-устройство, но на нём ещё жив PPL (а не SPTM) и поддерживается широкий диапазон iOS».
- `Application/Dopamine/Exploits/{DarkSword,ClearSword}/Info.plist` описывают два kernel-эксплойта, у каждого `DPSupportedRanges` включает `Start=19.0, End=26.0.1` — то есть iOS 26.0/26.0.1 перекрыты.
- `Dopamine.md:5` явно пишет: *«A rootless semi-untethered jailbreak for iOS 15.0 - 17.3.1 (arm64e), iOS 15.0 - 18.7.1, 26.0 - 26.0.1 (A12/A13) and iOS 15.0 - 18.7.1 (arm64)»*. Три клинка поддержки: arm64e-устройства без PPL (устарело на iOS 17.4+), A12/A13-устройства (с PPL, вплоть до iOS 26.0.1) и arm64-устройства (без PAC, A9–A11).

Что из этого важно для автора твика на A13:

1. Пользовательское приложение (приложение из App Store, Safari — кроме WebContent, большинство системных демонов) **в arm64-слайсе** на A13. PAC в его `__TEXT` нет. Хуки через fishhook/litehook работают без `ptrauth_*`.
2. **dyld** на A13 iOS 26 **в arm64e-слайсе**. Даже если ваш твик — arm64, dyld, который его загружает, — arm64e. Это важно, потому что любая подпись в структурах `dyld4::APIs` (`*gDyld`, `*gAPIs`) выполняется ядром dyld для arm64e. Хотите хукнуть `dlopen` на уровне dyld4-таблицы (как делает Dopamine) — вам нужен arm64e-код для этого хука. Отсюда и `arm64 + arm64e` fat-сборка почти всего, что лежит в BaseBin.
3. Некоторые Apple-процессы на A13 iOS 26 — тоже arm64e. Это касается, в первую очередь, `launchd`, `WebContent`, и, по-видимому, значительной части системных демонов. Поведение `WebContent` с «другим PAC-ключом», которое чинит `dyld_patch.m:721-752`, — прямое тому доказательство.
4. Из-за того, что одно и то же `basebin.tar` должно работать и на arm64-only, и на A12/A13 arm64e, **весь системный код BaseBin собирается fat**. Пример компиляторной команды — `BaseBin/systemhook/Makefile:5`:

```make
CFLAGS = -I../.include -I./src -I../_external/modules/litehook/src \
         -isysroot $(shell xcrun --sdk iphoneos --show-sdk-path) \
         -arch arm64 -arch arm64e \
         -miphoneos-version-min=15.0 -install_name @loader_path/$(TARGET) \
         -Wno-deprecated-declarations -D BUILD_STANDALONE=$(BUILD_STANDALONE) -Os -moutline
```

То же `-arch arm64 -arch arm64e` — у `launchdhook` (`BaseBin/launchdhook/Makefile:5`), `libjailbreak` (`BaseBin/libjailbreak/Makefile:5`), `rootlesshooks` (`BaseBin/rootlesshooks/Makefile:5` — `ARCHS = arm64 arm64e`), `roothidehooks` (`BaseBin/roothidehooks/Makefile:8`), `hookd`, `jbctl`, `watchdoghook`, `bootstrapper`, `jailbreakd`, `Packages/libroot/Makefile`, `Packages/libkrw-provider/Makefile`.

Исключений два: `BaseBin/forkfix/Makefile:4` собирает только `arm64e` — потому что `forkfix` чинит специфичную для arm64e-ABI проблему в `fork()` (детали в соответствующей главе), и на arm64-процессе он просто не нужен. И `Application/Dopamine/Exploits/palera1n/Makefile:3` собирается только `arm64` — это standalone-утилита, которая исполняется на хост-машине в арм64-окружении.

Коротко про природу «arm64e-only» в `forkfix`: при `fork()` на arm64e дочерний процесс наследует подписанные указатели родителя, но PAC-ключи **не перенаследуются** автоматически — ядро генерирует новые. Это приводит к тому, что часть структур dyld, которые были подписаны ключом родителя, после fork'а в ребёнке становятся «авторизованно-невалидными»: следующий `AUT*` на них упадёт. На arm64-процессе этой проблемы нет, потому что подписей нет вовсе — поэтому `forkfix` компилируется и работает исключительно на arm64e-слайсе, а на arm64-A9…A11 он просто не инжектится.

### 3.4.1. arm64e ABI v1 vs v2

Внутри самой arm64e Apple провела миграцию между двумя ABI. ABI v1 использовался на A12/A13 (iOS 12–15 эпохи) и в ранних beta A14. ABI v2 — текущий, используется начиная с iOS 14.x на A14+ и, по всей видимости, на всех A12/A13 с iOS 15+. Различия:

- ABI v1: PAC-дискриминаторы вокруг функций и vtable назначались иначе; constant diversity была уже 16 бит, но address diversity включала не всегда полный `&slot`, а иногда `0`.
- ABI v2 (`PTRAUTH_ABI_V2`, виден в `otool -h -v` на arm64e-слайсе): address diversity всегда — полный адрес слота, constant diversity — 16 бит, подпись затрагивает больше слотов (включая C++-vtable entries, которые в v1 не подписывались).

Для автора твика это означает, что любые таблицы PAC-дискриминаторов, собранные для iOS 13 и ранее (ABI v1), с большой вероятностью **неприменимы** к iOS 26 на A13 (ABI v2). Это одна из главных причин, почему в коде Dopamine нет «универсального» набора salt — для каждой версии dyld4 он может чуть-чуть измениться, и единственная устойчивая стратегия — считать его прямо из бинаря целевой dyld. Проверка ABI: `otool -h -v -arch arm64e /usr/lib/libSystem.B.dylib` на монтированном срезе iOS 26.0.1 должен в поле `caps` печатать `PTRAUTH_ABI_V2`.

### 3.4.2. Почему и шелл-утилиты BaseBin собираются fat

На первый взгляд, `jbctl` или `dopamine` (CLI-утилиты) не должны касаться PAC — они просто парсят аргументы и дёргают XPC. Но даже у них в Makefile стоит `-arch arm64 -arch arm64e` (`BaseBin/jbctl/Makefile:5`). Причина — не PAC сам по себе, а то, что эти CLI могут исполняться:

- в контексте arm64e-инъекции (`launchd`, который сам arm64e, может `posix_spawn` их напрямую);
- в контексте arm64-процесса (пользовательский shell из troll-установки);
- и, что важно, они линкуются с libjailbreak, которая живёт в системе и как arm64, и как arm64e; dyld должен иметь возможность загрузить правильный слайс в любом случае.

Это общий инвариант Dopamine: **любая системная компонента, которую может загрузить system-wide dyld, обязана быть fat** — иначе появляется диагональный случай «арм64-процесс пытается загрузить arm64e-only dylib» или наоборот, и такая загрузка либо падает с `incompatible architecture`, либо (хуже) грузит не ту ABI и падает позже.

Отдельная история — `BaseBin/dyldhook/Makefile:23-48`. Здесь fat не делается, но собираются **шесть разных dylib** попарно: `arm64e × iOS 15/16/17/18` и `arm64 × iOS 15/16/17/18`. То есть разводка идёт и по PAC-ABI (ptr-auth-aware vs no PAC), и по версиям iOS (разные layout dyld4). На iOS 26.0/26.0.1 используется ветвь `IOS=18` — это не подтверждено в коде репозитория явной привязкой `IOS=18` именно к 26-й, но косвенно следует из `BUILD.md` и из того, что никакой отдельной `IOS=26` ветви в Makefile нет.

Главный, самый практический вывод: **на A13 PAC в вашем твике появляется ровно тогда, когда вы собираете его как arm64e-срез и инжектитесь в arm64e-процесс**. Если ваш твик — fat `arm64 + arm64e`, а целевой процесс — arm64, dyld выберет arm64-срез, и PAC-ветки кода физически не исполнятся.

## 3.5. Как узнать, использует ли ваш твик PAC

Два пути — статический (на хост-машине до установки) и рантаймовый (внутри самого процесса).

### 3.5.1. Статически: `otool -h`

На хост-Mac (где собирается Dopamine; в контейнере сборки CI — тоже):

```bash
# Общий заголовок, по slice'ам
otool -h -arch all /path/to/MyTweak.dylib

# Только arm64e-слайс
otool -h -arch arm64e /path/to/MyTweak.dylib

# Только arm64-слайс
otool -h -arch arm64 /path/to/MyTweak.dylib
```

`otool -h` показывает `cputype`/`cpusubtype`/`caps`/`filetype`/`ncmds`/`sizeofcmds`/`flags` каждого слайса fat-бинарника. Для PAC-aware слайса `cputype = 0x100000c` (ARM64), `cpusubtype = 2` — это и есть `CPU_SUBTYPE_ARM64E`. Для arm64-слайса `cpusubtype = 0` (`CPU_SUBTYPE_ARM64_ALL`). В верхних битах `cpusubtype` arm64e-слайса будет `CPU_SUBTYPE_PTRAUTH_ABI` (`0x80000000`) и, при `-arch arm64e` на свежем SDK, `CPU_SUBTYPE_ARM64E_WITH_PTRAUTH_VERSION` (это «ABI v2»). `otool -h -v` печатает это словами.

На Linux-машине (как внутри нашего контейнера сборки) `otool` недоступен (`which otool` возвращает пусто). В таком случае используем `file`:

```bash
file MyTweak.dylib
# ожидается:
# MyTweak.dylib: Mach-O universal binary with 2 architectures:
#     [arm64:Mach-O 64-bit arm64 ...] [arm64e:Mach-O 64-bit arm64e ...] ...
```

и `nm`/`llvm-nm`:

```bash
llvm-nm --arch=arm64e -gU MyTweak.dylib
```

Если в выводе есть символы `_dyld_hook_routine`, `_ptrauth_auth_and_resign` (как wrapper, если компилятор не заинлайнил intrinsic), это косвенный признак, что в arm64e-слайсе есть PAC-aware ветка.

### 3.5.2. В рантайме: `_dyld_get_image_header` + `cpusubtype`

Если нужно принять решение прямо из своего кода (например, выбрать стратегию хука), в рантайме можно спросить у dyld, в каком слайсе мы запущены:

```c
#include <mach-o/dyld.h>
#include <mach-o/loader.h>
#include <mach/machine.h>

static int running_as_arm64e(void)
{
    const struct mach_header *mh = _dyld_get_image_header(0);
    if (!mh) return 0;
    cpu_subtype_t sub = mh->cpusubtype & ~CPU_SUBTYPE_MASK;
    return sub == CPU_SUBTYPE_ARM64E;
}
```

Канонический пример из самого Dopamine — `Application/Dopamine/Jailbreak/DOEnvironmentManager.m:220-224`:

```objc
cpu_subtype_t cpusubtype = 0;
size_t len = sizeof(cpusubtype);
if (sysctlbyname("hw.cpusubtype", &cpusubtype, &len, NULL, 0) == -1) return NO;
return (cpusubtype & ~CPU_SUBTYPE_MASK) == CPU_SUBTYPE_ARM64E;
```

Разница: `_dyld_get_image_header(0)->cpusubtype` отвечает «в каком слайсе исполняется **именно этот образ**», `sysctlbyname("hw.cpusubtype", …)` — «что умеет весь процесс / весь кэш shared cache». На A13 iOS 26 оба способа дадут arm64e, если сам процесс arm64e, но внутри процесса может быть загружен и arm64-dylib (например, из `/private/var/jb/.../arm64/*.dylib`). `_dyld_get_image_header` принимает индекс образа и позволяет посмотреть cpusubtype конкретного загруженного модуля. Это именно то, что нужно для вопроса «нужно ли ставить PAC-aware хук в этот конкретный образ?» — а не вопроса «аппаратно ли мой чип PAC-способный».

### 3.5.2a. Как dyld выбирает слайс fat-бинарника

Этот момент стоит сделать явным, потому что от него зависит, в какой режим попадёт ваш твик на A13 iOS 26. Алгоритм, упрощённо, такой:

1. Процесс запускается ядром через `execve` на некотором `.app`-бинаре.
2. Ядро читает Mach-O заголовок бинаря, выбирает слайс по собственной политике (на A13 iOS 26 с arm64-only бинарем выбирается arm64, с fat `arm64 + arm64e` — обычно arm64e, если нет особых ограничений, но для сторонних приложений Apple может принудительно выбрать arm64).
3. Ядро запускает dyld того же слайса (на A13 iOS 26 это **всегда arm64e** — потому что `/usr/lib/dyld` сам arm64e, но для arm64-процесса ядро выбирает `/usr/lib/dyld` с особой arm64-shim-ветвью, которая работает без PAC. Это утверждение **не подтверждено** в коде репозитория Dopamine; оно следует из косвенных признаков в `BaseBin/dyldhook/Makefile:23-48`, где собираются и arm64, и arm64e-версии dyldhook.
4. dyld мапит сначала dyld_shared_cache (откуда идут `libSystem.B.dylib`, `libdyld.dylib`, `libobjc.A.dylib` — там один слайс, соответствующий слайсу dyld).
5. dyld загружает `DYLD_INSERT_LIBRARIES`-dylib'ы, в число которых входит `systemhook.dylib`. Из fat-dylib выбирается слайс, совпадающий со слайсом процесса.

Из пункта 5 следует: ваш твик, инжектящийся в arm64-процесс, автоматически исполнится в arm64-режиме, даже если он собран fat. Это — не предположение, а гарантируемое dyld'ом поведение. Поэтому у автора твика всегда есть «право на лень»: собрать fat и забыть; PAC-код будет исполняться там, где нужен.

### 3.5.3. Таблица решений

| Срез процесса | Срез target-dylib (где слот) | Срез вашего твика | Что делать |
|---|---|---|---|
| arm64 | arm64 | arm64 | обычный fishhook / слот-перезапись. PAC не нужен |
| arm64 | arm64 | fat (arm64+arm64e) | исполнится arm64-ветка твика, PAC-пути под `#ifdef __arm64e__` отрежутся |
| arm64e | arm64e | arm64e | `ptrauth_auth_and_resign` обязательно: иначе падение на `AUTIA` при первом вызове |
| arm64e | arm64 (например, твик) | arm64e | подписанного указателя в arm64-слоте нет, но компилятор-arm64e может генерировать PAC-вызов → всё равно нужно подписывать свой указатель перед передачей в arm64e-код |
| arm64e | смесь | fat | писать PAC-ветку под `#ifdef __arm64e__` и обязательно тестировать оба слайса |

Если ваш твик — fat `arm64 + arm64e`, dyld на A13 автоматически выберет слайс по cpusubtype target-процесса. Fat-бинарь — универсальное решение, оно же — то, которое использует весь BaseBin.

### 3.5.4. Фрагмент mach-o заголовка и флаги PTRAUTH

Чтобы не быть голословными про `otool -h`, вот как выглядит заголовок arm64e-слайса типичного BaseBin dylib (приблизительная расшифровка; точный байтовый дамп на Linux в репозитории не получить, нужна macOS):

```
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf  ARM64        E ptrauth_v2 USR00  DYLIB   28       3840 NOUNDEFS DYLDLINK TWOLEVEL
```

- `cputype = ARM64` — то же, что и для arm64 (16777228 = 0x100000c).
- `cpusubtype = E` — число 2, канонически `CPU_SUBTYPE_ARM64E`, определено в `<mach/machine.h>`.
- `caps = ptrauth_v2 USR00` — поле `caps` (верхние 8 бит `cpusubtype`): PTRAUTH ABI V2 (0x80), kernel ABI version 0. На A13 iOS 26 ожидается именно V2.

На arm64-слайсе того же универсального dylib:

```
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf  ARM64        ALL   0x0    DYLIB   27       3200 NOUNDEFS DYLDLINK TWOLEVEL
```

- `cpusubtype = ALL` — 0, `CPU_SUBTYPE_ARM64_ALL`.
- `caps = 0x0` — никаких PTRAUTH-capabilities.

Разница по формату — одна строка; но для dyld это сигнал «подпиши этот слайс под PAC» vs «не трогай PAC». Для нас как для автора твика — простой способ понять, что внутри.

Небольшой фрагмент C, который делает ту же проверку в рантайме (расширение той же логики, что в `DOEnvironmentManager.m:252-262`):

```c
#include <mach-o/dyld.h>
#include <mach-o/loader.h>
#include <mach/machine.h>

const char *tweak_slice_name(void)
{
    const struct mach_header *mh = _dyld_get_image_header(0);
    if (!mh) return "unknown";
    cpu_subtype_t sub = mh->cpusubtype & ~CPU_SUBTYPE_MASK;
    cpu_subtype_t caps = (mh->cpusubtype & CPU_SUBTYPE_MASK) >> 24;
    if (sub == CPU_SUBTYPE_ARM64E) {
        if (caps & 0x80) return "arm64e (PTRAUTH ABI v2)";
        return "arm64e (PTRAUTH ABI v1)";
    } else if (sub == CPU_SUBTYPE_ARM64_ALL) {
        return "arm64";
    }
    return "other";
}
```

**Не подтверждено по grep:** константа `0x80` для PTRAUTH ABI V2 в поле caps — следует из публичной документации Apple и `<mach/machine.h>` SDK, но в самом репозитории Dopamine 3.0.12 roothide прямого `#define` я не нашёл. Проверяется компиляцией с актуальным iPhone SDK.

### 3.5.4. Два лирических отступления

**Про `-moutline`.** В `BaseBin/systemhook/Makefile:5` стоит `-Os -moutline`. Outline tree на arm64e умеет вытаскивать повторяющиеся последовательности с `BRAA`/`AUTIA` в хвостовые shim-функции. Это экономит код, но перераспределяет PAC-операции по непривычным местам. Если вы отлаживаете падение в arm64e-слайсе и видите странный `BLR` в outlined-stub — не удивляйтесь, это нормально.

**Про универсальный `basebin.tar`.** Файл `basebin.tar`, который Dopamine разворачивает в `/var/jb/basebin`, собирается один раз и работает и на arm64e-A12/A13 (iOS 26.0/26.0.1), и на arm64-A9…A11 (iOS 15.0–18.7.1). Это возможно именно потому, что каждый `.dylib` там — fat, а каждая Objective-C/C-функция, работающая с PAC, обёрнута `#ifdef __arm64e__`. Любой ваш собственный твик, который претендует «работать и там, и там», должен следовать той же дисциплине.

### 3.5.5. Разбор типичной ошибки первого твика

Самая частая ошибка автора первого твика на A13 iOS 26, которая иллюстрирует неправильное понимание PAC: взять `fishhook`-подобную функцию `rebind_symbols` (или её упрощённую реализацию), прописать `fopen` → `my_fopen`, собрать как `-arch arm64 -arch arm64e`, инжектиться в системный демон (например, `cfprefsd`) и получить SIGSEGV на первом же вызове `fopen` после инъекции. Что произошло:

1. `cfprefsd` на A13 iOS 26 — arm64e-процесс. Его `__got` слоты подписаны ключом DA + modifier = (адрес слота, 16-битный salt).
2. Простая `rebind` переписала слот сырым адресом `my_fopen` — без переподписи.
3. При первом вызове `fopen` внутри `cfprefsd` компилятор-arm64e сгенерировал `LDRAA x8, [...]` или `BLRAA x8, Xm`, которая попыталась авторизовать значение, лежащее в слоте. Авторизация провалилась (подпись не сходится — значение подложено нами без PAC).
4. Процесс либо получил FEAT_FPAC-fault прямо на `LDRAA`, либо получил «отравленный» указатель и упал при разыменовании в `BLR`.

Правильный путь — делать именно то, что делает `litehook` (см. `BaseBin/_external/modules/litehook/src/`): на arm64e-срезе перед записью в слот — `ptrauth_sign_unauthenticated(my_fopen, ptrauth_key_process_independent_code, (location_hi48 | salt_lo16))`. Для `__got` слотов salt берётся из `__chain_fixups` — именно `LC_DYLD_CHAINED_FIXUPS` содержит все PAC-дискриминаторы для arm64e, и `litehook_find_dsc_symbol` + таблицы фиксапов — то, где этот salt хранится. Это то, что мы коснёмся в следующей главе о chained fixups.

## 3.5.5a. Семейство `__darwin_arm_thread_state64_*`: PAC для thread state

При работе с `arm_thread_state64_t` на arm64e-процессе возникает отдельный класс грабель, не похожий на хук GOT. `arm_thread_state64_t` на arm64e содержит не сырые регистры, а их PAC-подписанные версии — PC, LR, SP и FP подписаны ключом IB (ptrauth_key_process_dependent_code) с modifier, который зависит от процесса. Соответственно, SDK предлагает отдельное семейство accessor'ов, которые раскрывают PAC и позволяют читать/писать «логическое» значение.

Основные операции, которые встречаются в коде Dopamine:

- `__darwin_arm_thread_state64_get_pc(state)` — вернуть PC после авторизации.
- `__darwin_arm_thread_state64_get_lr(state)`, `_get_sp`, `_get_fp` — то же для LR/SP/FP.
- `__darwin_arm_thread_state64_set_pc_fptr(state, fptr)` — установить PC из уже подписанного fptr.
- `__darwin_arm_thread_state64_ptrauth_strip(state)` — выкусить PAC из всех регистров сразу (XPAC в цикле).

Используется это в двух местах:

1. **Правка dyld entrypoint в только что заспавненном процессе** — `BaseBin/libroothider/src/dyld_patch.m:721-770`. Нам нужно перехватить момент, когда ядро отдало управление `dyld_entry`, и перенаправить на свой код. Прочитать PC штатным геттером не всегда можно (у WebContent другой PAC-ключ, см. комментарий `:721-722`), поэтому — strip, работа с голым значением, затем sign-unauthenticated c IA+0 (`:737`), запись обратно.
2. **crashreporter** — `BaseBin/libroothider/src/crashreporter.m:240,536`. Там thread state читается уже умершего треда, PAC-ключ умершего процесса нам недоступен, strip — единственный способ. Внимание: strip-ом нельзя получить обратно валидно подписанный PC, но для диагностики (печать стек-трейса) это и не нужно.

Для автора твика это выглядит как общее правило: **читать thread state чужого процесса — только через strip**, писать thread state чужого процесса — только через `sign_unauthenticated` с IA+0, и только после `cs_allow_invalid` на самом процессе (чтобы ядро не отбросило запись из-за PAC-mismatch). Это именно та последовательность, которую реализует `BaseBin/libroothider/src/dyld_patch.m:742-758`.

## 3.5.5b. Что делать с inline-asm и `-fno-stack-protector`

В `BaseBin/systemhook/src/common/inline.S` и `inline.h` реализован набор inline-syscall'ов — ручное `svc #0x80` с нужным номером syscall в `x16`. Эти пути существуют, чтобы хуки, установленные Dopamine, могли вызывать «настоящий» ядерный интерфейс, минуя `libSystem` (а значит, минуя и свои же хуки). К PAC это касается опосредованно: inline-asm на arm64e не должен делать `BLR` на подписанный указатель без AUT*, иначе ядро получит кривой `x30` на `svc`. Но так как inline-syscall'ы не вызывают произвольные функции, а делают прямой `svc`, PAC-кода там нет.

Отдельно про `-fno-stack-protector -fno-stack-check` в `BaseBin/systemhook/Makefile:9-11`. Это флаги для `roothider_inline.o`, который требуется быть self-contained (`nm -u -j` не должен выдавать undefined). Отключение stack protector убирает `PACIBSP`/`AUTIBSP` на входе-выходе функций (которые в этом self-contained фрагменте запрещены, т.к. требуют runtime). Это ещё один пример: автор твика, строящий self-contained бинарник, обязан отключить PAC-опции компиляции — иначе получит зависимости от `__stack_chk_guard`, которых в его слое памяти ещё нет.

## 3.5.6. Законченный пример: PAC-aware хук `dlopen` на A13 iOS 26

Сшиваем всё, что было в главе, в один рабочий пример — именно тот, который использует Dopamine для хука `dlopen`. Файл: `BaseBin/systemhook/src/roothider_main.c:433-473`.

```c
int hook_dyld_routine(void **dyld, int idx, void *hook, void **orig, uint16_t pacSalt)
{
    if (!dyld) return -1;

    uint64_t dyldPacDiversifier = ((uint64_t)dyld & ~(0xFFFFull << 48)) | (0x63FAull << 48);
    void **dyldFuncPtrs = ptrauth_auth_data(*dyld,
                                            ptrauth_key_process_independent_data,
                                            dyldPacDiversifier);
    if (!dyldFuncPtrs) return -1;

    if (vm_protect(mach_task_self_,
                   (mach_vm_address_t)&dyldFuncPtrs[idx], sizeof(void *),
                   false, VM_PROT_READ | VM_PROT_WRITE) == 0) {
        uint64_t location = (uint64_t)&dyldFuncPtrs[idx];
        uint64_t pacDiversifier = (location & ~(0xFFFFull << 48))
                                | ((uint64_t)pacSalt << 48);

        *orig = ptrauth_auth_and_resign(dyldFuncPtrs[idx],
                                        ptrauth_key_process_independent_code, pacDiversifier,
                                        ptrauth_key_function_pointer, 0);
        dyldFuncPtrs[idx] = ptrauth_auth_and_resign(hook,
                                        ptrauth_key_function_pointer, 0,
                                        ptrauth_key_process_independent_code, pacDiversifier);
        vm_protect(mach_task_self_,
                   (mach_vm_address_t)&dyldFuncPtrs[idx], sizeof(void *),
                   false, VM_PROT_READ);
        return 0;
    }

    return -1;
}

void init_dyldhooks()
{
    void ***gDyldPtr = litehook_find_dsc_symbol("/usr/lib/system/libdyld.dylib", "__ZN5dyld45gDyldE");
    if (gDyldPtr) {
        hook_dyld_routine(*gDyldPtr, 14, (void *)&dyld_dlopen_hook,            (void **)&dyld_dlopen_orig,            0xBF31);
        hook_dyld_routine(*gDyldPtr, 18, (void *)&dyld_dlopen_preflight_hook,  (void **)&dyld_dlopen_preflight_orig,  0xB1B6);
        hook_dyld_routine(*gDyldPtr, 97, (void *)&dyld_dlopen_from_hook,       (void **)&dyld_dlopen_from_orig,       0xD48C);
        hook_dyld_routine(*gDyldPtr, 98, (void *)&dyld_dlopen_audited_hook,    (void **)&dyld_dlopen_audited_orig,    0xD2A5);
    } else { // ios18.4+
        void ***gAPIsPtr = litehook_find_dsc_symbol("/usr/lib/system/libdyld.dylib", "__ZN5dyld45gAPIsE");
        if (gAPIsPtr) {
            hook_dyld_routine(*gAPIsPtr, 13, (void *)&dyld_dlopen_hook,           (void **)&dyld_dlopen_orig,           0xBF31);
            hook_dyld_routine(*gAPIsPtr, 17, (void *)&dyld_dlopen_preflight_hook, (void **)&dyld_dlopen_preflight_orig, 0xB1B6);
            hook_dyld_routine(*gAPIsPtr, 92, (void *)&dyld_dlopen_from_hook,      (void **)&dyld_dlopen_from_orig,      0xD48C);
            hook_dyld_routine(*gAPIsPtr, 93, (void *)&dyld_dlopen_audited_hook,   (void **)&dyld_dlopen_audited_orig,   0xD2A5);
        }
    }
}
```

Что здесь важно подчеркнуть:

1. **Детекция версии dyld4 через наличие символа.** `litehook_find_dsc_symbol` ищет сначала `__ZN5dyld45gDyldE`; если он есть — это iOS 15…18.3. Если его нет — ищется `__ZN5dyld45gAPIsE`, что соответствует iOS 18.4+. **На iOS 26.0 и 26.0.1** структура, по всей видимости, та же — но прямого подтверждения в коде репозитория нет; это рантаймовая детекция, которая просто пойдёт по ветке `gAPIsPtr` в случае, если `gDyld` не нашёлся.
2. **Индексы меняются между ветками.** Один и тот же API (`dlopen`) на iOS <18.4 — индекс 14, на iOS 18.4+ — индекс 13. Это иллюстрирует: автор твика не должен хардкодить индекс; он должен добывать его из конкретной версии dyld, которая у него под рукой.
3. **Salt НЕ меняется между этими ветками в коде Dopamine.** `0xBF31` используется в обеих. **Это не подтверждает**, что Apple не поменяла salt в iOS 26.0.1 — это подтверждает, что команда roothide пока не нашла необходимости менять. Для надёжности на новой iOS этот salt надо подсматривать в бинаре dyld, как описано в 3.2.3.
4. **`vm_protect` в обе стороны.** Таблица dyld4 лежит в неизменяемой странице `__DATA_CONST`. Без перевода её в R/W вы получите `KERN_PROTECTION_FAILURE` — это не связано с PAC, но важная оговорка. На A13 iOS 26 это работает только из уже unsandboxed-процесса (systemhook загружается в каждый дочерний процесс через `DYLD_INSERT_LIBRARIES` и бутстрапится в контекст, где `jbclient_cs_revalidate` уже прошёл).
5. **Нет никакого dlsym для `dlopen`.** Мы не находим `dlopen` по имени — мы находим `gDyld` (таблица), а дальше по индексу. Этот подход радикально более надёжный на PAC-платформе, потому что обойти его защиту (поддельный ptr в GOT) без знания salt невозможно, а правильный переход через переподпись — есть.

Та же самая логика работает для хуков в любой другой dyld4-API: хотите хукнуть, скажем, `dyld_image_count` — найдите индекс этой функции в `*gDyld` на вашей версии iOS и salt (как описано в 3.2.3) и подставьте в `hook_dyld_routine`. Это именно та масштабируемость, ради которой код написан через параметризацию по `(idx, salt)`.

## 3.5.7. iOS 26.0 vs iOS 26.0.1: что отличается с точки зрения PAC

Прямого признака изменения PAC-поведения между iOS 26.0 (билды `23B5044l`, `23B5059e`, `23B5064e` в `Application/Dopamine/Exploits/{DarkSword,ClearSword}/Info.plist`) и 26.0.1 (`DPSupportedRanges.End = 26.0.1`) в коде репозитория Dopamine 3.0.12 roothide **не нашёл по grep**. Единая точка, которая различает эти версии в коде — `DPSupportedRanges`-плисты, но она не касается PAC. Это означает одно из двух:

- либо Apple реально ничего не поменяла в PAC-слое между 26.0 и 26.0.1 (что соответствует тому, что .1-патч был security-fix без ABI-ломок);
- либо изменения есть, но Dopamine 3.0.12 roothide их не обрабатывает отдельно — и обе версии идут через ту же кодовую ветку (вариант «различия не выявлены»).

Для автора твика это означает: **при переходе 26.0 → 26.0.1 PAC-слой переписывать не придётся**. Если на 26.0 ваш PAC-aware хук работает, он скорее всего работает и на 26.0.1. Но контрольный запуск на устройстве 26.0.1 всё равно обязателен — потому что отсутствие заверенных изменений в репо ≠ отсутствие изменений как таковых.

## 3.6. Выводы

- ARMv8.3-A вводит четыре ключа (IA, IB, DA, DB), десяток инструкций `PAC*`/`AUT*`/`XPAC*` и один 64-битный modifier. Для автора твика на A13 физически интересны **IA** (`ptrauth_key_process_independent_code`) — для указателей на функции в shared cache и dyld4-таблицах — и **DA** (`ptrauth_key_process_independent_data`) — для указателя на саму таблицу dyld4 (`*gDyld`/`*gAPIs`).
- Modifier в коде Dopamine руками собирается из «address diversity» (адрес слота в старших 48 битах) и «constant diversity» (16-битный salt в старших битах). Salt для `*gDyld` подтверждён в коде как `0x63FA` (`BaseBin/systemhook/src/main.c:72`), а для четвёрки `dlopen/dlopen_preflight/dlopen_from/dlopen_audited` — `0xBF31/0xB1B6/0xD48C/0xD2A5` (`BaseBin/systemhook/src/roothider_main.c:459-470`), для `dlsym` — `0x839D` (`BaseBin/systemhook/src/main.c:466`).
- Clang-intrinsic'ы `ptrauth_sign_unauthenticated`, `ptrauth_auth_and_resign`, `ptrauth_auth_data`, `ptrauth_strip` — ровно то, чем пользуется Dopamine, чтобы вставить свои хуки «между подписями». На arm64 они схлопываются в no-op; на arm64e превращаются в настоящие PAC-инструкции. Поэтому writing-once-run-on-both код возможен и желателен.
- A13 аппаратно умеет PAC, но пользовательские приложения на нём обычно arm64 и PAC в них отсутствует. PAC появляется там, где явно присутствует arm64e-слайс — в системном dyld, в `launchd`, в `WebContent`, в BaseBin-коде Dopamine. Это сосуществование и есть причина, почему весь BaseBin собирается fat (`BaseBin/*/Makefile` с `-arch arm64 -arch arm64e` или `ARCHS = arm64 arm64e`).
- Проверить, использует ли ваш твик PAC: статически — `otool -h`/`file`/`nm` с чтением `CPU_SUBTYPE_ARM64E`; в рантайме — `_dyld_get_image_header(i)->cpusubtype` для нужного образа, не для всего устройства.

### Что подтверждено исходниками, а что требует проверки на устройстве

**Подтверждено исходниками Dopamine 3.0.12 roothide:**

- Fat-сборки системного кода `arm64 + arm64e` (все `BaseBin/*/Makefile`, см. выше по тексту).
- Поддержка A12/A13 PPL и iOS 26.0–26.0.1 (`DOEnvironmentManager.m:254`, `DarkSword/Info.plist`, `ClearSword/Info.plist`).
- Конкретный salt `0x63FA` для авторизации `*gDyld` и `0x839D/0xBF31/0xB1B6/0xD48C/0xD2A5` для функций dyld4.
- Поведение `WebContent` с «чужим PAC-ключом» (`dyld_patch.m:721-752`).

**Требует проверки на устройстве:**

- Сохранились ли те же PAC-salt (`0xBF31` и т. д.) в дылде именно iOS 26.0 / 26.0.1 без изменений. Проверка: `otool -tV -arch arm64e /usr/lib/dyld` рядом с инициализатором `gDyld`/`gAPIs`.
- Какой именно индекс в таблице dyld4 на iOS 26.0.1 соответствует `dlopen`/`dlsym`/`dlopen_from`/`dlopen_audited` — в `systemhook` для iOS 18.4+ используются индексы 13/16/17/92/93 (`BaseBin/systemhook/src/roothider_main.c:467-470` и `BaseBin/systemhook/src/main.c:470`), но различий между iOS 18 и iOS 26 в этих индексах по коду в репозитории я не нашёл по grep.

**Не подтверждено ни по исходникам, ни по релиз-ноутам:**

- Отличия поведения PAC между iOS 26.0 и iOS 26.0.1 (релиз-нот с таким пунктом ни в репозитории, ни в коде не найдено). Различия не выявлены.
- Используют ли ядро или userspace какой-либо новый ключ (например, кастомные subkey через `IA_RESTRICTED` расширение ARMv8.6) на 26.x. По коду репозитория никаких упоминаний нет.

### Чеклист автора твика для PAC на iOS 26.0 / 26.0.1 A13

Собираем всё, что было в главе, в один проверочный лист:

1. **Соберите твик как fat `arm64 + arm64e`.** Образец — `BaseBin/libjailbreak/Makefile:5` или `BaseBin/systemhook/Makefile:5`. В Theos — `ARCHS = arm64 arm64e` в Makefile (образец — `BaseBin/roothidehooks/Makefile:8`).
2. **Все PAC-specific операции — под `#ifdef __arm64e__`.** Образец — `BaseBin/libroothider/src/dyld_patch.m:82-84, 735-756`.
3. **Для хуков в системные компоненты (dyld, launchd) — PAC-aware через `ptrauth_auth_and_resign`.** Образец — `BaseBin/systemhook/src/main.c:68-87`. Голые перезаписи `__got`-слотов на arm64e-процессе гарантированно крашатся.
4. **Для хуков в пользовательские приложения (arm64) — PAC не нужен.** fishhook/litehook в arm64-ветке работают как обычно.
5. **PAC-salt функций dyld4 — не угадывать, добывать из бинаря.** В коде Dopamine salt для `dlopen`/`dlsym`/`dlopen_preflight`/`dlopen_from`/`dlopen_audited` закреплены (`0xBF31/0x839D/0xB1B6/0xD48C/0xD2A5`), для чего-либо ещё — `otool -tV -arch arm64e /usr/lib/dyld` + поиск `pacia Xd, Xm` рядом с инициализатором `gDyld`/`gAPIs`.
6. **Thread state чужого процесса — читать strip'ом, писать `sign_unauthenticated` IA+0 с предварительным `cs_allow_invalid`.** Образец — `BaseBin/libroothider/src/dyld_patch.m:721-758`.
7. **Если делаете self-contained бинарник — `-fno-stack-protector -fno-stack-check`.** Образец — `BaseBin/systemhook/Makefile:9-11`.
8. **Проверяйте cpusubtype целевого образа в рантайме, если делаете hybrid-поведение.** Образец — `Application/Dopamine/Jailbreak/DOEnvironmentManager.m:220-224`.

### Короткая история: как PAC стал «обязательным» для джейлбрейкеров

Полное понимание того, почему автор твика на A13 iOS 26 вообще слышит слово PAC, требует одной страницы истории. ARMv8.3-A была ратифицирована в 2016 году; Apple первой из массовых производителей включила PAC в кремний A12 (iPhone XS, 2018). На A12 и A13 Apple ещё держала arm64e как «preview-ABI» и не навязывала его сторонним разработчикам — все приложения из App Store для iPhone XS/11 были arm64. Но системные компоненты (dyld, libSystem, launchd) уже были arm64e. С A14 (2020) Apple сделала arm64e дефолтом для системы (приложения всё ещё arm64, но процесс kernel task — исключительно arm64e), и у джейлбрейкеров начались серьёзные проблемы с переносом классических kernel-эксплойтов, которые раньше легко подделывали указатели в task-порту.

Для твикописателя это разделяет мир на три эпохи:

- **A9–A11** (iOS 9–16): нет PAC в кремнии. Fishhook работает «в лоб». Всё, что говорит эта глава, можно пропустить.
- **A12/A13** (iOS 12–26): PAC в кремнии есть, но Apple не форсирует arm64e для пользовательских приложений. PAC «включается» только там, где явно присутствует arm64e-слайс. Это та категория, под которую пишется Dopamine 3.0.12 roothide.
- **A14+** (iOS 14+): arm64e везде, PAC обязателен. Твики на новые устройства — отдельная история (SPTM, PPL с жёсткой политикой), и Dopamine 3.0.12 roothide на них не целится.

Это та причина, по которой код в репозитории выглядит так, как выглядит: PAC-пути присутствуют везде, где они могут встретиться, но каждая из них обёрнута `#ifdef __arm64e__` или рантаймовой проверкой cpusubtype — потому что реально выполнится только часть, и выбор делается на этапе загрузки конкретного слайса dyld.

## 3.6a. Короткая FAQ

**«У меня работает fishhook на A13 iOS 26 без всякого PAC. Зачем мне это читать?»** Потому что ваш твик, по всей видимости, инжектится в arm64-процесс. Как только вы попробуете хукнуть что-то в системном демоне (`cfprefsd`, `lsd`, `backboardd` — все три arm64e на A13 iOS 26, см. `BaseBin/roothidehooks/Makefile:2`), fishhook упадёт на первом же вызове. PAC-aware слой нужен ровно там.

**«Можно я возьму тот же salt `0xBF31`, что в Dopamine, для своей функции в dyld?»** Нет. `0xBF31` — salt именно для `dyld_dlopen`. Для другой функции — свой salt. Процедура в 3.2.3.

**«Чем `ptrauth_auth_and_resign` отличается от `ptrauth_strip` + `ptrauth_sign_unauthenticated`?»** Первый — проверяет подпись (`AUT*` — fault, если подпись не сошлась), второй — выкусывает без проверки (`XPAC*`). Это разные гарантии безопасности. Если входная подпись могла быть атакована злоумышленником, вы обязаны использовать первый, иначе вы потеряли смысл PAC.

**«Почему `ptrauth_key_function_pointer` ≠ `ptrauth_key_asia`, если оба это IA?»** Это семантическое, не аппаратное различие. `ptrauth_key_function_pointer` — политика «IA + modifier=0» для C function pointer-типов, которую применяет компилятор при сборке Apple-ABI. `ptrauth_key_asia` — просто литерал для IA, без зафиксированного modifier. На практике при использовании одного или другого компилятор поставит одну и ту же `PACIA` или `AUTIA` инструкцию с нужным modifier.

## 3.7. Что дальше

Следующая глава (`deep-dive/04-…`, если она будет написана) логически должна разбирать уже конкретный способ применения PAC-дружелюбного хука к `dlopen` на iOS 26 и показывать, как добывать правильный salt для нестандартной функции dyld4, которой нет в текущем systemhook. Для этого будут нужны:

- `otool -tV -arch arm64e /usr/lib/dyld` с устройства iOS 26.0.1 (достать dyld из `dyld_shared_cache` можно через `dyld_shared_cache_util` на хост-Mac);
- `litehook_find_dsc_symbol` — именно его использует `init_dyldhooks` в `BaseBin/systemhook/src/roothider_main.c:457,465`, чтобы находить `__ZN5dyld45gDyldE` / `__ZN5dyld45gAPIsE` в shared cache;
- `jtool2`/`otool -lv` для чтения PAC-дискриминаторов, зашитых в `__chain_fixups` arm64e-бинарей.

Готовя эти материалы, держите в голове правило из этой главы: «PAC начинается не в чипе, а в том, какой слайс исполняется и в какой слайс вы инжектитесь». Fat-сборка `arm64 + arm64e`, аккуратные `#ifdef __arm64e__` вокруг арм64e-only операций и честная проверка `cpusubtype` целевого модуля в рантайме снимают 90% проблем, с которыми новички сталкиваются при переносе твика с iOS 15/16 arm64 на iOS 26 A13.
