# Атлас: LocalAuthentication

Face ID, Touch ID, device passcode validation.

## 1. Главный класс — LAContext

```objc
@interface LAContext : NSObject

- (BOOL)canEvaluatePolicy:(LAPolicy)policy error:(NSError **)error;
- (void)evaluatePolicy:(LAPolicy)policy
         localizedReason:(NSString *)localizedReason
                   reply:(void (^)(BOOL success, NSError *error))reply;
- (void)evaluateAccessControl:(SecAccessControlRef)accessControl
                     operation:(LAAccessControlOperation)operation
                 localizedReason:(NSString *)localizedReason
                          reply:(void (^)(BOOL success, NSError *error))reply;

@property (readonly) LABiometryType biometryType;
@property (strong) NSString *localizedFallbackTitle;
@property (strong) NSString *localizedCancelTitle;
@end
```

## 2. Policies

| Policy | Что проверяет |
|---|---|
| `LAPolicyDeviceOwnerAuthenticationWithBiometrics` | Только биометрия (Face/Touch ID) |
| `LAPolicyDeviceOwnerAuthentication` | Биометрия + passcode fallback |
| `LAPolicyDeviceOwnerAuthenticationWithWatch` (watchOS) | С Apple Watch |

## 3. BiometryType

```objc
typedef NS_ENUM(NSInteger, LABiometryType) {
    LABiometryTypeNone = 0,
    LABiometryTypeTouchID = 1,
    LABiometryTypeFaceID = 2,
    LABiometryTypeOpticID = ???  // Vision Pro — iPhone 11 нет
};
```

На A13 (iPhone 11) → `LABiometryTypeFaceID`. На SE 2nd gen → `LABiometryTypeTouchID`.

## 4. Хук для своего app

Рецепт 8 Cookbook — три режима (`PassThrough`, `ForceSuccess`, `ForceFailure`), биометрия через PreferenceBundle toggle.

## 5. Что хукать нельзя

### 5.1. biometrickitd (daemon)

`/usr/libexec/biometrickitd` — обрабатывает Face/Touch ID. **Не хукать**:

- Защищает все приложения.
- iOS считает biometrickitd critical.
- Крах = перезагрузка в recovery.

### 5.2. SEP (Secure Enclave)

Биометрия matching происходит в SEP. SEP недоступен для hook'ов — hardware isolated. Любой «обход» в userland bypass'ит только application-level checks, но не биометрию саму.

## 6. Этика

Биометрия — главная защита пользовательских данных. Hook в чужом app = обход этой защиты. Юридически — обычно нарушение; этически — всегда.

Для **своего** app — valid для:

- Automated UI testing.
- Screenshots для App Store (где биометрия всегда «проходит»).
- Debug error paths.

## 7. iOS 26 заметки

- LAContext API стабилен с iOS 8.
- Нет breaking changes на 26.
- Possibility: новые policies (например, `LAPolicyDeviceOwnerAuthenticationWithWristDetection` для watchOS).

## 8. См. также

- Рецепт 8 — LAContext hook.
- Глава II.2 — экран блокировки и аутентификация.
- iOS 26 changes 8 — Privacy.
