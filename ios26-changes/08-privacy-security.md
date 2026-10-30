# Privacy и Security в iOS 26

Apple каждый релиз усиливает privacy. Для автора твика это — **граница**: что раньше работало без indicator'а, теперь показывает banner / требует разрешения. Эта глава — обзор + этика.

## 1. Главное: iOS не любит скрытое наблюдение

Направление Apple с iOS 14+:

- Pasteboard access → banner «AppX pasted from...».
- Camera active → yellow/green dot в status bar.
- Microphone active → yellow dot.
- Location → precise/approximate opt-in, banner при background use.
- Local Network → explicit prompt при первом использовании.
- Clipboard check → banner.
- Nearby devices → prompt.

Для автора твика это — **constraint**: подавить эти indicators технически возможно через хуки в SpringBoard, но:

- Нестабильно (iOS 26 скорее всего продолжает усиление, hook может сломаться на 26.0.1).
- Этически — вы отключаете защиту пользователя. Для чужого app = tracking tool.
- Юридически — Zone грубого нарушения в большинстве юрисдикций.

Этот текст **не описывает**, как убрать эти indicators. Описывает — как использовать iOS 26 Privacy correctly в своём app.

## 2. TCC (Transparency, Consent, Control)

TCC — framework разрешений iOS. При первом обращении app к chaувствительному ресурсу — iOS показывает prompt. Ответ сохраняется в `TCC.db`.

На iOS 26 TCC:

- Fine-grained Photos access (специфические фотографии вместо всех).
- Fine-grained Contacts.
- Microphone / Camera / Location по-прежнему.
- Local Network.
- Bluetooth (с 13+).

Для автора твика в своём app:

- Запросы разрешения — через public API (`AVCaptureDevice.requestAccessForMediaType:`, `CLLocationManager.requestWhenInUseAuthorization`, etc.).
- Хукать для логирования — безопасно для своего app.
- Хукать для **обхода** — незаконно (нарушение privacy framework Apple).

## 3. Camera / Microphone indicators

Жёлтая / зелёная точка в status bar:

- Рендерится SpringBoard (не ваше app).
- Сигнал исходит от `mediaserverd` / `avconferenced`.
- Отключить = hook в SpringBoard + подавить сигнал.

**Технически возможно**, **этически неприемлемо**. Эта функция защищает пользователя от скрытой записи. Автор tweak-книги не должен давать recipe.

## 4. App Attestation / DeviceCheck

iOS 14+ дало `DCDevice` для attestation (приложение подтверждает «я подлинный на подлинном устройстве»). iOS 26 продолжает.

Для jailbroken device — attestation **ломается**: ядро skewed, DCDevice возвращает ошибки или reject.

Для автора твика:

- Нельзя обойти DCDevice legitimately.
- Твики для «обхода attestation» — gray area; обычно для банкинг-apps = нарушение.

## 5. Что заявляет Apple для iOS 26

Направления:

- **AI** защиты — Apple Intelligence работает on-device, но controls от Apple.
- **Image Playground** / **Writing Tools** — privacy prompts.
- **Advanced Data Protection** — расширение end-to-end encryption.
- **Lockdown Mode** — обновления.

Для автора твика — ни одно из этих не open'ит новых hook-points. Все требуют user consent и protect user.

## 6. Этические правила для автора твика

Три правила:

1. **Собственное приложение** — всё разрешено (вы владелец данных).
2. **Чужое приложение**:
   - Observation (log) — gray area; для personal research — OK, для публичного твика — обычно нет.
   - Modification (замена values) — gray area; зависит от цели.
   - Обход защит — обычно **нет**.
3. **Системные indicators** (camera dot, mic dot, pasteboard banner) — **не трогать**. Это core user protection Apple.

## 7. Hardening вашего собственного твика

Если ваш твик обрабатывает чувствительные данные:

- `os_log` с `%{private}` для sensitive values (iOS automatically redacts).
- Clearing memory после use: `memset_s(buffer, size, 0, size)` вместо `memset`.
- Не кэшировать credentials в файлы.
- Keychain для storage credentials.

Пример:

```c
char password[256];
read_password(password, sizeof(password));
// use it
process_password(password);
// clear:
memset_s(password, sizeof(password), 0, sizeof(password));
```

`memset_s` — не optimize-able компилятором. `memset` может быть удалён оптимизацией как «dead code».

## 8. Dopamine и privacy

Dopamine сам:

- Не записывает trajectory пользователя.
- Не отправляет данные на серверы.
- Не tracks installs.

Автор твика наследует эту репутацию — но **своим** кодом. Если ваш твик шлёт telemetry — делайте это transparent (privacy policy, opt-in).

## 9. Чеклист для твика с чувствительными данными

```
[ ] Нет записи в syslog `%{public}` для credentials
[ ] Нет записи в файлы, которые видны другим приложениям
[ ] Нет отправки на внешние серверы без consent
[ ] Используется memset_s для clearing
[ ] Нет keylogger-подобного поведения
[ ] Нет захвата screenshot чужих окон
[ ] Нет записи pasteboard в фоне
[ ] Нет обхода TCC prompts
[ ] Documented privacy policy (если распространяете)
```

## 10. Что подтверждено

- **Подтверждено Apple docs**: TCC framework, DCDevice, UNUserNotificationCenter authorization.
- **Подтверждено Apple**: camera/mic indicators — hard-wired в SpringBoard.
- **Этически**: эти indicators — not for bypass.
- **Юридически**: обход privacy protection может нарушать GDPR / CCPA / local laws.

## 11. Выводы

Главный тезис: **privacy в iOS 26 — ещё строже, чем в 18**. Это ограничение автору твика; но и **возможность** построить doverие пользователя через соблюдение.

Правильный твик уважает privacy framework. Неправильный — попытается обойти; обычно ломается на 26.0.1 patch, оставляя пользователя с broken device.

## 12. См. также

- Глава II.13 — Accessibility (этика).
- Глава I.5 — что jailbreak может и не может.
- Рецепт 03 — pasteboard (с этикой).
- Рецепт 06 — keychain (с этикой).
