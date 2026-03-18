<div align="center">

# 💰 Personal Expense Tracker

### Self-Hosted · Google OAuth2 · Smart Budget Analytics

[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge&logo=mongodb&logoColor=white)](https://mongodb.com)
[![JavaScript](https://img.shields.io/badge/Vanilla%20JS-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)](#)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.com)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://docker.com)
[![OAuth2](https://img.shields.io/badge/Google%20OAuth2-4285F4?style=for-the-badge&logo=google&logoColor=white)](#)

*A production-deployed, multi-user personal finance tracker with smart budget modelling, carry-over logic, and visual analytics — built from scratch and self-hosted on a personal VPS.*

</div>

---

## 📌 What This Does

A full-featured personal finance management application that goes beyond simple expense logging. It models your monthly finances using the **50/30/20 budget framework**, handles income from multiple sources, tracks carry-over savings between months, and gives you visual breakdowns by category, payment method, and time period.

| Capability | Details |
|---|---|
| 🔐 Authentication | Google OAuth2 via OAuth2 Proxy (no passwords) |
| 💳 Expense Tracking | Category + payment method + section tagging |
| 📊 Budget Model | Needs / Wants / Savings (50/30/20) |
| 🔁 Carry-Over | Unspent budget rolls into next month |
| 💼 Income Tracking | Salary + multiple additional income sources |
| 📈 Analytics | Filter by year / month / category / payment method |
| 👤 Multi-User | Isolated data per Google account |
| 🔄 CI/CD | Zero-downtime auto-deploy via GitHub Actions |

---

## 🏗️ Architecture

```
Browser
  │
  │  HTTPS
  ▼
Nginx (Reverse Proxy)
  │
  ├── /oauth2/* ──────────► OAuth2 Proxy Container
  │                              │
  │                    Google OAuth2 (accounts.google.com)
  │                              │
  │                    Injects headers into request:
  │                    X-User-Name, X-User-Email
  │
  └── /* ─────────────────► FastAPI Backend (port 8001)
                                │
                      Reads X-User-Email header
                      Maps to internal user_id
                                │
                                ▼
                           MongoDB (port 27018)
                           Database: expense_tracker
                           Collection: expenses_data
```

### Key Design Decision: Header-Based Auth
Rather than building a custom auth system, Nginx + OAuth2 Proxy handles the entire authentication flow. By the time a request reaches FastAPI, the user's identity is already verified and injected as HTTP headers — **zero auth code in the application layer**.

---

## 🗂️ Data Model

### Expense Document (MongoDB)
```json
{
  "_id": "ObjectId",
  "user_id": 1,
  "date": "2024-03-15",
  "amount": 2500.00,
  "category": "Groceries",
  "payment_method": "UPI",
  "section": "Needs",
  "description": "Monthly groceries",
  "month": "2024-03",
  "created_at": "2024-03-15T10:30:00"
}
```

> **Key Architecture Note:** The `section` field (Needs / Wants / Savings) is **stamped at insert time** using `config.csv` as the source of truth — it is never recomputed. This ensures historical integrity even if category-to-section mappings change later.

### Config-Driven Business Logic
```
config.csv              ← Defines: Category → Section → Allowed Payment Methods
budget_ratio.json       ← Defines: 50/30/20 split percentages (customisable)
```

Both files are the **single source of truth** for all business rules. No hardcoded logic in application code.

---

## 💡 Feature Deep-Dive

### 📊 Budget Page
The core of the application. Shows three cards — **Needs**, **Wants**, **Savings** — each with:
- Allocated budget (from income × ratio)
- Spent so far this month
- Remaining balance
- Carry-over from previous month (if any)

**Carry-over Logic:** If last month's Savings budget was not fully spent, the surplus rolls into this month's starting balance — mimicking real-world budgeting behaviour.

### 💼 Income Page
- Record monthly salary
- Add multiple additional income sources (freelance, rental, etc.)
- Total income auto-generates the monthly budget split using `budget_ratio.json`
- Historical income records preserved per month

### 📈 Analysis Tab
Interactive analytics with 4-level filtering:

```
Year → Month → Category → Payment Method
```

All filters are additive. The view re-fetches data on every tab switch to ensure freshness without caching stale state.

### 🔐 Multi-User Isolation
Each Google account gets a unique `user_id` (integer). All MongoDB queries are automatically scoped to the authenticated user — no data leakage between users.

---

## 🛠️ API Reference

### Expense Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/expenses` | List expenses (current month, filtered) |
| `POST` | `/api/expenses` | Add new expense |
| `PUT` | `/api/expenses/{id}` | Update expense |
| `DELETE` | `/api/expenses/{id}` | Delete expense |

### Budget Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/current-month` | Canonical month info (frontend date source) |
| `GET` | `/api/budget` | Budget cards with carry-over |
| `GET` | `/api/income` | Income sources for current month |
| `POST` | `/api/income` | Add income source |

### Analysis Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/analysis` | Filtered expense analytics |
| `GET` | `/api/categories` | Category list from config |
| `GET` | `/api/payment-methods` | Payment methods from config |

### Admin Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/admin/recalc-budget` | Recalculate all budgets |
| `POST` | `/api/admin/backfill-section` | Backfill section on old records |

---

## ⚙️ Infrastructure

```
Contabo VPS (Ubuntu)
│
└── Docker Compose (network: client_mode)
    ├── nginx                  (reverse proxy + SSL)
    ├── oauth2-proxy           (Google OAuth2 authentication)
    ├── expense-fastapi        (port 8001)
    └── expense-mongodb        (port 27018)
        └── DB: expense_tracker
            └── Collection: expenses_data
```

### CI/CD
```
git push main
    │
    └── GitHub Actions
        ├── Lint + basic checks
        ├── Build Docker image
        ├── Push to Docker Hub
        └── SSH into VPS → docker compose pull → docker compose up -d
```

---

## 📁 Repository Structure

```
expense-tracker/
├── backend/
│   ├── main.py                  # FastAPI app, route registration
│   ├── routers/
│   │   ├── expenses.py          # Expense CRUD
│   │   ├── budget.py            # Budget + carry-over logic
│   │   ├── income.py            # Income sources
│   │   ├── analysis.py          # Analytics queries
│   │   └── admin.py             # Admin utilities
│   ├── db/
│   │   └── mongo.py             # MongoDB connection + collection
│   ├── config/
│   │   ├── config.csv           # Category → Section → Payment mapping
│   │   └── budget_ratio.json    # 50/30/20 split configuration
│   └── models/
│       └── schemas.py           # Pydantic models
├── frontend/
│   ├── index.html
│   ├── app.js                   # SPA routing + switchPage()
│   ├── pages/
│   │   ├── budget.js
│   │   ├── expenses.js
│   │   ├── income.js
│   │   └── analysis.js
│   └── styles/
│       └── main.css
├── docker-compose.yml
├── nginx.conf
└── .github/
    └── workflows/
        └── deploy.yml
```

---

## 🗺️ Completed Features

- [x] Google OAuth2 authentication (zero-password)
- [x] Expense CRUD with category + payment + section tagging
- [x] 50/30/20 budget model with monthly budget cards
- [x] Carry-over logic (unspent budget rolls to next month)
- [x] Multi-source income tracking
- [x] Visual analytics with 4-level drill-down filters
- [x] Config-driven business logic (no hardcoded rules)
- [x] Admin endpoints for bulk operations
- [x] Full Docker + Nginx + CI/CD deployment
- [x] Multi-user data isolation

---

## 🧰 Tech Stack Summary

| Layer | Technology |
|---|---|
| **Backend** | FastAPI, Python, Pydantic |
| **Database** | MongoDB (Docker container) |
| **Frontend** | Vanilla JavaScript SPA |
| **Auth** | Google OAuth2 via OAuth2 Proxy |
| **Proxy** | Nginx (reverse proxy + SSL) |
| **Infra** | Docker Compose, Contabo VPS |
| **CI/CD** | GitHub Actions → Docker Hub → VPS SSH |

---

<div align="center">

*Code is private. Architecture, design, and documentation are public.*
*Built as a personal tool, engineered to production standards.*

</div>