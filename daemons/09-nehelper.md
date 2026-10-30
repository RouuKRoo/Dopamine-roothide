# Daemon: nehelper (Network Extension helper)

Привилегированный помощник для `NetworkExtension.framework`. Приложения с VPN/Content Filter проходят через него.

## 1. Что это

- **Путь**: `/usr/libexec/nehelper`
- **Executable name**: `nehelper`
- **Запуск**: launchd on-demand
- **Привилегии**: root + VPN config entitlements

## 2. Что делает

- Создание/запуск VPN tunnels для `NEVPNManager`.
- Регистрация/запуск Content Filter (`NEFilterManager`).
- Запуск DNS Proxy (`NEDNSProxyManager`).
- Хранит NetworkExtension configurations в `/private/var/preferences/com.apple.networkextension.plist`.

## 3. Framework

- `NetworkExtension.framework` — public клиент API.
- `NEHelperClient.framework` — private для `nehelper` протокола.

## 4. Хуки Dopamine

**Нет прямых** хуков `nehelper`. Но помните (глава II.9, III.5):

- `systemhook` в **каждом процессе** хукает NECP syscalls через `litehook_hook_function`.
- NECP — kernel subsystem, которую nehelper использует.

То есть процессы, использующие NetworkExtension, пройдут через правильно-повторно-подписанный NECP, но nehelper сам не трогается.

## 5. Что хукать

### 5.1. Public NetworkExtension API в приложении

```objc
%hook NEVPNManager

+ (NEVPNManager *)sharedManager {
    NEVPNManager *m = %orig;
    os_log(gLog, "NEVPNManager sharedManager: %p", m);
    return m;
}

%end

%hook NEVPNConnection

- (BOOL)startVPNTunnelAndReturnError:(NSError **)error {
    os_log(gLog, "startVPNTunnel");
    return %orig(error);
}

%end
```

Это **в приложении-клиенте**, не в nehelper. Безопасно.

### 5.2. NetworkExtension providers (ваш `.appex`)

Если ваше приложение имеет `NEPacketTunnelProvider` — hook его методы в `.appex`:

```objc
%hook YourTunnelProvider

- (void)startTunnelWithOptions:(NSDictionary *)options
             completionHandler:(void (^)(NSError *))completionHandler
{
    os_log(gLog, "startTunnel: options=%{public}@", options);
    %orig(options, completionHandler);
}

%end
```

Filter plist — на bundle id `.appex`.

## 6. Что хукать нельзя

### 6.1. nehelper сам

Platform-restricted. Инъекция туда — специализированное, обычно не стоит.

### 6.2. kernel NECP syscalls

Dopamine уже хукает (`systemhook/main.c:128–156`). Не дублируйте.

### 6.3. Private NEHelperClient API

Нестабильно между iOS, trippy entitlements. Для автора твика — табу.

## 7. Debugging VPN в своём app

```bash
ssh mobile@<ip>
log stream --process nehelper --device <udid>
# И:
log stream --process YourAppTunnel --device <udid>
```

Увидите handshake, установка tunnel, DNS resolution.

## 8. Configurations

На устройстве VPN configs:

```bash
ssh mobile@<ip>
plutil -p /private/var/preferences/com.apple.networkextension.plist
# иерархический plist с saved configs
```

Автору твика редко нужно читать это напрямую — через public `NEVPNManager.loadFromPreferencesWithCompletionHandler:` в приложении.

## 9. Что подтверждено

- **Подтверждено Dopamine**: nehelper не хукается напрямую.
- **Подтверждено Dopamine**: NECP syscalls хукаются в `systemhook` для arm64 (глава III.5).
- **Подтверждено Apple docs**: `NetworkExtension.framework` — stable public API.

## 10. Выводы

Главный тезис: **nehelper — platform daemon, не трогается**. Для VPN/Filter твика — работайте через public NEVPNManager / NEFilter* API в приложении.

## 11. См. также

- Глава II.9 — Networking.
- Глава III.5 — systemhook с NECP хуками.
- iOS 26 changes 9 — Networking на iOS 26.
