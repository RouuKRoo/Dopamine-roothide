# Daemon: lsd (Launch Services)

Реестр установленных приложений, URL-схем, UTI, document types. С хуками Dopamine.

## 1. Что это

- **Путь**: `/usr/libexec/lsd`
- **Executable name**: `lsd`
- **Запуск**: launchd
- **Два инстанса**: system + per-user

## 2. Что делает

- Хранит `LSApplicationProxy` для каждого установленного app.
- Регистрирует URL schemes (`cydia://`, `sileo://`, пользовательские).
- Отвечает на `canOpenURL:`, `openURL:`.
- Регистрирует UTI (file types).
- Поддерживает `.lsregister` database.

## 3. Framework

- `CoreServices.framework` — public LS* API.
- `LaunchServices.framework` — private internal.
- Классы: `LSApplicationProxy`, `LSApplicationWorkspace`, `_LSBundleID`, `_LSApplicationRecord`.

## 4. Хуки Dopamine

Из `BaseBin/rootlesshooks/lsd.x` (глава III.9):

- **`_LSGetInboxURLForBundleIdentifier`** — для не-apple apps редиректит путь inbox в rootless-пространство.
- **`_LSServer_RebuildApplicationDatabases`** — после ребилда LS DB вызывает `uicache -a` для re-регистрации jailbreak apps.

Из `BaseBin/roothidehooks/src/lsd.x`:

- **`_LSURLOverride.initWithOriginalURL:`** — блокирует jailbreak URL schemes.
- **`_LSCanOpenURLManager.getIsURL:alwaysCheckable:hasHandler:`** — скрывает jb schemes от detection.

Это **существующие** хуки. Автор твика в них встраивается, не ломает.

## 5. Что это значит для автора

- URL scheme вашего jailbreak-приложения автоматически «спрятана» от `canOpenURL:` → работает как detection bypass для некоторых apps.
- Пути inbox редиректятся — не нужно вручную path translation.
- `uicache -a` вызывается — ваше приложение появится в SpringBoard после установки.

Всё это — автоматически. Автор tweak-а ничего не делает.

## 6. LSApplicationWorkspace — публичный интерфейс

```objc
@interface LSApplicationWorkspace : NSObject
+ (LSApplicationWorkspace *)defaultWorkspace;
- (NSArray *)allInstalledApplications;
- (NSArray *)applicationsAvailableForHandlingURLScheme:(NSString *)scheme;
- (BOOL)installApplication:(NSString *)appIdentifier
             withOptions:(NSDictionary *)options
                   error:(NSError **)error;
- (BOOL)uninstallApplication:(NSString *)appIdentifier
             withOptions:(NSDictionary *)options
                   error:(NSError **)error;
@end
```

Это **public private** API — объявлен не в SDK, но используется многими tweak авторами (из class-dump).

Хукать можно для observation:

```objc
%hook LSApplicationWorkspace

- (NSArray *)allInstalledApplications {
    NSArray *result = %orig;
    os_log(gLog, "allInstalledApplications: count=%lu", (unsigned long)result.count);
    return result;
}

%end
```

## 7. LSApplicationProxy

Для одного приложения:

```objc
@interface LSApplicationProxy : NSObject
- (NSString *)bundleIdentifier;
- (NSString *)applicationType;    // User, System, Internal
- (NSString *)localizedName;
- (NSString *)teamID;
- (NSURL *)bundleURL;
- (NSDate *)installDate;
@end
```

Используется для получения info о конкретном app.

## 8. Регистрация URL schemes вашего твика

Если ваш твик включает `.app` с кастомной URL scheme:

- В Info.plist приложения — `CFBundleURLTypes` + schemes.
- После `dpkg -i` вашего `.deb` — вызовите `uicache -a` в postinst.
- lsd подхватит.

Для автора твика — standard flow, никакого хука не нужно.

## 9. Диагностика

```bash
ssh mobile@<ip>
log stream --process lsd --device <udid>

# Увидеть зарегистрированные apps:
# (нужен dumping tool — `lsdump` или аналог; не всегда установлен)
```

## 10. Что подтверждено

- **Подтверждено исходниками**: Dopamine хуки в lsd (`BaseBin/rootlesshooks/lsd.x`, `BaseBin/roothidehooks/src/lsd.x`).
- **Подтверждено**: `uicache -a` вызывается после `_LSServer_RebuildApplicationDatabases` (глава III.9).
- **Требует проверки**: конкретные private классы `_LSURLOverride`, `_LSCanOpenURLManager` на iOS 26 — через class-dump CoreServices.

## 11. Выводы

Главный тезис: **lsd — активно хукается Dopamine; не дублируйте его работу**. Для observations — через public `LSApplicationWorkspace`.

## 12. См. также

- Глава II.12 — установка приложений.
- Глава III.9 — rootless/roothidehooks.
- Daemon: installd — связанный процесс.
