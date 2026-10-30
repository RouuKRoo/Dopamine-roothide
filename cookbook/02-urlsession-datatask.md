# Рецепт 2. Логирование URLSession-запросов в своём приложении

**Цель**: твик, логирующий все HTTP-запросы через `NSURLSession` в своём тестовом приложении.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = URLLog
URLLog_FILES = Tweak.x
URLLog_CFLAGS = -fobjc-arc
URLLog_FRAMEWORKS = Foundation
URLLog_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## control

```
Package: com.example.urllog
Name: URLLog
Version: 0.0.1
Architecture: iphoneos-arm
Description: Log all URLSession requests in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## URLLog.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Filter</key>
    <dict>
        <key>Bundles</key>
        <array>
            <string>com.example.testbed</string>
        </array>
    </dict>
</dict>
</plist>
```

## Tweak.x

```objc
#import <Foundation/Foundation.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.urllog", "http");
    os_log(gLog, "URLLog loaded");
}

%hook NSURLSession

- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                             completionHandler:(void (^)(NSData *, NSURLResponse *, NSError *))handler
{
    os_log(gLog, "dataTaskWithRequest: %{public}@ %{public}@",
           request.HTTPMethod ?: @"GET", request.URL.absoluteString ?: @"");
    return %orig(request, handler);
}

- (NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url
                        completionHandler:(void (^)(NSData *, NSURLResponse *, NSError *))handler
{
    os_log(gLog, "dataTaskWithURL: %{public}@", url.absoluteString);
    return %orig(url, handler);
}

- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                     completionHandler:(void (^)(NSURL *, NSURLResponse *, NSError *))handler
{
    os_log(gLog, "downloadTaskWithRequest: %{public}@", request.URL.absoluteString);
    return %orig(request, handler);
}

- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                           fromData:(NSData *)bodyData
                                  completionHandler:(void (^)(NSData *, NSURLResponse *, NSError *))handler
{
    os_log(gLog, "uploadTaskWithRequest: %{public}@ (%lu bytes)",
           request.URL.absoluteString, (unsigned long)bodyData.length);
    return %orig(request, bodyData, handler);
}

%end
```

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"

log stream --predicate 'subsystem == "com.example.urllog"' --device <udid>
```

## Расширение: логировать заголовки

```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                             completionHandler:(void (^)(NSData *, NSURLResponse *, NSError *))handler
{
    NSDictionary *headers = request.allHTTPHeaderFields;
    os_log(gLog, "dataTask: %{public}@ %{public}@ headers=%{public}@",
           request.HTTPMethod, request.URL.absoluteString, headers);
    return %orig(request, handler);
}
```

## Расширение: логировать response

Для перехвата response — хукаем completion handler:

```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                             completionHandler:(void (^)(NSData *, NSURLResponse *, NSError *))origHandler
{
    NSURL *url = request.URL;
    void (^wrappedHandler)(NSData *, NSURLResponse *, NSError *) = ^(NSData *data, NSURLResponse *response, NSError *error) {
        os_log(gLog, "response for %{public}@: %ld bytes, err=%{public}@",
               url.absoluteString, (long)data.length, error.localizedDescription ?: @"nil");
        if (origHandler) origHandler(data, response, error);
    };
    return %orig(request, wrappedHandler);
}
```

## Этика

Этот рецепт — для **собственного** приложения. Логирование трафика **чужого** приложения — privacy issue даже на jailbroken устройстве.

## См. также

- Глава II.9 — Networking и NECP.
- Глава IV.4 — первый ObjC hook.
- Рецепт 13 — переопределение User-Agent.
