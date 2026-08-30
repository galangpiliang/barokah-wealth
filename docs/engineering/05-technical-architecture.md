# 05 — Technical Architecture

**Barokah Wealth**

---

## 1. Overview

This document describes how the system components connect: frontend, backend, database, external APIs, and hosting.

---

## 2. System Diagram (described)

```
React (Vite)  ──HTTP/JSON──►  Django REST Framework  ──►  PostgreSQL
     │                              │
     │                              └──►  logam-mulia-api (gold price)
     │
     └── deployed on Vercel          Django deployed on Railway
                                     PostgreSQL hosted on Supabase (prod)
```

---

## 3. Authentication

- **Method:** JWT (JSON Web Token)
- **Library:** `djangorestframework-simplejwt`
- **Flow:**
  1. User logs in with email + password → receives access token + refresh token
  2. Access token sent in `Authorization: Bearer <token>` header on every API request
  3. Refresh token used to obtain new access token when expired
- **Token lifetime:** Access token short-lived (e.g. 15–30 min), refresh token longer (e.g. 7 days)

---

## 4. Local Development Setup (Docker)

**docker-compose services:**

| Service | Purpose |
|---|---|
| `db` | Local PostgreSQL container — mirrors production schema |
| `backend` | Django + DRF |
| `frontend` | React + Vite |

**Why local Postgres instead of Supabase in dev:**
- No risk of touching production data
- Works offline
- Faster iteration, no network latency

**Environment files:**
```
.env.local   → local Postgres credentials (used in Docker dev)
.env.prod    → Supabase credentials (used by Railway in production, never committed)
```

---

## 5. Production Setup

| Component | Hosting |
|---|---|
| Frontend | Vercel |
| Backend | Railway |
| Database | Supabase (PostgreSQL) |

- Railway backend connects to Supabase via connection string stored in Railway's environment variables
- Vercel frontend calls Railway backend via public API URL stored in frontend environment variables

---

## 6. External API Integration

**Gold Price — logam-mulia-api**
- Backend calls this API server-side (not directly from frontend)
- Reason: keeps API logic centralized, allows caching/rate-limit control, avoids CORS issues
- Price fetched on-demand when a gold pocket loads, cached for 24 hours

> **Addendum (2026-08-30):** Changed to a daily scheduled job instead of fetching on each request — avoids slow first-requests and duplicate calls. The API now just reads the latest saved price. Full details in Doc 07.

---

## 7. Backend Structure (Django apps)

```
backend/
├── config/          # Django settings, urls, wsgi
├── users/           # Auth, user model
├── pockets/         # Pocket model (Cash/Gold), transactions
├── wealth/          # Nisab calculation, Haul tracking, Zakat logic
└── goldprice/       # Gold price fetching + caching logic
```

---

## 8. Frontend Structure (React)

```
frontend/
├── src/
│   ├── pages/         # Dashboard, Pockets, Transactions, Settings
│   ├── components/    # Reusable UI (Flowbite-based)
│   ├── api/           # API client, axios instance with JWT handling
│   └── hooks/         # Custom hooks (e.g. useWealthSummary)
```

---

## 9. Key Technical Decisions

| Decision | Reasoning |
|---|---|
| JWT over session auth | Frontend and backend are decoupled apps on different domains |
| Server-side gold price fetch | Avoids CORS, centralizes caching |
| Local Postgres in Docker | Safe, fast local dev without touching prod data |
| Separate Django apps by domain | Keeps codebase modular as features grow |

---

*Document status: Approved*
*Last updated: 2026-08-30*