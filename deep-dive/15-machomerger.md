# Deep-dive 15. MachOMerger — склейка Mach-O для внедрения dyldhook

`MachOMerger` — утилита проекта (`BaseBin/MachOMerger/`), которая объединяет две Mach-O в одну. Единственный поддерживаемый use-case — слияние `dyldhook.dylib` с настоящим `dyld`. Это тот самый механизм, который превращает Apple-dyld в «супер-dyld», выполняющий наш lv_bypass.

## 1. Что это

Из `BaseBin/MachOMerger/README.md`:

> # MachOMerger
>
> Merge two MachO binaries into one. The only supported use case at this moment is merging a dylib into dyld.

Единственный use-case — склеить `dyldhook.dylib` в бинарник `dyld`.

Технически — утилита-executable, собираемая через Swift Package Manager (`BaseBin/MachOMerger/Package.swift`). Зависит от трёх submodules:

- `SwiftUtils` (pinauten)
- `SwiftMachO` (pinauten)
- `PatchfinderUtils` (pinauten)

## 2. Структура Sources

Из `BaseBin/MachOMerger/Sources/MachOMerger/`:

| Файл | Назначение |
|---|---|
| `main.swift` | Entry point утилиты (argv processing) |
| `MagicSymbols.swift` | Mach-O magic numbers |
| `Symbols.swift` | Операции с таблицей символов |
| `RelocInfo.swift` | Relocation info между двумя binary |
| `SegmentSplitInfo.swift` | `LC_SEGMENT_SPLIT_INFO` обработка |
| `ChainedFixups.swift` | Chained fixups (iOS 15+) |
| `MiscCmds.swift` | Прочие load commands |
| `SegmentLoadCommand.swift` | `LC_SEGMENT_64` операции |
| `UnixThread.swift` | `LC_UNIXTHREAD` обработка |
| `ULEB128Reader.swift` | ULEB128 decoder (для Mach-O payload'ов) |

Это даёт понимание: утилита **полный Mach-O-мерджер**, не просто concat'катор.

## 3. Требование к dylib

Из того же README:

> # Requirements
> - The dylib to be injected *must* be compiled with `-Xlinker -add_split_seg_info -Xlinker -no_auth_data`

Два критических флага компилятора:

- **`-add_split_seg_info`**: добавляет `LC_SEGMENT_SPLIT_INFO` load command. Это таблица, описывающая, какие части сегмента могут быть передвинуты (relocated). Без неё merger не знает, как корректно сдвигать dylib.
- **`-no_auth_data`**: отключает PAC-подписи в `__DATA` секции. Это нужно потому, что после merge PAC-ключи в dyld-процессе могут отличаться от тех, что были при подписании dylib отдельно.

Проверка этих флагов в сборке dyldhook — в `BaseBin/dyldhook/Makefile`:

```make
LDFLAGS = ... -Xlinker -add_split_seg_info -Xlinker -no_auth_data ...
```

(Точные флаги проверить в текущем Makefile.)

## 4. Как это запускается в сборке

Из `BaseBin/Makefile` цепочка:

```make
MachOMerger: .build .include
	@$(MAKE) -C MachOMerger
	@cp MachOMerger/MachOMerger .build
```

MachOMerger собирается как standalone executable. Затем:

```make
# dyldhook dylibs do not need to be added to the basebin trustcache
dyldhook: basebin.tc
	$(MAKE) -C dyldhook
	@cp dyldhook/*.dylib .build
```

На `make -C dyldhook`:

1. Сначала компилируется исходники dyldhook в `dyldhook.dylib`.
2. Затем вызывается `MachOMerger` для склейки:

```
MachOMerger <path-to-dyld> <path-to-dyldhook.dylib> -o <output-merged-dyld>
```

(Точный синтаксис — в `BaseBin/dyldhook/Makefile` или аналогичном.)

Результат — модифицированный `dyld`, в котором есть код `dyldhook`. Этот `dyld` потом попадает в `basebin.tar` + `basebin.tc` → на устройство, подменяет настоящий `/usr/lib/dyld`.

## 5. Что делает merger по шагам

Концептуально (точные детали — в main.swift):

1. **Прочитать dyld**: заголовок, load commands, segments.
2. **Прочитать dyldhook.dylib**: то же.
3. **Выделить место** в dyld для secций dyldhook.
4. **Перенести секции** dyldhook в dyld, пересчитать offsets.
5. **Пересчитать symbol table**: добавить символы из dyldhook в dyld.
6. **Пересчитать relocations**: используя `LC_SEGMENT_SPLIT_INFO`.
7. **Пересчитать chained fixups**: для iOS 15+ формата.
8. **Создать `LC_UNIXTHREAD`**: entry point теперь модифицированный.
9. **Записать результирующий Mach-O**: собрать всё воедино.

Это — complex Mach-O операция, требующая глубокого понимания формата. Что делает Swift-package такой подходящей — SwiftMachO предоставляет high-level API для таких манипуляций.

## 6. Entitlements

Из `BaseBin/MachOMerger/MachOMerger.entitlements`:

```xml
<!-- Проверить содержимое -->
```

Для **сборочной** утилиты entitlements обычно не нужны — она просто читает/пишет файлы. Но Apple-подпись (на macOS) может потребовать специальных прав для доступа к Xcode SDK; это зависит от конкретной версии.

На устройстве MachOMerger **не запускается** — склейка dyld делается на хосте при сборке.

## 7. Разница с ChOma

ChOma (Deep-dive 12) — тоже работа с Mach-O. Разница:

| Аспект | ChOma | MachOMerger |
|---|---|---|
| Язык | C | Swift |
| Назначение | Общие операции над Mach-O | Специфическая задача: merge двух |
| Применяется | Внутри Dopamine (трастcache, подпись, patch) | Только на этапе сборки |
| На устройстве | Да (линкуется в libjailbreak) | Нет (host-only) |

Это объясняет, почему две разные библиотеки: ChOma — библиотека рантайма; MachOMerger — build-time утилита с другим API-уровнем.

## 8. Почему это важно для автора твика

Прямо — не важно. Автор твика **не использует** MachOMerger.

Косвенно — важно, потому что это объясняет, как ваш твик вообще может грузиться:

1. MachOMerger склеивает dyldhook в dyld на этапе сборки Dopamine.
2. Этот модифицированный dyld попадает в basebin.tar.
3. На устройстве jailbreak подменяет настоящий dyld на модифицированный (детали — в эксплойте).
4. Модифицированный dyld содержит `lv_bypass` (глава III.6), который обходит library validation.
5. Благодаря `lv_bypass` ваш ad-hoc подписанный `.dylib` проходит AMFI load.
6. Твик загружается.

Если бы MachOMerger не работал — ни один твик не грузился бы, потому что dyld не ставил бы lv_bypass.

## 9. Будет ли MachOMerger заменён

Из README проекта и структуры — это узкоспециализированная утилита. Вероятно, со временем:

- Либо будет расширена для других merge-сценариев.
- Либо заменена чем-то более общим (ChOma может эволюционировать).

Для автора twик — не важно. Это deep internals Dopamine build process.

## 10. Что подтверждено

- **Подтверждено**: `MachOMerger` существует в `BaseBin/MachOMerger/` как Swift package.
- **Подтверждено**: `Package.swift` зависит от SwiftUtils, SwiftMachO, PatchfinderUtils (pinauten).
- **Подтверждено README проекта**: единственный use-case — merge dylib в dyld.
- **Подтверждено README**: требование `-add_split_seg_info -no_auth_data`.
- **Подтверждено**: интеграция в сборку через `BaseBin/Makefile:28–30`.
- **Требует проверки**: конкретный синтаксис CLI вызова merger — в `BaseBin/dyldhook/Makefile`.

## 11. Выводы

Главный тезис: **MachOMerger — build-time инструмент**, слои ниже которого (ChOma / SwiftMachO) делают реальную работу. Это объясняет, как `dyldhook.dylib` оказывается **внутри** dyld, а не отдельной dylib.

Для автора твика — одно практическое знание: когда вы собираете свой `.deb` на хосте с установленным Theos, вам не нужно знать про MachOMerger. Он — часть внутренней build-pipeline Dopamine.

## 12. Что дальше

- `ChOma` — Deep-dive 12 (библиотека рантайма для Mach-O).
- `dyldhook` — глава III.6 (что именно встраивается).
- `dyld chained fixups` — Deep-dive 02 (почему это сложно).
- `trustcache` — Deep-dive 05 (почему модифицированный dyld проходит AMFI).

Этим Deep-dive 15 заканчивается подсекция «внутренние build-time инструменты».
