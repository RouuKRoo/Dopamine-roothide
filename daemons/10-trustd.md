# Daemon: trustd (TLS trust evaluation)

Централизованный процесс проверки TLS-цепочек, OCSP, CT, pinning.

## 1. Что это

- **Путь**: `/usr/libexec/trustd` или `/System/Library/Frameworks/Security.framework/Versions/A/XPCServices/trustd.xpc/`
- **Executable name**: `trustd`
- **Запуск**: launchd on-demand
- **Привилегии**: root

## 2. Что делает

- Принимает запросы `SecTrustEvaluate*` от приложений.
- Проверяет chain certificates.
- Выполняет OCSP (online certificate status) requests.
- Проверяет Certificate Transparency (CT) logs.
- Проверяет SSL pinning (включая Apple-specific pins).
- Отвечает trust/no-trust.

Для каждого HTTPS-запроса на iOS trustd (в некоторых формах) задействован.

## 3. Framework

- `Security.framework` — public клиент API.
- Классы: `SecTrust`, `SecCertificate`.
- Функции: `SecTrustEvaluate`, `SecTrustEvaluateWithError`, `SecTrustGetCertificateCount`.

## 4. Хуки Dopamine

**Нет** прямых хуков trustd (глава II.14).

## 5. SSL pinning bypass — популярная тема

Для **собственного** приложения автор хочет видеть HTTPS трафик. Самый простой путь — хук `SecTrust*` в приложении-клиенте:

```objc
#import <Security/Security.h>

%hookf(bool, SecTrustEvaluateWithError, SecTrustRef trust, CFErrorRef *error) {
    os_log(gLog, "SecTrustEvaluateWithError called");
    if (error) *error = NULL;
    return true;  // всегда trust
}

%hookf(OSStatus, SecTrustEvaluate, SecTrustRef trust, SecTrustResultType *result) {
    OSStatus r = %orig(trust, result);
    os_log(gLog, "SecTrustEvaluate: real=%u", (unsigned)*result);
    *result = kSecTrustResultProceed;  // всегда OK
    return errSecSuccess;
}
```

Это **в приложении**, не в trustd. Хукается стандартным Logos.

Filter plist на bundle id вашего приложения.

## 6. Для чужого приложения

**Серая зона**. SSL pinning защищает:

- Банковские приложения.
- Password managers.
- Enterprise MFA.

Обход для research вашего собственного трафика — обычно OK. Публикация tweak «bypass SSL pinning for all apps» — обычно нет; этически и юридически.

## 7. Что хукать нельзя

### 7.1. trustd сам

Platform-process с строгими entitlements. Хук опасен — ломает TLS для всех приложений.

### 7.2. Apple-signed server validation

iOS verifies connections к Apple-серверам специально. Любой хук здесь = приложения не могут обращаться к Apple services (iCloud, App Store, Push).

## 8. mitmproxy workflow для своего app

```
1. Поставить mitmproxy на хосте: mitmproxy --listen-port 8080
2. На устройстве Settings → Wi-Fi → network → Configure Proxy → Manual
     Server: <host-ip>
     Port: 8080
3. Открыть http://mitm.it в Safari на устройстве
4. Установить mitmproxy CA certificate
5. Trust CA в Settings → General → About → Certificate Trust Settings
6. С SSL pinning tweak (глава V.3) — HTTPS трафик видим в mitmproxy
```

Полезный workflow для debugging вашего собственного app.

## 9. Что подтверждено

- **Подтверждено Dopamine**: trustd не хукается.
- **Подтверждено Apple docs**: `SecTrust*` API — public.
- **Этика**: SSL pinning bypass — только для своего app.

## 10. Выводы

Главный тезис: **trustd — не хукаем. `SecTrust*` в приложении — хукаем для своего app только**.

Для production — SSL pinning защищает пользователя; его обход в чужом приложении = нарушение privacy.

## 11. См. также

- Глава II.9 — Networking.
- Глава V.3 — runtime патчи.
- iOS 26 changes 9 — Networking на 26.

На этом **daemons секция закрыта**. 10 глав — главные системные демоны iOS с точки зрения автора твика.
