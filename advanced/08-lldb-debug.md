# Продвинуто 8. Отладка твика через LLDB

Хуки сложно отлаживать через `os_log` одно, иногда нужен **полноценный debugger**. На jailbroken device можно подключить LLDB к любому процессу через `debugserver`. Эта глава — как.

## 1. Что нужно

- **debugserver** — Apple-поставляемая утилита в Xcode (`/Applications/Xcode.app/Contents/.../debugserver`). На jailbroken device должна быть установлена и подписана с `com.apple.private.cs.debugger`.
- **LLDB** — на хосте macOS (из Xcode command-line tools).
- **Dopamine `ptrace_hook`** активен в debugserver (`BaseBin/systemhook/src/main.c:504`) — Dopamine прописывает debuggable флаг для любого attach'аемого процесса. Это ключевой enabler.

## 2. Установка debugserver на устройство

На iOS debugserver обычно есть в DeveloperDiskImage, но нужна copy с правильными entitlements:

### 2.1. Через Xcode (простой путь)

Подключите устройство к Mac с Xcode. Запустите любое debug build приложение → Xcode автоматически монтирует DDI и копирует debugserver.

После этого на устройстве:

```bash
ssh mobile@<ip>
ls /Developer/usr/bin/debugserver
# должен быть
```

### 2.2. Через ldid перепаковка

Если `debugserver` на устройстве без нужных entitlements:

```bash
# на хосте с Xcode:
cp /Applications/Xcode.app/Contents/SharedFrameworks/LLDB.framework/Resources/debugserver /tmp/
# ldid применить entitlements:
cat > /tmp/debugserver-ents.xml <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.springboard.debugapplications</key><true/>
    <key>run-unsigned-code</key><true/>
    <key>get-task-allow</key><true/>
    <key>task_for_pid-allow</key><true/>
    <key>com.apple.private.cs.debugger</key><true/>
    <key>platform-application</key><true/>
</dict>
</plist>
EOF
ldid -S/tmp/debugserver-ents.xml /tmp/debugserver

# Скопировать на устройство:
scp /tmp/debugserver mobile@<ip>:/var/jb/usr/bin/debugserver
ssh mobile@<ip>
sudo chmod +x /var/jb/usr/bin/debugserver
# Трастить в cache:
sudo /basebin/jbctl trustcache insert /var/jb/usr/bin/debugserver  # или через libjailbreak API
```

Теперь debugserver готов к работе.

## 3. Подключение LLDB к процессу

На устройстве:

```bash
ssh mobile@<ip>
pid=$(pgrep -f TestbedApp)
/var/jb/usr/bin/debugserver 0.0.0.0:1234 --attach=$pid
```

Debugserver слушает TCP 1234, ждёт LLDB.

На хосте:

```bash
lldb
(lldb) platform select remote-ios
(lldb) process connect connect://<device-ip>:1234
# Процесс подключён, остановлен.
```

Или в одну строку:

```bash
lldb -o "platform select remote-ios" \
     -o "process connect connect://<device-ip>:1234"
```

## 4. Базовые команды LLDB

```
(lldb) continue                    # c — продолжить
(lldb) breakpoint set --name method    # b — поставить breakpoint
(lldb) breakpoint set --name "-[MyClass method:]"    # ObjC selector
(lldb) breakpoint set -r "pattern"       # regex pattern

(lldb) image list                  # загруженные dylib
(lldb) image lookup -n some_function    # найти символ

(lldb) register read x0            # регистр x0
(lldb) register read                   # все регистры
(lldb) memory read 0x12345             # память по адресу

(lldb) po [self description]          # Objective-C print
(lldb) p someVariable                  # C print
(lldb) frame variable                   # все локальные переменные
(lldb) bt                              # stack trace

(lldb) thread list                    # потоки
(lldb) thread backtrace all           # bt всех потоков

(lldb) expr -- (void)[someView removeFromSuperview]    # выполнить Objective-C
```

## 5. Отладка хука в твике

Проблема: ваш твик в `.dylib`, которая **загружена** в процесс. Её символы — доступны в LLDB только если .dylib не strip'нута.

### 5.1. Debug build твика

Собирать с `DEBUG=1` (не `FINALPACKAGE=1`):

```bash
make clean package DEBUG=1 THEOS_PACKAGE_SCHEME=roothide
```

Это даст:

- `-O0` вместо `-O2` (нет inline).
- Без `strip` — все символы остаются.
- `-g` — DWARF debug info (может потребоваться дополнительный флаг в Makefile: `MyTweak_CFLAGS += -g`).

### 5.2. Поставить breakpoint на свой хук

```
(lldb) breakpoint set --name "_logos_method$_ungrouped$NSFileManager$fileExistsAtPath$"
```

Logos превращает `-[NSFileManager fileExistsAtPath:]` в такое длинное имя. `nm -gU` на вашей `.dylib` покажет точное имя.

Или просто искать по regex:

```
(lldb) breakpoint set -r "_logos_method.*NSFileManager"
```

Теперь при вызове хука LLDB остановит, вы увидите аргументы, можете пошагово step.

### 5.3. Пример

```
(lldb) breakpoint set -r "_logos_method.*fileExistsAtPath"
Breakpoint 1: 2 locations.

(lldb) continue
Process 1234 resuming
# в приложении какое-то действие...

Process 1234 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
   frame #0: 0x12345678 MyTweak.dylib`_logos_method$...$fileExistsAtPath$(self=0x600000c40000, _cmd="fileExistsAtPath:", path=@"/tmp/foo")

(lldb) po path
/tmp/foo

(lldb) finish    # дойти до конца хука
...
```

## 6. Отладка креша в твике

Если твик падает — LLDB attach и смотрим stack:

```
(lldb) bt
* thread #1, queue = 'com.apple.main-thread'
  * frame #0: 0x1234 MyTweak.dylib`myFunction + 20
    frame #1: 0x5678 UIKit`-[UIViewController viewDidLoad] + 123
    ...

(lldb) frame select 0
(lldb) frame variable
# все переменные frame'а
```

Это полный snapshot момента крэша.

## 7. Статический анализ без отладки

Иногда проще — без LLDB:

```bash
# Пойти в crash report:
scp mobile@<ip>:/var/mobile/Library/Logs/CrashReporter/TestbedApp-*.ips /tmp/
# Прочитать (см. главу VII.2)
```

LLDB — для live-отладки race conditions, странного поведения в рантайме.

## 8. Удобный helper-скрипт

```bash
#!/usr/bin/env bash
# attach.sh <process-name>
PROCNAME="$1"
DEVICE="${THEOS_DEVICE_IP:-192.168.1.100}"
PORT=1234

ssh mobile@$DEVICE "pkill -f debugserver 2>/dev/null; sleep 1" || true

pid=$(ssh mobile@$DEVICE "pgrep -f \"$PROCNAME\"")
if [ -z "$pid" ]; then
    echo "process not found: $PROCNAME"
    exit 1
fi

echo "Attaching to pid $pid..."
ssh mobile@$DEVICE "/var/jb/usr/bin/debugserver 0.0.0.0:$PORT --attach=$pid" &
sleep 2

lldb -o "platform select remote-ios" \
     -o "process connect connect://$DEVICE:$PORT"
```

Запуск:

```bash
./attach.sh TestbedApp
```

## 9. Ограничения

- **Нельзя** attach к процессам с `com.apple.security.app-sandbox` без AMFI-approval. Dopamine обходит через ptrace_hook в debugserver.
- **Нельзя** attach к `launchd`, `cfprefsd`, `logd` — некоторые system процессы отвергают debugger.
- **PAC** на arm64e — усложняет backtrace у system frameworks (но Apple часть символизирует).
- **Производительность** — debug build медленнее production. Не для release.

## 10. Что подтверждено

- **Подтверждено Dopamine**: ptrace_hook в systemhook активен в `debugserver` (`BaseBin/systemhook/src/main.c:504-506`).
- **Подтверждено Apple docs**: LLDB remote debugging через debugserver — стандарт для iOS development.
- **Подтверждено Theos**: `DEBUG=1` отключает strip и `-O2`.
- **Требует проверки**: конкретное местоположение debugserver на вашем устройстве — зависит от того, как установлен.

## 11. Выводы

Главный тезис: **LLDB доступен на jailbroken device для отладки твиков**. Dopamine уже обработал ptrace_hook так, чтобы любой attach проходил. Для debug build твика — все символы на месте, breakpoints работают.

Для стабильной работы — всегда FINAL build в релизе, DEBUG build для диагностики.

## 12. См. также

- Глава III.5 — systemhook и ptrace_hook.
- Глава VII.2 — crash reports.
- Advanced 9 — static analysis.
- Шпаргалка 9 — инспекционные команды.
