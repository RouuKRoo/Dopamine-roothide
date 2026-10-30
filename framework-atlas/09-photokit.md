# Атлас: PhotoKit

Framework доступа к Photos. Public, но privacy-sensitive.

## 1. Главные классы

### 1.1. PHAsset

Один медиа-asset (фото/видео):

```objc
PHAsset *asset = ...;
asset.localIdentifier;
asset.mediaType;        // image, video, audio
asset.mediaSubtypes;    // panorama, HDR, screenshot
asset.pixelWidth / pixelHeight;
asset.creationDate;
asset.location;         // CLLocation если есть
```

### 1.2. PHFetchResult, PHFetchOptions

```objc
PHFetchOptions *options = [PHFetchOptions new];
options.predicate = [NSPredicate predicateWithFormat:@"mediaType == %d",
                                                      PHAssetMediaTypeImage];
PHFetchResult<PHAsset *> *assets = [PHAsset fetchAssetsWithOptions:options];
```

### 1.3. PHImageManager

Загрузка изображения:

```objc
[[PHImageManager defaultManager] requestImageForAsset:asset
                                            targetSize:CGSizeMake(320, 320)
                                           contentMode:PHImageContentModeAspectFit
                                               options:nil
                                         resultHandler:^(UIImage *result, NSDictionary *info) {
    // use result
}];
```

### 1.4. PHPickerViewController (iOS 14+)

Публичный picker:

```objc
PHPickerConfiguration *config = [[PHPickerConfiguration alloc]
                                   initWithPhotoLibrary:[PHPhotoLibrary sharedPhotoLibrary]];
config.filter = [PHPickerFilter imagesFilter];
config.selectionLimit = 1;

PHPickerViewController *picker = [[PHPickerViewController alloc] initWithConfiguration:config];
picker.delegate = self;
[self presentViewController:picker animated:YES completion:nil];
```

## 2. Хуки

Рецепт 16 cookbook — полный пример.

Что можно:

- `PHPickerViewController initWithConfiguration:` — ловить открытие picker.
- `PHImageManager requestImageForAsset:...` — что приложение запрашивает.
- `PHFetchResult` iteration (сложнее — generic protocol).
- Delegate `-picker:didFinishPicking:` — что выбрал user.

## 3. Privacy (iOS 14+ limited photo access)

Приложение запрашивает:

- **Full access** — все photos.
- **Limited** — только выбранные пользователем.
- **None**.

iOS 26 продолжает fine-grained модель.

Для автора твика в своём app:

- Можете использовать все levels.
- Хук запросов — для диагностики.
- **Не** обход privacy — iOS не даст иначе через public API.

## 4. Этические границы

| Задача | Собственное app | Чужое app |
|---|---|---|
| Логировать PhotoPicker usage | OK | Серая |
| Логировать какие assets запрашиваются | OK | Серая |
| Копировать фото в фоне | **Нет** | **Абсолютно нет** |
| Снятие limited photo access | **Нет** | **Нет** |

Photos — privacy сенситив. Правила строгие.

## 5. iOS 26 заметки

- PHPickerViewController — стабилен с iOS 14.
- PhotoKit API — стабилен.
- Public read access через picker остаётся main путь.
- AI-related (Image Playground и т. п.) — через новые API, не ломают existing.

## 6. См. также

- Рецепт 16 — PHPicker.
- iOS 26 changes 8 — Privacy.
- Атлас LocalAuthentication (связанный).
