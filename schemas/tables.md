# Data table schemas

All tables live in the n8n personal project `{{N8N_PROJECT_ID}}` on {{N8N_HOST}}. Row `id`, `createdAt`, `updatedAt` are auto-managed by n8n.

## trade_ledger ({{TABLE_TRADE_LEDGER_ID}})

Every order event parsed from IndMoney transaction emails. Append-only; deduped by `gmail_message_id`.

| column | type | notes |
|---|---|---|
| event_ts | date | email receive time (UTC ISO) |
| event_type | string | BUY_FILL / SELL_FILL / ORDER_CANCELLED / DIVIDEND / OTHER |
| ticker | string | symbol mapped from company name via static dictionary; "?" if unknown (row still logged) |
| company | string | as printed in the email "Ticker:" field (which actually holds the company name) |
| shares | number | null for cancels/dividends |
| price | number | null for cancels/dividends |
| amount | number | order USD amount; dividend USD for DIVIDEND |
| order_type | string | Market / stop / Trigger / limit; null for dividends |
| account | string | e.g. XX106 |
| gmail_message_id | string | dedup key |
| raw_subject | string | original email subject |
| source | string | gmail:transactions@transactions.indmoney.com or backfill |

## positions_state ({{TABLE_POSITIONS_STATE_ID}})

One row per position. Source of truth for the ratchet + alerts.

| column | type | notes |
|---|---|---|
| symbol | string | primary lookup key |
| company | string | |
| layer | string | Core / Amplifier / Tactical |
| units | number | fractional shares held |
| hard_stop | number | the LIVE broker stop order on IndMoney |
| watch_sl | number | formula/watch stop level |
| tp_order | number | take-profit level, nullable |
| trail_pct | number | ratchet: formula_stop = high_20d * (1 - trail_pct/100), never lowered |
| manual_override | boolean | true = never auto-recommend from formula (e.g. TSM stop held above formula, OUST volatility overlay) |
| high_20d | number | 20-day high; maintained as intraday high-water mark by Market Watch |
| last_price | number | last TwelveData poll |
| pending_action | string | e.g. "raise stop to 179.28"; cleared by the `done` Telegram command |
| pending_since | date | |
| status | string | OPEN / CLOSED |
| last_updated | date | |
| last_stop_alert | date | once-per-day dedup for near-stop alerts |
| last_tp_alert | date | once-per-day dedup for TP alerts |
| next_earnings | date | maintained by the Claude brain via MCP - Brief Intake (TwelveData /earnings is paywalled) |

## daily_brief ({{TABLE_DAILY_BRIEF_ID}})

One row per day, written by the Claude judgment layer via MCP - Brief Intake. Upserted by date.

| column | type | notes |
|---|---|---|
| date | string | YYYY-MM-DD (IST day) |
| brief_text | string | condensed brief, max 1500 chars; rendered in the Daily Digest if <24h old |
| author | string | e.g. claude |
| created_at | date | |
