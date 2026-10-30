# Глава V.4. Изменения Mach-O и метаданных

Эта глава — о четвёртой категории патчей: модификация **статического** артефакта (Mach-O бинарник, .deb, .ipa). В отличие от runtime, здесь мы меняем файл на диске: load commands, зависимости, подпись, entitlements, встроенные ресурсы. Это нужно, когда runtime-подход не подходит — например, вам нужно убрать зависимость от отсутствующей dylib, добавить `LC_RPATH`, или подписать произвольный бинарник так, чтобы Dopamine его принял.

## 1. Когда это нужно

Три типовых случая:

1. **Чужая dylib ссылается на отсутствующую зависимость**. Например, старый твик линкуется с `/usr/lib/libsubstrate.dylib`, которой нет на новом Dopamine. Нужно переписать её ссылку на `@rpath/libellekit.dylib` без пересборки.
2. **Подпись отсутствует или истекла**. Произвольный бинарник (чужая утилита, прекомпилированный helper) нужно сделать «валидным» для AMFI через ad-hoc подпись + trust cache.
3. **Удалить `platform-application` entitlement**. Для `.ipa`, которую вы хотите установить, но боитесь предоставить ей platform-права.

Для таких случаев есть две группы инструментов: **ChOma** (программный API для Mach-O) и **ldid** (CLI для подписи).

## 2. Инструменты

| Инструмент | Что умеет | Когда применять |
|---|---|---|
| **ldid** | Ad-hoc подпись, чтение/запись entitlements, пересчёт cdhash | Любой изменённый бинарник нужно переподписать |
| **ChOma** (BaseBin/ChOma/, API) | Чтение/модификация Mach-O на уровне load commands | Программные правки — изменить `LC_LOAD_DYLIB`, добавить `LC_RPATH` |
| **install_name_tool** | Готовая Apple-утилита: `-change`, `-id`, `-add_rpath` | Простые правки без своего кода |
| **otool / nm / vtool** | Чтение Mach-O | Инспекция |
| **lipo** | Fat binary операции (thin, create, extract) | Разделить/собрать arm64/arm64e slices |
| **plutil** | Чтение/запись plist | Info.plist, Entitlements |

На macOS всё это есть; на Linux — ldid доступен, otool/install_name_tool/lipo обычно отсутствуют (есть замены в cctools).

## 3. install_name_tool — простые изменения

### 3.1. Переназначить install_name

```bash
install_name_tool -id "@rpath/MyLib.dylib" MyLib.dylib
```

Записывает в `LC_ID_DYLIB` новое имя. Полезно, если dylib была собрана с абсолютным путём, а нужен `@rpath`.

### 3.2. Переназначить зависимость

```bash
install_name_tool -change \
    "/usr/lib/libsubstrate.dylib" \
    "@rpath/libellekit.dylib" \
    MyTweak.dylib
```

Теперь `otool -L` покажет `@rpath/libellekit.dylib` вместо `/usr/lib/libsubstrate.dylib`. При загрузке dyld будет искать её через LC_RPATH.

Полный проход через все зависимости:

```bash
otool -L MyTweak.dylib
# найти строки с libsubstrate
install_name_tool -change /usr/lib/libsubstrate.dylib @rpath/libellekit.dylib MyTweak.dylib
install_name_tool -change /Library/MobileSubstrate/MobileSubstrate.dylib @rpath/libellekit.dylib MyTweak.dylib
```

### 3.3. Добавить LC_RPATH

```bash
install_name_tool -add_rpath "/var/jb" MyTweak.dylib
```

Если dylib ищет `@rpath/libroot.dylib`, но её нет в rpath-пути — добавить `/var/jb` (который на RootHide — symlink в jbroot) решит.

### 3.4. Удалить LC_RPATH

```bash
install_name_tool -delete_rpath "/some/path" MyTweak.dylib
```

Полезно, если старый rpath указывает на несуществующий путь.

### 3.5. После любого install_name_tool — переподписать

```bash
ldid -Sentitlements.xml MyTweak.dylib
```

Или ad-hoc без entitlements:

```bash
ldid -Sadhoc MyTweak.dylib
# или проще:
codesign --force --sign - MyTweak.dylib
```

Без переподписи — изменённый Mach-O имеет мертвую подпись (не соответствует cdhash), AMFI откажет.

## 4. ldid — подпись и entitlements

### 4.1. Ad-hoc подпись

```bash
ldid -Sadhoc MyBinary
```

Создаёт `__LINKEDIT` segment с ad-hoc подписью. На jailbroken-устройстве валидируется через trust cache.

### 4.2. С entitlements

```bash
# Entitlements.plist:
cat > ent.xml <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>platform-application</key>
    <true/>
    <key>com.apple.private.security.no-container</key>
    <true/>
</dict>
</plist>
EOF

# Подписать с entitlements:
ldid -Sent.xml MyBinary
```

### 4.3. Чтение entitlements

```bash
ldid -e MyBinary
```

Выдаст XML с текущими entitlements.

Или через codesign:

```bash
codesign -d --entitlements - MyBinary 2>&1 | tail -n +3
```

### 4.4. Удаление подписи

```bash
ldid -S- MyBinary
```

Полезно для debug: собрать `.deb` с полностью unsigned бинарником и переподписать на устройстве через `jbctl` (не стандартно, редко нужно).

## 5. ChOma — программный API

ChOma — subproject Dopamine (BaseBin/ChOma/), который даёт C API для чтения и модификации Mach-O. Используется внутри Dopamine (например, в `codesign.m` для пересчёта cdhash). Автор твика может использовать ChOma, если нужны сложные операции batch.

### 5.1. Пример: чтение заголовка

```c
#include <choma/MachO.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    MachO *macho = macho_init_from_path(argv[1]);
    if (!macho) {
        fprintf(stderr, "failed to open %s\n", argv[1]);
        return 1;
    }

    struct mach_header_64 hdr = macho_get_header(macho);
    printf("magic: 0x%08x\n", hdr.magic);
    printf("cputype: 0x%x\n", hdr.cputype);
    printf("ncmds: %u\n", hdr.ncmds);
    printf("flags: 0x%x\n", hdr.flags);

    macho_free(macho);
    return 0;
}
```

Собрать:

```bash
clang -I/path/to/ChOma/include -L/path/to/ChOma/output/macos/lib \
      -lchoma -o inspect inspect.c
```

Для автора твика это нужно редко — большинство операций проще сделать через `install_name_tool` + `otool`.

### 5.2. Пример: пересчёт cdhash

Это основная функция ChOma, которая используется Dopamine. После любой модификации Mach-O cdhash становится неверным. ChOma даёт API `codesign_recompute` (или аналогичное), которое пересчитывает все хеши и обновляет SuperBlob.

Для автора твика проще использовать `ldid -S` — он делает то же самое.

### 5.3. Пример: добавление LC_RPATH программно

```c
#include <choma/MachO.h>

// Добавить LC_RPATH "/var/jb"
MachO *m = macho_init_from_path("MyTweak.dylib");
macho_add_rpath(m, "/var/jb");    // гипотетический API — проверьте имя
macho_save(m, "MyTweak-patched.dylib");
macho_free(m);
```

Это аналогично `install_name_tool -add_rpath`, но в коде. Полезно, если вы автоматизируете процесс сборки и хотите batch-обработку многих файлов.

## 6. Типовой сценарий: ремонт чужого .deb

Представим: вы скачали `.deb` твика, который работал на старом Dopamine, но на новом не загружается. Задача — переназначить зависимости.

### 6.1. Шаги

```bash
# 1. Распаковать .deb
mkdir -p /tmp/repair
cd /tmp/repair
dpkg-deb -x /path/to/original.deb unpacked
dpkg-deb -e /path/to/original.deb unpacked/DEBIAN

# 2. Найти проблемные dylib
cd unpacked
find . -name "*.dylib" -exec otool -L {} \; | grep substrate
# ./var/jb/Library/MobileSubstrate/DynamicLibraries/OldTweak.dylib:
#     /Library/MobileSubstrate/MobileSubstrate.dylib (compatibility ...)

# 3. Переназначить зависимость
install_name_tool -change \
    "/Library/MobileSubstrate/MobileSubstrate.dylib" \
    "@rpath/libellekit.dylib" \
    "./var/jb/Library/MobileSubstrate/DynamicLibraries/OldTweak.dylib"

# 4. Проверить
otool -L ./var/jb/Library/MobileSubstrate/DynamicLibraries/OldTweak.dylib | head -4

# 5. Добавить rpath если нужно
install_name_tool -add_rpath "/var/jb" \
    "./var/jb/Library/MobileSubstrate/DynamicLibraries/OldTweak.dylib"

# 6. Переподписать
ldid -Sadhoc ./var/jb/Library/MobileSubstrate/DynamicLibraries/OldTweak.dylib

# 7. (опционально) Обновить control, добавив ellekit в Depends
# edit unpacked/DEBIAN/control:
#   Depends: ellekit | mobilesubstrate | substitute

# 8. Пересобрать .deb
cd /tmp/repair
dpkg-deb --build unpacked repaired.deb
```

Это полный цикл ремонта без исходников твика. Три сценария применимости:

- Твик давно заброшен автором.
- У вас нет возможности пересобрать (нет среды Theos).
- Нужна быстрая срочная починка.

### 6.2. Проверка результата

```bash
# Инспекция (глава IV.10)
./inspect.sh repaired.deb > inspect-report.txt
# Проверить:
#  - otool -L не содержит старых зависимостей
#  - codesign -dv показывает adhoc
#  - control содержит правильные Depends
```

Установить на устройство и проверить работу.

## 7. Entitlements для своих helper-утилит

Если вам нужен helper-бинарник внутри `.deb` твика (например, CLI-утилита, которую твик будет вызывать через `posix_spawn`), и ей нужны entitlements — вы подписываете её с конкретными правами.

### 7.1. Example entitlements

Для twic-помощника, которому нужен доступ к файлам системы:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>platform-application</key>
    <true/>
    <key>com.apple.private.security.no-container</key>
    <true/>
    <key>com.apple.private.security.storage.AppBundles</key>
    <true/>
</dict>
</plist>
```

Применение:

```bash
ldid -Sent.xml my-helper
```

**Внимание**: не раздавайте вашему helper'у entitlements, которых ему реально не нужно. Это принцип least-privilege. `platform-application` даёт очень широкие возможности; если helper'у нужен только доступ к `/var/mobile/Containers/...`, достаточно `storage.AppBundles`.

### 7.2. Trust cache

После подписи — помните: ad-hoc подпись сама по себе не делает бинарник валидным для AMFI. Нужен trust cache.

На Dopamine в staging ваш `.deb` при установке вызывает `jbclient_trust_file_by_path` для каждого бинарника внутри (через postinst-скрипт или автоматически через dpkg-хуки Dopamine). Это добавляет cdhash в dynamic trust cache.

Если helper запускается не через dpkg, а ваш твик делает `posix_spawn("/path/to/helper", ...)` — ваш твик должен сам вызвать `jbclient_trust_file_by_path("/path/to/helper")` перед spawn'ом.

## 8. Обход library validation для чужой dylib

Задача: загрузить чужую dylib (например, скачанный плагин), которой нет в вашем trust cache.

Два пути:

### 8.1. Через jbclient_trust_file

```c
#include <libjailbreak/jbclient_xpc.h>
#include <dlfcn.h>

int result = jbclient_trust_file_by_path("/path/to/plugin.dylib");
if (result != 0) {
    // не удалось доверить — не можем загружать
    return;
}
void *handle = dlopen("/path/to/plugin.dylib", RTLD_NOW);
```

### 8.2. Через переподпись

```bash
ldid -Sadhoc /path/to/plugin.dylib
# затем ваш твик вызовет jbclient_trust_file_by_path
```

Этот путь требует правильной структуры подписи — старая подпись должна быть сначала удалена через `ldid -S-` или `codesign --remove-signature`.

## 9. Fat binary — работа с arm64 + arm64e

### 9.1. Разделить

```bash
lipo -thin arm64 MyTweak.dylib -output MyTweak-arm64.dylib
lipo -thin arm64e MyTweak.dylib -output MyTweak-arm64e.dylib
```

### 9.2. Собрать

```bash
lipo -create MyTweak-arm64.dylib MyTweak-arm64e.dylib -output MyTweak-fat.dylib
```

### 9.3. Проверить slices

```bash
lipo -archs MyTweak.dylib
# arm64 arm64e

file MyTweak.dylib
# Mach-O universal binary with 2 architectures: [arm64:...] [arm64e:...]
```

Для твиков на RootHide Dopamine fat binary — норма (ARCHS = arm64 arm64e в Makefile). Если нужно thin — используйте `lipo -thin`.

## 10. Проверка интегритета патча

Полный скрипт проверки `.dylib` после модификаций:

```bash
#!/usr/bin/env bash
set -e
DYLIB="${1:?usage: verify.sh file.dylib}"

echo "=== $DYLIB ==="

echo "-- file --"
file "$DYLIB"

echo "-- arch --"
lipo -archs "$DYLIB"

echo "-- dependencies --"
otool -L "$DYLIB"

echo "-- install_name --"
otool -D "$DYLIB"

echo "-- rpath --"
otool -l "$DYLIB" | grep -A2 LC_RPATH | grep "path" || echo "  (no LC_RPATH)"

echo "-- codesign --"
codesign -dv "$DYLIB" 2>&1 | head -5

echo "-- entitlements --"
ldid -e "$DYLIB" 2>/dev/null || echo "  (no entitlements)"

echo "-- symbols (undefined) --"
nm -u "$DYLIB" 2>/dev/null | head -10

echo "-- strings | /var/jb hardcode? --"
if strings "$DYLIB" | grep -q '/var/jb'; then
    echo "  WARN: hardcoded /var/jb paths"
    strings "$DYLIB" | grep '/var/jb' | head -5
else
    echo "  OK: no /var/jb"
fi
```

Запускайте после любой модификации `.dylib` через `install_name_tool`/`ldid`.

## 11. Таблица «мне нужно → инструмент»

| Задача | Инструмент |
|---|---|
| Переназначить `LC_LOAD_DYLIB` | `install_name_tool -change` |
| Переназначить `LC_ID_DYLIB` | `install_name_tool -id` |
| Добавить `LC_RPATH` | `install_name_tool -add_rpath` |
| Удалить `LC_RPATH` | `install_name_tool -delete_rpath` |
| Прочитать entitlements | `ldid -e` или `codesign -d --entitlements -` |
| Подписать ad-hoc | `ldid -Sadhoc` или `codesign --sign -` |
| Подписать с entitlements | `ldid -Sent.xml` |
| Удалить подпись | `ldid -S-` |
| Thin из fat | `lipo -thin <arch>` |
| Собрать fat | `lipo -create ...` |
| Прочитать cdhash | `codesign -d` или ChOma API |
| Пересчитать cdhash после правок | `ldid -S*` (сделает автоматически) |
| Прочитать load commands | `otool -l` |
| Прочитать зависимости | `otool -L` |
| Прочитать символы | `nm -g` / `nm -u` |

## 12. Что подтверждено

- **Подтверждено Apple docs**: `install_name_tool`, `lipo`, `otool` — стандартные Xcode-утилиты.
- **Подтверждено практикой**: `ldid` — стандарт для jailbreak-подписей.
- **Подтверждено проектом**: ChOma — Dopamine subproject (BaseBin/ChOma/), используется внутри для работы с Mach-O (см. главу III.11).
- **Подтверждено Dopamine**: `jbclient_trust_file_by_path` — публичный API для добавления cdhash в dynamic trust cache (глава III.2).
- **Требует проверки на устройстве**: что после ваших правок dylib корректно загружается — зависит от того, не нарушена ли совместимость.

## 13. Выводы

Главный тезис: **Mach-O patching — это стыковка несовместимых deps без пересборки**. Инструменты простые: `install_name_tool` для метаданных, `ldid` для подписи. Для сложного — ChOma как C API.

Три якорных правила:

1. **После любой правки — переподписать** через `ldid -S*`. Иначе AMFI отвергнет.
2. **Для твиков используйте ARCHS = arm64 arm64e** (fat binary) — работает на всех устройствах.
3. **Entitlements — least privilege**. Не раздавайте `platform-application` «на всякий случай».

## 14. Что дальше

- Контроль качества патча — глава V.5.
- Исследование — глава VI.1.
- Детальный разбор Mach-O — Deep-dive 01 (внутри раздела глубоких погружений).
- Детальный разбор ChOma — Deep-dive 12.
