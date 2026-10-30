# Атлас: Network.framework и URLSession

Два современных Apple networking APIs. URLSession (iOS 7+) и Network.framework (iOS 12+).

## 1. URLSession — главный HTTP-клиент

### 1.1. Классы

```objc
NSURLSession *session = [NSURLSession sharedSession];
// или:
NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
NSURLSession *session = [NSURLSession sessionWithConfiguration:config];
```

Task types:

- `NSURLSessionDataTask` — обычный HTTP request.
- `NSURLSessionDownloadTask` — скачивание в файл.
- `NSURLSessionUploadTask` — upload данных.
- `NSURLSessionWebSocketTask` (iOS 13+) — WebSocket.
- `NSURLSessionStreamTask` — bidirectional stream (устарело в iOS 15+).

### 1.2. Хуки

Главная точка:

```objc
%hook NSURLSession

- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                             completionHandler:(void (^)(NSData *, NSURLResponse *, NSError *))handler
{
    os_log(gLog, "dataTaskWithRequest: %{public}@", request.URL);
    return %orig(request, handler);
}

%end
```

Рецепты 2, 13 cookbook.

### 1.3. Configuration

```objc
NSURLSessionConfiguration *config;
config.HTTPAdditionalHeaders = @{@"User-Agent": @"Custom"};
config.connectionProxyDictionary = @{...};
config.URLCredentialStorage = ...;
config.allowsCellularAccess = YES;
```

Хуки для подмены headers / proxy — рецепт 13.

## 2. Network.framework

Низкоуровневый C API (iOS 12+):

```c
nw_endpoint_t endpoint = nw_endpoint_create_host("example.com", "443");
nw_parameters_t params = nw_parameters_create_secure_tcp(...);
nw_connection_t conn = nw_connection_create(endpoint, params);
nw_connection_set_state_changed_handler(conn, ^(nw_connection_state_t state, nw_error_t err) { ... });
nw_connection_start(conn);
```

### 2.1. Хуки

C API, через `%hookf` / ellekit:

```objc
%hookf(void, nw_connection_start, nw_connection_t conn) {
    os_log(gLog, "nw_connection_start: %p", conn);
    %orig(conn);
}
```

Используется редко в tweak'ах — URLSession сверху достаточна для 95% случаев.

## 3. TLS

iOS автоматически:

- Minimum TLS 1.2 (iOS 11+).
- TLS 1.3 preferred (iOS 13+).
- Certificate Transparency check.
- SSL pinning validation (если app его использует).

Hook `SecTrustEvaluate*` для своего app (глава V.3, атлас Security).

## 4. HTTP/2 и HTTP/3

URLSession автоматически использует HTTP/2 (iOS 9+) и HTTP/3 (iOS 15+, если сервер поддерживает).

На iOS 26 — продолжается оптимизация HTTP/3 (QUIC).

Для автора твика это **невидимо**:

- Вы хукаете `dataTaskWithRequest:`, iOS внутри решает, какой протокол.
- Если нужен конкретный — через `URLSessionConfiguration.supportsHTTP3 = YES/NO` (если доступно).

## 5. WebSocket

```objc
NSURLSessionWebSocketTask *task = [session webSocketTaskWithURL:url];
[task sendMessage:[[NSURLSessionWebSocketMessage alloc] initWithString:@"hi"]
    completionHandler:^(NSError *err) { ... }];
[task receiveMessageWithCompletionHandler:^(NSURLSessionWebSocketMessage *msg, NSError *err) { ... }];
```

Хукается стандартно:

```objc
%hook NSURLSessionWebSocketTask

- (void)sendMessage:(NSURLSessionWebSocketMessage *)message
    completionHandler:(void (^)(NSError *))completionHandler
{
    os_log(gLog, "WS send: %{public}@", message.string ?: @"<data>");
    %orig(message, completionHandler);
}

%end
```

## 6. Что подтверждено

- **Подтверждено Apple docs**: URLSession, Network.framework.
- **Подтверждено Dopamine**: NECP syscalls хукаются в `systemhook` для arm64 (глава III.5).
- **Требует проверки**: конкретные Network.framework iOS 26 extensions.

## 7. См. также

- Рецепты 2, 7, 13.
- Глава II.9 — Networking и фоновые задачи.
- iOS 26 changes 9 — Networking.
- Daemon: nehelper.
