# 06 — Database Schema

**Barokah Wealth**

---

## 1. Overview

This document defines the core data models for the MVP: Users, Pockets, Transactions, Gold Price Logs, and Zakat/Haul tracking.

The schema is designed to be **denomination-agnostic** — Cash (IDR) and Gold (grams) share the same Pocket and Transaction structure. This allows future currencies (USD, SGD, MYR) to be added without schema changes, only a new denomination type.

---

## 2. Entity Relationship (described)

```
User
 └── Pocket (denomination: IDR | GRAM | future: USD, SGD, MYR...)
      └── Transaction (belongs to any pocket, any denomination)

User
 └── HaulPeriod
      └── linked to wealth snapshots over time

GoldPriceLog (independent, shared across all users)
```

---

## 3. Models

### User
| Field | Type | Notes |
|---|---|---|
| id | UUID / PK | |
| email | string, unique | login |
| password | hashed | Django default |
| created_at | datetime | |

---

### Pocket

Unified model for all wealth types. No separate Cash/Gold model — differentiated by `denomination`.

| Field | Type | Notes |
|---|---|---|
| id | UUID / PK | |
| user_id | FK → User | |
| name | string | user-defined, e.g. "Main Account", "Gold Savings" |
| denomination | enum | `IDR`, `GRAM` (future: `USD`, `SGD`, `MYR`, etc.) |
| initial_balance | decimal | in native denomination |
| current_balance | decimal | in native denomination — recalculated from transactions |
| auto_fetch_price | boolean | applies to non-IDR denominations — default true |
| manual_price_override | decimal, nullable | used if `auto_fetch_price` is false |
| created_at | datetime | |

> IDR pockets always have an implicit conversion rate of 1 — no price fetching needed. Non-IDR pockets (Gold now, foreign currency later) use `auto_fetch_price` or `manual_price_override` to determine current IDR value.

---

### Transaction

Applies to **any pocket type** — a gold purchase, a salary deposit, or a business expense are all transactions.

| Field | Type | Notes |
|---|---|---|
| id | UUID / PK | |
| pocket_id | FK → Pocket | any denomination |
| direction | enum | `in` or `out` (replaces income/expense — works for both cash and gold buy/sell) |
| amount | decimal | in pocket's native denomination |
| price_at_transaction | decimal, required | IDR value per unit — pre-filled with today's price from GoldPriceLog (or live fetch if cache expired), user must confirm or edit before saving. If fetch fails, falls back to last known GoldPriceLog entry. If no log exists at all, field is blank but still required — user must manually enter. |
| category | string | user-defined, defaults: Salary, Food, Transport, Other |
| tags | array of strings | optional, user-defined |
| note | text, nullable | optional |
| date | date | |
| created_at | datetime | |

> **Gain/loss tracking (deferred to v2):** price_at_transaction is captured now so historical data exists when the feature is built later. No calculation or UI is built on this field in the MVP.

---

### GoldPriceLog
| Field | Type | Notes |
|---|---|---|
| id | UUID / PK | |
| source | string | fixed: `galeri24` |
| price_per_gram | decimal | IDR |
| fetched_at | datetime | |

> Shared across all users — one row per fetch, cached for 24 hours before re-fetching. Also used to backfill `price_at_transaction` via the historical price endpoint when a user logs a past-dated transaction.

---

### HaulPeriod
| Field | Type | Notes |
|---|---|---|
| id | UUID / PK | |
| user_id | FK → User | |
| start_date | date | date wealth first exceeded Nisab |
| status | enum | `in_progress`, `completed`, `reset` |
| completed_at | date, nullable | set when Haul reaches 354 days |
| zakat_amount | decimal, nullable | calculated when completed |
| paid_at | date, nullable | set when user marks Zakat as paid |
| created_at | datetime | |

---

## 4. Computed Values (not stored, calculated on request)

| Value | Formula |
|---|---|
| Total net wealth | sum of all pocket balances converted to IDR (IDR pockets: balance × 1; non-IDR pockets: balance × current price) |
| Nisab threshold | current gold price per gram × 85 |
| Above/Below Nisab | total net wealth ≥ Nisab threshold |
| Haul days remaining | 354 − (today − HaulPeriod.start_date) |
| Zakat amount | total net wealth × 0.025 |
| Gain/loss (deferred to v2) | (current price − price_at_transaction) × amount, per transaction |

---

## 5. Relationships Summary

- One **User** has many **Pockets** (any denomination)
- One **Pocket** has many **Transactions**
- One **User** has many **HaulPeriod** records (history of past cycles)
- **GoldPriceLog** is global, not tied to a user

---

## 6. Notes for Implementation

- `current_balance` on a pocket is derived from `initial_balance` + sum of transactions (`in` − `out`), denormalized and stored for performance, recalculated on each transaction write
- When a user logs a transaction on a non-IDR pocket, backend pre-fills price_at_transaction with today's current price (from cache or live fetch). If fetch fails, falls back to the most recent GoldPriceLog entry. If no log exists, field is blank but the user must still manually enter a value before saving — it cannot be left empty.
- If historical price fetch fails, `price_at_transaction` remains editable and empty — this is the only case it stays null
- Nisab and wealth checks should run as a backend service function, not duplicated across multiple views
- Adding a new denomination (e.g. USD) in the future requires no schema change — only a new enum value and a corresponding price-fetching service

---

*Document status: Approved*
*Last updated: 2026-08-27*