# Продвинуто 5. Собственный jbserver-клиент

Для твика, который хочет работать с публичными jbserver-доменами (`SYSTEMWIDE`, в особых случаях `PLATFORM`) — через стандартный `jbclient_*` API из `libjailbreak`. Эта глава — pattern'ы и подводные камни.

## 1. Что доступно

Напомним из главы III.2 и шпаргалки 7:

- **SYSTEMWIDE** — все процессы: `get_jbroot`, `trust_file_by_path`, `cs_revalidate`, `jbsettings_get`.
- **PLATFORM** — платформенные процессы: `jbsettings_set`, `set_process_debugged`.
- **ROOT** — root-процессы: kernel primitives.

Для 95% твиков — SYSTEMWIDE. Это глава про те 5%, когда нужно больше.

## 2. Простейший клиент (SYSTEMWIDE)

Из обычного твика:

```c
#include <libjailbreak/jbclient_xpc.h>

%ctor {
    char *jbroot = jbclient_get_jbroot();
    if (jbroot) {
        os_log(gLog, "jbroot=%s", jbroot);
        free(jbroot);
    }

    char *bootUUID = jbclient_get_boot_uuid();
    if (bootUUID) {
        os_log(gLog, "boot UUID=%s", bootUUID);
        free(bootUUID);
    }

    // Trust свою вспомогательную .dylib:
    int r = jbclient_trust_file_by_path(JBROOT_PATH("/Library/MyTweak/plugin.dylib"));
    if (r == 0) {
        os_log(gLog, "trusted ok");
    } else {
        os_log_error(gLog, "trust failed: %d", r);
    }
}
```

Это **готовое** решение. Linking: `MyTweak_LIBRARIES = root` (re-exports libroothider, которая re-exports libjailbreak).

## 3. Доступ к PLATFORM-домену

Требования:

- Процесс должен иметь `platform-application` entitlement, или
- Иметь CS_PLATFORMIZED flag в codesign, или
- Быть сам Dopamine.app.

Обычный tweak-container этого **не имеет**. Как получить:

### 3.1. Через helper-бинарник с ad-hoc entitlements

Собрать отдельный tool (не tweak!) с ad-hoc подписью и правильным entitlements:

```xml
<!-- helper.entitlements -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>platform-application</key>
    <true/>
    <key>com.apple.private.security.container-required</key>
    <false/>
</dict>
</plist>
```

```make
# Makefile для helper
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TOOL_NAME = my-helper
my-helper_FILES = helper.c
my-helper_CFLAGS = -fobjc-arc
my-helper_LIBRARIES = jailbreak root
my-helper_INSTALL_PATH = /usr/local/bin
my-helper_CODESIGN_FLAGS = -Shelper.entitlements

include $(THEOS_MAKE_PATH)/tool.mk
```

Для того чтобы entitlements приняли:

1. ldid подпишет с указанным XML.
2. Dopamine добавит cdhash в trust cache при установке .deb.
3. AMFI увидит entitlements как валидные.

### 3.2. Код helper'а

```c
// helper.c
#include <stdio.h>
#include <stdlib.h>
#include <libjailbreak/jbclient_xpc.h>

int main(int argc, char **argv) {
    if (argc < 3) {
        fprintf(stderr, "usage: %s <key> <value>\n", argv[0]);
        return 1;
    }

    const char *key = argv[1];
    bool value = (atoi(argv[2]) != 0);

    int r = jbclient_platform_jbsettings_set_bool(key, value);
    if (r == 0) {
        printf("ok\n");
        return 0;
    } else {
        fprintf(stderr, "failed: %d\n", r);
        return 1;
    }
}
```

Запуск с устройства:

```bash
ssh mobile@<ip>
sudo my-helper tweaks-enabled 0
# → ok
```

### 3.3. Твик → helper → jbserver

Из твика (который сам не имеет platform-прав) запустить helper:

```objc
#import <spawn.h>

- (void)disableTweaksViaHelper {
    char *argv[] = { "my-helper", "tweaks-enabled", "0", NULL };
    pid_t pid;
    posix_spawn(&pid, "/var/jb/usr/local/bin/my-helper",
                NULL, NULL, argv, environ);
    waitpid(pid, NULL, 0);
}
```

Twic → spawn helper → helper имеет platform entitlement → jbserver принимает запрос PLATFORM-домена.

## 4. Нюанс: что проверяет jbserver

Из `BaseBin/launchdhook/src/jbserver/jbdomain_platform.c` (концептуально):

1. Из mach-сообщения достать `audit_token_t`.
2. Из токена — PID + codesign flags.
3. Проверить:
   - CS_PLATFORMIZED в flags, или
   - Entitlement `platform-application` = true.
4. Если OK — выполнить action; иначе — return `-EPERM`.

Эта проверка — на стороне **сервера** (launchdhook). Клиент не может её обойти — максимум подать запрос с правильными credentials.

## 5. Чего нельзя

| Нельзя | Почему |
|---|---|
| Вызывать `jbclient_root_*` из обычного твика | ROOT домен требует uid=0, не выполняется |
| Создать новый домен | Это код внутри `launchdhook/jbserver/`, то есть part of Dopamine |
| Обойти audit_token проверку | Mach-level, не фальсифицируется |
| Пропустить trust cache | Ваш helper без cdhash в tc не загрузится вообще |

## 6. Edge case: TrollStore + jbclient

Если ваш helper запускается через TrollStore (persistent signing), а не из jailbreak-deb:

- TrollStore может прописать `platform-application` в его подписи.
- Запущенный через TrollStore процесс имеет этот entitlement.
- `jbclient_platform_*` вызовы работают.

Это полезно для «admin-utility» приложений, которые хотят управлять jailbreak-настройками.

## 7. mach vs XPC транспорт

В `libjailbreak`:

- `jbclient_xpc.h` — XPC transport, самый стандартный.
- `jbclient_mach.h` — mach messages напрямую, для раннего boot (используется `dyldhook`).

Для автора твика — всегда XPC. Mach — внутренняя магия.

## 8. Пример полного helper'а

Файл `jbsettings-cli.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <libjailbreak/jbclient_xpc.h>

static void usage(void) {
    fprintf(stderr,
            "usage:\n"
            "  jbsettings-cli get-bool <key>\n"
            "  jbsettings-cli set-bool <key> <0|1>\n"
            "  jbsettings-cli get-int  <key>\n"
            "  jbsettings-cli set-int  <key> <value>\n");
}

int main(int argc, char **argv) {
    if (argc < 3) { usage(); return 1; }

    const char *cmd = argv[1];
    const char *key = argv[2];

    if (strcmp(cmd, "get-bool") == 0) {
        bool v = jbclient_jbsettings_get_bool(key);
        printf("%d\n", v);
        return 0;
    }
    if (strcmp(cmd, "set-bool") == 0) {
        if (argc < 4) { usage(); return 1; }
        bool v = (atoi(argv[3]) != 0);
        int r = jbclient_platform_jbsettings_set_bool(key, v);
        if (r != 0) { fprintf(stderr, "error: %d\n", r); return 1; }
        return 0;
    }
    if (strcmp(cmd, "get-int") == 0) {
        uint64_t v = jbclient_jbsettings_get_uint64(key);
        printf("%llu\n", v);
        return 0;
    }
    if (strcmp(cmd, "set-int") == 0) {
        if (argc < 4) { usage(); return 1; }
        uint64_t v = strtoull(argv[3], NULL, 0);
        int r = jbclient_platform_jbsettings_set_uint64(key, v);
        if (r != 0) { fprintf(stderr, "error: %d\n", r); return 1; }
        return 0;
    }

    usage();
    return 1;
}
```

Полноценная CLI утилита для jbsettings. Установите с правильным entitlement — и используете из скриптов.

## 9. Что подтверждено

- **Подтверждено**: все API в `BaseBin/libjailbreak/src/jbclient_xpc.h`.
- **Подтверждено**: entitlements example — `BaseBin/hookd/entitlements.plist`, `BaseBin/jailbreakd/entitlements.plist` (глава III.7, III.8).
- **Подтверждено**: `platform-application` даёт доступ к PLATFORM-домену (глава III.2).
- **Требует проверки**: что ваш helper с ad-hoc подписью и custom entitlements реально принимается — зависит от правильного попадания в trust cache через dpkg-хуки Dopamine.

## 10. Выводы

Главный тезис: **для большинства задач SYSTEMWIDE-домен достаточен**. PLATFORM-домен — для специализированных helper-утилит с правильной подписью.

Не пытайтесь обойти audit_token проверку — это уровень mach-security, за который Apple следит очень внимательно.

## 11. См. также

- Глава III.2 — libjailbreak.
- Глава III.4 — launchdhook (где серверная часть).
- Шпаргалка 6 — публичные функции libjailbreak.
- Шпаргалка 7 — jbserver домены.
- Кейс 14 — trustcache inject helper.
