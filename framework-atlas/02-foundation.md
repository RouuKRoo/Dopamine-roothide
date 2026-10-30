# Атлас: Foundation

Фундамент iOS runtime. `NSString`, `NSArray`, `NSDictionary`, `NSFileManager`, `NSUserDefaults`, `NSURLSession`.

## 1. Классы для хука

### 1.1. File system

| Класс.Метод | Описание |
|---|---|
| `NSFileManager -fileExistsAtPath:` | Проверка существования |
| `NSFileManager -contentsOfDirectoryAtPath:error:` | Содержимое папки |
| `NSFileManager -attributesOfItemAtPath:error:` | Атрибуты |
| `NSFileManager -removeItemAtPath:error:` | Удаление |
| `NSFileManager -createDirectoryAtPath:...` | Создание |
| `NSFileManager -copyItemAtPath:toPath:error:` | Копирование |
| `NSFileManager -moveItemAtPath:toPath:error:` | Перемещение |

Рецепт 1 Cookbook — полный пример.

### 1.2. User defaults

| Класс.Метод | Описание |
|---|---|
| `NSUserDefaults -objectForKey:` | Чтение |
| `NSUserDefaults -setObject:forKey:` | Запись |
| `NSUserDefaults -boolForKey:` / `-setBool:forKey:` | Bool shortcut |
| `NSUserDefaults -integerForKey:` / `-setInteger:forKey:` | Int shortcut |

### 1.3. Networking

| Класс.Метод | Описание |
|---|---|
| `NSURLSession -dataTaskWithRequest:completionHandler:` | HTTP request |
| `NSURLSessionDownloadTask -resume` | Download start |
| `NSURLRequest -valueForHTTPHeaderField:` | Header |
| `NSMutableURLRequest -setValue:forHTTPHeaderField:` | Set header |

Рецепты 2, 13.

### 1.4. Date / Time

| Класс.Метод | Описание |
|---|---|
| `NSDate +now`, `+date` | Current |
| `NSDate -timeIntervalSinceDate:` | Delta |
| `NSDateFormatter -stringFromDate:` | Format |

Рецепт 14.

### 1.5. XPC

| Класс.Метод | Описание |
|---|---|
| `NSXPCConnection -initWithServiceName:` | Создать |
| `NSXPCConnection -resume` | Start |
| `NSXPCConnection -remoteObjectProxy` | Get proxy |

Рецепт 19.

## 2. Что НЕ хукать

| Класс | Почему |
|---|---|
| `NSString` initWithUTF8String: | Hot path, тысячи вызовов/сек |
| `NSString -isEqualToString:` | Hot path |
| `NSArray -objectAtIndex:` | Hot path |
| `NSDictionary -objectForKey:` | Hot path |
| `NSObject -init` | Hot path, рекурсия возможна |
| `NSAutoreleasePool` | Критично для ARC runtime |

Правило: **immutable collection accessors — не хукаем**.

## 3. Thread safety

Foundation immutable классы (`NSString`, `NSArray`, `NSDictionary`) — thread-safe для read.

Mutable (`NSMutableString`, `NSMutableArray`, `NSMutableDictionary`) — **НЕ** thread-safe. Если хукаете их — используйте serial queue или `@synchronized`.

## 4. iOS 26 заметки

- Все перечисленные выше классы — стабильны.
- Новые API (например, `NSString(RegexExtensions)`) — доступны через `@available` check.
- Нет breaking changes в public Foundation между 18 → 26.

## 5. См. также

- Рецепты Cookbook 1, 2, 13, 14, 19.
- Глава IV.4 — первый ObjC hook.
- Глава IV.5 — C-функции (многие Foundation внутри используют).
