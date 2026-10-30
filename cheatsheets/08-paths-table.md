# Шпаргалка 8. Таблица путей rootless vs RootHide

Полный справочник путей в трёх моделях jailbreak: classic, rootless, RootHide. Для каждого — как корректно получить путь в твике.

## Три модели

| Модель | iOS | Корень |
|---|---|---|
| Classic | до 15 | `/` (rootfs писуемо) |
| Rootless | 15.0–17.x (Dopamine 1.x, Taurine, Unc0ver с patches) | `/var/jb` |
| **RootHide** | 15.0+ и 26.0/26.0.1 (Dopamine 3) | `/var/<rand1>/<rand2>/.jbroot` |

Для нашей целевой среды (A13 + iOS 26) — только **RootHide**. Классическая и чистая rootless модель не применимы.

## Таблица путей

Главная таблица: что было в classic → что стало в rootless → RootHide.

| Классический путь | rootless (Dopamine 2) | RootHide (Dopamine 3) | Как получить в твике |
|---|---|---|---|
| `/usr/bin/jbctl` | `/var/jb/usr/bin/jbctl` | `/var/<rand>/<rand>/.jbroot/usr/bin/jbctl` или `/basebin/jbctl` | `JBROOT_PATH("/usr/bin/jbctl")` |
| `/usr/lib/libroot.dylib` | `/var/jb/usr/lib/libroot.dylib` | ... jbroot/usr/lib/... | обычно линкуется автоматически через `-lroot` |
| `/Library/MobileSubstrate/DynamicLibraries/MyTweak.dylib` | `/var/jb/Library/MobileSubstrate/DynamicLibraries/MyTweak.dylib` | ... /jbroot/Library/MobileSubstrate/... | Theos сам правильно установит |
| `/Library/PreferenceBundles/MyPrefs.bundle` | `/var/jb/Library/PreferenceBundles/MyPrefs.bundle` | ... /jbroot/Library/PreferenceBundles/... | Theos (`BUNDLE_NAME_INSTALL_PATH = /Library/PreferenceBundles`) |
| `/Library/PreferenceLoader/Preferences/MyTweak.plist` | `/var/jb/Library/PreferenceLoader/Preferences/MyTweak.plist` | ... аналогично | Theos `internal-stage::` |
| `/Library/LaunchDaemons/com.example.plist` | `/var/jb/Library/LaunchDaemons/com.example.plist` | ... аналогично | `JBROOT_PATH("/Library/LaunchDaemons/...")` |
| `/Applications/MyApp.app` | `/var/jb/Applications/MyApp.app` | ... /jbroot/Applications/... | не трогайте вручную; через `uicache` |
| `/var/mobile/Library/Preferences/com.example.plist` | `/var/jb/var/mobile/Library/Preferences/com.example.plist` (через cfprefsd redirect) | аналог в jbroot | Theos сам редиректит через `cfprefsd` |
| `/basebin/*` | `/var/jb/basebin/*` | `/basebin → <jbroot>/basebin` (symlink) | `/basebin/<binary>` работает везде |

## Пути приложения (не меняются)

Эти пути **не** редиректируются — они остаются в rootfs:

| Назначение | Путь |
|---|---|
| Бандл приложения | `/private/var/containers/Bundle/Application/<UUID>/<App>.app/` |
| Контейнер данных | `/private/var/mobile/Containers/Data/Application/<UUID>/` |
| Shared container | `/private/var/mobile/Containers/Shared/AppGroup/<UUID>/` |
| System приложения | `/Applications/*.app/` |
| System frameworks | `/System/Library/Frameworks/*.framework/` |
| Private frameworks | `/System/Library/PrivateFrameworks/*.framework/` |
| Preference bundles (system) | `/System/Library/PreferenceBundles/*.bundle/` |

Для доступа к ним в твике — обычные пути без `JBROOT_PATH`.

## Как получить контейнер приложения в твике

```c
const char *home = getenv("HOME");
// На iOS: /private/var/mobile/Containers/Data/Application/<UUID>/
// Внутри: Documents/, Library/, tmp/, Preferences/
```

Или через API:

```c
NSString *docDir = NSSearchPathForDirectoriesInDomains(
    NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
```

Для своего приложения эти пути всегда доступны (sandbox даёт их).

## Структура jbroot

```
<jbroot>/
├── basebin/              — компоненты Dopamine
│   ├── libjailbreak.dylib
│   ├── libroothider.dylib
│   ├── systemhook.dylib
│   ├── jailbreakd
│   ├── hookd
│   ├── jbctl
│   ├── basebin.tc        — static trust cache
│   ├── .safe_mode        — маркер (если присутствует)
│   └── .version
├── bin/                  — launchctl и др.
├── etc/
├── sbin/
├── Applications/         — jailbreak-приложения
├── Library/
│   ├── MobileSubstrate/
│   │   └── DynamicLibraries/   — ваши твики (.dylib + .plist)
│   ├── PreferenceBundles/      — PreferenceBundles
│   ├── PreferenceLoader/
│   │   └── Preferences/        — entry.plist для Settings.app
│   ├── LaunchDaemons/
│   ├── LaunchAgents/
│   └── Application Support/
├── private/
│   └── var/
│       └── mobile/
│           ├── Library/
│           │   └── Preferences/   — .plist настроек (cfprefsd redirect)
│           └── Documents/
├── usr/
│   ├── bin/
│   ├── lib/
│   │   ├── libroot.dylib
│   │   ├── libellekit.dylib
│   │   └── TweakLoader.dylib
│   └── sbin/
└── var/                  — тот же, что /private/var (симлинк)
```

## Проверка jbroot на устройстве

```bash
ssh mobile@<ip>

# Через libroot (всегда работает):
/basebin/jbctl jbroot
# или:
echo "$(jbroot)"

# Руками:
ls /basebin               # symlink в jbroot/basebin
readlink /basebin         # покажет настоящий путь

# Другой способ:
find / -maxdepth 3 -name ".jbroot" 2>/dev/null
```

## Пути для твика — чеклист

```
[ ] Используется JBROOT_PATH(...) или libroot_get_jbroot_prefix()
[ ] НЕТ хардкода "/var/jb/..."
[ ] НЕТ хардкода "/private/var/..." для твик-данных
[ ] CFPreferences с доменом вашего пакета (не прямой fopen)
[ ] NSBundle через bundleWithPath: + JBROOT_PATH
[ ] НЕ используется [NSBundle mainBundle] для доступа к твик-ресурсам
```

## См. также

- Глава II.8 — файловая система и контейнеры.
- Глава III.10 — libroot в деталях.
- Глава IV.9 — RootHide-совместимость (расширенный разбор).
- Шпаргалка 5 — libroot API.
