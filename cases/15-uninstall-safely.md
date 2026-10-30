# Кейс 15. Безопасный uninstall твика без доступа к устройству

Пользователь установил ваш твик, он сломал систему или просто ему разонравился. Задача: обеспечить, чтобы `dpkg -r` чисто откатил **всё**, что твик наставил, без ручной работы на устройстве.

Этот кейс — финальный в разделе case-studies. Он не про новый код, а про **правильную гигиену** установщика.

## 1. Что остаётся после `dpkg -r` без заботы

Если вы не позаботились о чистом uninstall, после `dpkg -r com.example.mytweak`:

- **`.dylib`, `.plist`, bundle'ы** удаляются (их dpkg знает).
- **Preferences `.plist`** в cfprefsd — **остаётся** (dpkg не знает про них).
- **Файл-маркеры** (`.cache`, `.state`), созданные самим твиком — **остаются**.
- **Launchd plist** вашего daemon'а в `/Library/LaunchDaemons/` — **удаляется** если был в .deb; но demon может продолжить работать до перезагрузки.
- **Darwin notification subscriptions** — ваших нет, но для чистоты важно.
- **Иконки приложения** (если ваш пакет включал .app) — требуется `uicache -a` для их обновления.

Задача maintainer scripts (`preinst`/`postinst`/`prerm`/`postrm`) — закрыть эти пробелы.

## 2. Полный set скриптов

### 2.1. postinst (после установки)

```bash
#!/bin/bash
# postinst: вызывается после распаковки файлов.
# $1 = "configure" (новая установка или upgrade)

case "$1" in
    configure)
        # 1. Регистрация приложения, если ваш пакет его содержит
        if [ -d /var/jb/Applications/MyApp.app ]; then
            /var/jb/usr/bin/uicache -a
        fi

        # 2. Запустить daemon, если есть
        if [ -f /var/jb/Library/LaunchDaemons/com.example.mytweak.plist ]; then
            launchctl bootstrap system /var/jb/Library/LaunchDaemons/com.example.mytweak.plist 2>/dev/null \
                || launchctl load /var/jb/Library/LaunchDaemons/com.example.mytweak.plist 2>/dev/null
        fi

        # 3. Перезапустить cfprefsd, чтобы подхватил новые preferences
        killall -9 cfprefsd 2>/dev/null || true

        # 4. (опционально) Респринг — если твик для SpringBoard
        # killall -9 SpringBoard 2>/dev/null || true
        ;;
    *)
        ;;
esac

exit 0
```

### 2.2. prerm (до удаления)

```bash
#!/bin/bash
# prerm: вызывается до удаления файлов.
# $1 = "remove" или "upgrade"

case "$1" in
    remove|upgrade|deconfigure)
        # 1. Остановить daemon если есть
        if [ -f /var/jb/Library/LaunchDaemons/com.example.mytweak.plist ]; then
            launchctl bootout system /var/jb/Library/LaunchDaemons/com.example.mytweak.plist 2>/dev/null \
                || launchctl unload /var/jb/Library/LaunchDaemons/com.example.mytweak.plist 2>/dev/null
        fi

        # 2. (опционально) Отправить Darwin notification, чтобы твик знал о выключении
        /var/jb/usr/bin/notifyutil -p "com.example.mytweak.shutting-down" 2>/dev/null || true
        ;;
    *)
        ;;
esac

exit 0
```

### 2.3. postrm (после удаления)

```bash
#!/bin/bash
# postrm: вызывается после удаления файлов.
# $1 = "remove" (чистая деинсталляция) или "purge" (полная очистка)

case "$1" in
    purge)
        # Удалить всё, включая настройки и state

        # 1. Settings .plist (cfprefsd)
        rm -f /var/jb/var/mobile/Library/Preferences/com.example.mytweak.plist 2>/dev/null
        # Для rootless-redirect путь обычно такой:
        rm -f /var/mobile/Library/Preferences/com.example.mytweak.plist 2>/dev/null

        # 2. State-файлы вашего твика в контейнере приложения — нет доступа отсюда (sandboxed)
        # Пользователь удалит сам через настройки приложения.

        # 3. Логи
        rm -f /var/jb/var/mobile/Library/Logs/MyTweak.log 2>/dev/null

        # 4. Файлы в /tmp
        rm -rf /tmp/mytweak-* 2>/dev/null

        # 5. Если был Dopamine notification lock
        rm -f /var/jb/var/mobile/.mytweak-lock 2>/dev/null
        ;;
    remove)
        # Чистая деинсталляция — настройки оставляем на случай переустановки
        ;;
    upgrade)
        # Upgrade — вообще ничего не удаляем
        ;;
    *)
        ;;
esac

# 6. Перезапустить cfprefsd для подхвата изменений
killall -9 cfprefsd 2>/dev/null || true

# 7. Если ваш пакет включал приложение
if [ "$1" = "purge" ]; then
    /var/jb/usr/bin/uicache -a 2>/dev/null || true
fi

exit 0
```

### 2.4. preinst (до установки)

Обычно не нужен, но на всякий:

```bash
#!/bin/bash
# preinst: до распаковки файлов.
# $1 = "install" или "upgrade"

case "$1" in
    upgrade)
        # Остановить daemon перед заменой файлов
        if [ -f /var/jb/Library/LaunchDaemons/com.example.mytweak.plist ]; then
            launchctl bootout system /var/jb/Library/LaunchDaemons/com.example.mytweak.plist 2>/dev/null
        fi
        ;;
    install)
        # Проверить требования — например, что ellekit установлен
        if [ ! -f /var/jb/usr/lib/TweakLoader.dylib ]; then
            echo "warning: no TweakLoader found — install ellekit first"
        fi
        ;;
    *)
        ;;
esac

exit 0
```

## 3. Как подключить скрипты в Theos

Положить в проекте:

```
MyTweak/
├── Makefile
├── Tweak.x
├── MyTweak.plist
├── control
└── layout/
    └── DEBIAN/
        ├── preinst       (chmod +x)
        ├── postinst      (chmod +x)
        ├── prerm         (chmod +x)
        └── postrm        (chmod +x)
```

Theos автоматически добавляет их в `.deb`. Проверить:

```bash
dpkg-deb -I packages/*.deb
# Выведет control + список maintainer scripts
```

## 4. Что проверить

### 4.1. Чистая установка

```bash
sudo dpkg -i your.deb
# postinst отработал? daemon запущен?
launchctl list | grep com.example.mytweak

# Файлы на месте?
ls /var/jb/Library/MobileSubstrate/DynamicLibraries/
```

### 4.2. Апгрейд

```bash
sudo dpkg -i your-new-version.deb
# preinst (upgrade) → остановил daemon
# файлы заменены
# postinst (configure) → daemon запущен снова
```

### 4.3. Чистая деинсталляция

```bash
sudo dpkg -r com.example.mytweak
# prerm → daemon остановлен
# файлы удалены
# postrm (remove) → cfprefsd restarted, preferences сохранены

# Settings .plist остался (ожидаемо):
ls /var/jb/var/mobile/Library/Preferences/com.example.mytweak.plist
```

### 4.4. Полная очистка

```bash
sudo dpkg --purge com.example.mytweak
# то же + postrm (purge) → preferences удалены

# Проверить — пусто:
ls /var/jb/var/mobile/Library/Preferences/com.example.mytweak.plist 2>/dev/null
# → No such file
```

## 5. Автотест uninstall

Простой скрипт для CI:

```bash
#!/usr/bin/env bash
set -e

DEB="$1"
DEVICE="${THEOS_DEVICE_IP:-192.168.1.123}"
PKG=$(dpkg-deb -I "$DEB" | awk '/^ Package:/ {print $2}')

echo "=== Install test ==="
scp "$DEB" "mobile@$DEVICE:/tmp/"
ssh "mobile@$DEVICE" "sudo dpkg -i /tmp/$(basename $DEB)"

echo "=== Verify files installed ==="
ssh "mobile@$DEVICE" "ls -l /var/jb/Library/MobileSubstrate/DynamicLibraries/" | grep "$PKG"

echo "=== Uninstall ==="
ssh "mobile@$DEVICE" "sudo dpkg -r $PKG"

echo "=== Verify files gone ==="
if ssh "mobile@$DEVICE" "ls /var/jb/Library/MobileSubstrate/DynamicLibraries/" | grep -q "${PKG##com.example.}"; then
    echo "FAIL: dylib still present"
    exit 1
fi

echo "=== Purge ==="
ssh "mobile@$DEVICE" "sudo dpkg --purge $PKG"

echo "=== Verify preferences gone ==="
if ssh "mobile@$DEVICE" "ls /var/jb/var/mobile/Library/Preferences/${PKG}.plist" 2>/dev/null; then
    echo "FAIL: preferences still present after purge"
    exit 1
fi

echo "=== OK ==="
```

Запускайте этот тест перед каждым релизом в CI pipeline.

## 6. Советы по user experience

### 6.1. Сообщить пользователю, что произойдёт

В Description пакета можно добавить:

```
Description: Short description
 This tweak modifies X behavior in Y app.
 .
 Settings: Open Settings.app → MyTweak to configure.
 .
 Uninstall: dpkg -r com.example.mytweak removes files but keeps your
 preferences. To fully remove including preferences, use
 "dpkg --purge com.example.mytweak".
```

### 6.2. Сохранение данных

Если ваш твик хранит важные данные пользователя (не настройки, а, скажем, записи):

- При `remove` — **сохраните** (пользователь может переустановить).
- При `purge` — **удалите**.

Пользователю понятно: `remove` — временно, `purge` — навсегда.

### 6.3. Логирование uninstall

В postrm добавьте:

```bash
echo "$(date): mytweak $1" >> /var/jb/var/log/mytweak-uninstall.log
```

Это помогает диагностировать «пользователь удалил и переустановил, но что-то не работает».

## 7. Таблица «что в каком скрипте»

| Операция | preinst | postinst | prerm | postrm |
|---|---|---|---|---|
| Проверить зависимости | install | — | — | — |
| Остановить daemon | upgrade | — | remove/upgrade | — |
| Запустить daemon | — | configure | — | — |
| Регистрация в LaunchServices (uicache) | — | configure | — | purge |
| Перезапустить cfprefsd | — | configure | — | always |
| Удалить preferences | — | — | — | purge |
| Удалить state-файлы | — | — | — | purge |
| Респринг SpringBoard | — | configure (если нужен) | — | — |
| Darwin notification клиентам | — | configure (settings-changed) | remove (shutting-down) | — |

## 8. Распространённые ошибки

| Ошибка | Решение |
|---|---|
| Скрипт не исполняется | `chmod +x layout/DEBIAN/*` |
| Daemon остался в памяти | `launchctl bootout` в prerm |
| Preferences не удалены при purge | Explicit `rm` в postrm |
| Twic продолжает работать после uninstall | Респринг или kill target-процесса |
| Иконка приложения осталась | `uicache -a` в postrm/purge |
| Нужны права root в скрипте | dpkg вызывает скрипты от root — всё ок |

## 9. Что подтверждено

- **Подтверждено Debian Policy**: 4 maintainer script (preinst, postinst, prerm, postrm) — стандарт.
- **Подтверждено Theos**: `layout/DEBIAN/*` структура — автоматически добавляется в .deb.
- **Подтверждено**: `launchctl bootstrap`/`bootout` — iOS 11+, рекомендованные команды.
- **Требует проверки на устройстве**: что `uicache -a` на RootHide находится по `/var/jb/usr/bin/uicache` — это зависит от того, где установлен пакет (на RootHide через симлинк).
- **Не подтверждено**: что все пути в скриптах корректны для randomized jbroot — на RootHide dpkg должен справиться через symlink `/var/jb → <jbroot>`, но проверяйте на реальной установке.

## 10. Выводы

Главный тезис: **чистый uninstall — это ответственность автора твика**. maintainer scripts — ваша возможность не оставлять мусор на устройстве пользователя.

Три якорных правила:

1. **Preferences при `remove` сохраняются**, при `purge` — удаляются. Это стандартная конвенция Debian.
2. **Daemon останавливается в prerm**, не оставляется работающим.
3. **cfprefsd перезапускается в postinst/postrm** — чтобы изменения подхватились немедленно.

Этот кейс закрывает секцию case-studies. 15 кейсов — набор рабочих паттернов, из которых можно собирать свои твики.

## 11. См. также

- Debian Policy Manual, Chapter 10.
- Шпаргалка 3 — control-файл.
- Глава IV.10 — сборка и инспекция.
- Кейс 3 — rebootable launcher daemon.
