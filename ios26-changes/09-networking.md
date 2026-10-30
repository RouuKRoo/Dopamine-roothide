# Networking в iOS 26

Эта глава — расширение главы II.9. Там разбирали архитектуру и NECP; здесь — что меняется на iOS 26 для автора твика.

## 1. Главное — NECP на A13 всё ещё хукается Dopamine

Напомним (глава III.5, `BaseBin/systemhook/src/main.c:128–156, 515–521`):

- На arm64 (A13) `systemhook` хукает все 5 NECP syscalls.
- Это нужно, чтобы `CS_VALID` revalidation прошла до каждого syscall.
- Без этого сеть **не работает** в процессах под jailbreak.

На iOS 26.0 и 26.0.1 этот механизм **активен**. Автор твика ничего не делает — всё работает автоматически.

Проверка на устройстве:

```bash
ssh mobile@<ip>
# Запустить любое приложение, сделать сетевой запрос
# Если работает — NECP revalidation OK
# Если нет — проверить /var/jb/basebin/systemhook.dylib и .safe_mode
```

## 2. Public API — стабильно

На iOS 26 fundamental networking API:

- `URLSession` — работает так же.
- `URLSessionConfiguration` — те же опции.
- `URLRequest` — тот же API.
- `Network.framework` (`nw_connection_t` et al) — стабильно с iOS 12.
- `CFNetwork` — legacy, но работает.

Хукается стандартным Logos. Все рецепты из cookbook (2, 7, 13) — на 26 работают.

## 3. iOS 26 изменения (ожидаемые)

Направления развития:

- **HTTP/3 и QUIC** — Apple продолжает улучшать.
- **Multi-path TCP** (iOS 11+) — расширение support.
- **Low Data Mode** — больше контроля.
- **Private Relay** (iCloud+) — обновления.
- **TLS 1.3** — всегда есть; возможно TLS 1.4 если выпустят RFC.
- **Post-quantum crypto** — Apple начинает integration.

Для автора твика ни одно из этих не меняет fundamental workflow хука. URL/data level — то же.

## 4. VPN и Network Extension

VPN на iOS реализуется через `NetworkExtension.framework`:

- `NEPacketTunnelProvider` — tunnel VPN.
- `NEAppProxyProvider` — per-app proxy.
- `NEDNSProxyProvider` — DNS proxy.
- `NEFilterDataProvider` / `NEFilterPacketProvider` — content filter.

Это **extensions** (`.appex`), не обычные приложения. Отдельный process с особыми entitlements.

Для автора твика:

- Собственный VPN provider — public API, работает без твика.
- Hook чужого provider — через bundle id `.appex`, если filter plist правильный.
- `nehelper` demon — Dopamine его не хукает (глава II.14), поэтому open для tweak experimentation, но осторожно.

## 5. TLS pinning и SSL inspection

Общий сценарий — хотите видеть HTTPS трафик собственного приложения.

Approach 1: подмена SecTrustEvaluate (глава V.3):

```objc
%hookf(OSStatus, SecTrustEvaluateWithError, SecTrustRef trust, CFErrorRef *error) {
    os_log(gLog, "SecTrustEvaluateWithError called");
    if (error) *error = NULL;
    return errSecSuccess;  // всегда trust'им (только в своём app!)
}
```

**Для чужого app** — pinning bypass tool, скользкая этика.

Approach 2: системный proxy через `URLSessionConfiguration.connectionProxyDictionary`:

```objc
%hook NSURLSessionConfiguration

- (NSDictionary *)connectionProxyDictionary {
    return @{
        (NSString *)kCFNetworkProxiesHTTPSEnable: @YES,
        (NSString *)kCFNetworkProxiesHTTPSProxy: @"192.168.1.100",
        (NSString *)kCFNetworkProxiesHTTPSPort: @8888,
    };
}

%end
```

Это направит трафик через ваш mitmproxy. Для собственного тестирования.

## 6. DNS override

iOS 14+ дало custom DNS через Network.framework:

```swift
NWParameters.privateMode  // в приложении
```

Для твика — хук `nw_connection_set_viability_changed_handler` или подобное — сложно; обычно не нужно. Для DNS модификации — проще через VPN provider или HTTP proxy.

## 7. Таблица «что хукать»

| Задача | Способ | На iOS 26 |
|---|---|---|
| URLSession.dataTask | `%hook NSURLSession` | ✓ стабильно |
| Request/Response headers | `%hook NSURLRequest`/`setValue:` | ✓ |
| Response data | wrapped completion handler | ✓ |
| TLS verification | `%hookf SecTrustEvaluate*` | ✓ (этически — своё app) |
| Network.framework `nw_connection` | fishhook/ellekit | ✓ (сложно) |
| NECP syscalls | — | **Делает Dopamine, не трогать** |
| nehelper, apsd | hook в demon | **Опасно, не рекомендовано** |

## 8. Nitpick: Apple Private API в сетевой подсистеме

iOS 26 имеет multiple private классы:

- `_CFNetwork*` — legacy
- `NWConcrete_*` — Network.framework internal
- Прочие

Хукать их — нестабильно. Используйте public URLSession level.

## 9. Testing без устройства

Для тестирования network-tweaks:

- **mitmproxy** на хосте + proxy configuration на устройстве:
  ```bash
  mitmproxy --listen-port 8080
  # На устройстве: Settings → Wi-Fi → network → Configure Proxy → Manual
  ```
- **Charles Proxy** / **Proxyman** — graphical альтернативы.
- **Burp Suite** — для security тестирования.

Все работают с jailbroken device + ваш твик с подменой `SecTrustEvaluate` (для HTTPS inspection).

## 10. Что подтверждено

- **Подтверждено исходниками Dopamine**: NECP хуки в `systemhook/main.c:128–156, 515–521`.
- **Подтверждено Apple docs**: URLSession, Network.framework, NetworkExtension — stable public API.
- **Требует проверки**: конкретные новые features в iOS 26 (QUIC, HTTP/3 extensions) — не меняют hook workflow для автора твика.
- **Не подтверждено**: что принципиально новое networking API появилось в 26 — всё ожидаемо эволюционно.

## 11. Выводы

Главный тезис: **networking на iOS 26 — стабильный, хукается легко через public API**. NECP уже делается Dopamine. Public URL* layer — лучший выбор для автора твика.

Для advanced сценариев (TLS inspection, VPN provider) — Apple даёт public frameworks; хуки — только для своего приложения с этическими границами.

## 12. См. также

- Глава II.9 — Networking и фоновые задачи.
- Глава III.5 — systemhook (NECP hooks).
- Рецепты 02, 07, 13 — URLSession-related.
- Атлас Network.framework (framework-atlas, если будет).
