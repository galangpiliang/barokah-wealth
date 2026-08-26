# 01 — Product Requirements Document (PRD)

**Barokah Wealth — MVP**
**Version:** 1.0
**Status:** Draft
**Last updated:** 2026-08-26

---

## 1. Overview

Barokah Wealth is a wealth tracking companion for Muslims in Indonesia. The MVP focuses on one core loop: help the user know their total net wealth in IDR, compare it against the current Nisab threshold, and track the 12-month Haul period to determine Zakat obligation.

---

## 2. Goals

- User can see their total net wealth in IDR at any time
- User knows whether they have reached Nisab
- User can track how long they have been above Nisab (Haul)
- User can calculate how much Zakat they owe when Haul is complete

---

## 3. Non-Goals

### Out of Scope (will not be built)
- Bank or e-wallet integration
- Offline mode — this is a web app only
- Zakat on crops, livestock, trade goods — different fiqh, different product
- Investment portfolio tracking (stocks, crypto) — requires separate pricing APIs
- Social or sharing features

### Deferred to Next Version
- **Multi-currency display and foreign currency pockets** — e.g. a user holds $500 USD savings; the app would store it in USD, convert to IDR daily via exchange rate API, and include it in total wealth. Currently everything is IDR only.
- **Multi-market expansion** — e.g. a Malaysian user tracking wealth in MYR against Malaysian Nisab threshold. Requires locale settings, different gold price sources, and different currency defaults.
- **Recurring transaction templates** — e.g. salary of Rp 8.000.000 auto-logs every 1st of the month without manual entry.
- **Data export (CSV/PDF)** — e.g. user downloads full transaction history as a spreadsheet or exports a Zakat report as PDF.

---

## 4. User Persona

**Name:** Galang
**Age:** Late 20s — mid 30s
**Status:** Married, small family
**Work:** Full-time employee + small side business
**Wealth types:** Cash savings, gold
**Pain point:** Cannot easily answer "do I owe Zakat this year?" without manual calculation

---

## 5. Wealth Categories (MVP)

| Category | Description | Stored as | Displayed as |
|---|---|---|---|
| Cash | Any IDR holdings — personal savings, salary, business capital, daily expenses | IDR | IDR |
| Gold | Physical or digital gold | Grams | IDR (auto-converted via Galeri24 price or manual entry) |

> All asset types convert to IDR for total wealth and Nisab comparison.

> Users can create multiple cash pockets and name them freely (e.g. "Main Account", "Warung Galang", "Emergency Fund"). There is no separate business pocket type — business transactions are simply logged under a cash pocket.

---

## 6. Features

### 6.1 Dashboard

- Display total net wealth in IDR
- Display current Nisab threshold in IDR (based on live gold price × 85g)
- Display wealth vs Nisab status: **Below Nisab / Above Nisab**
- Display Haul status: **Not started / In progress (X days remaining) / Complete**
- Display Zakat amount due (if Haul complete and above Nisab)

---

### 6.2 Pockets

All wealth is organized into **pockets**. Each pocket has:
- A name (user-defined)
- A type: **Cash** or **Gold**
- An initial balance setting
- Its own transaction log

Users can create multiple pockets of any type.

---

### 6.3 Cash Pocket

**Two modes — user chooses per pocket:**

**Simple mode:**
- User manually updates cash balance at any frequency (daily, weekly, monthly)
- Every update triggers a Nisab recalculation

**Transaction mode (for advanced users):**
- User logs individual income and expense entries
- App calculates running cash balance automatically
- Every income entry triggers a Nisab recalculation

**Transaction entry fields:**
- Type: Income / Expense
- Amount (IDR)
- Category (optional, user-defined — defaults: Salary, Food, Transport, Other)
- Tags (optional, user-defined, multiple allowed)
- Note (optional)
- Date

---

### 6.4 Gold Pocket

- User enters gold holdings in **grams**
- Gold value in IDR is calculated automatically

**Gold pocket settings:**
- **Auto-fetch toggle:** On/Off
  - If **On:** app fetches current buyback price from Galeri24 automatically. User can verify the price at galeri24.co.id
  - If **Off:** user manually enters gold buyback price per gram in IDR
- Price is refreshed on app open / login and every 24 hours when auto-fetch is on

---

### 6.5 Nisab Calculation

- Nisab threshold = gold buyback price per gram × 85
- Recalculated automatically whenever:
  - Gold price is refreshed from API
  - User updates any pocket balance or logs a transaction
- Displayed in IDR on the dashboard

---

### 6.6 Haul Tracking

- Haul starts automatically on the **first date the user's total wealth exceeds Nisab**
- Haul duration: **12 Hijri months (~354 days)**
- If wealth drops below Nisab at any point during Haul, the Haul **resets**
- Haul restarts when wealth exceeds Nisab again
- Dashboard shows: days elapsed, days remaining, or "Haul Complete"

---

### 6.7 Zakat Calculation

- Triggered when Haul is complete and wealth is still above Nisab
- Zakat amount = total net wealth × 2.5%
- Displayed prominently on dashboard
- User marks Zakat as paid — resets Haul and starts a new cycle

---

### 6.8 Notifications (MVP — basic)

- "You have reached Nisab — Haul has started"
- "Your Haul is complete — Zakat is now due"
- "Your wealth dropped below Nisab — Haul has been reset"

---

## 7. User Flows

### Flow 1 — First time setup
1. User opens app → lands on dashboard (empty state)
2. User creates first pocket (Cash or Gold)
3. User sets initial balance or logs first transaction
4. App calculates total wealth and compares to Nisab
5. If above Nisab → Haul starts, user is notified

### Flow 2 — Daily/regular use
1. User opens app → sees dashboard with current wealth and Nisab status
2. User logs new income or updates a pocket balance
3. Nisab recalculates instantly
4. Haul timer updates

### Flow 3 — Zakat payment
1. Haul completes → user receives notification
2. User opens app → sees Zakat amount due
3. User pays Zakat externally
4. User marks as paid in app → Haul resets

---

## 8. Technical Notes

| Concern | Decision |
|---|---|
| Gold price API | logam-mulia-api.iamutaki.workers.dev (free, no auth) |
| Gold price source | Galeri24 buyback price |
| Gold price refresh | On app open / login + every 24 hours |
| Haul calendar | Hijri calendar (354 days) |
| Currency (MVP) | IDR only |
| Auth | Email + password (MVP) |
| Platform | Web only |

---

## 9. Success Metrics (MVP)

- User can complete first pocket setup in under 3 minutes
- Nisab status is always visible on dashboard without extra navigation
- Haul tracking is automatic — zero manual input required beyond pocket entries

---

*Document status: Draft*
*Last updated: 2026-08-26*