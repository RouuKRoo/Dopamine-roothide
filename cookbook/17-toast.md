# Рецепт 17. Добавить тост в любой `UIViewController` своего приложения

**Цель**: расширить `UIViewController` новым методом `-showToast:` через `%new` в твике — не меняя исходный код приложения.

## Makefile

```make
TARGET := iphone:clang:latest:15.0
ARCHS = arm64 arm64e
INSTALL_TARGET_PROCESSES = TestbedApp
THEOS_PACKAGE_SCHEME = roothide

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = ToastExt
ToastExt_FILES = Tweak.x
ToastExt_CFLAGS = -fobjc-arc
ToastExt_FRAMEWORKS = UIKit Foundation
ToastExt_LIBRARIES = root

include $(THEOS_MAKE_PATH)/tweak.mk
```

## ToastExt.plist

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
Package: com.example.toastext
Name: ToastExt
Version: 0.0.1
Architecture: iphoneos-arm
Description: Add -showToast: category to UIViewController in TestbedApp
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

## Tweak.x

```objc
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>
#import <os/log.h>

static os_log_t gLog;

%ctor {
    gLog = os_log_create("com.example.toastext", "toast");
    os_log(gLog, "ToastExt loaded");
}

%hook UIViewController

// %new добавляет метод, которого не было в оригинальном классе
%new
- (void)showToast:(NSString *)message {
    [self showToast:message duration:2.0];
}

%new
- (void)showToast:(NSString *)message duration:(NSTimeInterval)duration {
    if (!message || message.length == 0) return;

    // Главный поток для UI
    dispatch_async(dispatch_get_main_queue(), ^{
        UIView *host = self.view;
        if (!host) return;

        // Контейнер тоста
        UIView *toast = [[UIView alloc] init];
        toast.translatesAutoresizingMaskIntoConstraints = NO;
        toast.backgroundColor = [UIColor.blackColor colorWithAlphaComponent:0.75];
        toast.layer.cornerRadius = 12;
        toast.alpha = 0;
        toast.clipsToBounds = YES;

        // Лейбл
        UILabel *label = [[UILabel alloc] init];
        label.translatesAutoresizingMaskIntoConstraints = NO;
        label.text = message;
        label.textColor = [UIColor whiteColor];
        label.font = [UIFont systemFontOfSize:14];
        label.textAlignment = NSTextAlignmentCenter;
        label.numberOfLines = 0;

        [toast addSubview:label];
        [host addSubview:toast];

        [NSLayoutConstraint activateConstraints:@[
            // Toast — внизу, по центру, с отступами
            [toast.bottomAnchor constraintEqualToAnchor:host.safeAreaLayoutGuide.bottomAnchor constant:-50],
            [toast.centerXAnchor constraintEqualToAnchor:host.centerXAnchor],
            [toast.leadingAnchor constraintGreaterThanOrEqualToAnchor:host.leadingAnchor constant:20],
            [toast.trailingAnchor constraintLessThanOrEqualToAnchor:host.trailingAnchor constant:-20],

            // Label — c padding'ом
            [label.topAnchor constraintEqualToAnchor:toast.topAnchor constant:10],
            [label.bottomAnchor constraintEqualToAnchor:toast.bottomAnchor constant:-10],
            [label.leadingAnchor constraintEqualToAnchor:toast.leadingAnchor constant:16],
            [label.trailingAnchor constraintEqualToAnchor:toast.trailingAnchor constant:-16],
        ]];

        // Анимация появления
        [UIView animateWithDuration:0.3 animations:^{
            toast.alpha = 1.0;
        } completion:^(BOOL finished) {
            // Удержать видимым и исчезнуть
            [UIView animateWithDuration:0.3
                                   delay:duration
                                 options:0
                              animations:^{
                toast.alpha = 0;
            } completion:^(BOOL done) {
                [toast removeFromSuperview];
            }];
        }];

        os_log(gLog, "toast shown on %{public}@: %{public}@",
               NSStringFromClass(self.class), message);
    });
}

%end
```

Разбор:

- `%new` — Logos директива, добавляющая метод в класс (которого не было в оригинале).
- Теперь **любой** `UIViewController` в вашем приложении имеет `-showToast:`.
- Второй `%new` — overload с duration.
- `dispatch_async(main)` — безопасно создаём UI на main thread.
- `translatesAutoresizingMaskIntoConstraints = NO` — требование Auto Layout.
- Анимация появления-исчезновения — Fade.
- `removeFromSuperview` — освобождаем ресурсы.

## Использование из приложения

В коде вашего тестового приложения:

```swift
// Swift
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        // Вызываем через -performSelector, т. к. метод добавлен через %new
        self.perform(Selector(("showToast:")), with: "Hello from tweak!")
    }
}
```

Или через Objective-C bridging:

```objc
// Если приложение на ObjC:
[self performSelector:@selector(showToast:) withObject:@"Hello"];
```

Для Swift лучше явно:

```swift
import UIKit

extension UIViewController {
    @objc func showToast(_ message: String) {
        // Метод объявлен в твике — приложение видит только selector,
        // прямой вызов не компилируется. Используйте perform:
        self.perform(#selector(NSObject.performSelector(_:)), with: Selector(("showToast:")))
    }
}

// Или проще:
let sel = Selector(("showToast:"))
if viewController.responds(to: sel) {
    viewController.perform(sel, with: "Hello")
}
```

## Сборка и тест

```bash
make clean package THEOS_PACKAGE_SCHEME=roothide
scp packages/*.deb mobile@<ip>:/tmp/
ssh mobile@<ip> "sudo dpkg -i /tmp/*.deb && killall -9 TestbedApp"
```

В TestbedApp вызвать метод → тост появится.

## Retain cycles?

В этом рецепте — нет:

- `toast` создаётся как local variable, добавляется в hierarchy, удаляется в completion.
- Блоки захватывают `toast` (local), не `self` → no cycle.
- `host` — self.view; но мы не keep retain.

Если хотите, чтобы тост «следовал за VC» (переустанавливался при повороте) — нужен другой подход через associated object.

## См. также

- Шпаргалка 1 — `%new` directive.
- Глава IV.7 — threads/ARC/retain cycles.
- Рецепт 9 — PreferenceBundle для custom-настроек стиля toast.
