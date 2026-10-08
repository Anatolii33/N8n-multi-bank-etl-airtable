# Multi-Bank Transactions ETL to Airtable (n8n)

🇬🇧 [English](README.md)

n8n-воркфлоу, який забирає транзакції з **двох банків із різними форматами даних**, приводить їх до єдиної схеми, доповнює даними про країни та завантажує в Airtable через **idempotent upsert** – тож повторний запуск ніколи не створює дублікатів.

## Яку проблему вирішує

Кожен банк віддає транзакції у своєму форматі: різні назви полів (`amount` проти `transaction_amount`), різні формати дат, країна — повною назвою в одному банку і ISO-кодом в іншому. Перш ніж аналізувати дані, їх треба звести в одну чисту таблицю.

Воркфлоу автоматизує це: **extract → transform → load**, батчами та в межах лімітів Airtable API.

## Як це працює

```
Manual Trigger
   ├─► Fetch Bank 1 ──┐
   ├─► Fetch Bank 2 ──┼─► (доповнення списком країн) ─► Нормалізація схеми ─► Combine Banks
   └─► Country list ──┘
                                                             │
        Унікальний ключ ─► Limit ─► Loop по 10 ─► Format ─► Build records array
                                          ▲                            │
                                          └── Upsert to Airtable ◄── Wait 1s ◄┘
```

| Крок | Ноди | Що відбувається |
|---|---|---|
| 1. Extract | `Fetch Bank 1 Transactions`, `Fetch Bank 2 Transactions` | Два HTTP-запити до мок-API [Mockaroo](https://www.mockaroo.com) (по 1000 рядків), що імітують дані двох банків |
| 2. Довідник | `Country Reference List`, `Split Countries` | Статичний список країн (назва, телефонний код, ISO-код) розбивається на окремі елементи |
| 3. Збагачення | `Add Country Code (Bank 1)`, `Add Country Name (Bank 2)` | Банк 1 має **назву** країни → join за назвою дає ISO-код. Банк 2 має ISO-**код** → join за кодом дає назву |
| 4. Нормалізація | `Normalize Bank 1 Schema`, `Normalize Bank 2 Schema`, `Format Date (Bank 1)` | Обидва потоки приводяться до однієї схеми; дати — у формат `yyyy-MM-dd` |
| 5. Об'єднання | `Combine Banks` | Потоки зливаються в один список |
| 6. Ключ дедуплікації | `Generate Unique Key` | Формується рядок `uuid` з id, країни, валюти, номера рахунку та мерчанта |
| 7. Load | `Limit Records` → `Loop in Batches of 10` → `Wrap in Fields` → `Build Records Array` → `Wait (Rate Limit)` → `Upsert to Airtable` | Записи надсилаються в Airtable пачками по 10 (максимум API) з паузою 1 секунда між запитами |

### Чому upsert?

Запит до Airtable використовує `performUpsert` з `fieldsToMergeOn: ["uuid"]`. Якщо запис із таким `uuid` уже існує — він **оновлюється**, інакше **створюється**. Воркфлоу можна запускати скільки завгодно разів без дублювання даних.

### Єдина схема

| Поле | Банк 1 | Банк 2 |
|---|---|---|
| `id` | `transaction_id` | `transaction_id` |
| `date` | `transaction_date` | `transaction_date` |
| `transaction_amount` | `transaction_amount` | `amount` |
| `transaction_type` | `transaction_type` | `transaction_type` |
| `account_number` | `account_number` | `account_number` |
| `merchant` | `merchant_name` | `merchant_name` |
| `description` | `transaction_description` | `transaction_description` |
| `transaction_category` | `transaction_category` | `transaction_category` |
| `card_type` | `card_type` | `card_type` |
| `location` | `location` | `location` |
| `currency` | `currency` | `transaction_currency` |
| `country_name` | `country` | визначається за ISO-кодом |
| `code` | визначається за назвою країни | `country_code` |
| `uuid` | генерується | генерується |

## Технології

- [n8n](https://n8n.io) – Manual Trigger, HTTP Request, Edit Fields (Set), Merge, Split Out, Date & Time, Limit, Loop Over Items, Aggregate, Wait
- [Airtable](https://airtable.com) REST API (upsert)
- [Mockaroo](https://www.mockaroo.com) – тестові дані банків

## Налаштування

### Що потрібно
- Інстанс n8n (cloud або self-hosted)
- Безкоштовний акаунт [Mockaroo](https://www.mockaroo.com) (для генерації двох тестових потоків)
- Акаунт Airtable

### Кроки

1. **Створіть дві схеми в Mockaroo**, що повертають JSON-масиви:
   - *Банк 1*: `transaction_id`, `transaction_date`, `transaction_amount`, `transaction_type`, `account_number`, `merchant_name`, `transaction_description`, `transaction_category`, `card_type`, `location`, `country` (повна назва країни), `currency`
   - *Банк 2*: `transaction_id`, `transaction_date`, `amount`, `transaction_type`, `account_number`, `merchant_name`, `transaction_description`, `transaction_category`, `card_type`, `location`, `transaction_currency`, `country_code` (ISO, 2 літери)
2. **Створіть таблицю в Airtable** з полями: `uuid`, `id`, `date`, `transaction_amount`, `transaction_type`, `account_number`, `merchant`, `description`, `transaction_category`, `card_type`, `location`, `currency`, `country_name`, `code`. Поле `uuid` зробіть першим (primary, текстове).
3. **Створіть Airtable Personal Access Token** зі scope `data.records:read` та `data.records:write` для цієї бази.
4. **Імпортуйте** `n8n-multi-bank-etl-airtable.json` в n8n (Workflows → Import from file).
5. **Створіть credential** в n8n: *Airtable Personal Access Token* і виберіть його в ноді `Upsert to Airtable`.
6. **Замініть плейсхолдери:**
   - `YOUR_BANK_1_SCHEMA_ID` / `YOUR_BANK_2_SCHEMA_ID` – у двох URL Mockaroo
   - `YOUR_MOCKAROO_KEY` – ваш ключ Mockaroo (query-параметр `key`)
   - `YOUR_AIRTABLE_BASE_ID` / `YOUR_AIRTABLE_TABLE_ID` – в URL ноди `Upsert to Airtable`
7. **Запустіть** воркфлоу через Manual Trigger.

> 🔐 Ніколи не вставляйте API-ключі напряму в поля нод чи заголовки – n8n не прибирає їх при експорті воркфлоу. Завжди використовуйте credentials.

## Нотатки та обмеження

- `Limit Records` встановлено на **100** записів для швидких тестів. Для повного завантаження збільште або приберіть його (2000 записів ≈ 200 запитів ≈ 3–4 хвилини з паузою 1 с).
- Airtable приймає максимум **10 записів на запит** і ~5 запитів/секунду на базу — тому батчі по 10 і пауза.
- `uuid` складається з кількох полів. Якщо банк може прислати дві однакові транзакції (той самий id, рахунок, мерчант, валюта, країна), вони перезапишуть одна одну — за можливості використовуйте справжній унікальний ID транзакції від банку.
- Дані тестові; список країн статичний (вбудований у ноду `Country Reference List`).

## Можливі покращення

- Замінити Manual Trigger на **Schedule Trigger** (щоденна синхронізація)
- Підключити реальні API банків / CSV замість Mockaroo
- Обробка помилок: retry при `429` від Airtable, сповіщення в Slack/Telegram
- Крок валідації відсутніх або некоректних полів перед завантаженням
- Конвертація валют в одну базову

## Автор

Створено [@Anatolii33](https://github.com/Anatolii33). Доступний для проєктів з автоматизації n8n та інтеграції даних.
