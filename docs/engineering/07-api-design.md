# 07 — API Design

**Barokah Wealth**

---

## 1. Overview

REST API built with Django REST Framework. All endpoints (except auth) require a JWT access token in the `Authorization: Bearer <token>` header.

Base URL (dev): `http://localhost:8000/api/`
Base URL (prod): TBD (Railway URL)

**Response shape rule:** every object of the same resource type (e.g. every Pocket) returns the same JSON shape, regardless of denomination. Fields that don't apply to a given denomination are still present in the response, set to `null`. This keeps frontend types consistent — no conditional field-checking per denomination, only conditional *rendering* based on `denomination`.

Request payloads (what the client sends) may omit fields that don't apply — see field tables below for what's required per case.

All primary keys are UUID4. Rationale: prevents ID enumeration (a known vulnerability class for financial data — sequential numeric IDs let someone guess adjacent records like `/transactions/43/` after seeing `/transactions/42/`). Native Django/Postgres support, no extra dependencies.

All models include `updated_at` alongside `created_at`. No `created_by`/`updated_by` — every record already has a `user_id`, and MVP has no multi-editor scenario needing separate audit fields.

---

## 2. Auth

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/auth/register/` | No | Create new user |
| POST | `/auth/login/` | No | Returns access + refresh token |
| POST | `/auth/refresh/` | No | Exchange refresh token for new access token |
| POST | `/auth/logout/` | Yes | Invalidate refresh token |

**POST `/auth/register/` request:**
```json
{
  "email": "galang@example.com",
  "password": "••••••••",
  "confirm_password": "••••••••"
}
```

**POST `/auth/login/` response:**
```json
{
  "access": "eyJ...",
  "refresh": "eyJ...",
  "user": { "id": "9f8e7d6c-5b4a-4c3d-8e2f-1a0b9c8d7e6f", "email": "galang@example.com" }
}
```

---

## 3. Pockets

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/pockets/` | Yes | List all pockets for logged-in user |
| POST | `/pockets/` | Yes | Create a new pocket |
| GET | `/pockets/{id}/` | Yes | Get single pocket detail |
| PATCH | `/pockets/{id}/` | Yes | Update pocket |
| DELETE | `/pockets/{id}/` | Yes | Delete a pocket |

### Field reference

| Field | Type | Required on create | Applies to |
|---|---|---|---|
| name | string | Yes | all |
| denomination | enum (`IDR`, `GRAM`) | Yes | all |
| initial_balance | decimal | Yes | all |
| auto_fetch_price | boolean | No — defaults `true` | GRAM only, `null` for IDR |
| manual_price_override | decimal | No | GRAM only, `null` for IDR |

### Request example — create IDR pocket
```json
{ "name": "Main Account", "denomination": "IDR", "initial_balance": 5000000 }
```

### Request example — create Gold pocket
```json
{ "name": "Gold Savings", "denomination": "GRAM", "initial_balance": 10.5, "auto_fetch_price": true }
```

### Response shape (GET /pockets/) — same shape for every pocket
```json
[
  {
    "id": "a3f1c2e4-9b2d-4a1e-8f3c-1d2e3f4a5b6c",
    "name": "Main Account",
    "denomination": "IDR",
    "initial_balance": 5000000,
    "current_balance": 6200000,
    "auto_fetch_price": null,
    "manual_price_override": null,
    "created_at": "2026-08-01T09:00:00Z",
    "updated_at": "2026-08-20T14:30:00Z"
  },
  {
    "id": "d8e2b7a1-4c3f-4e9a-b2d1-7f8a9c0e1b2d",
    "name": "Gold Savings",
    "denomination": "GRAM",
    "initial_balance": 10.5,
    "current_balance": 12.0,
    "auto_fetch_price": true,
    "manual_price_override": null,
    "created_at": "2026-08-05T09:00:00Z",
    "updated_at": "2026-08-25T11:00:00Z"
  }
]
```

---

## 4. Transactions

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/pockets/{pocket_id}/transactions/` | Yes | List transactions for a pocket |
| POST | `/pockets/{pocket_id}/transactions/` | Yes | Add a new transaction |
| PATCH | `/transactions/{id}/` | Yes | Edit a transaction |
| DELETE | `/transactions/{id}/` | Yes | Delete a transaction |

### Field reference

| Field | Type | Required on create | Applies to |
|---|---|---|---|
| direction | enum (`in`, `out`) | Yes | all |
| amount | decimal | Yes | all |
| price_at_transaction | decimal | Yes for GRAM pockets, not applicable for IDR | GRAM only |
| category | string | No | all |
| tags | array of strings | No | all |
| note | string | No | all |
| date | date | Yes | all |

> `tags` can hold recurring labels (e.g. "monthly") or the money source — bank/e-wallet name like "BCA", "Jago", "Mandiri", "OVO", "Cash". Multiple tags allowed per transaction, e.g. `["monthly", "BCA"]`.

> `price_at_transaction` is required when the parent pocket's denomination is not IDR. The frontend fetches the current price from `GET /gold-price/current/` and pre-fills this value before submit; the user can edit it before saving. Omitting it on a GRAM pocket returns a `price_required` validation error.

### Request example — IDR pocket transaction
```json
{
  "direction": "in",
  "amount": 8000000,
  "category": "Salary",
  "tags": ["monthly", "BCA"],
  "note": "August salary",
  "date": "2026-08-27"
}
```

### Request example — Gold pocket transaction
```json
{
  "direction": "in",
  "amount": 5,
  "price_at_transaction": 1950000,
  "category": "Gold Purchase",
  "note": "Bought from Galeri24 store",
  "date": "2026-08-27"
}
```

### Response shape (GET /pockets/{id}/transactions/) — same shape for every transaction
```json
[
  {
    "id": "f4a1e2c3-8b7d-4a6e-9c1f-2e3a4b5c6d7e",
    "pocket_id": "a3f1c2e4-9b2d-4a1e-8f3c-1d2e3f4a5b6c",
    "direction": "in",
    "amount": 8000000,
    "price_at_transaction": null,
    "category": "Salary",
    "tags": ["monthly", "BCA"],
    "note": "August salary",
    "date": "2026-08-27",
    "created_at": "2026-08-27T10:00:00Z",
    "updated_at": "2026-08-27T10:00:00Z"
  }
]
```

---

## 5. Gold Price

**Architecture:** a scheduled daily job fetches the price from `logam-mulia-api` once per day and writes a new `GoldPriceLog` row. This endpoint only ever reads from the database — never a live external call inside a user request.

**Two separate use cases for this price, handled differently:**

- **Dashboard / Nisab display** — needs *a* current price to show. If today's price is missing, show the most recent available price with its date clearly labeled (e.g. "Price as of Aug 26 — today's update unavailable"). If no price has ever been logged, the dashboard shows "Gold price unavailable" and Nisab comparison is paused until it resolves.
- **Transaction creation** — frontend tries to prefill `price_at_transaction` from this endpoint. If it returns null, the field is simply left empty and required — the user manually enters a price. No special fallback logic needed since it's already an interactive, required field.
- **Admin alert on missing price (deferred to next version):** an email notification when the scheduled job fails is a reasonable production safeguard, deferred for now — not required at current portfolio scale.

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/gold-price/current/` | Yes | Returns the most recent `GoldPriceLog` row |

**Response fields:**

| Field | Meaning |
|---|---|
| price_per_gram | IDR price at time of logging |
| price_date | The calendar date this price represents |
| cached_at | Timestamp when this row was written to the database |
| is_fallback | `true` if today's scheduled job hasn't run yet or failed, and this is an older cached row |

**Response example — price is current (today's job ran successfully):**
```json
{
  "source": "galeri24",
  "price_per_gram": 1950000,
  "price_date": "2026-08-27",
  "cached_at": "2026-08-27T06:00:00Z",
  "is_fallback": false
}
```

**Response example — fallback (today's job hasn't run or failed):**
```json
{
  "source": "galeri24",
  "price_per_gram": 1930000,
  "price_date": "2026-08-26",
  "cached_at": "2026-08-26T06:00:00Z",
  "is_fallback": true
}
```

**Response example — no log exists at all:**
```json
{
  "source": "galeri24",
  "price_per_gram": null,
  "price_date": null,
  "cached_at": null,
  "is_fallback": true
}
```

---

## 6. Wealth Summary

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/wealth/summary/` | Yes | Returns total wealth, Nisab, status, Haul info |

**Why this endpoint is not cached — computed fresh on every request:** the gold price is fetched once a day and stored (Section 5), but wealth totals depend on the user's live pocket balances, which can change at any moment as they log transactions. Since this calculation is just arithmetic over data already in the database (no external API call involved), it's cheap to compute fresh on every request rather than storing a snapshot that could go stale the moment a new transaction is logged.

> This is a deliberate MVP choice, not a limitation. If usage scale ever made this expensive to compute per request, the fix would be to store a snapshot updated on each transaction write — similar to how `current_balance` is denormalized on Pocket. Not needed at current scale.

**Response example — below Nisab:**
```json
{
  "total_wealth_idr": 125000000,
  "nisab_threshold_idr": 165750000,
  "is_above_nisab": false,
  "haul": {
    "status": "not_started",
    "start_date": null,
    "days_remaining": null
  },
  "zakat_due_idr": null
}
```

**Response example — above Nisab, Haul in progress:**
```json
{
  "total_wealth_idr": 200000000,
  "nisab_threshold_idr": 165750000,
  "is_above_nisab": true,
  "haul": {
    "status": "in_progress",
    "start_date": "2026-01-15",
    "days_remaining": 180
  },
  "zakat_due_idr": null
}
```

**Response example — Haul complete, Zakat due:**
```json
{
  "total_wealth_idr": 210000000,
  "nisab_threshold_idr": 165750000,
  "is_above_nisab": true,
  "haul": {
    "status": "completed",
    "start_date": "2025-08-27",
    "days_remaining": 0
  },
  "zakat_due_idr": 5250000
}
```

---

## 7. Zakat / Haul Actions

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/haul/history/` | Yes | List past Haul periods for user |
| POST | `/haul/{id}/mark-paid/` | Yes | Mark Zakat as paid, resets Haul cycle |

**Reset rule:** the new Haul cycle starts from `completed_at` (the original due date), not `paid_at` (when the user actually pays). This matches standard Zakat fiqh — Haul tracks the time wealth stayed above Nisab, not the user's payment timing. Paying a few days or weeks late does not push back the next cycle's start date.

**Response example (GET /haul/history/):**
```json
[
  {
    "id": "c1d2e3f4-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
    "start_date": "2025-01-15",
    "status": "completed",
    "completed_at": "2026-01-04",
    "zakat_amount": 4800000,
    "paid_at": "2026-01-11"
  }
]
```

**Response example (POST /haul/{id}/mark-paid/):**

User pays 7 days after the due date. The next Haul cycle starts from `completed_at` (Jan 4), not `paid_at` (Jan 11).

```json
{
  "id": "c1d2e3f4-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
  "status": "reset",
  "paid_at": "2026-01-11",
  "next_haul_start_date": "2026-01-04"
}
```

---

## 8. Error Format (standard across all endpoints)

```json
{
  "error": {
    "code": "invalid_denomination",
    "message": "Pocket denomination must be a supported type."
  }
}
```

**Common error codes:**

| Code | When |
|---|---|
| `price_required` | `price_at_transaction` missing on a non-IDR transaction |
| `invalid_denomination` | Unsupported denomination value on pocket creation |
| `pocket_not_found` | Pocket ID doesn't exist or doesn't belong to the user |
| `unauthorized` | Missing or expired JWT |

---

## 9. Notes

- All monetary values are integers in IDR (no decimals — Rupiah has no subunits in practice)
- Gold amounts use decimal with up to 2 decimal places (grams)
- All list endpoints support pagination (`?page=1&page_size=20`)
- Every resource type returns a consistent response shape regardless of denomination — inapplicable fields are `null`, never omitted
- Gold price is never fetched live inside a user request — always read from the daily-updated `GoldPriceLog` table

---

*Document status: Approved*
*Last updated: 2026-08-30*