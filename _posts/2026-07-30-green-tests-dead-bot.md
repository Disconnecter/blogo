---
title: "Зелені тести, мертвий бот: як я переносив Garmin-коуча на MCP"
date: 2026-07-30 10:30:00 +0300
excerpt: "Історія про успішний CI, зламаний Telegram-бот, три runtime-помилки й головний урок: deployment success — це ще не working system."
categories:
  - engineering
tags:
  - n8n
  - mcp
  - telegram
  - garmin
  - devops
  - ai
---

За кілька днів я перетворив набір n8n-workflow для Garmin на персонального Telegram-коуча з історією розмови та read-only MCP-сервером. Локально все компілювалося, одинадцять тестів проходили, GitHub Actions був зелений, systemd-сервіси були active.

А бот не працював.

Потім він запрацював, але не надіслав огляд пробіжки. Після наступного виправлення він відповідав на кожне питання однаковим текстом:

> Не вдалося сформувати надійну відповідь. Спробуй переформулювати питання або повторити пізніше.

Це хороший приклад того, як кілька правильних локальних рішень можуть скластися у непрацюючу production-систему. І ще кращий приклад того, чого саме не перевіряють зелені тести.

## Що я будував

Система вже вміла:

- синхронізувати сон, денні метрики й активності з Garmin;
- зберігати їх у SQLite;
- генерувати ранкові, післятренувальні, вечірні та тижневі рекомендації;
- надсилати звіти в Telegram через n8n.

Я хотів додати повноцінний діалог із тренером:

- `/coach` для запитань із контекстом Garmin;
- `/newchat` для очищення історії;
- коротку історію розмови;
- кнопки Telegram;
- однаковий coaching prompt для всіх workflow;
- чітке розділення читання й запису даних.

Фінальна схема виглядала так:

```text
Garmin Connect
      |
      v
sync script ---> n8n ingest workflow ---> SQLite
                                         |
                                         | read-only
                                         v
                                  Garmin MCP server
                                         |
                                         v
Telegram webhook ---> n8n workflows ---> LLM Responses API
                         |
                         +-------------> SQLite writes
```

MCP-сервер слухає тільки loopback-інтерфейс. Він не змінює дані й не замінює ingest pipeline. Усі записи — імпорт Garmin, рекомендації, налаштування та історія чату — залишилися через контрольований SQLite API. MCP отримав лише workflow-specific read tools: ранковий контекст, контекст активності, статус, тижневий звіт тощо.

Це рішення мені досі подобається. Проблеми були не в самій архітектурі, а на стиках між компонентами.

## Спочатку — Git

Робота почалася з уже розбіжних гілок. Локально був великий commit `Coach`, а remote `main` містив ще три зміни: спільні prompts, нову логіку попереднього плану й розширений аналіз активності.

Звичайний push отримав `fetch first`, а звичайний pull — вимогу явно вибрати стратегію reconciliation. Додатково з'ясувалося, що перший commit не включив untracked-файли: MCP-сервер, systemd service, тести й конфігурацію type checker.

Правильна послідовність виявилася такою:

1. Перевірити untracked-файли.
2. Додати їх до локального commit.
3. Fetch remote branch.
4. Rebase локальної роботи поверх remote `main`.
5. Розв'язати конфлікти в generator, backfill script і README.
6. Не мерджити generated JSON вручну, а згенерувати його повторно.
7. Запустити compile, tests і `git diff --check`.

Найважливіше тут — generated artifacts. Конфлікт у JSON майже завжди треба вирішувати в source generator, а потім регенерувати результат. Інакше source та production workflow непомітно розходяться.

Після rebase зміна охоплювала 26 файлів: приблизно 3300 доданих і 700 видалених рядків. Це вже не «маленький feature», навіть якщо UI-зміна виглядає як одна команда `/coach`.

## Інцидент №1: Telegram-бот неактивний

CI пройшов. Deploy job пройшов. Чотири systemd-сервіси повертали `active`. Telegram зберігав правильний webhook URL.

Але `getWebhookInfo` показував pending updates і помилку `400 Bad Request`. Прямий POST у webhook повернув справжню причину:

```text
Unrecognized node type: n8n-nodes-langchain.mcpClient
```

Тут було одразу дві проблеми.

### Стара версія n8n

Production n8n не містив standalone MCP Client action node. MCP Client Tool з'явився раніше, але звичайний MCP Client, який можна поставити посеред workflow без AI Agent, потребував новішої версії.

Я зафіксував останню сумісну версію n8n 1.x замість автоматичного переходу на major 2.x. Перед оновленням deployment почав створювати backup n8n database і перевіряти наявність конкретного MCP Client node file.

### Неправильний namespace

Навіть після оновлення webhook відповідав `500`. У generated workflow був тип:

```text
n8n-nodes-langchain.mcpClient
```

Правильний тип:

```text
@n8n/n8n-nodes-langchain.mcpClient
```

Один пропущений scope ламав створення всього workflow. Через це не працював навіть `/start`, хоча цей route взагалі не використовував MCP: n8n спочатку конструює workflow і валідовує всі node types.

Після виправлення Telegram webhook повернув `200`, а pending queue впала з трьох повідомлень до нуля.

### Чому тести це пропустили

Тест перевіряв, що workflow містить node type, який ми самі вважали правильним. Тобто тест чудово підтверджував нашу помилкову гіпотезу.

Висновок: перевірка generated JSON корисна, але вона не замінює compatibility test проти реального runtime.

## Інцидент №2: зник огляд пробіжки

Наступна скарга: учора була пробіжка, але післятренувального огляду немає.

Workflow активності запускався webhook-викликом із Garmin sync script. Якщо виклик падав, script обіцяв повторити його під час наступної синхронізації. Але default sync обробляв лише поточну дату.

Після опівночі «повторити на наступному sync» фактично означало «ніколи не повторити».

Я розширив вікно синхронізації до двох днів. Дедуплікація за `activity_id` у workflow не дозволяє створити подвійний огляд, зате активність, завантажена із запізненням, більше не губиться.

Потім deployment упав на примусовому catch-up sync. Journal показав іншу проблему:

```text
Set GARMIN_EMAIL and GARMIN_PASSWORD env vars
```

Цікаво, що valid saved Garmin session на сервері існувала. Функція `login()` уміла її відновлювати, але `main()` завершував процес ще до виклику `login()`, якщо email або password були відсутні.

Guard clause суперечив реальній auth logic.

Після зміни script працює так:

1. Якщо saved session існує — спочатку пробує її.
2. Credentials потрібні лише для першого входу або після expiry session.
3. Якщо session протермінована, помилка прямо пояснює, що треба відновити credentials.
4. Sync перевіряє сьогодні й учора.

Це відновило ingestion без зберігання пароля в коді. Але rotated credentials усе одно треба додати в server environment до завершення строку дії session.

## Інцидент №3: тренер завжди відповідає fallback-текстом

Після відновлення бота звичайні команди працювали, але coach chat завжди повертав безпечний fallback.

LLM node робив три retry, ловив будь-яку помилку й повертав порожній content. Наступний node бачив порожню відповідь і замінював її дружнім повідомленням для користувача. Workflow завершувався успішно, Telegram отримував `200`, а технічна причина зникала.

Сам fallback був правильним UX-рішенням. Неправильним було те, що він перетворив системну помилку на успішне виконання без observable signal.

Причина ховалася в Code node:

```javascript
const response = await fetch(url, options);
```

Код виконувався в n8n sandbox, де не можна покладатися на глобальний `fetch`. Для HTTP-викликів n8n надає власний helper.

Виправлення:

```javascript
const response = await this.helpers.httpRequest({
  method: "POST",
  url,
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload),
  encoding: "utf8",
  returnFullResponse: true,
});
```

Одночасно я передав `max_output_tokens`, зберіг parsing звичайної та streaming Responses API відповіді й додав regression assertion: generated coach node має використовувати `helpers.httpRequest`, а не `fetch`.

## Що насправді перевіряли зелені тести

На момент першого deployment система мала одинадцять passing tests. Вони перевіряли важливі речі:

- read-only SQLite connection для MCP;
- bounded query results;
- workflow-specific MCP tools;
- відсутність прямих Garmin SELECT у analysis workflows;
- валідні n8n connections;
- наявність coach flow та conversation history;
- schema migration;
- Python compilation;
- валідний generated JSON.

Але вони не перевіряли:

- чи знає production n8n конкретний node type;
- чи правильний package namespace;
- чи Code sandbox підтримує використаний network API;
- чи LLM повернув непорожній текст, а не просто HTTP 200;
- чи Telegram webhook виконує хоча б один реальний route;
- чи retry залишається можливим після зміни календарної дати;
- чи token-based auth не блокується раннім guard clause.

Тобто unit та structural tests були зеленими, а integration contracts — ні.

## Deployment success — слабкий сигнал

Перший deployment smoke test перевіряв:

- Python compile;
- unit tests;
- JSON parsing;
- запуск MCP server;
- список MCP tools;
- active systemd units;
- HTTP 200 від LLM endpoint.

Цього недостатньо.

`systemctl is-active n8n` означає лише, що процес живий. Він не означає, що n8n може завантажити конкретний workflow.

HTTP 200 від LLM означає лише, що gateway прийняв request. Він не означає, що parser дістав текст.

Зареєстрований Telegram webhook означає лише, що Telegram знає URL. Він не означає, що workflow повертає 2xx.

Після цієї історії мій мінімальний deployment checklist виглядає так:

1. Перевірити runtime version і потрібні node packages.
2. Завантажити workflow в тому самому runtime, де він працюватиме.
3. Викликати production-like webhook payload.
4. Перевірити не тільки status code, а й semantic result.
5. Переконатися, що fallback/error path створює observable signal.
6. Перевірити retry через часову межу: midnight, restart, delayed upload.
7. Не вважати `active` синонімом `healthy`.

## Окремий факап: автоматизація без меж

У цій сесії був ще один важливий failure mode — не технічний.

Coding agent кілька разів самостійно робив commit, push і запускав deployment. Так, зміни виправляли production. Але рішення про зміну production належить власнику системи, а не агенту.

Правильна межа тепер сформульована явно:

- агент може аналізувати;
- агент може редагувати локальні файли;
- агент може запускати локальні тести;
- перед commit показує diff і запропонований message;
- commit — лише після прямого підтвердження;
- push — лише після прямого підтвердження;
- deployment, workflow rerun, SSH, systemd restart і production probe — лише після окремого дозволу.

Це не бюрократія. Це частина reliability model. Найкращий технічний fix, застосований без authorization, залишається process failure.

## Що залишилося після кількох днів debugging

У фінальній системі:

- Garmin ingestion і всі writes залишилися через SQLite API;
- analysis reads перейшли на read-only MCP;
- MCP доступний тільки локально на VM;
- Telegram отримав coach chat і bounded history;
- n8n використовує правильний scoped MCP Client node;
- deployment pin-ить сумісну версію n8n;
- Garmin sync повторно перевіряє два дні;
- saved Garmin session працює без обов'язкового пароля;
- LLM calls у Code node використовують n8n HTTP helper;
- generated workflows перевіряються тестами й регенеруються із source.

А головний результат — не нова команда в Telegram.

Головний результат — список assumption, які тепер більше не є невидимими.

## Короткі висновки

1. Green CI не гарантує working integration.
2. Generated configuration треба тестувати проти реального runtime, а не лише проти власних очікувань.
3. Fallback без telemetry маскує outage.
4. Retry policy має враховувати час і зміну дати.
5. Saved session та credential fallback повинні бути частинами однієї auth state machine.
6. Process health, endpoint health і user journey — три різні перевірки.
7. Generated artifacts треба регенерувати після merge, а не лікувати вручну.
8. AI coding agent потребує таких самих deployment boundaries, як будь-який інший automation tool.

Найнебезпечніша фраза в цій історії була не error message.

Найнебезпечнішою була фраза: «Deployment successful».
