# RE 5. Как сравнить файловую систему 26.0 и 26.0.1

Для автора твика важно знать, что изменилось в системных файлах между minor-releases. Процедура.

## 1. Задача

- Pозаказать diff: `/System/*` на iOS 26.0 vs iOS 26.0.1.
- Увидеть, какие frameworks обновлены, какие daemons перекомпилированы, какие plist изменены.
- Прогнозировать impact на свои твики.

## 2. Инструменты

- `ipsw` CLI (как в RE 4).
- `diff` (стандартный).
- Два `.ipsw` файла — для 26.0 и 26.0.1.

## 3. Шаг 1: Скачать оба IPSW

- От Apple: `developer.apple.com/download/` (с developer account).
- От community: `ipsw.me` (unofficial mirror).

Для iPhone 11 (A13):

```
iPhone11,2_26.0_23B5044l_Restore.ipsw   — iOS 26.0
iPhone11,2_26.0.1_23B5064e_Restore.ipsw  — iOS 26.0.1 (гипотетический билд)
```

Каждый — ~5 GB.

## 4. Шаг 2: Извлечь filesystem из каждого

```bash
mkdir -p /tmp/ios26-0 /tmp/ios26-0-1

# Извлечь filesystem image:
ipsw extract --filesystem /path/to/iPhone11,2_26.0_23B5044l_Restore.ipsw --output /tmp/ios26-0/
ipsw extract --filesystem /path/to/iPhone11,2_26.0.1_23B5064e_Restore.ipsw --output /tmp/ios26-0-1/

# Результат — DMG файлы.
```

## 5. Шаг 3: Смонтировать DMG

На macOS:

```bash
# Смонтировать:
hdiutil attach /tmp/ios26-0/<file>.dmg -mountpoint /tmp/mnt-26-0 -readonly
hdiutil attach /tmp/ios26-0-1/<file>.dmg -mountpoint /tmp/mnt-26-0-1 -readonly
```

На Linux — через `apfs-fuse` (сложнее). Для iOS filesystem APFS с encryption; работа на Linux ограничена.

## 6. Шаг 4: Diff

### 6.1. Файловая структура

```bash
diff -r --brief /tmp/mnt-26-0/System /tmp/mnt-26-0-1/System > /tmp/diff.txt
wc -l /tmp/diff.txt
# Обычно сотни строк — куча frameworks updated.
```

### 6.2. Фокус на бинарники

```bash
# Какие frameworks изменились:
diff -r --brief /tmp/mnt-26-0/System/Library/Frameworks /tmp/mnt-26-0-1/System/Library/Frameworks

# Какие PrivateFrameworks:
diff -r --brief /tmp/mnt-26-0/System/Library/PrivateFrameworks /tmp/mnt-26-0-1/System/Library/PrivateFrameworks
```

На minor update — обычно 10-50 frameworks отличаются (security fixes + bug fixes).

### 6.3. plist diff

```bash
# LaunchDaemons:
diff -r /tmp/mnt-26-0/System/Library/LaunchDaemons /tmp/mnt-26-0-1/System/Library/LaunchDaemons

# Info.plist's of apps:
diff -r /tmp/mnt-26-0/Applications /tmp/mnt-26-0-1/Applications

# Settings bundles:
diff -r /tmp/mnt-26-0/System/Library/PreferenceBundles /tmp/mnt-26-0-1/System/Library/PreferenceBundles
```

## 7. Шаг 5: Для конкретного framework — binary diff

Если видите, что `SpringBoardFoundation` изменился:

```bash
# File sizes:
ls -la /tmp/mnt-26-0/System/Library/PrivateFrameworks/SpringBoardFoundation.framework/
ls -la /tmp/mnt-26-0-1/System/Library/PrivateFrameworks/SpringBoardFoundation.framework/

# Отличается ли бинарник:
md5sum /tmp/mnt-26-0/.../SpringBoardFoundation
md5sum /tmp/mnt-26-0-1/.../SpringBoardFoundation
```

Если да — class-dump обеих версий и diff:

```bash
class-dump-dyld /tmp/mnt-26-0/.../SpringBoardFoundation -o /tmp/dump-26-0/
class-dump-dyld /tmp/mnt-26-0-1/.../SpringBoardFoundation -o /tmp/dump-26-0-1/

diff -r /tmp/dump-26-0 /tmp/dump-26-0-1 > /tmp/sbf-headers-diff.txt
```

Что ищем в diff:

- **Renamed classes** — старое имя исчезло, появилось новое.
- **New methods** — добавлены.
- **Removed methods** — удалены (ваш хук сломается).
- **Changed signatures** — редко, но ломает хуки.

## 8. Шаг 6: Через dyld shared cache (альтернатива)

Вместо DMG mount — извлечь dyld_shared_cache из каждого IPSW и diff через ipsw CLI (RE 4):

```bash
ipsw extract --dyld /path/to/iPhone11,2_26.0_...ipsw -o /tmp/dyld-26-0/
ipsw extract --dyld /path/to/iPhone11,2_26.0.1_...ipsw -o /tmp/dyld-26-0-1/

# Compare — но dyld cache — single file, сложно diff'ить напрямую.
# Проще — extract specific framework из каждого и compare.
```

## 9. Шаг 7: Changelog Apple

Публичный security content:

- `support.apple.com/<HT-ID>` — список CVE fixed.
- Иногда — mentions конкретных frameworks.

Это дополняет файловый diff: вы видите _что_ изменилось и _почему_.

## 10. Что ищет автор твика

Приоритеты при diff:

1. **Frameworks моего target** — если я хукаю SpringBoard, SBF должен быть примерно тот же.
2. **cfprefsd, lsd, runningboardd** — Dopamine их хукает; изменения в их бинарниках могут требовать re-adjust Dopamine самого.
3. **AMFI / codesign** — если изменились, может сломать trust cache.
4. **dyld** — если изменился, dyldhook может потребовать re-merge.

## 11. Workflow для production проекта

Регулярно (при каждом Apple release):

```
1. Скачать new IPSW.
2. diff против previous.
3. Проверить, затронут ли ваш target framework.
4. Если да — class-dump + diff headers.
5. Обновить код твика при необходимости.
6. Выпустить patch release.
```

Это рутина для tweak maintainer на долго-живущих проектах.

## 12. Что подтверждено

- **Подтверждено**: `ipsw` CLI поддерживает `extract --filesystem`.
- **Подтверждено Apple**: filesystem image в IPSW — APFS.
- **Требует**: macOS для convenient APFS mount; Linux — ограничено.
- **Не подтверждено**: конкретные размеры и structure iOS 26.0.1 — зависит от Apple release.

## 13. Выводы

Главный тезис: **FS diff — обязательная рутина для серьёзного tweak maintainer**. На minor update занимает час; результат — знание, что менять.

На этом **re-walkthroughs секция закрыта**. 5 глав — base kit RE-инструментов для автора твика.
