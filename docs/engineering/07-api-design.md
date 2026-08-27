# 07 — API Design

**Barokah Wealth**

---

## 1. Overview

REST API built with Django REST Framework. All endpoints (except auth) require a JWT access token in the `Authorization: Bearer <token>` header.

Base URL (dev): `http://localhost:8000/api/`
Base URL (prod): TBD (Railway URL)

---

## 2. Auth

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/register/` | Create new user |
| POST | `/auth/login/` | Returns access + refresh token |
| POST | `/auth/refresh/` | Exchange refresh token for new access token |
| POST | `/auth/logout/` | Invalidate refresh token |

---

## 3. Pockets

| Method | Endpoint | Description |
|---|---|---|
| GET | `/pockets/` | List all pockets for logged-in user |
| POST | `/pockets/` | Create a new pocket |
| GET | `/pockets/{id}/` | Get single pocket detail |
| PATCH | `/pockets/{id}/` | Update pocket (name, balance, price settings) |
| DELETE | `/pockets/{id}/` | Delete a pocket |

**Create pocket payload example (IDR):**
```json
{
  "name": "Main Account",
  "denomination": "IDR",
  "initial_balance": 5000000
}
```

**Create pocket payload example (Gold):**
```json
{
  "name": "Gold Savings",
  "denomination": "GRAM",
  "initial_balance": 10.5,
  "auto_fetch_price": true
}
```

---

## 4. Transactions

| Method | Endpoint | Description |
|---|---|---|
| GET | `/pockets/{pocket_id}/transactions/` | List transactions for a pocket |
| POST | `/pockets/{pocket_id}/transactions/` | Add a new transaction |
| PATCH | `/transactions/{id}/` | Edit a transaction |
| DELETE | `/transactions/{id}/` | Delete a transaction |

**Create transaction payload example (IDR pocket):**
```json
{
  "direction": "in",
  "amount": 8000000,
  "category": "Salary",
  "tags": ["monthly"],
  "note": "August salary",
  "date": "2026-08-27"
}
```

**Create transaction payload example (Gold pocket):**

`price_at_transaction` is required. The frontend pre-fills this value by calling `GET /gold-price/current/` before submitting — user can edit before saving.

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

> If `price_at_transaction` is omitted for a non-IDR pocket, the API returns a validation error — this field cannot be blank on save.

---

## 5. Gold Price

| Method | Endpoint | Description |
|---|---|---|
| GET | `/gold-price/current/` | Returns latest price — live fetch if cache expired, falls back to last cached `GoldPriceLog` entry if fetch fails |

**Response example:**
```json
{
  "source": "galeri24",
  "price_per_gram": 1950000,
  "fetched_at": "2026-08-27T06:00:00Z",
  "is_fallback": false
}
```

**Fallback response example (live fetch failed, using last known log):**
```json
{
  "source": "galeri24",
  "price_per_gram": 1930000,
  "fetched_at": "2026-08-26T06:00:00Z",
  "is_fallback": true
}
```

**No data available at all:**
```json
{
  "source": "galeri24",
  "price_per_gram": null,
  "fetched_at": null,
  "is_fallback": true
}
```
> Frontend must handle `price_per_gram: null` by requiring manual user entry — this is the rare edge case where no cached price exists yet.

---

## 6. Wealth Summary

| Method | Endpoint | Description |
|---|---|---|
| GET | `/wealth/summary/` | Returns total wealth, Nisab, status, Haul info |

**Response example:**
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

**When above Nisab and Haul in progress:**
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

---

## 7. Zakat / Haul Actions

| Method | Endpoint | Description |
|---|---|---|
| GET | `/haul/history/` | List past Haul periods for user |
| POST | `/haul/{id}/mark-paid/` | Mark Zakat as paid, resets Haul cycle |

---

## 8. Error Format (standard across all endpoints)

```json
{
  "error": {
    "code": "invalid_pocket_type",
    "message": "Pocket denomination must be a supported type."
  }
}
```

**Validation error example — missing required price:**
```json
{
  "error": {
    "code": "price_required",
    "message": "price_at_transaction is required for non-IDR pockets."
  }
}
```

---

## 9. Notes

- All monetary values are integers in IDR (no decimals — Rupiah has no subunits in practice)
- Gold amounts use decimal with up to 2 decimal places (grams)
- All list endpoints support pagination (`?page=1&page_size=20`)
- Wealth summary recalculates in real-time — not cached (only gold price is cached)
- `price_at_transaction` is always required for non-IDR pockets, pre-filled by the frontend from `/gold-price/current/`, never auto-filled server-side based on transaction date

---

*Document status: Draft*
*Last updated: 2026-08-27*