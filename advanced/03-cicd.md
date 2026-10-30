# Продвинуто 3. CI/CD для твика

Как настроить автоматическую сборку `.deb` при push, тестирование, публикацию в репозиторий. Пример для GitHub Actions.

## 1. Что должен делать pipeline

Минимум:

1. На push/PR — собрать `.deb`.
2. Прогнать unit tests (XCTest, Advanced 2).
3. Проверить .deb (inspect.sh из главы IV.10).
4. На tag — опубликовать как GitHub Release.

Расширенно:

- Nightly builds с FINALPACKAGE=0 (debug версия).
- Автоматический changelog.
- Коммит обновлённого repo index (если есть repository).

## 2. Структура

```
MyTweak/
├── Makefile
├── Tweak.x
├── control
├── MyTweak.plist
├── .github/
│   └── workflows/
│       ├── ci.yml          — сборка + тесты на push
│       └── release.yml     — публикация на tag
└── Tests.xcodeproj         — unit tests (Advanced 2)
```

## 3. .github/workflows/ci.yml

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '15.4'

      - name: Run unit tests
        run: |
          xcodebuild test \
            -project Tests.xcodeproj \
            -scheme MyTweakLogicTests \
            -destination 'platform=macOS' \
            CODE_SIGNING_ALLOWED=NO

  build-deb:
    runs-on: macos-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3

      - name: Setup Theos (roothide fork)
        run: |
          git clone --recursive https://github.com/roothide/Theos.git ~/theos
          echo "THEOS=$HOME/theos" >> $GITHUB_ENV
          brew install ldid dpkg xz

      - name: Build .deb
        run: |
          make clean package \
            THEOS_PACKAGE_SCHEME=roothide \
            FINALPACKAGE=1

      - name: Inspect .deb
        run: |
          cd packages
          DEB=$(ls *.deb)
          echo "=== $DEB ==="
          dpkg-deb -c "$DEB"
          dpkg-deb -I "$DEB"

      - name: Verify no /var/jb hardcode
        run: |
          cd packages
          DEB=$(ls *.deb)
          mkdir -p /tmp/x
          dpkg-deb -x "$DEB" /tmp/x
          if find /tmp/x -name "*.dylib" -exec strings {} \; | grep -q "/var/jb/"; then
              echo "FAIL: hardcoded /var/jb paths found"
              exit 1
          fi
          echo "OK: no hardcode"

      - name: Upload .deb artifact
        uses: actions/upload-artifact@v3
        with:
          name: deb-package
          path: packages/*.deb
          retention-days: 30
```

Разбор:

- Два job'а: `unit-tests` и `build-deb`. `build-deb` depends on `unit-tests`.
- Theos клонируется заново на каждый run (можно кэшировать через `actions/cache` для скорости).
- `brew install ldid dpkg xz` — зависимости.
- `make clean package` — сборка.
- `inspect` + `strings` — проверки качества.
- `upload-artifact` — сохраняет .deb для download.

## 4. .github/workflows/release.yml

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Setup Theos
        run: |
          git clone --recursive https://github.com/roothide/Theos.git ~/theos
          echo "THEOS=$HOME/theos" >> $GITHUB_ENV
          brew install ldid dpkg xz

      - name: Build .deb
        run: |
          make clean package \
            THEOS_PACKAGE_SCHEME=roothide \
            FINALPACKAGE=1

      - name: Extract version from tag
        id: version
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          echo "version=$VERSION" >> $GITHUB_OUTPUT

      - name: Generate changelog
        id: changelog
        run: |
          # Простой changelog из git log между tag'ами:
          PREV_TAG=$(git describe --abbrev=0 --tags HEAD^ 2>/dev/null || echo "")
          if [ -n "$PREV_TAG" ]; then
            git log --pretty=format:'- %s (%h)' "$PREV_TAG"..HEAD > CHANGELOG.md
          else
            git log --pretty=format:'- %s (%h)' > CHANGELOG.md
          fi
          cat CHANGELOG.md

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          files: packages/*.deb
          body_path: CHANGELOG.md
          draft: false
          prerelease: false
```

Разбор:

- Триггер — tag с prefix'ом `v*` (например, `v1.0.0`).
- Собирает .deb, вычисляет changelog, создаёт GitHub Release с .deb attachment.

Workflow для tag:

```bash
# Автор:
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0
# → GitHub Actions автоматически собирает и публикует.
```

## 5. Кэширование Theos для скорости

```yaml
      - name: Cache Theos
        uses: actions/cache@v3
        with:
          path: ~/theos
          key: theos-roothide-${{ hashFiles('**/*.yml') }}
          restore-keys: theos-roothide-

      - name: Setup Theos (if not cached)
        run: |
          if [ ! -d "$HOME/theos" ]; then
            git clone --recursive https://github.com/roothide/Theos.git ~/theos
          fi
          echo "THEOS=$HOME/theos" >> $GITHUB_ENV
          brew install ldid dpkg xz
```

Сокращает время сборки с 5+ минут до 1 минуты.

## 6. Nightly builds

```yaml
on:
  schedule:
    - cron: '0 2 * * *'   # каждый день 02:00 UTC

jobs:
  nightly:
    # ... то же что CI, но c FINALPACKAGE=0 ...
```

Полезно, если репозиторий активно разрабатывается.

## 7. Публикация в собственный .deb-репозиторий

Если у вас свой web-repo (Procursus-compatible):

```yaml
      - name: Publish to repo
        env:
          REPO_SSH_KEY: ${{ secrets.REPO_SSH_KEY }}
        run: |
          echo "$REPO_SSH_KEY" > /tmp/ssh_key
          chmod 600 /tmp/ssh_key
          scp -i /tmp/ssh_key -o StrictHostKeyChecking=no \
              packages/*.deb user@repo.example.com:/var/www/debs/

          ssh -i /tmp/ssh_key -o StrictHostKeyChecking=no \
              user@repo.example.com \
              "cd /var/www && dpkg-scanpackages debs > Packages && gzip -9 -k Packages && apt-ftparchive release . > Release"
```

Это обновляет Packages index на вашем сервере, после чего Sileo/Zebra подхватывают.

## 8. Запуск integration tests из CI (сложно)

CI без physical device обычно не делает integration tests. Варианты:

- **Self-hosted runner** — ваш Mac mini с подключённым iPhone.
- **Cloud CI с устройствами** — Firebase Test Lab, Browserstack (ограничения на jailbreak).
- **Device farm** — redacted, редко применимо для jailbreak.

Для большинства — unit tests + manual integration на одном устройстве.

## 9. Secrets

Чувствительные данные:

- `REPO_SSH_KEY` — для публикации.
- `APP_SIGNING_CERT` — для TrollStore-pack'ов (если применимо).
- `DEPIFY_API_KEY` — для сторонних репозиториев.

Хранятся в GitHub → Settings → Secrets. Доступны через `${{ secrets.NAME }}`.

**Не** коммитьте в git. Если ошиблись — rotate секрет немедленно.

## 10. Статус-бейдж в README

```markdown
![CI](https://github.com/you/mytweak/actions/workflows/ci.yml/badge.svg)
![Release](https://github.com/you/mytweak/actions/workflows/release.yml/badge.svg)
```

Удобно для пользователя увидеть «проходит ли CI».

## 11. Таблица «CI vs manual»

| Задача | CI | Manual |
|---|---|---|
| Сборка .deb | ✓ (на каждый push) | — только для dev iteration |
| Unit tests | ✓ | — |
| Inspect .deb (no hardcode) | ✓ | — |
| Integration test | manual (на устройстве) | ✓ |
| Релиз | ✓ (на tag) | — |
| Changelog | ✓ (из git log) | — |
| Публикация в repo | ✓ (с секретами) | — |

## 12. Что подтверждено

- **Подтверждено GitHub Actions docs**: `actions/checkout`, `actions/upload-artifact`, `actions/cache`.
- **Подтверждено**: `maxim-lobanov/setup-xcode` — популярный action для Xcode selection.
- **Подтверждено проектом Dopamine**: `.github/workflows/main.yml` в репозитории — похожий паттерн; см. `BUILD.md`.
- **Требует проверки**: `softprops/action-gh-release` версия — может обновляться.

## 13. Выводы

Главный тезис: **CI превращает «забыл запустить тест» в невозможное**. Pipeline стоит один раз, работает годами, экономит часы отладки.

Для production твика без CI — риск «сборка пропустила баг». С CI — автоматика ловит до релиза.

## 14. См. также

- Advanced 2 — testing.
- Глава IV.10 — сборка и инспекция.
- `BUILD.md` в репозитории Dopamine — пример CI для jailbreak-проекта.
