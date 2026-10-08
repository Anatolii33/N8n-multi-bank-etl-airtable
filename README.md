# Multi-Bank Transactions ETL to Airtable (n8n)

🇺🇦 [Українською](README.uk.md)

An n8n workflow that pulls transaction data from **two banks with different data formats**, normalizes it into one unified schema, enriches it with country data, and loads it into Airtable using **idempotent upserts** – so re-running the workflow never creates duplicates.

![Workflow overview](<img width="1287" height="370" alt="image" src="https://github.com/user-attachments/assets/45a643c1-4594-497b-b2a1-2b9faca82aa1" />
)

## The problem

Every bank exports transactions in its own shape: different field names (`amount` vs `transaction_amount`), different date formats, country as a full name in one feed and as an ISO code in another. Before the data can be analysed, it has to be merged into one clean table.

This workflow automates that: **extract → transform → load**, in batches, within Airtable's API limits.

## How it works

```
Manual Trigger
   ├─► Fetch Bank 1 ──┐
   ├─► Fetch Bank 2 ──┼─► (enrich with country list) ─► Normalize schema ─► Combine Banks
   └─► Country list ──┘
                                                             │
        Generate unique key ─► Limit ─► Loop in batches of 10 ─► Format ─► Build records array
                                              ▲                                    │
                                              └── Upsert to Airtable ◄── Wait 1s ◄┘
```

| Step | Nodes | What happens |
|---|---|---|
| 1. Extract | `Fetch Bank 1 Transactions`, `Fetch Bank 2 Transactions` | Two HTTP requests to [Mockaroo](https://www.mockaroo.com) mock APIs (1000 rows each) simulating two bank feeds |
| 2. Reference data | `Country Reference List`, `Split Countries` | A static list of countries (name, dial code, ISO code) is split into one item per country |
| 3. Enrich | `Add Country Code (Bank 1)`, `Add Country Name (Bank 2)` | Bank 1 has a country **name** → join by name to get the ISO code. Bank 2 has an ISO **code** → join by code to get the country name |
| 4. Normalize | `Normalize Bank 1 Schema`, `Normalize Bank 2 Schema`, `Format Date (Bank 1)` | Both feeds are mapped to one schema; dates are converted to `yyyy-MM-dd` |
| 5. Combine | `Combine Banks` | Both streams are merged into a single list |
| 6. Deduplication key | `Generate Unique Key` | Builds a `uuid` string from id, country, currency, account number and merchant |
| 7. Load | `Limit Records` → `Loop in Batches of 10` → `Wrap in Fields` → `Build Records Array` → `Wait (Rate Limit)` → `Upsert to Airtable` | Records are sent to Airtable in batches of 10 (the API maximum) with a 1-second pause between requests |

### Why upsert?

The Airtable request uses `performUpsert` with `fieldsToMergeOn: ["uuid"]`. If a record with the same `uuid` already exists it is **updated**, otherwise it is **created**. You can run the workflow as many times as you like without duplicating data.

### Unified schema

| Field | Bank 1 source | Bank 2 source |
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
| `country_name` | `country` | looked up by ISO code |
| `code` | looked up by country name | `country_code` |
| `uuid` | generated | generated |

## Tech stack

- [n8n](https://n8n.io) – Manual Trigger, HTTP Request, Edit Fields (Set), Merge, Split Out, Date & Time, Limit, Loop Over Items, Aggregate, Wait
- [Airtable](https://airtable.com) REST API (upsert)
- [Mockaroo](https://www.mockaroo.com) – mock bank data

## Setup

### Requirements
- An n8n instance (cloud or self-hosted)
- A free [Mockaroo](https://www.mockaroo.com) account (to generate the two test bank feeds)
- An Airtable account

### Steps

1. **Create two Mockaroo schemas** that return JSON arrays:
   - *Bank 1* fields: `transaction_id`, `transaction_date`, `transaction_amount`, `transaction_type`, `account_number`, `merchant_name`, `transaction_description`, `transaction_category`, `card_type`, `location`, `country` (full country name), `currency`
   - *Bank 2* fields: `transaction_id`, `transaction_date`, `amount`, `transaction_type`, `account_number`, `merchant_name`, `transaction_description`, `transaction_category`, `card_type`, `location`, `transaction_currency`, `country_code` (ISO 2-letter)
2. **Create an Airtable table** with these fields: `uuid`, `id`, `date`, `transaction_amount`, `transaction_type`, `account_number`, `merchant`, `description`, `transaction_category`, `card_type`, `location`, `currency`, `country_name`, `code`.
   Set `uuid` as the first/primary text field.
3. **Create an Airtable Personal Access Token** with scopes `data.records:read` and `data.records:write` for that base.
4. **Import** `n8n-multi-bank-etl-airtable.json` into n8n (Workflows → Import from file).
5. **Create the credential** in n8n: *Airtable Personal Access Token* and select it in the `Upsert to Airtable` node.
6. **Replace the placeholders:**
   - `YOUR_BANK_1_SCHEMA_ID` / `YOUR_BANK_2_SCHEMA_ID` – in the two Mockaroo URLs
   - `YOUR_MOCKAROO_KEY` – your Mockaroo API key (query parameter `key`)
   - `YOUR_AIRTABLE_BASE_ID` / `YOUR_AIRTABLE_TABLE_ID` – in the `Upsert to Airtable` URL (find them in the Airtable API docs of your base)
7. **Run** the workflow with the manual trigger.

> 🔐 Never paste API keys directly into node fields or headers – n8n does not remove them when you export a workflow. Always use credentials.

## Notes & limitations

- `Limit Records` is set to **100** items to keep test runs short. Increase or remove it for full loads (2000 records ≈ 200 requests ≈ 3–4 minutes with the 1 s wait).
- Airtable accepts a maximum of **10 records per request** and ~5 requests/second per base, which is why the workflow batches by 10 and waits.
- The `uuid` is built from several fields. If a bank can send two identical transactions (same id, account, merchant, currency, country), they would overwrite each other – use the bank's real unique transaction ID if available.
- Data is mock data; the country list is static (embedded in the `Country Reference List` node).

## Possible improvements

- Replace the manual trigger with a **Schedule Trigger** (daily sync)
- Connect real bank APIs / CSV exports instead of Mockaroo
- Error handling: retry on Airtable `429`, send an alert to Slack/Telegram on failure
- Validation step for missing or malformed fields before loading
- Currency conversion to a single base currency

## Author

Built by [@Anatolii33](https://github.com/Anatolii33). Available for n8n automation and data integration projects.
