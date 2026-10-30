# Рецепт 16. Логирование `PHPickerViewController` в своём приложении

**Цель**: твик, логирующий, когда и с какой конфигурацией приложение открывает photo picker, и какие ассеты пользователь выбрал.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = PhotoPickerTrace
PhotoPickerTrace_FILES = Tweak.x
PhotoPickerTrace_CFLAGS = -fobjc-arc
PhotoPickerTrace_FRAMEWORKS = PhotosUI Photos UIKit Foundation
PhotoPickerTrace_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## PhotoPickerTrace.plist

```xml
<key>Filter</key>
<dict>
    <key>Bundles</key>
    <array>
        <string>com.example.testbed</string>
    </array>
</dict>
```

## control

```
Package: com.example.photopickertrace
Name: PhotoPickerTrace
Version: 0.0.1
Architecture: iphoneos-arm
Description: Trace PHPickerViewController in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## Tweak.x

```objc
#import <PhotosUI/PhotosUI.h>
#import <Photos/Photos.h>
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.photopickertrace", "picker");
    os_log(gLog, "PhotoPickerTrace loaded");
}

%hook PHPickerViewController

- (instancetype)initWithConfiguration:(PHPickerConfiguration *)configuration {
    os_log(gLog, "init with config: selectionLimit=%ld, filter=%{public}@",
           (long)configuration.selectionLimit, configuration.filter);
    return %orig(configuration);
}

- (void)viewDidLoad {
    os_log(gLog, "picker viewDidLoad");
    %orig;
}

- (void)viewDidDisappear:(BOOL)animated {
    os_log(gLog, "picker viewDidDisappear");
    %orig(animated);
}

%end

// Хукаем delegate-метод — нужен конкретный класс делегата.
// Универсальный способ: swizzling в метод-добавление через runtime.

// Простой способ — хукнуть на уровне любого класса, который implementsPHPickerViewControllerDelegate
// (но мы не знаем, какой это класс). Используем _сам_ UIViewController presentation:

%hook UIViewController

- (void)presentViewController:(UIViewController *)viewControllerToPresent
                      animated:(BOOL)flag
                    completion:(void (^)(void))completion
{
    if ([viewControllerToPresent isKindOfClass:[PHPickerViewController class]]) {
        PHPickerViewController *picker = (PHPickerViewController *)viewControllerToPresent;
        os_log(gLog, "presenting PHPickerViewController, delegate=%{public}@",
               picker.delegate);

        // Swizzle delegate method:
        if (picker.delegate) {
            Class delegateCls = object_getClass(picker.delegate);
            SEL sel = @selector(picker:didFinishPicking:);
            Method m = class_getInstanceMethod(delegateCls, sel);
            if (m) {
                // Этот подход хрупкий: swizzle делегата конкретного экземпляра
                // Для production — лучше использовать KVO или другую технику
                os_log(gLog, "delegate class = %{public}s", class_getName(delegateCls));
            }
        }
    }
    %orig(viewControllerToPresent, flag, completion);
}

%end

// Для PHImageManager — часто используется для загрузки выбранных ассетов
%hook PHImageManager

- (PHImageRequestID)requestImageForAsset:(PHAsset *)asset
                               targetSize:(CGSize)targetSize
                              contentMode:(PHImageContentMode)contentMode
                                  options:(PHImageRequestOptions *)options
                            resultHandler:(void (^)(UIImage *, NSDictionary *))resultHandler
{
    os_log(gLog, "requestImageForAsset: id=%{public}@ size=%.0fx%.0f",
           asset.localIdentifier, targetSize.width, targetSize.height);
    return %orig(asset, targetSize, contentMode, options, resultHandler);
}

%end
```

Разбор:

- `PHPickerViewController` новый API (iOS 14+). Для старых версий — `UIImagePickerController` (хукается аналогично).
- `-initWithConfiguration:` — ловим настройки picker'а (selection limit, filter images/videos/etc).
- `-viewDidLoad`/`-viewDidDisappear:` — для timing.
- `-presentViewController:` на `UIViewController` — универсальный способ поймать показ picker'а.
- `PHImageManager requestImageForAsset:...` — часто используется для загрузки выбранного ассета.

## Что сложно

Delegate method `picker:didFinishPicking:` — метод протокола, не класса. Хукнуть его в абстрактном виде через Logos нельзя — нужно знать конкретный класс делегата.

Для своего приложения — просто добавьте логирование прямо в код делегата. Для чужого — ищите класс через `delegate`-property и свизлите динамически (сложно; обычно избегайте).

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"
```

В TestbedApp открыть photo picker → выбрать фото. В логе:

```
init with config: selectionLimit=5, filter=<PHPickerFilter>
presenting PHPickerViewController, delegate=<MyPickerDelegate>
picker viewDidLoad
...
requestImageForAsset: id=ABC-DEF... size=320x320
picker viewDidDisappear
```

## Этика

Picker — публичное API; трэйс в собственном app = valid. Для чужого app — tracking пользовательских photos privacy issue.

## См. также

- Атлас PhotoKit (framework-atlas, если будет).
- Рецепт 8 — LAContext (похожий approach).
