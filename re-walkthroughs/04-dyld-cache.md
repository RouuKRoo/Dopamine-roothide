# RE 4. Работа с dyld shared cache

Все iOS frameworks живут в одном огромном файле. Как извлечь нужный и работать с ним.

## 1. Что такое dyld_shared_cache

- **Путь на устройстве**: `/System/Library/dyld/dyld_shared_cache_arm64` (на A13) или `_arm64e` (A14+).
- **Размер**: несколько gigabyte (все public + private frameworks iOS).
- **Назначение**: один shared mapping для всех процессов → экономия памяти и startup time.
- **Formatformat**: специальный Apple cache format, не стандартный Mach-O.

## 2. Для автора твика — зачем лезть

На устройстве framework-ы **не существуют** как отдельные файлы (кроме stubs для linker). Все собраны в dyld_shared_cache.

Если хотите class-dump чего-то — нужно сначала извлечь из cache.

## 3. Инструмент: ipsw (blacktop/ipsw)

```bash
# Установка на macOS:
brew install blacktop/tap/ipsw

# или Linux: смотрите github.com/blacktop/ipsw
```

## 4. Workflow

### 4.1. Получить IPSW

- Apple делает IPSW доступным для download на `ipsw.me` (community-mirror) или через Apple-signed URLs.
- Для вашей версии iOS 26.0.1 на iPhone 11 — скачать соответствующий `.ipsw` файл.

**Внимание**: IPSW — большой (~5-7 GB). Нужен быстрый интернет.

### 4.2. Извлечь dyld_shared_cache из IPSW

```bash
ipsw extract --dyld /path/to/iPhone11,2_26.0_23B5044l_Restore.ipsw
# Создаст dyld_shared_cache_arm64 в текущей директории
```

Этот файл — несколько gigabyte. На disk требуется.

### 4.3. Список frameworks в cache

```bash
ipsw dyld info dyld_shared_cache_arm64 | head -30
# Выводит все images в cache (сотни):
# /System/Library/Frameworks/UIKit.framework/UIKit
# /System/Library/Frameworks/Foundation.framework/Foundation
# /System/Library/PrivateFrameworks/SpringBoardFoundation.framework/SpringBoardFoundation
# ... и т. д.
```

### 4.4. Извлечь один framework

```bash
# По path в cache:
ipsw dyld extract dyld_shared_cache_arm64 \
    --image "/System/Library/PrivateFrameworks/SpringBoardFoundation.framework/SpringBoardFoundation" \
    -o /tmp/sbf/

# Теперь у вас:
ls /tmp/sbf/
# SpringBoardFoundation  — стандартный Mach-O
```

### 4.5. Работать с extracted binary

Как обычный Mach-O:

```bash
# Информация:
file /tmp/sbf/SpringBoardFoundation
# Mach-O 64-bit dynamically linked shared library arm64

# Load commands:
otool -l /tmp/sbf/SpringBoardFoundation | head

# Dependencies:
otool -L /tmp/sbf/SpringBoardFoundation

# Symbols:
nm -gU /tmp/sbf/SpringBoardFoundation

# Class-dump:
class-dump-dyld /tmp/sbf/SpringBoardFoundation -o /tmp/sbfdump/
```

## 5. Shared cache format особенности

Dyld shared cache — не просто concat Mach-O'в. Это:

- Единый shared read-only mapping в virtual memory.
- Pre-linked chained fixups.
- Shared stubs (экономия size).
- Codesign'ed as single blob.

Это **означает**: `ipsw dyld extract` не просто копирует bytes — делает полный «un-cache» (восстанавливает standalone Mach-O structure).

Результирующий binary иногда отличается от того, что был бы standalone — например, inlined stubs могут быть разными. Но для class-dump / symbol inspection — работает идентично.

## 6. Альтернативы ipsw CLI

### 6.1. jtool2

```bash
jtool2 -d objc /tmp/sbf/SpringBoardFoundation
```

Дает ObjC class/methods info без class-dump.

### 6.2. dsc_extractor (Apple)

Включён в Xcode:

```bash
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/bin/dsc_extractor \
    dyld_shared_cache_arm64 \
    /tmp/output-dir/
```

Создаёт директорию со всеми framework-ами в правильной иерархии.

### 6.3. Hopper / Ghidra / IDA Pro

Открывают dyld shared cache напрямую, позволяют navigate без extraction. Удобно для interactive RE.

## 7. На устройстве — работа с cache

Если dyld_shared_cache на устройстве нужен (например, для dynamic class-dump):

```bash
ssh mobile@<ip>
ls -la /System/Library/dyld/
# dyld_shared_cache_arm64 — размер несколько GB
```

Копировать на хост:

```bash
scp mobile@<ip>:/System/Library/dyld/dyld_shared_cache_arm64 /tmp/
```

(Это медленно и файл большой; обычно проще скачать IPSW с Apple.)

## 8. iOS 26 заметки

- Format dyld_shared_cache остался тот же (с iOS 15 — chained fixups в cache).
- На A13 (arm64) — обычный arm64 cache.
- Размер растёт с каждой версией — на iOS 26 ожидаемо 5-8 GB.

## 9. Что подтверждено

- **Подтверждено**: `ipsw` CLI (blacktop/ipsw) — публичный tool, работает с dyld cache.
- **Подтверждено Apple**: формат dyld shared cache — Apple opensource dyld; детали в `dyld/launch-cache/`.
- **Требует проверки**: что ваша конкретная версия ipsw CLI работает с iOS 26 cache — проверьте README на github.
- **Не подтверждено**: что IPSW для вашего устройства/версии доступен на Apple signed URL — зависит от того, какой текущий «current signed» для вашего устройства.

## 10. См. также

- Deep-dive 02 — dyld 4+ и chained fixups.
- Deep-dive 15 — MachOMerger (похожий low-level работа).
- RE 1, 2, 3 — используют extracted framework.
- Глава VI.2 — static vs device.
