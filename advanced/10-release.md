# Продвинуто 10. Релиз твика: подписание, версионирование, host-репозитории

Финальная глава раздела Advanced. От первого release до регулярных обновлений: что проверить, где опубликовать, как не потерять пользователей.

## 1. Pre-release чеклист

Перед каждым релизом:

```
[ ] Version в control инкрементирована (semver: MAJOR.MINOR.PATCH)
[ ] Changelog обновлён (CHANGELOG.md в репозитории)
[ ] README актуален
[ ] Все известные bugs либо исправлены, либо документированы
[ ] Unit tests проходят (Advanced 2)
[ ] Static analysis чистый (Advanced 9)
[ ] inspect.sh показывает OK (глава IV.10)
[ ] Нет strings "/var/jb" в собранной .dylib
[ ] Протестировано на iOS 26.0 и 26.0.1 (если это ваша matrix)
[ ] Протестировано на нескольких устройствах (хотя бы 2 для flakiness)
[ ] Depends в control корректны
[ ] FINALPACKAGE=1 использован при сборке
[ ] prerm/postrm maintainer scripts работают (кейс 15)
[ ] Нет чувствительных данных в .deb (ldid -e; strings)
```

## 2. Semantic versioning

Semver: `MAJOR.MINOR.PATCH`:

- **MAJOR** (`1.x.x`) — несовместимые изменения API/поведения. Старые пользователи должны знать.
- **MINOR** (`x.1.x`) — новая функциональность, обратно-совместимая.
- **PATCH** (`x.x.1`) — bug fixes, нет новой функциональности.

Правила:

- Начинайте с `0.0.1` (pre-release, нестабильно).
- `0.1.0` — первая feature-complete beta.
- `1.0.0` — stable, API заморожен.
- Pre-release suffix: `1.0.0-rc1`, `1.0.0-beta.2`.

Пример жизненного цикла:

```
0.0.1 — initial commit
0.1.0 — работающий тwик, но только базовые хуки
0.2.0 — добавлены PreferenceBundle и Darwin notifications
0.9.0 — функционально полный
1.0.0-rc1 — release candidate
1.0.0-rc2 — fixes
1.0.0 — stable
1.0.1 — bug fix
1.1.0 — новая feature
2.0.0 — несовместимое изменение
```

## 3. Changelog

```markdown
# Changelog

Все заметные изменения в проекте — здесь.

## [1.1.0] - 2026-10-30

### Added
- Поддержка iOS 26.0.1 (билды 23B5064e)
- Toggle через PreferenceBundle

### Changed
- NSBundle paths теперь через libroot (RootHide совместимость)

### Fixed
- Крэш при одновременном открытии двух modal'ов

## [1.0.0] - 2026-09-15

### Added
- Первый stable релиз
- Хуки для UIViewController viewDidLoad/viewDidAppear
- PreferenceBundle с 5 опциями

[1.1.0]: https://github.com/user/repo/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/user/repo/releases/tag/v1.0.0
```

Формат — [Keep a Changelog](https://keepachangelog.com/). Полезно иметь — пользователь видит, что нового.

## 4. Host-репозитории

Где опубликовать `.deb`.

### 4.1. GitHub Releases

Самый простой:

```bash
git tag v1.0.0
git push origin v1.0.0
# → CI (Advanced 3) создаёт GitHub Release с .deb файлом attachment
```

Пользователь скачивает `.deb` вручную, устанавливает через Filza / dpkg -i.

**Минус**: нет авто-updates. Пользователь должен сам узнать про новую версию.

### 4.2. Собственный .deb-репозиторий

Файловая структура (на вашем web-сервере):

```
https://repo.example.com/
├── Packages.gz          — index всех .deb (gzipped)
├── Packages.bz2         — то же, bzipped
├── Release              — метаданные репозитория
├── CydiaIcon.png        — иконка
└── debs/
    ├── com.example.mytweak_1.0.0_iphoneos-arm.deb
    ├── com.example.mytweak_1.1.0_iphoneos-arm.deb
    └── ...
```

Создание:

```bash
# На хосте с .deb файлами:
mkdir -p /var/www/myrepo/debs
cp *.deb /var/www/myrepo/debs/
cd /var/www/myrepo
dpkg-scanpackages debs > Packages
gzip -k Packages              # keep .Packages тоже
bzip2 -k Packages

# Release:
cat > Release <<EOF
Origin: My Tweaks
Label: My Tweaks
Suite: stable
Version: 1.0
Codename: ios
Architectures: iphoneos-arm
Components: main
Description: My personal tweak repository
EOF
```

Пользователь добавляет через Sileo/Zebra:
`Sources` → `+` → `https://repo.example.com/`

Теперь Sileo показывает ваши tweaks + автоматически уведомляет об обновлениях.

### 4.3. Публичные репозитории

- **Havoc** (`havoc.app`) — популярный, с submission процессом.
- **Chariz** (`chariz.com`) — профессиональный.
- **Packix** (`packix.com`) — исторически популярный.
- **BigBoss** (`apt.thebigboss.org`) — legacy но большой.

Процесс — варьируется, обычно:

1. Регистрация developer account.
2. Submit `.deb` + metadata.
3. Review (manual).
4. Publishing.

Minuses — manual process, может быть rejection. Plus — доверие пользователей (многие ставят tweaks только через известные repos).

### 4.4. Open-source distribution

Если ваш твик open-source:

```
GitHub repo
├── README.md           — описание + install instructions
├── CHANGELOG.md
├── LICENSE
├── Makefile
├── Tweak.x
├── control
└── releases/
    └── v1.0.0/
        └── com.example.mytweak_1.0.0_iphoneos-arm.deb
```

Это даёт сообществу возможность форкнуть и собрать самостоятельно, если ваш .deb не подходит.

## 5. Пакет для разных схем

Если хотите поддержать и rootless, и roothide:

```bash
# Сборка для roothide:
make clean package THEOS_PACKAGE_SCHEME=roothide FINALPACKAGE=1
mv packages/*.deb release/com.example.mytweak_1.0.0_iphoneos-arm_roothide.deb

# Сборка для rootless (простой Dopamine):
make clean package THEOS_PACKAGE_SCHEME=rootless FINALPACKAGE=1
mv packages/*.deb release/com.example.mytweak_1.0.0_iphoneos-arm_rootless.deb
```

Пользователь выбирает нужный .deb.

Для нашей целевой среды (iOS 26 на A13 с Dopamine 3 RootHide) — только `roothide` нужен.

## 6. Communication

### 6.1. Twitter / fediverse

Короткое announcement:

> MyTweak 1.1.0 released!
>
> ✓ iOS 26.0.1 support
> ✓ New PreferenceBundle toggle
> ✓ Fixed modal crash
>
> Repo: https://repo.example.com/
> Changelog: https://github.com/.../CHANGELOG.md

### 6.2. Reddit / forums

- r/jailbreak — на многие релизы.
- iPhone Dev Wiki — для архивации.
- Discord серверы (конкретных jailbreak проектов).

Полезно для discoverability.

### 6.3. In-tweak notification (осторожно)

Можно показать in-app alert «новая версия доступна». Технически:

```objc
%ctor {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0), ^{
        NSURL *url = [NSURL URLWithString:@"https://repo.example.com/version.txt"];
        NSString *latest = [NSString stringWithContentsOfURL:url encoding:NSUTF8StringEncoding error:nil];
        if (latest && ![latest isEqualToString:@MY_CURRENT_VERSION]) {
            // Показать alert — но только раз в N дней
        }
    });
}
```

**Осторожно**:

- Любой network в `%ctor` — анти-pattern (слоу start приложения).
- Отправка IP/device-fingerprint к вашему серверу = tracking, нужен consent.

Для production — лучше оставить update discovery пользователю (Sileo/Zebra делают это сами).

## 7. Rollback plan

Если новая версия сломала:

1. **Repository**: удалить новую версию из `Packages`-index, оставить старую.
2. **GitHub Release**: пометить как «yanked», но файл оставить (кому-то может помочь).
3. **In-app**: нет механизма; пользователь должен сам даунгрейдить.

Для production — всегда держите предыдущую working версию доступной.

## 8. Lifetime support

Для серьёзного твика:

- **Current major (1.x)** — bug fixes + minor features.
- **Previous major (0.x)** — critical bug fixes only.
- **Older** — EOL, не поддерживается.

Документируйте в README: «Supported versions: 1.x. 0.x — security fixes only. iOS < 15: unsupported».

## 9. Privacy policy

Если ваш твик:

- Шлёт данные на сервер (telemetry, update checks) — **нужен** privacy policy.
- Читает чужие данные (log) — нужен consent.
- Для своего приложения, no telemetry — обычно не требуется, но хорошо иметь.

Простой текст в README:

```
# Privacy
MyTweak does NOT send any data to external servers.
MyTweak logs file operations only within your own container.
No analytics, no telemetry.
```

Это **sell** — пользователи ценят.

## 10. Что подтверждено

- **Подтверждено**: `dpkg-scanpackages` + `apt-ftparchive` — стандартный Debian repo workflow.
- **Подтверждено практикой**: Sileo/Zebra принимают стандартный Debian repo format.
- **Подтверждено Semver**: https://semver.org — стандарт.
- **Подтверждено Keep a Changelog**: https://keepachangelog.com — convention.

## 11. Выводы

Главный тезис: **релиз — это не последняя кнопка «готово», это начало поддержки**. Semver, changelog, repository, communication, privacy — всё это части professional workflow.

Для твика, который хотят ставить сотни/тысячи людей — всё это обязательно. Для твика «только для себя» — минимум `make package FINALPACKAGE=1` + .deb на устройство.

Книга закончена. Вы прошли полный путь от «что такое Mach-O» до «как опубликовать твик в community».

## 12. См. также

- Advanced 3 — CI/CD (автоматизация pipeline).
- Advanced 9 — static analysis (качество кода).
- Глава IV.10 — сборка и инспекция.
- Приложение D — библиография (ресурсы).

На этом **раздел Advanced полностью закрыт**. 10 глав — набор инженерных практик, превращающих хобби-твик в production-grade проект.
