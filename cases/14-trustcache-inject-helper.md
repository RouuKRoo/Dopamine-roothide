# Кейс 14. Собственный helper, вносящий cdhash в trustcache

Задача: отдельный helper-бинарник (CLI-утилита или daemon), который получает путь к `.dylib` и просит Dopamine добавить его cdhash в dynamic trust cache. Удобно для:

- Инструментов, которые загружают произвольные user-provided dylib (plugin loader вашего приложения).
- Dev-итерационных сценариев.
- Сторонних менеджеров twик-пакетов, которые хотят интегрироваться с trust cache Dopamine.

## 1. Как это работает

Dopamine предоставляет `jbclient_trust_file_by_path(const char *path)` в `libjailbreak`. Эта функция XPC-вызывает jbserver, который считает cdhash файла и добавляет его в dynamic trust cache ядра. После этого AMFI воспринимает файл как «подписанный».

Для автора твика — вот и всё. Один вызов, одна строка.

## 2. Проект: `jbtrust-cli`

Небольшая CLI утилита. Принимает путь к файлу, доверяет его.

### 2.1. Файловое дерево

```
jbtrust-cli/
├── Makefile
├── jbtrust.c
└── layout/
    └── DEBIAN/
        └── control
```

### 2.2. jbtrust.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <libjailbreak/jbclient_xpc.h>

static void usage(const char *prog) {
    fprintf(stderr, "usage: %s <file>\n", prog);
    fprintf(stderr, "  Add cdhash of <file> to Dopamine dynamic trust cache.\n");
}

int main(int argc, char **argv) {
    if (argc < 2) {
        usage(argv[0]);
        return 2;
    }

    const char *path = argv[1];
    if (access(path, R_OK) != 0) {
        fprintf(stderr, "error: cannot read %s\n", path);
        return 1;
    }

    int ret = jbclient_trust_file_by_path(path);
    if (ret == 0) {
        printf("ok: %s is now trusted\n", path);
        return 0;
    } else {
        fprintf(stderr, "error: trust_file_by_path returned %d\n", ret);
        return 1;
    }
}
```

Разбор:

- `access(path, R_OK)` — предварительная проверка, что файл читаем.
- `jbclient_trust_file_by_path(path)` — главная строка. Делает XPC через jbserver.
- Возврат: 0 успех, иное — error code.

### 2.3. Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TOOL_NAME = jbtrust

jbtrust_FILES = jbtrust.c
jbtrust_CFLAGS = -fobjc-arc
jbtrust_LIBRARIES = jailbreak root
jbtrust_INSTALL_PATH = /usr/local/bin

include $(THEOS_MAKE_PATH)/tool.mk
```

Ключевое:

- `TOOL_NAME` вместо `TWEAK_NAME` — собираем CLI.
- `tool.mk` вместо `tweak.mk`.
- `_LIBRARIES = jailbreak root` — libjailbreak для jbclient, libroot для путей.
- `_INSTALL_PATH = /usr/local/bin` — куда класть в .deb.

### 2.4. control

```
Package: com.example.jbtrust
Name: jbtrust
Version: 0.0.1
Architecture: iphoneos-arm
Description: CLI to trust files via Dopamine dynamic trust cache
Maintainer: You
Author: You
Section: Development
Depends: firmware (>= 15.0), libroot-dopamine
```

### 2.5. Сборка и установка

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide FINALPACKAGE=1
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/com.example.jbtrust_*.deb"
```

### 2.6. Использование на устройстве

```bash
ssh mobile@<ip>
# Произвольный .dylib, который нужно доверить:
jbtrust /var/mobile/Documents/myplugin.dylib
# ok: /var/mobile/Documents/myplugin.dylib is now trusted

# Теперь любой процесс может dlopen его:
```

## 3. Расширение: helper-процесс

Задача: ваше приложение хочет загружать plugin .dylib из своего контейнера. Своё приложение само может вызывать `jbclient_trust_file_by_path`, но полезно вынести в helper, если:

- Плагин грузит отдельный demon-процесс.
- Хочется централизовать логику подписей/верификации.

### 3.1. Daemon version

```c
// jbtrust-daemon.c
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <sys/stat.h>
#include <libjailbreak/jbclient_xpc.h>
#include <libroot.h>

// Простой watcher-daemon: читает "команды" из FIFO и trust'ит.

#define FIFO_PATH_SUFFIX "/var/mobile/Documents/.jbtrust-fifo"

int main(void) {
    char fifoPath[PATH_MAX];
    snprintf(fifoPath, sizeof(fifoPath), "%s%s",
             libroot_get_jbroot_prefix(), FIFO_PATH_SUFFIX);

    if (mkfifo(fifoPath, 0600) != 0 && errno != EEXIST) {
        fprintf(stderr, "mkfifo failed: %s\n", strerror(errno));
        return 1;
    }

    printf("jbtrust-daemon: listening on %s\n", fifoPath);

    while (1) {
        FILE *f = fopen(fifoPath, "r");
        if (!f) {
            sleep(1);
            continue;
        }

        char path[PATH_MAX];
        while (fgets(path, sizeof(path), f)) {
            // Удалить trailing newline
            size_t len = strlen(path);
            if (len > 0 && path[len-1] == '\n') path[len-1] = '\0';
            if (strlen(path) == 0) continue;

            if (access(path, R_OK) != 0) {
                fprintf(stderr, "jbtrust-daemon: cannot read %s\n", path);
                continue;
            }

            int r = jbclient_trust_file_by_path(path);
            printf("jbtrust-daemon: %s → %d\n", path, r);
        }

        fclose(f);
    }

    return 0;
}
```

Разбор:

- Создаёт FIFO в jbroot-пути.
- Loop: читать строки, доверять файлы.
- Любой клиент (ваше приложение) записывает в FIFO путь к `.dylib` → helper доверяет.

### 3.2. launchd plist

Чтобы helper запускался при userspace boot — положить в `/Library/LaunchDaemons/`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.jbtrust-daemon</string>
    <key>Program</key>
    <string>/usr/local/bin/jbtrust-daemon</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/var/mobile/Documents/jbtrust-daemon.log</string>
    <key>StandardErrorPath</key>
    <string>/var/mobile/Documents/jbtrust-daemon.err</string>
</dict>
</plist>
```

Поместить в `/var/jb/Library/LaunchDaemons/com.example.jbtrust-daemon.plist` (Theos сделает автоматом, если проект содержит эту структуру).

### 3.3. Клиент из своего приложения

```c
#import <libroot.h>
#import <Foundation/Foundation.h>

- (void)trustPluginAtPath:(NSString *)path {
    NSString *fifoPath = [NSString stringWithFormat:@"%s/var/mobile/Documents/.jbtrust-fifo",
                                                      libroot_get_jbroot_prefix()];
    NSString *msg = [path stringByAppendingString:@"\n"];
    NSFileHandle *fh = [NSFileHandle fileHandleForWritingAtPath:fifoPath];
    [fh writeData:[msg dataUsingEncoding:NSUTF8StringEncoding]];
    [fh closeFile];
}
```

После вызова — ваш helper обработает, trust'ит, можно `dlopen`-уть.

## 4. Безопасность

### 4.1. Что проверять в helper'е

Daemon-helper с правами trust cache — повышенная ответственность. Проверьте:

- **Путь** находится в sandbox-разрешённой зоне (ваш контейнер приложения, не произвольно).
- **Файл принадлежит владельцу приложения** (не root).
- **Размер и cdhash логируются** — для audit trail.

Пример расширенной проверки:

```c
#include <sys/stat.h>

int validate_path(const char *path) {
    struct stat st;
    if (stat(path, &st) != 0) return -1;

    // Файл должен быть обычным
    if (!S_ISREG(st.st_mode)) return -1;

    // Размер разумный (< 100 MB)
    if (st.st_size > 100 * 1024 * 1024) return -1;

    // Путь ограничен определённой зоной
    const char *allowed_prefix = "/var/mobile/Containers/Data/Application/";
    if (strncmp(path, allowed_prefix, strlen(allowed_prefix)) != 0) {
        return -1;
    }

    return 0;
}
```

Вызовите `validate_path(path)` в helper до `jbclient_trust_file_by_path`.

### 4.2. Этически

Этот helper позволяет вам доверять произвольные dylib. Это тот же механизм, через который Dopamine добавляет свои cdhash. Применяйте только:

- Для dylib **собственного** авторства / проекта.
- Не для «mod loader» произвольных чужих файлов.

## 5. Полный маршрут от сборки до работы

```bash
# На хосте:
cd jbtrust-cli
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/

cd ../jbtrust-daemon      # отдельный проект
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/

# На устройстве:
ssh mobile@<ip>
sudo dpkg -i /tmp/com.example.jbtrust_*.deb
sudo dpkg -i /tmp/com.example.jbtrust-daemon_*.deb

# Daemon сам стартует через launchd:
launchctl list | grep jbtrust

# Тест:
jbtrust /var/mobile/Documents/test.dylib
# → ok

# Или через FIFO:
echo "/var/mobile/Documents/test.dylib" > /var/jb/var/mobile/Documents/.jbtrust-fifo
# daemon логирует в /var/mobile/Documents/jbtrust-daemon.log
```

## 6. Что подтверждено

- **Подтверждено**: `jbclient_trust_file_by_path` — публичный API libjailbreak (`BaseBin/libjailbreak/src/jbclient_xpc.h`).
- **Подтверждено**: доступен из SYSTEMWIDE домена, т. е. из любого процесса (глава III.2 и шпаргалка 7).
- **Подтверждено Theos**: `TOOL_NAME` + `tool.mk` для CLI утилит.
- **Требует проверки на устройстве**: что `jbclient_trust_file_by_path` работает без дополнительных entitlements для вашего инструмента — обычно да, но проверьте.
- **Не подтверждено**: что FIFO-подход — лучший для вашего use-case. Для production рассмотрите XPC или mach ports с правильной authentication.

## 7. Выводы

Главный тезис: **trust cache доступен из любого процесса** — через `jbclient_trust_file_by_path`. Один вызов — файл доверен.

Для твика, который должен грузить произвольные dylib — это самый прямой путь. CLI утилита + helper daemon даёт flexible архитектуру для интеграции с другими инструментами.

## 8. См. также

- Deep-dive 05 — trust cache в деталях.
- Глава III.2 — libjailbreak публичное API.
- Шпаргалка 6 — публичные функции libjailbreak.
- Кейс 15 — безопасный uninstall.
