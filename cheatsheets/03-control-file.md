# Шпаргалка 3. control-файл Debian-пакета

`control` — метаданные `.deb` пакета. Располагается внутри `.deb` → `DEBIAN/control`. Theos создаёт его из вашего `control`-файла в корне проекта.

## Обязательные поля

| Поле | Пример | Описание |
|---|---|---|
| `Package` | `com.example.mytweak` | Reverse-DNS id пакета. Уникальный. |
| `Name` | `MyTweak` | Human-readable имя (в Sileo/Zebra). |
| `Version` | `1.0.0` | Semver. Инкрементировать при релизе. |
| `Architecture` | `iphoneos-arm` | Для rootless/RootHide на arm64. |
| `Description` | `My tweak` | Короткое описание. |

## Рекомендованные поля

| Поле | Пример | Описание |
|---|---|---|
| `Maintainer` | `You <you@example.com>` | Контакт сопровождающего. |
| `Author` | `You` | Автор. |
| `Section` | `Tweaks` | Категория: Tweaks, System, Development, Themes, Utilities. |
| `Depends` | см. ниже | Зависимости. |
| `Depiction` | `https://repo.example.com/depictions/com.example.mytweak` | URL страницы пакета. |
| `Icon` | `file:///Library/MyTweak/icon.png` | Иконка для Sileo. |

## Опциональные поля

| Поле | Назначение |
|---|---|
| `Priority` | `optional`, `required` — для системных пакетов. |
| `Essential` | `yes` — не удалять. |
| `Replaces` | Какой пакет заменяет (при upgrade). |
| `Conflicts` | С каким пакетом конфликтует. |
| `Breaks` | Какие версии других пакетов ломает. |
| `Pre-Depends` | Зависимости до разворачивания. |
| `Provides` | Виртуальный пакет (альтернатива). |
| `Suggests` | Рекомендуется, но не требуется. |
| `Recommends` | Сильно рекомендовано. |
| `Enhances` | Расширяет функциональность другого. |
| `Tag` | Теги: `role::hacker`, `purpose::extension`. |
| `Sponsor` | Кто спонсировал разработку. |

## Поле Depends — подробно

Синтаксис:

- Простые зависимости: `pkgA, pkgB`
- Альтернативы (OR): `pkgA | pkgB` (любой подходит)
- Версия: `pkgA (>= 1.0)`, `pkgA (<< 2.0)`, `pkgA (= 1.0.0)`
- Комбинация: `firmware (>= 15.0) & firmware (<= 26.9)`

### Для твика на RootHide Dopamine

```
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine
```

Разбор:

- `firmware (>= 15.0)` — минимальная iOS.
- `ellekit | mobilesubstrate | substitute` — hook-менеджер (любой подойдёт).
- `libroot-dopamine` — libroot.

### Если твик имеет PreferenceBundle

```
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine, preferenceloader
```

### Если нужны специфические версии

```
Depends: firmware (>= 15.0), ellekit (>= 1.0), libroot-dopamine (>= 1.0.1)
```

## Полный пример для твика

```
Package: com.example.mytweak
Name: MyTweak
Version: 1.0.0
Architecture: iphoneos-arm
Description: Example tweak demonstrating preference bundle and Darwin notifications
Maintainer: Your Name <your@email>
Author: Your Name
Section: Tweaks
Depends: firmware (>= 15.0), ellekit | mobilesubstrate | substitute, libroot-dopamine, preferenceloader
Depiction: https://example.com/depictions/com.example.mytweak
Icon: file:///Library/PreferenceBundles/MyTweakPrefs.bundle/icon.png
Tag: role::hacker, purpose::extension
```

## Правила именования

- **Package** всегда lowercase, точки-разделители, без пробелов и спецсимволов.
- **Reverse DNS**: ваш домен задом наперёд + имя твика (`com.example.mytweak`).
- **Name** может содержать пробелы и большие буквы.
- **Version** строго semver: `MAJOR.MINOR.PATCH` или `MAJOR.MINOR.PATCH-<pre>`.

## Многострочные значения

Для Description:

```
Description: Short line (shown first)
 This is a longer description that spans
 multiple lines. Each continuation line must
 start with a single space.
 .
 Empty paragraph separator is " ." (space + dot).
```

## maintainer scripts

В `.deb` можно добавить скрипты, выполняемые при install/remove:

| Скрипт | Когда вызывается |
|---|---|
| `preinst` | До распаковки |
| `postinst` | После распаковки |
| `prerm` | До удаления файлов |
| `postrm` | После удаления файлов |

Theos автоматически добавляет их, если файлы `layout/DEBIAN/preinst` (etc.) существуют в проекте.

Типовой `postinst` для твика с PreferenceBundle:

```bash
#!/bin/bash
case "$1" in
    configure)
        # Перезапустить cfprefsd для подхвата новых preferences
        killall -9 cfprefsd 2>/dev/null || true
        # Перезапустить SpringBoard если установлен твик для него
        # killall -9 SpringBoard 2>/dev/null || true
        ;;
esac
exit 0
```

## Инспекция собранного .deb

```bash
dpkg-deb -I your.deb              # control + maintainer scripts
dpkg-deb -c your.deb              # список файлов
dpkg-deb -x your.deb /tmp/x       # распаковать
dpkg-deb -e your.deb /tmp/meta    # извлечь DEBIAN/*
cat /tmp/meta/control             # прочитать control
```

## Типовые ошибки

| Ошибка | Диагноз |
|---|---|
| `conflicts with X` при dpkg -i | Другой пакет имеет тот же Package. Переименуйте. |
| `dependency problems` | `Depends` не удовлетворены — установите недостающее. |
| Пакет не появляется в Sileo | Section нестандартный или нет Depiction. |
| Апдейт не ставит новую версию | Version не инкрементирована. |
| `.deb` сломался при dpkg -i | Нет финального `\n` в control. Theos обычно добавляет. |

## См. также

- Debian Policy Manual: https://www.debian.org/doc/debian-policy/
- Шпаргалка 2 — Makefile.
- Глава IV.2 — минимальный проект.
- Глава IV.10 — сборка и инспекция.
