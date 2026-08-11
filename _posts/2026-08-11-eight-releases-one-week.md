---
title: "Вісім релізів за тиждень: mute, Swift 6 і fallback, який не врятував"
date: 2026-08-11 12:30:00 +0300
excerpt: "Як маленька macOS-утиліта пройшла шлях від першого commit до v0.0.8, чому gain zero — не mute, а резервний Twitter-клієнт заблокував чергу з 34 постів."
categories:
  - engineering
tags:
  - macos
  - swift
  - swiftui
  - coreaudio
  - concurrency
  - ci
  - homebrew
  - analytics
  - devops
  - ai
---

На початку серпня я хотів зробити маленьку native macOS-утиліту: показувати стан мікрофона в menu bar і перемикати mute глобальним hotkey. Через тиждень [MicStatusAI](https://github.com/Disconnecter/MicStatusAI) мав вісім релізів, Swift 6 strict concurrency, локалізацію, налаштовуваний HUD, Homebrew-дистрибуцію, privacy-first analytics і значно більше нюансів, ніж передбачав початковий опис.

Паралельно з цим production-бот, який переносить дописи з Telegram у X, накопичив чергу з 34 повідомлень. Основний API перестав працювати, fallback був на місці — і теж не врятував систему.

Обидві історії виявилися про одне: назва механізму ще не гарантує його семантику. `volume = 0` не обов'язково означає mute. `service active` не означає working pipeline. Наявність fallback не означає, що черга рухається.

## День перший: «маленька утиліта»

Початкові вимоги вміщувалися в кілька пунктів:

- стежити за default input device;
- показувати однакові за стилем status bar icons;
- дозволяти зупинити monitoring;
- mute/unmute через menu і configurable global hotkey;
- не запитувати доступ до запису аудіо;
- створити локальний Git-репозиторій і робити commits у логічних точках.

Перша версія використовувала SwiftUI для menu bar і Settings, CoreAudio для стану input device та Carbon для global hotkey. Дуже швидко з'ясувалося, що навіть кольори потребують точної моделі станів. Спочатку червоний означав активний мікрофон, потім зелений. Окремо були `monitoring stopped`, `input level is zero` і справжній mute.

Це не косметика. Якщо UI стискає кілька різних технічних станів до двох кольорів, користувач отримує впевненість, якої система не може гарантувати.

## Інфраструктура раніше за polish

Ще до першого публічного релізу проєкт отримав:

- XcodeGen замість hand-maintained project file;
- SwiftLint і build checks;
- PR та issue templates;
- Dependabot;
- protected `main` із required macOS build;
- squash merge без force-push і deletion;
- release workflow;
- окремий Homebrew tap із write-only deploy key.

Перший tag запускав повний ланцюжок:

```text
tag
  -> macOS Release build
  -> ZIP + SHA-256
  -> GitHub Release
  -> Homebrew cask update
```

Після [v0.0.1](https://github.com/Disconnecter/MicStatusAI/releases/tag/v0.0.1) я прибрав x86_64, додав MIT license і випустив v0.0.2. Далі з'явилися localization через String Catalog, короткі typed keys, коректне locale-aware форматування відсотків і суворіші lint rules.

Рання автоматизація не зробила подальші зміни безпомилковими. Вона зробила їх дешевшими для перевірки й випуску. Це важлива різниця.

## Swift 6 без аварійних виходів

Global hotkey поєднав старий C API Carbon із `@MainActor`-ізольованим Swift-кодом. Найпростіший спосіб змусити compiler замовкнути — додати `nonisolated(unsafe)` або `@unchecked Sendable`. Я свідомо заборонив обидва escape hatch.

Фінальна межа виглядала так:

- `HotKeyManager` лишився `@MainActor`;
- Carbon handles перейшли в окремий non-Sendable RAII owner;
- C callback передавав подію через `Task { @MainActor in ... }`;
- lint окремо перевіряв відсутність unsafe concurrency annotations;
- CI компілював проєкт у Swift 6 complete concurrency mode.

Це був корисний урок: strict concurrency найцінніша не тоді, коли додає annotations, а коли змушує явно намалювати ownership і межу між legacy callback та UI state.

## Gain zero — це не mute

Перші версії «вимикали» мікрофон, встановлюючи input volume у нуль. Візуально все працювало. Але це лише gain zero. Драйвер, інший channel або інша програма можуть поводитися інакше.

Сильніший варіант у CoreAudio — `kAudioDevicePropertyMute` на input scope:

```swift
AudioObjectPropertyAddress(
    mSelector: kAudioDevicePropertyMute,
    mScope: kAudioDevicePropertyScopeInput,
    mElement: kAudioObjectPropertyElementMain
)
```

Практична реалізація потребувала більше, ніж одного write:

1. Знайти mute property на main element або окремих channels.
2. Встановити значення.
3. Прочитати його назад і перевірити.
4. Використати gain zero лише для пристроїв без native mute.
5. Перенести бажаний стан на новий input device після перемикання.

Так з'явився [v0.0.6](https://github.com/Disconnecter/MicStatusAI/releases/tag/v0.0.6).

Але навіть CoreAudio mute — не гарантований hardware mute. macOS не дає public API, який глобально блокує мікрофон для всіх процесів. Реальну гарантію дають фізичний switch, від'єднання пристрою або firmware-controlled mute.

Текст у UI має відображати цю межу. «Native mute verified» і «hardware muted» — не синоніми.

## Teams і проблема власника стану

Після переходу на native mute з'явився складніший bug: старт Microsoft Teams call повертав мікрофон в unmuted state.

Перший fix зробив бажаний стан MicStatusAI авторитетним. Polling помічав external unmute і повертав mute приблизно за пів секунди. Це технічно працювало, але породило нову проблему: System Settings та інші програми більше не могли навмисно змінити стан.

CoreAudio повідомляє нове значення, але не пояснює намір і джерело зміни. Неможливо надійно відрізнити:

- Teams самовільно ввімкнув мікрофон;
- користувач натиснув unmute в Teams;
- користувач змінив state у System Settings.

Правильна модель мала б окрему політику:

```text
Monitor external state
    приймати зміни з інших програм

Enforce mute
    блокувати external unmute,
    доки lock не знято в MicStatusAI
```

Без явного product decision такий «fix» занадто агресивний. Я відкотив його. Revert тут був не поразкою, а правильним результатом перевірки семантики.

## П'ять ітерацій ширини Settings

UI теж не підкорився одному числу. Спочатку transparency slider був надто широкий. Потім широким лишалося все Settings window. Fixed width прибрав порожній простір, але почав обрізати content. Dynamic width повернув адаптивність, однак усе ще виглядав завеликим.

Рішення склалося з кількох малих змін:

- `Form` і native `Section` замість вкладених `GroupBox`;
- `ViewThatFits` для horizontal/vertical layout;
- slider із діапазоном ширини, а не одним hardcoded value;
- wrapping для інструкцій і About links;
- content-sized window;
- narrow layout для hotkey та overlay controls;
- live HUD preview;
- явний recording state і Cancel для hotkey;
- Reduce Motion support;
- retry UI замість terminal error state.

Важливим був сам процес. Фрази «still wide» і «UI is cut» містили більше корисної інформації, ніж попередня впевненість у конкретному `frame(width:)`. Responsive UI перевіряється поведінкою на межах, а не красою одного screenshot.

## Analytics для програми про мікрофон

Для microphone utility telemetry легко перетворити на проблему довіри. Тому вимога була не «зібрати все, а потім вирішити», а одразу визначити заборонені дані.

MicStatusAI не надсилає:

- audio;
- назви мікрофонів;
- hotkey values;
- raw errors;
- дані про використання інших програм;
- device fingerprint.

Aptabase отримує лише bounded product events: launch, monitoring toggle, джерело mute action, input-level bucket, зміни HUD settings, preview, hotkey change і дедупліковану категорію помилки. У Settings є Privacy toggle, а без configured key analytics взагалі не стартує.

Client key додається під час release build через GitHub Actions secret. Але тут важлива чесна межа: key, потрібний клієнтській програмі, зрештою потрапляє в binary і може бути витягнутий. Secret захищає CI storage та logs, а не перетворює client credential на server secret.

Після UI та analytics змін вийшли [v0.0.7](https://github.com/Disconnecter/MicStatusAI/releases/tag/v0.0.7) і [v0.0.8](https://github.com/Disconnecter/MicStatusAI/releases/tag/v0.0.8). Кожен release пройшов protected PR, required build, публікацію ZIP/checksum і автоматичне оновлення Homebrew.

## Fallback, який заблокував 34 повідомлення

Поки MicStatusAI рухався до v0.0.8, інший production-сервіс перестав публікувати Telegram-дописи в X.

На перший погляд усе було добре:

- systemd service працював;
- failed posts зберігалися на disk;
- background worker повторював спробу кожні 20 секунд;
- після official API існував Twikit fallback;
- pending posts переживали restart.

Насправді official API повертав `402 Payment Required`: закінчилися credits. Fallback брав перший елемент черги, але відхиляв його через власне обмеження довжини. Повідомлення мало 279 символів, а безпечний ліміт fallback виявився 250.

Черга зберігала strict ordering. Один невдалий head item повторювався безкінечно, тому наступні 33 повідомлення навіть не пробували публікуватися.

```text
official API -> 402
       |
       v
Twikit fallback -> head item too long
       |
       v
retry same item every 20 s
       |
       v
34 queued posts, zero progress
```

Fix обрізав текст спеціально для fallback до 250 символів і зберігав Telegram link. Перед restart я перевірив перший реальний queued item: `279 -> 250`. Після deployment перші п'ять повідомлень успішно вийшли, черга почала зменшуватися, а fallback errors зникли.

Головний урок: fallback має проходити ті самі end-to-end перевірки, що й primary path. Особливо якщо вони мають різні rate limits, правила довжини та формати помилок.

## Що об'єднало ці сесії

За тиждень я кілька разів зустрів одну й ту саму помилку мислення — підміну гарантії зручною назвою:

| Назва | Реальна гарантія |
| --- | --- |
| `volume = 0` | gain zero, не hardware mute |
| CoreAudio mute | driver-controlled native mute, не глобальний lock |
| polling fix | синхронізація state, але без знання intent |
| dynamic window | адаптивність лише для перевірених constraints |
| GitHub secret | захист CI, не client key усередині binary |
| active service | живий process, не working user journey |
| fallback exists | запасний code path існує, але може не робити progress |
| queue persisted | дані не втрачені, але head-of-line blocking лишається |

## Короткі висновки

1. Починайте release automation рано, але не плутайте repeatability із correctness.
2. У stateful utility явно визначайте, хто володіє desired state.
3. Якщо platform API не передає intent, не вигадуйте його з polling result.
4. `Revert` — нормальний інструмент product discovery.
5. Strict concurrency має формувати ownership, а не додавати unsafe annotations.
6. Responsive UI потребує перевірки вузьких і довгих localized layouts.
7. Privacy analytics починається зі списку даних, які ніколи не збираються.
8. Client key не стає секретом лише тому, що прийшов із GitHub Secret.
9. Для queue важливий не retry count, а гарантований progress.
10. Fallback без production-like test — лише оптимістична гілка коду.

За цей тиждень найбільше змінився не номер версії. Змінився список речей, яким я більше не вірю без перевірки.
