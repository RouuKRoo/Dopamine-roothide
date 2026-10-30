# Daemon: mediaserverd

Центральный процесс мультимедиа. **Табу для хуков**.

## 1. Что это

- **Путь**: `/usr/sbin/mediaserverd` (или `/usr/libexec/mediaserverd` на новых iOS)
- **Executable name**: `mediaserverd`
- **Запуск**: launchd
- **Привилегии**: root + extensive entitlements (DRM, codec access, hardware)

## 2. Что делает

- Единая точка доступа к аудио- и видео-ресурсам.
- Обслуживает все `AVFoundation` клиентские запросы.
- Управляет audio sessions (`AVAudioSession` global state).
- Routing звука (speaker, earphones, Bluetooth, AirPlay).
- DRM для защищённого контента.
- Hardware codec access.
- Platform-wide volume state.

## 3. Главные frameworks (загруженные в сам mediaserverd)

Отдельная экосистема, десятки private frameworks:

- `MediaExperience.framework`
- `MediaRemote.framework`
- `AVKitCore.framework`
- `MediaPlayerUI.framework`
- `MusicLibrary.framework`
- `CoreMediaStream.framework`
- Codec plugins (H264, HEVC, AV1, Dolby Audio...).

Общий размер — сотни megabytes всего кода.

## 4. Хуки Dopamine

**НЕТ** (глава II.10).

```bash
grep -rln "mediaserverd" /home/user/Dopamine-roothide/BaseBin/
# пусто
```

Это **намеренное** решение проекта. mediaserverd слишком хрупкий, цена ошибки слишком высока.

Для автора твика — **НЕ ХУКАЙТЕ mediaserverd**. Повторите вслух.

## 5. Почему это так опасно

Крах mediaserverd:

- Пропадает **весь** звук системы (включая звонки).
- Теряется state audio sessions всех приложений.
- Routing сбрасывается.
- Активные audio tracks обрываются.
- Входящий звонок прерывается.

launchd перезапустит mediaserverd, но все audio sessions умрут. Пользователь будет думать, что устройство сломалось.

## 6. Безопасные альтернативы

Что бы вы ни хотели сделать с mediaserverd — есть **public AVFoundation** в приложении:

| Хочу | Public API |
|---|---|
| Логировать audio session changes | `AVAudioSession` notifications |
| Перехватить volume change | `AVAudioSession.outputVolume` KVO |
| Логировать play/pause | `AVPlayer -play`/`-pause` hook в приложении |
| Routing control | `AVAudioSession.routeChangeNotification` |
| Snapshot current state | `AVAudioSession.sharedInstance` properties |

Эти — в приложении, не в mediaserverd. Хукается стандартно.

## 7. coreaudiod (iOS 17+)

С iOS 17 часть обязанностей mediaserverd выделилась в отдельный **coreaudiod**:

- **Путь**: `/usr/libexec/coreaudiod`
- **Что делает**: низкоуровневый audio graph, HAL.

coreaudiod — так же хрупкий, так же **не хукается** Dopamine. Правило то же.

## 8. Диагностика без хуков

```bash
ssh mobile@<ip>
log stream --process mediaserverd --device <udid>
# или:
log stream --process coreaudiod --device <udid>
```

Логи mediaserverd богаты — все audio events системы. Фильтруйте по вашему приложению.

Для мониторинга audio session в своём app:

```objc
[[NSNotificationCenter defaultCenter] addObserver:self
                                          selector:@selector(audioRouteChanged:)
                                              name:AVAudioSessionRouteChangeNotification
                                            object:nil];
```

## 9. Что подтверждено

- **Подтверждено исходниками**: Dopamine не содержит хуков mediaserverd (glava II.10).
- **Подтверждено практикой**: mediaserverd — исторически fragile across iOS versions.
- **Подтверждено Apple docs**: AVFoundation как рекомендованная public API.

## 10. Выводы

Главный тезис: **mediaserverd — абсолютное табу**. Dopamine сам не трогает, автор твика — тем более.

Для audio-related tweak — всё делается в приложении через AVFoundation. Без исключений.

## 11. См. также

- Глава II.10 — мультимедиа, камера, звук, haptics.
- Daemon: backboardd — связанный процесс для haptics.
