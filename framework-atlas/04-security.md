# Атлас: Security framework

Keychain, SecTrust, SecCertificate. Для автора твика — осторожная зона.

## 1. Классы и функции

### 1.1. Keychain

```c
OSStatus SecItemCopyMatching(CFDictionaryRef query, CFTypeRef *result);
OSStatus SecItemAdd(CFDictionaryRef attributes, CFTypeRef *result);
OSStatus SecItemUpdate(CFDictionaryRef query, CFDictionaryRef attributesToUpdate);
OSStatus SecItemDelete(CFDictionaryRef query);
```

Рецепт 6 Cookbook — полный пример hook'а.

### 1.2. Trust (TLS validation)

```c
OSStatus SecTrustEvaluate(SecTrustRef trust, SecTrustResultType *result);
bool SecTrustEvaluateWithError(SecTrustRef trust, CFErrorRef *error);
OSStatus SecTrustEvaluateAsyncWithError(...);
```

Для SSL pinning bypass (своё приложение) — хук этих функций.

### 1.3. Certificates

```c
SecCertificateRef SecCertificateCreateWithData(allocator, data);
CFDataRef SecCertificateCopyData(certificate);
```

Используется под capотом.

### 1.4. Code signing

```c
OSStatus SecStaticCodeCreateWithPath(path, flags, &codeRef);
OSStatus SecCodeCheckValidity(code, flags, requirement);
```

Внутренние проверки iOS. Для автора твика — редко интересно (Dopamine сам занимается codesign).

## 2. Этические границы

| Задача | Собственное app | Чужое app |
|---|---|---|
| Логирование keychain queries | OK (debug) | **Нет** (privacy) |
| Подмена keychain values | OK (testing) | **Нет** (data integrity) |
| SSL pinning bypass | OK | Серая зона |
| Trust SeC certificates | OK (dev) | **Нет** (bypass security) |
| Удаление keychain items | OK | **Нет** |

Правило: Security framework защищает **пользователя**. Хук — только в своём приложении для legitimate dev/testing.

## 3. iOS 26 заметки

Apple продолжает усиливать:

- **Advanced Data Protection** — расширение E2EE.
- **Post-quantum crypto** — начинается.
- **Attestation** — DCDevice, DeviceCheck.

Для автора твика — public SecItem/SecTrust API стабильны. Internal — Apple-only.

## 4. См. также

- Рецепт 6 — keychain logging.
- Daemon: trustd.
- iOS 26 changes 8 — Privacy и Security.
- Глава V.3 — runtime patches.
