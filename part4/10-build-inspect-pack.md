# Глава IV.10. Сборка, инспекция и упаковка

Эта глава — финальный технический этап: от исходников к готовому `.deb`, с чекером «а что внутри?» и правилами, которые отделяют «вроде собралось» от «готово к релизу».

## 1. Полная команда сборки

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide FINALPACKAGE=1
```

Разбор каждого аргумента:

- **`clean`** — удалить `.theos/obj/` (кеш сборки). Нужно, если:
  - вы поменяли `ARCHS` или `TARGET`;
  - вы меняли `.plist` или `control` (Theos кэширует это);
  - вы хотите быть уверенным в чистоте результата.
- **`package`** — собрать и упаковать в `.deb`.
- **`THEOS_PACKAGE_SCHEME=roothide`** — для RootHide (глава IV.9).
- **`FINALPACKAGE=1`** — финальная сборка:
  - применяется `-O2` оптимизация (вместо `-O0`);
  - удаляется debug info через `strip`;
  - `.deb` сжимается плотнее;
  - добавляются макросы `-DNDEBUG`.

**Для разработки** (быстрая итерация, с отладкой):

```bash
make package THEOS_PACKAGE_SCHEME=roothide
```

**Для релиза**:

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide FINALPACKAGE=1
```

Разница — debug строит в 3–5 раз быстрее, но получает больше кода и строк (не критично для dev, критично для users).

## 2. Что получается

После успешной сборки:

```
MyTweak/
├── .theos/
│   ├── obj/
│   │   ├── MyTweak.dylib         — финальная fat (arm64 + arm64e)
│   │   ├── arm64/
│   │   │   └── MyTweak.dylib     — slice arm64
│   │   └── arm64e/
│   │       └── MyTweak.dylib     — slice arm64e
│   └── _/                        — staging для dpkg-deb
└── packages/
    └── com.example.mytweak_0.0.1_iphoneos-arm.deb
```

Путь к .deb — `./packages/<name>_<version>_<arch>.deb`.

## 3. Инспекция .deb — что внутри

### 3.1. Содержимое

```bash
dpkg-deb -c packages/com.example.mytweak_0.0.1_iphoneos-arm.deb
```

Ожидаемый вывод для твика с filter plist (без PreferenceBundle):

```
drwxr-xr-x root/wheel  0 ... ./
drwxr-xr-x root/wheel  0 ... ./var/
drwxr-xr-x root/wheel  0 ... ./var/jb/
drwxr-xr-x root/wheel  0 ... ./var/jb/Library/
drwxr-xr-x root/wheel  0 ... ./var/jb/Library/MobileSubstrate/
drwxr-xr-x root/wheel  0 ... ./var/jb/Library/MobileSubstrate/DynamicLibraries/
-rwxr-xr-x root/wheel  ... ./var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.dylib
-rw-r--r-- root/wheel  ... ./var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.plist
```

Два важных пункта:

- **`.dylib` имеет mode `0755`** (owner-executable). Если `-rw-r--r--` — не исполнимый, dyld откажется грузить.
- **`.plist` имеет mode `0644`**.
- **owner — root:wheel**. При некорректной сборке иногда бывает `your-user:your-group`, что ломает установку.

### 3.2. control и прочие метаданные

```bash
dpkg-deb -e packages/com.example.mytweak_0.0.1_iphoneos-arm.deb /tmp/pkg-meta
ls /tmp/pkg-meta
# control
# [preinst] (если есть)
# [postinst] (если есть)
# [prerm] (если есть)
# [postrm] (если есть)
cat /tmp/pkg-meta/control
```

Ожидаемый `control`:

```
Package: com.example.mytweak
Name: MyTweak
Version: 0.0.1
Architecture: iphoneos-arm
Description: A tweak
Maintainer: Me
Author: Me
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

Проверьте, что всё, что вы писали в `control`, действительно попало сюда.

## 4. Инспекция `.dylib` — Mach-O layer

### 4.1. Fat binary и slices

```bash
file .theos/obj/MyTweak.dylib
```

Ожидаемо:

```
.theos/obj/MyTweak.dylib: Mach-O universal binary with 2 architectures: [arm64:...] [arm64e:...]
```

Если `ARCHS = arm64` только — одна архитектура. На RootHide + Dopamine это работает для A13, но для переносимости на A14+ нужно и arm64e.

Отдельная инспекция одного slice:

```bash
lipo -thin arm64 .theos/obj/MyTweak.dylib -o /tmp/MyTweak-arm64.dylib
file /tmp/MyTweak-arm64.dylib
```

### 4.2. Зависимости

```bash
otool -L .theos/obj/MyTweak.dylib
```

Ожидаемо:

```
.theos/obj/MyTweak.dylib (architecture arm64):
    /var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.dylib (compatibility version 0.0.0, current version 0.0.0)
    @rpath/libroot.dylib (compatibility version ...)
    @rpath/libellekit.dylib (compatibility version ...)
    /System/Library/Frameworks/Foundation.framework/Foundation (compatibility version ...)
    /System/Library/Frameworks/UIKit.framework/UIKit (compatibility version ...)
    /usr/lib/libobjc.A.dylib (compatibility version ...)
    /usr/lib/libSystem.B.dylib (compatibility version ...)
```

Проверочный чеклист:

| Пункт | Что должно быть |
|---|---|
| Первая строка — ваш install_name | `/var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.dylib` |
| `@rpath/libroot.dylib` | Если вы линкуете с `-lroot` |
| `@rpath/libellekit.dylib` | Если ellekit вас стыкует |
| Нет `@rpath` без rpath entry | Иначе dyld не найдёт |
| Нет странных путей `/Users/...` | Иначе где-то неправильный Makefile |

### 4.3. rpath

```bash
otool -l .theos/obj/MyTweak.dylib | grep -A2 LC_RPATH
```

Ожидаемо (для RootHide):

```
      cmd LC_RPATH
  cmdsize 24
     path /var/jb (offset 12)
```

Это rpath на `/var/jb/`, который на RootHide — симлинк в randomized jbroot (через `/basebin` → ...). При загрузке `@rpath/libroot.dylib` dyld ищет `/var/jb/libroot.dylib` (или `/var/jb/usr/lib/libroot.dylib`), что через symlink раскрывается в jbroot.

Если `LC_RPATH` вообще отсутствует — ваш твик упадёт на `@rpath/`-зависимостях.

### 4.4. install_name

```bash
otool -l .theos/obj/MyTweak.dylib | grep -A2 LC_ID_DYLIB
```

Ожидаемо:

```
      cmd LC_ID_DYLIB
  cmdsize 128
     name /var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.dylib (offset 24)
```

install_name — это «как dyld найдёт эту dylib, если её кто-то захочет загрузить по имени». Для твиков это информационное поле.

### 4.5. Подпись

```bash
codesign -dv .theos/obj/MyTweak.dylib
```

Ожидаемо:

```
Executable=/.../MyTweak.dylib
Identifier=... (adhoc)
Format=Mach-O universal (arm64 arm64e)
CodeDirectory v=... size=... flags=0x2(adhoc) hashes=... location=embedded
...
Signature=adhoc
...
```

**Ключевое — `Signature=adhoc`**. Это ad-hoc подпись (без доверенной CA), которую Dopamine делает валидной через trust cache. Если `Signature=none` — ваш `.dylib` не подписан, и dyld откажется грузить.

### 4.6. Экспортированные символы

```bash
nm -gU .theos/obj/MyTweak.dylib
```

Для твика обычно пусто или минимум. Если ваш .dylib **экспортирует** много символов — проверьте: не забыли `-fvisibility=hidden` в CFLAGS?

### 4.7. Неразрешённые символы

```bash
nm -u .theos/obj/MyTweak.dylib
```

Это список символов, которые ваш .dylib ожидает найти в своих зависимостях. Если вы видите что-то вроде `_SomeStrangeAPIYouDoNotRecognize` — проверьте: может, вы забыли линковку с нужным framework.

## 5. Проверка filter plist

```bash
plutil -p .theos/_/var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.plist
```

Ожидаемо:

```
{
  "Filter" => {
    "Bundles" => [
      0 => "com.example.testbed"
    ]
  }
}
```

Если `.plist` отсутствует в staging — вы не сможете найти проблему. Проверьте: имя файла должно быть `<TWEAK_NAME>.plist` (ровно так же, как `TWEAK_NAME` в Makefile).

## 6. Полный скрипт инспекции

Собираем все проверки в один скрипт `inspect.sh`:

```bash
#!/usr/bin/env bash
set -e
DEB="${1:?usage: inspect.sh file.deb}"
TMP=$(mktemp -d)
trap "rm -rf $TMP" EXIT

echo "=== $DEB ==="

echo
echo "--- deb contents ---"
dpkg-deb -c "$DEB"

echo
echo "--- control ---"
dpkg-deb -e "$DEB" "$TMP/meta"
cat "$TMP/meta/control"

echo
echo "--- dylib analysis ---"
dpkg-deb -x "$DEB" "$TMP/x"
for DYLIB in $(find "$TMP/x" -name "*.dylib"); do
    echo "- $DYLIB"
    file "$DYLIB" | sed 's/^/    /'
    echo "    otool -L:"
    otool -L "$DYLIB" | tail -n +2 | sed 's/^/      /'
    echo "    LC_RPATH:"
    otool -l "$DYLIB" | grep -A2 LC_RPATH | sed 's/^/      /'
    echo "    codesign:"
    codesign -dv "$DYLIB" 2>&1 | head -5 | sed 's/^/      /'
    echo "    hardcoded /var/jb:"
    N=$(strings "$DYLIB" | grep -c '/var/jb' || echo 0)
    echo "      count: $N"
    if [ "$N" -gt 0 ]; then
        strings "$DYLIB" | grep '/var/jb' | head -3 | sed 's/^/        /'
    fi
done

echo
echo "--- filter plists ---"
for PLIST in $(find "$TMP/x" -name "*.plist"); do
    echo "- $PLIST"
    plutil -p "$PLIST" | sed 's/^/    /'
done
```

Запускайте перед релизом:

```bash
chmod +x inspect.sh
./inspect.sh packages/com.example.mytweak_0.0.1_iphoneos-arm.deb > inspect-report.txt
```

Сохраняйте `inspect-report.txt` в git — это снимок состояния вашего пакета на момент релиза.

## 7. Упаковка в `.tipa`

Dopamine может принимать `.tipa` файлы — обёртку над `.deb`, которую можно устанавливать через `Dopamine.app`. Собрать .tipa — опциональный шаг:

```bash
# после make package:
# положить .deb внутрь .ipa-подобной структуры и переименовать
mkdir -p tipa-stage/Payload/MyTweak.app
cp packages/*.deb tipa-stage/Payload/MyTweak.app/debians/mytweak.deb
# (детали зависят от версии инструмента)
```

На практике для твиков `.tipa` используется редко — обычный `.deb` + Sileo/Zebra удобнее.

Если нужен конкретный путь производства .tipa — смотрите `BUILD.md` в корне репозитория Dopamine (глава I.5).

## 8. make install — через Theos

Если `THEOS_DEVICE_IP` задан:

```bash
export THEOS_DEVICE_IP=192.168.0.123
export THEOS_DEVICE_PORT=22
make package install THEOS_PACKAGE_SCHEME=roothide
```

Theos:

1. Соберёт пакет.
2. `scp` на устройство в `/tmp/_theos_install/`.
3. SSH + `dpkg -i`.
4. `killall -9 <INSTALL_TARGET_PROCESSES>`.

Для первой настройки SSH нужен ключ:

```bash
# на хосте:
ssh-keygen -t ed25519 -f ~/.ssh/mytweak-dev
# скопировать публичный ключ на устройство:
ssh-copy-id -i ~/.ssh/mytweak-dev.pub mobile@192.168.0.123
# затем в ~/.ssh/config:
Host mytweak-device
    HostName 192.168.0.123
    User mobile
    IdentityFile ~/.ssh/mytweak-dev
```

После — `make package install` работает без пароля.

## 9. Отладка «не собирается»

Таблица частых ошибок:

| Ошибка | Причина | Фикс |
|---|---|---|
| `clang: error: no such file or directory: 'Tweak.x'` | Logos не найден | Проверьте, что `.x`/`.xm` указан в `_FILES` |
| `undefined reference to _logos_orig$...` | Logos preprocessor сломался | `make clean` и пересоберите |
| `undefined reference to _CFPreferencesCopy...` | Не линкуется CoreFoundation | Добавьте `MyTweak_FRAMEWORKS = CoreFoundation` |
| `library not found for -lroot` | libroot не установлена в SDK | Theos должен иметь libroot в своих include-путях (идёт с roothide/Theos) |
| `clang: unsupported architecture arm64e` | SDK слишком старый | Обновите iOS SDK до 16+ |
| `ldid: ... Resource busy` | На устройстве ldid захватил файл | `killall -9 ldid`; редкий случай |

## 10. Отладка «собралось, но на устройстве не работает»

| Симптом | Диагностика | Решение |
|---|---|---|
| После `dpkg -i` файлы нет на устройстве | `dpkg-deb -c` показывает странные пути | Проверьте `THEOS_PACKAGE_SCHEME=roothide` |
| Файлы есть, но твик не загружается | Нет filter plist | Проверьте `<TWEAK_NAME>.plist` рядом с `.dylib` |
| Загружается, но ничего не делает | Хук на несуществующий класс | class-dump на iOS 26 — проверьте имена |
| Падает при загрузке | Dependency not found | `otool -L` на устройстве; `dpkg -l | grep <libroot>` |

Детальная диагностика — глава VII.1.

## 11. Чек-лист перед релизом

```
[ ] FINALPACKAGE=1 в сборке
[ ] THEOS_PACKAGE_SCHEME = roothide
[ ] Version в control инкрементирована
[ ] Все хуки работают на тестовом устройстве
[ ] inspect.sh не показывает WARN
[ ] /var/jb хардкода нет (strings | grep /var/jb — пусто)
[ ] codesign -dv показывает Signature=adhoc
[ ] otool -L не показывает странных зависимостей
[ ] filter plist содержит правильный Bundles/Executables
[ ] Depends в control корректен (libroot-dopamine, ellekit|...)
[ ] prerm/postrm если есть — тестирован на удаление
[ ] README/CHANGELOG обновлён (если выкладываете на репозиторий)
```

Пройдите этот список — и ваш `.deb` готов к публикации.

## 12. Что подтверждено

- **Подтверждено практикой**: `otool`/`nm`/`codesign`/`dpkg-deb` — стандартные macOS/Linux утилиты.
- **Подтверждено Theos docs**: `make package`, `FINALPACKAGE`, `THEOS_DEVICE_IP`.
- **Подтверждено Dopamine**: ad-hoc подпись + trust cache — модель, в которой работает jailbreak (глава III.11).
- **Требует проверки на устройстве**: ваш конкретный `.deb` действительно работает на целевой iOS 26.0/26.0.1 — финальный тест на реальной машине.

## 13. Выводы

Главный тезис: **собранный `.deb` без инспекции — это тест-пилот без парашюта**. Пять минут проверки через `otool`/`strings`/`plutil`/`codesign` экономят часы диагностики на устройстве.

Три команды, которые нужно запомнить:

```bash
dpkg-deb -c file.deb       # что внутри
otool -L .../dylib         # зависимости
strings .../dylib | grep /var/jb   # нет ли хардкода
```

Это ваш базовый QA. Расширенный — скрипт `inspect.sh` из раздела 6.

## 14. Что дальше

- Полный учебный проект №1 (всё, что обсуждалось в IV.1–IV.10) — глава IV.11.
- Проект №2 — SpringBoard twik — глава IV.12.
- Проект №3 — системный хук — глава IV.13.
- Патчи самого jailbreak — глава V.1.
