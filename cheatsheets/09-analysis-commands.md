# Шпаргалка 9. Инспекционные команды для твиков и пакетов

Полный набор CLI команд для анализа `.deb`, `.dylib`, `.plist`, Mach-O. С примерами.

## .deb (Debian package)

### Содержимое

```bash
dpkg-deb -c file.deb       # список файлов + permissions
dpkg-deb -I file.deb       # control + maintainer scripts
dpkg-deb -R file.deb /tmp/x    # полная распаковка в x/{DEBIAN,...}
dpkg-deb -x file.deb /tmp/data # распаковка только данных
dpkg-deb -e file.deb /tmp/meta # извлечение DEBIAN/*
dpkg-deb --info file.deb       # то же, что -I
```

### Проверка структуры для RootHide

```bash
dpkg-deb -c MyTweak.deb | grep -v '/$' | awk '{print $NF}'
# ./var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.dylib
# ./var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.plist
```

## Mach-O (.dylib, executable)

### Общая информация

```bash
file MyTweak.dylib
# Mach-O universal binary with 2 architectures: [arm64:...] [arm64e:...]

lipo -info MyTweak.dylib
# или:
lipo -archs MyTweak.dylib
# arm64 arm64e
```

### Load commands

```bash
otool -h MyTweak.dylib           # header
otool -l MyTweak.dylib           # все load commands (длинно)
otool -l MyTweak.dylib | grep -A3 LC_LOAD_DYLIB    # зависимости
otool -l MyTweak.dylib | grep -A3 LC_RPATH         # rpath'ы
otool -l MyTweak.dylib | grep -A3 LC_ID_DYLIB      # install_name
otool -l MyTweak.dylib | grep -A3 LC_UUID          # UUID
otool -l MyTweak.dylib | grep -A3 LC_CODE_SIGNATURE # где подпись
otool -l MyTweak.dylib | grep -A3 LC_BUILD_VERSION # min version
```

### Shortcuts

```bash
otool -L MyTweak.dylib          # зависимости (кратко)
otool -D MyTweak.dylib          # install_name
```

### Дизассемблер

```bash
otool -tV MyTweak.dylib         # inline disasm всех секций
otool -tV MyTweak.dylib -arch arm64 | head -50
```

### Содержимое строк

```bash
strings MyTweak.dylib | head -50
strings MyTweak.dylib | grep '/var/jb'   # хардкоды путей (должно быть пусто)
strings MyTweak.dylib | grep 'MyClass'   # ObjC классы и селекторы
strings MyTweak.dylib | grep -i 'api\|key\|password'  # потенциальные leak'и
```

## Символы

```bash
nm MyTweak.dylib               # все символы
nm -g MyTweak.dylib            # только external (globals)
nm -gU MyTweak.dylib           # external + undefined
nm -u MyTweak.dylib            # только undefined (нужны от зависимостей)
nm -arch arm64 MyTweak.dylib   # одного slice
```

### Полезные grep

```bash
nm -gU MyTweak.dylib | grep '[+-]\['       # ObjC методы
nm -gU MyTweak.dylib | grep 'OBJC_CLASS'    # ObjC классы
nm -u MyTweak.dylib | sort -u              # все внешние зависимости
```

## Codesign

### Чтение

```bash
codesign -dv MyTweak.dylib             # базовая инфо
codesign -dvvv MyTweak.dylib           # verbose (все blobs)
codesign -d --entitlements - MyTweak.dylib    # entitlements (XML)
codesign -d --entitlements :- MyTweak.dylib   # entitlements (print)
codesign -d --signatures MyTweak.dylib        # все signatures
```

Что искать:

- `Signature=adhoc` — нужно для jailbreak.
- `Identifier=...` — должен совпадать с install_name.
- `CodeDirectory v=20400` — версия CD.
- `Hash type=sha256` — тип хеша.

### Подпись

```bash
codesign --force --sign - MyTweak.dylib        # ad-hoc
codesign --remove-signature MyTweak.dylib       # убрать
```

## ldid (альтернатива codesign)

```bash
ldid -Sadhoc MyTweak.dylib       # ad-hoc подпись
ldid -Sent.xml MyTweak.dylib     # ad-hoc + entitlements
ldid -e MyTweak.dylib            # вывести entitlements
ldid -S- MyTweak.dylib           # удалить подпись
ldid -h MyTweak.dylib            # cdhash
```

## install_name_tool

```bash
install_name_tool -id "@rpath/MyLib.dylib" MyLib.dylib
install_name_tool -change OLD_PATH NEW_PATH MyTweak.dylib
install_name_tool -add_rpath "/var/jb" MyTweak.dylib
install_name_tool -delete_rpath "/old/path" MyTweak.dylib
```

**После любого install_name_tool — переподписать через `ldid -S*`.**

## lipo (fat binary)

```bash
lipo -archs MyTweak.dylib                    # arm64 arm64e
lipo -thin arm64 MyTweak.dylib -output MyTweak-arm64.dylib
lipo -create a.dylib b.dylib -output fat.dylib
lipo -extract arm64e MyTweak.dylib -output arm64e.dylib
```

## plist

```bash
plutil -p file.plist         # pretty print (ascii)
plutil -lint file.plist      # проверка валидности XML
plutil -convert xml1 file.plist   # в XML
plutil -convert binary1 file.plist   # в binary (меньше размер)
plutil -convert json file.plist -o file.json   # в JSON

# Чтение значений:
/usr/libexec/PlistBuddy -c "Print :SomeKey" file.plist
/usr/libexec/PlistBuddy -c "Set :SomeKey newValue" file.plist
```

## vtool (альтернатива otool -l)

```bash
vtool -show MyTweak.dylib         # version info
vtool -set-build-version iphoneos 15.0 15.0 -replace MyTweak.dylib
```

## Инспекция одной командой

Полный скрипт:

```bash
#!/usr/bin/env bash
DYLIB="$1"
echo "=== $DYLIB ==="
echo "-- file --";          file "$DYLIB"
echo "-- lipo --";          lipo -archs "$DYLIB"
echo "-- otool -L --";      otool -L "$DYLIB"
echo "-- otool -D --";      otool -D "$DYLIB"
echo "-- rpath --";         otool -l "$DYLIB" | grep -A2 LC_RPATH
echo "-- codesign --";      codesign -dv "$DYLIB" 2>&1 | head -5
echo "-- entitlements --";  ldid -e "$DYLIB" 2>/dev/null
echo "-- hardcode /var/jb --"
strings "$DYLIB" | grep '/var/jb' | head -5 || echo "  (clean)"
echo "-- undefined symbols --"
nm -u "$DYLIB" 2>/dev/null | head -10
```

## Диагностика на устройстве

```bash
# Загруженные dylib в процесс:
ssh mobile@<ip>
vmmap <pid> 2>/dev/null | grep '\.dylib$' | head
# или через /proc (может не работать на iOS):
ls -l /proc/<pid>/maps 2>/dev/null

# Список процессов:
ps -A
launchctl list
launchctl list | grep com.apple

# Какой executable у PID:
ps -p <pid> -o comm=

# Открытые файлы:
sudo lsof -p <pid> 2>/dev/null | head -20
```

## log stream

```bash
# С хоста macOS:
log stream --device <udid> --predicate 'subsystem == "com.example.mytweak"'
log stream --device <udid> --predicate 'process == "SpringBoard"' --style compact
log stream --device <udid> --predicate 'eventMessage CONTAINS "keyword"'

# На устройстве через SSH:
log stream --predicate 'subsystem == "com.example"' 2>&1
# или:
log show --predicate '...' --last 1h
```

## IPC инспекция (на устройстве)

```bash
# XPC services известные launchd:
launchctl print system | grep -i xpc

# Mach ports процесса:
launchctl print pid/<pid>

# Файловые дескрипторы:
sudo lsof -p <pid> 2>/dev/null
```

## Crash reports

```bash
ssh mobile@<ip>
ls -t /var/mobile/Library/Logs/CrashReporter/ | head -5

# JSON формат (iOS 15+):
cat SomeApp-2026-10-30-100012.ips | tail -n +2 | python3 -m json.tool | less
```

## См. также

- Глава IV.10 — сборка + inspection.
- Глава V.4 — Mach-O patches.
- Глава VII.1 — диагностика «не загружается».
- Глава VII.2 — crash reports.
