# Глава III.9. rootlesshooks и roothidehooks — хуки системных процессов

Эти две dylib — главный мотор «тихой работы» rootless + roothide на iOS 26. Они инъектируются в конкретные системные процессы (SpringBoard, cfprefsd, lsd, runningboardd) и переадресуют стандартные пути так, чтобы приложения из jailbreak-окружения выглядели для iOS «нормально». В этой главе — что и где они делают, с указанием, что из этого **нельзя дублировать** сторонним твикам.

## 1. Две dylib — две задачи

Dopamine поставляет **две** dylib хуков системных процессов:

- **`rootlesshooks.dylib`** (`BaseBin/rootlesshooks/`) — базовый слой для rootless. Переадресует пути `/var/jb/*`, обеспечивает корректную работу cfprefsd, SpringBoard, lsd в rootless-режиме.
- **`roothidehooks.dylib`** (`BaseBin/roothidehooks/`) — слой **поверх** rootless, специфичный для RootHide: обрабатывает randomized jbroot (`/var/<rand>/<rand>/.jbroot`), скрывает URL-схемы jailbreak, патчит runningboardd для корректной работы r^x-файловой системы.

Обе инжектируются в системные процессы через стандартный Logos-механизм (`.plist` filter), но результат — часть базовой инфраструктуры jailbreak, а не «обычный твик».

## 2. Как они выбирают процесс — `main.x`

Структура обеих одинакова: главный файл `main.x` в `%ctor` смотрит имя процесса и вызывает конкретный init-функцию.

`BaseBin/rootlesshooks/main.x:17–40`:

```objc
%ctor
{
    NSString *processName = getProcessName();
    /*if ([processName isEqualToString:@"installd"]) {
        extern void installdInit(void);
        installdInit();
    }
    else*/ if ([processName isEqualToString:@"cfprefsd"]) {
        extern void cfprefsdInit(void);
        cfprefsdInit();
    }
    else if ([processName isEqualToString:@"SpringBoard"]) {
        extern void springboardInit(void);
        springboardInit();
    }
    else if ([processName isEqualToString:@"lsd"]) {
        extern void lsdInit(void);
        lsdInit();
    }
}
```

Два факта:

1. **`installd` исключён** — закомментировано, с пометкой в самом `installd.x` «BOOTLOOP RISK, DO NOT TOUCH» (глава II.12).
2. Остальные три — `cfprefsd`, `SpringBoard`, `lsd` — активно хукаются.

Аналогично в roothidehooks (`BaseBin/roothidehooks/src/main.x:25–47`):

```objc
%ctor
{
    NSString *processName = getProcessName();
    if ([processName isEqualToString:@"cfprefsd"]) { cfprefsdInit(); }
    else if ([processName isEqualToString:@"lsd"]) { lsdInit(); }
    else if ([processName isEqualToString:@"runningboardd"]) { rbdInit(); }
    else if ([processName isEqualToString:@"SpringBoard"]) { sbInit(); }
}
```

Добавлен `runningboardd` — специфика RootHide.

## 3. rootlesshooks — хук за хуком

### 3.1. SpringBoard — `rootlesshooks/SpringBoard.x`

Два активных хука.

**Снапшоты приложений**. `XBSnapshotContainerIdentity.snapshotContainerPath` возвращает путь к контейнеру снапшотов:

```objc
%hook XBSnapshotContainerIdentity
- (NSString *)snapshotContainerPath
{
    NSString *path = %orig;
    if([path hasPrefix:@"/var/mobile/Library/SplashBoard/Snapshots/"]
       && ![self.bundleIdentifier hasPrefix:@"com.apple."]) {
        return JBROOT_PATH_NSSTRING(path);
    }
    return path;
}
%end
```

Разбор:

- SpringBoard хранит снапшоты (preview изображений, которые видно в переключателе приложений) в `/var/mobile/Library/SplashBoard/Snapshots/<bundleid>/...`.
- Для jailbreak-приложений (не-apple) этот путь перенаправляется в `JBROOT_PATH_NSSTRING(...)` — т. е. в `<jbroot>/var/mobile/Library/SplashBoard/Snapshots/<bundleid>/...`.
- Без этого хука снапшоты jailbreak-приложений либо не сохранялись бы, либо попадали в rootfs и были бы удалены при system update.

**fcntl + F_SETPROTECTIONCLASS**. Второй хук:

```objc
%hookf(int, fcntl, int fildes, int cmd, ...) {
    if (cmd == F_SETPROTECTIONCLASS) {
        char filePath[PATH_MAX];
        if (fcntl(fildes, F_GETPATH, filePath) != -1) {
            if (string_has_prefix(filePath,
                JBROOT_PATH_CSTRING("/var/mobile/Library/SplashBoard/Snapshots"))) {
                return 0;
            }
        }
    }
    ...
    return %orig(fildes, cmd, arg1, arg2, ..., arg10);
}
```

Разбор:

- При сохранении снапшота SpringBoard вызывает `fcntl(fd, F_SETPROTECTIONCLASS, ...)` чтобы задать Data Protection-класс (например, «Available after first unlock»).
- Для файлов внутри jbroot/SplashBoard/Snapshots — пропускаем вызов (возвращаем 0 успешно). Комментарий в коде: «doesn't work and causes snapshots to not be saved correctly».
- Из-за того что `fcntl` — варг-функция, нужно пробросить до 10 аргументов. Это стандартный паттерн для хуков vararg C-функций.

### 3.2. cfprefsd — `rootlesshooks/cfprefsd.x`

Главный хук — `_CFPrefsGetPathForTriplet`, определяющий путь к .plist для (bundle_id, user, host) triplet:

```objc
%hookf(BOOL, _CFPrefsGetPathForTriplet, CFStringRef bundleIdentifier,
       CFStringRef user, BOOL byHost, CFStringRef path, UInt8 *buffer)
{
    BOOL orig = %orig(bundleIdentifier, user, byHost, path, buffer);
    if(orig && buffer && !access("/var/jb", F_OK))
    {
        NSString* origPath = [NSString stringWithUTF8String:(char*)buffer];
        BOOL needsRedirection = preferencePlistNeedsRedirection(origPath);
        if (needsRedirection) {
            strcpy((char*)buffer, "/var/jb");
            strcat((char*)buffer, origPath.UTF8String);
        }
    }
    return orig;
}
```

Логика (упрощённо):

1. Получить оригинальный путь к .plist.
2. Проверить, нужна ли переадресация через `preferencePlistNeedsRedirection(origPath)`.
3. Если да — перенаправить в `/var/jb/<orig_path>`.

Функция `preferencePlistNeedsRedirection` исключает:
- пути, уже начинающиеся с `/private/var/mobile/Containers`, `/var/db`, `/var/jb`;
- .plist, названия которых начинаются с `com.apple.`, `systemgroup.com.apple.`, `group.com.apple.`;
- длинный whitelist системных .plist (`.GlobalPreferences.plist`, `bluetoothaudiod.plist`, `NetworkInterfaces.plist`, `OSThermalStatus.plist`, `preferences.plist`, `wifid.plist`, `nfcd.plist`, …).

Все остальные .plist (пользовательские, твиковые) переадресуются в rootless-пространство.

Для автора твика это ключевой момент: **ваши `CFPreferences` для своего твика автоматически попадают в jbroot**. Вам не нужно писать свой path-translation.

### 3.3. lsd — `rootlesshooks/lsd.x`

Разбор был в главе II.12: два хука (`_LSGetInboxURLForBundleIdentifier` + `_LSServer_RebuildApplicationDatabases` → вызов `uicache -a`).

### 3.4. installd — выключен (bootloop risk)

`rootlesshooks/installd.x` содержит закомментированный хук `MIGlobalConfiguration._bundleIDMapForBundlesInDirectory:` с явной пометкой «BOOTLOOP RISK, DO NOT TOUCH». См. главу II.12.

## 4. roothidehooks — расширение для randomized jbroot

### 4.1. pathhook — универсальная переадресация

`BaseBin/roothidehooks/src/pathhook.x` даёт функцию `pathhook()`, хукающую `_CFCopyHomeDirURLForUser`:

```c
CFURLRef new__CFCopyHomeDirURLForUser(const char *username, bool fallBackToHome)
{
    CFURLRef url = orig__CFCopyHomeDirURLForUser(username, fallBackToHome);
    char path[PATH_MAX]={0};
    if(CFURLGetFileSystemRepresentation(url, 0, (UInt8*)path, sizeof(path)))
    {
        const char* jbpath = rootfs(path);
        if(strncmp(jbpath, "/rootfs/", sizeof("/rootfs/")-1) == 0)
        {
            CFRelease(url);
            const char* newpath = jbroot(path);
            url = CFURLCreateFromFileSystemRepresentation(
                kCFAllocatorDefault, (const UInt8*)newpath, strlen(newpath), true);
        }
    }
    return url;
}
```

Разбор:

- `rootfs(path)` — функция из `libroot`, возвращает «rootfs-view» пути (путь, каким он виден со стороны настоящей rootfs).
- Если результат начинается с `/rootfs/` — путь переадресуется через `jbroot(path)` в jbroot-пространство.
- Это универсальный путь для процессов, использующих `_CFCopyHomeDirURLForUser` — важно для системных демонов, которые проверяют user's home.

### 4.2. runningboardd — исправление r^x-архитектуры

`BaseBin/roothidehooks/src/runningboardd.x` исправляет случай «RUNNINGBOARD 3735883980» из комментария к файлу. Он возникает, когда roothide uicache переадресует переменные среды jailbreak-приложений в jbroot (в r^x-архитектуре файловой системы), а runningboardd затем пытается lock'нуть эти файлы в memory.

Хук `fixAllowedPaths` переписывает список `allowed locked file paths`:

```objc
NSSet<NSString*>* fixAllowedPaths(pid_t pid, NSSet<NSString*>* paths)
{
    char* procpath = proc_get_path(pid, NULL);
    if(strncmp(rootfs(procpath), "/rootfs/", sizeof("/rootfs/")-1) == 0)
        return paths; // не jailbroken-приложение

    NSMutableSet* result = [NSMutableSet new];
    for (NSString* path in paths) {
        NSString* newPath = path;
        if([rootfs(path) hasPrefix:@"/rootfs/"]) {
            newPath = jbroot(path);
        }
        char real_path[PATH_MAX]={0};
        if(realpath(newPath.fileSystemRepresentation, real_path))
            [result addObject:@(real_path).stringByStandardizingPath];
    }
    return result;
}
```

Разбор:

- Для jailbreak-приложения все пути из `paths` конвертируются из rootfs-view в jbroot-view.
- Через `realpath` затем приводятся к канонической форме.
- Для системных приложений — возвращается оригинальный набор.

### 4.3. springboard — скрытие и инструментовка SpringBoard

`BaseBin/roothidehooks/src/springboard.x` содержит хуки на private классы SpringBoard — детальный разбор не делаем, потому что это внутренние селекторы, которые требуют class-dump для сверки на iOS 26.

Общий смысл: скрыть jailbreak-приложения от определённых детектов SpringBoard и обеспечить корректное отображение.

### 4.4. cfprefsd и lsd — расширение rootless-слоя

- `roothidehooks/src/cfprefsd.x` — дополнительные хуки сверх rootless, учитывающие randomized путь.
- `roothidehooks/src/lsd.x` — скрытие jailbreak URL-схем (разбор в главе II.12: `_LSURLOverride`, `_LSCanOpenURLManager`).

### 4.5. palera1n.x

`BaseBin/roothidehooks/src/palera1n.x` — хуки, специфичные для palera1n-режима (checkra1n-based). На A13 + iOS 26 это обычно не используется, но хуки присутствуют для совместимости с установками, где Dopamine устанавливается поверх palera1n.

## 5. Как они собираются и ставятся

Обе dylib собираются как Theos-tweaks:

- `BaseBin/rootlesshooks/Makefile` — собирает `.theos/obj/rootlesshooks.dylib`.
- `BaseBin/roothidehooks/Makefile` — аналогично для `roothidehooks.dylib`.

Из `BaseBin/Makefile:66–70`:

```make
rootlesshooks: .build .include libjailbreak
	$(MAKE) -C rootlesshooks
	@cp rootlesshooks/.theos/obj/rootlesshooks.dylib .build

roothidehooks: .build .include libjailbreak libroothider
	$(MAKE) -C roothidehooks
	@cp roothidehooks/.theos/obj/roothidehooks.dylib .build
```

После сборки обе .dylib попадают в `basebin.tar`, который разворачивается на устройстве в `<jbroot>/basebin/`. Их filter plist (`roothidehooks.plist` — пустой, значит фильтруется через механизм вне плоскости стандартного Logos; на самом деле они инъектируются через загрузку в системные процессы, которые их прямо `dlopen`-ят из TweakLoader или analogous механизмов).

## 6. Что НЕЛЬЗЯ дублировать сторонним твикам

Таблица «уже сделано — не повторять»:

| Задача | Что уже делает Dopamine |
|---|---|
| Переадресация `cfprefsd` путей для твиков | `rootlesshooks/cfprefsd.x` + `roothidehooks/src/cfprefsd.x` |
| Снапшоты SpringBoard для jailbreak-приложений | `rootlesshooks/SpringBoard.x` |
| `fcntl F_SETPROTECTIONCLASS` для snapshots | тот же файл |
| `_LSGetInboxURLForBundleIdentifier` → jbroot | `rootlesshooks/lsd.x` |
| Запуск `uicache -a` после ребилда LS БД | тот же файл |
| Скрытие jailbreak URL-схем | `roothidehooks/src/lsd.x` |
| r^x path fix для runningboardd | `roothidehooks/src/runningboardd.x` |
| `_CFCopyHomeDirURLForUser` → jbroot | `roothidehooks/src/pathhook.x` |

Если ваш твик собрался хукнуть что-то из этого списка — скорее всего, вы делаете лишнюю работу или входите в конфликт.

## 7. Что подтверждено

- **Подтверждено исходниками**:
  - `main.x` в обеих dylib — цепочка диспатча по имени процесса.
  - `cfprefsd` хук `_CFPrefsGetPathForTriplet` + whitelist — `rootlesshooks/cfprefsd.x:1–65`.
  - `SpringBoard` хук `XBSnapshotContainerIdentity.snapshotContainerPath` — `rootlesshooks/SpringBoard.x:28–40`.
  - `fcntl F_SETPROTECTIONCLASS` обход — тот же файл.
  - `pathhook` для `_CFCopyHomeDirURLForUser` — `roothidehooks/src/pathhook.x:11–40`.
  - `runningboardd` фикс для «RUNNINGBOARD 3735883980» — `roothidehooks/src/runningboardd.x:7–45`.
- **Подтверждено Makefile**: `rootlesshooks` зависит от `libjailbreak`; `roothidehooks` — от `libjailbreak` и `libroothider`.
- **Требует проверки на устройстве**: все хуки active (не выпали из загрузки) — проверяется `otool -L` на процессе SpringBoard/cfprefsd/lsd на jailbroken устройстве.
- **Не подтверждено**: что весь `springboard.x` работает идентично на iOS 26.0 и 26.0.1 — private классы SpringBoard меняются между минорными версиями.

## 8. Выводы

Главный тезис: `rootlesshooks` + `roothidehooks` — это **системная инфраструктура, работающая внутри четырёх ключевых системных процессов** (SpringBoard, cfprefsd, lsd, runningboardd). Их задача — сделать так, чтобы jailbreak-приложения жили в rootless + randomized-jbroot пространстве «как родные». Для автора стороннего твика это означает:

- Вам **не надо** переадресовывать пути `/var/jb/*` вручную — для `CFPreferences` это уже делается.
- Вам **не надо** регистрировать свой snapshot path — SpringBoard хук уже переадресует.
- Вам **не надо** решать проблему hidden URL-schemes — roothide это уже делает.
- И ни в коем случае **не хукайте** `installd` — проект сам отступил от этого после bootloop.

Что это даёт: твик, корректно использующий `libroot` и `JBROOT_PATH`, автоматически получает все эти бенефиты.

## 9. Что дальше

- `libroot` — публичный API путей для сторонних твиков — глава III.10.
- `basebin` build + bootstrap — как это всё разворачивается на устройстве — глава III.11.
- Сводная таблица задач — глава III.12.
