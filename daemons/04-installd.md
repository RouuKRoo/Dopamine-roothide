# Daemon: installd

Инсталлятор .ipa/.app. Запрещён для хуков по решению проекта Dopamine.

## 1. Что это

- **Путь**: `/usr/libexec/installd`
- **Executable name**: `installd`
- **Запуск**: launchd (`/System/Library/LaunchDaemons/com.apple.mobile.installd.plist`)
- **Привилегии**: root + entitlements для управления установленными приложениями

## 2. Что делает

- Принимает `.ipa`/`.app` для установки.
- Проверяет подпись (через AMFI).
- Распаковывает в `/private/var/containers/Bundle/Application/<UUID>/`.
- Создаёт контейнеры (`Data/Application/<UUID>/`, `Shared/`).
- Регистрирует в lsd.
- Отвечает за uninstall.

## 3. Главный framework

- `MobileInstallation.framework` — public API для `installd` клиентов (System tools).

Классы:

- `MIGlobalConfiguration` — глобальные правила.
- `MIInstallableBundle` — single bundle to install.
- `MIInstallInfo` — результат операции.

## 4. ВАЖНО: Dopamine НЕ хукает installd

Напомним главу II.12 и III.9. В `BaseBin/rootlesshooks/installd.x`:

```objc
// BOOTLOOP RISK, DO NOT TOUCH
/*%hook MIGlobalConfiguration
- (NSMutableDictionary *)_bundleIDMapForBundlesInDirectory:(NSURL *)directoryURL
                                             withExtension:(NSString *)extension
                                     loadingAdditionalKeys:(NSSet *)additionalKeys
{ ...
}
%end*/

void installdInit(void) { %init(); }
```

Хук **закомментирован**. Разработчики проекта столкнулись с bootloop при хуке `_bundleIDMapForBundlesInDirectory:` и **отказались**.

Для автора твика это:

**НЕ ХУКАТЬ installd. Серьёзно. Никогда.**

Повторите вслух: не хукать installd.

## 5. Почему это так опасно

Крах installd:

- iOS не может загрузить приложения.
- Watchdog перезапустит installd — но если ваш хук в его памяти, он снова упадёт.
- Эскалирует до kernel panic или DFU-only recovery.
- Пользователь теряет устройство.

Для сравнения — SpringBoard crash — просто respring. Installd crash — потеря данных.

## 6. Что делать вместо хуков

Если вам кажется, что нужны хуки в installd — почти всегда есть **альтернатива**:

| Хочу | Альтернатива |
|---|---|
| Логировать установку app | Hook `LSApplicationWorkspace` в SpringBoard |
| Модифицировать bundle перед install | Pre-process `.ipa` до передачи installd |
| Изменить install path | Impossible, да и нежелательно |
| Поймать «новое приложение установлено» | `LSApplicationProxy` KVO или Darwin notification |

## 7. Как установленно приложение на jailbroken device

Для автора вспомогательной утилиты, которая **ставит** приложение:

- CLI: `installd` doesn't have public CLI; обычно через `ideviceinstaller` (libimobiledevice) или через Sileo/Zebra для .deb-based.
- Через `MobileInstallation.framework` API — требует entitlements.

Если ваш твик хочет уcтановить приложение (например, companion .app) — делает это через `.deb` с `_INSTALL_PATH = /Applications` и `uicache -a` в postinst (глава II.12).

## 8. Логирование установок — через lsd, не installd

Для автора твика:

```objc
// в SpringBoard tweak:
%hook LSApplicationWorkspace

- (BOOL)installApplication:(NSString *)applicationIdentifier
                withOptions:(NSDictionary *)options
                      error:(NSError **)error
{
    BOOL r = %orig(applicationIdentifier, options, error);
    os_log(gLog, "LSApplicationWorkspace installApplication: %{public}@ → %d",
           applicationIdentifier, (int)r);
    return r;
}

%end
```

Это **безопасно**: `LSApplicationWorkspace` — public-ish API, многократно используется.

## 9. Проверка «можно ли вообще дёрнуть installd как клиент»

Из своей app:

```swift
import MobileInstallation  // undocumented API

// Отправить query — просто чтобы увидеть, что получится:
let info = MICreateDictionaryWithQueryKeys(queryKeys)
```

Для production — не стоит. Для research — можно, через LLDB attached к процессу с правильными entitlements.

## 10. Что подтверждено

- **Подтверждено исходниками**: `installd` не хукается активно в Dopamine (`BaseBin/rootlesshooks/installd.x:1–24`).
- **Подтверждено комментарием в коде**: «BOOTLOOP RISK, DO NOT TOUCH».
- **Этический вывод**: автор твика повторяет это решение.

## 11. Выводы

Главный тезис: **installd — табу**. Даже разработчики Dopamine отступили. Для автора твика — всегда альтернативы (lsd, LSApplicationWorkspace, Darwin notifications).

## 12. См. также

- Глава II.12 — установка приложений.
- Глава III.9 — rootless/roothidehooks.
- Daemon: lsd — следующая глава.
- Шпаргалка 3 — control file (`maintainer scripts` для integration).
