<div align="center">

# 📈 NSE Stock Intelligence Platform

### Automated Stock Screening · AI-Powered Insights · Self-Hosted Production System

[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![React](https://img.shields.io/badge/React%20+%20Vite-61DAFB?style=for-the-badge&logo=react&logoColor=black)](https://react.dev)
[![TimescaleDB](https://img.shields.io/badge/TimescaleDB-FDB515?style=for-the-badge&logo=timescale&logoColor=black)](https://www.timescale.com)
[![Airflow](https://img.shields.io/badge/Apache%20Airflow-017CEE?style=for-the-badge&logo=apacheairflow&logoColor=white)](https://airflow.apache.org)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://docker.com)
[![Claude AI](https://img.shields.io/badge/Claude%20AI-D97757?style=for-the-badge&logo=anthropic&logoColor=white)](https://anthropic.com)

*A production-grade platform that automatically screens ~296 NSE-listed stocks, scores them on fundamentals + technicals, identifies optimal entry zones, and delivers AI-written portfolio newsletters — running 24/7 on a self-hosted VPS.*

</div>

---

## 📌 What This Does

This platform eliminates the manual effort of stock research by automating the full pipeline — from raw market data ingestion to an AI-generated daily brief that tells you exactly which stocks to watch and why.

| Capability | Details |
|---|---|
| 🏦 Universe | ~296 NSE-listed stocks tracked daily |
| 🎯 Active Candidates | ~35 high-conviction stocks in the entry zone |
| 🤖 AI Newsletters | Daily portfolio + candidate digest via Claude AI |
| 📡 Live UI | WebSocket-powered real-time updates |
| 🔄 Automation | Apache Airflow DAGs, fully scheduled |
| 🐳 Deployment | Docker Compose + Watchtower on Contabo VPS |

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          DATA INGESTION                              │
│                                                                      │
│   yfinance / NSE APIs  →  JupyterHub Notebooks  →  Airflow DAGs     │
│                                    │                                 │
│                          Apache Airflow (Orchestrator)               │
│                          ┌─────────┴──────────────┐                 │
│                          │                        │                 │
│               Fact Processor DAG         AI Newsletter DAG          │
│               (NB_optimized_stock_       (NB_AI_Portfolio_           │
│                fact_processor)            Newsletter)                │
└────────────────────┬──────────────────────────────┬─────────────────┘
                     │                              │
                     ▼                              ▼
┌─────────────────────────────┐    ┌────────────────────────────────┐
│       TimescaleDB            │    │         AI PIPELINE            │
│                              │    │                                │
│  dim_stock_info (tickers)    │    │  Perplexity Sonar              │
│  dim_portfolio               │◄───│  (News Fetcher)                │
│  fact_ranking (~296 stocks)  │    │         │                      │
│  fact_active_candidates      │    │         ▼                      │
│  fact_candidate_outcomes     │    │  Claude Sonnet 3.5             │
│  fact_newsletter_log (RAG)   │    │  (Portfolio Newsletter)        │
│  fact_candidate_log (RAG)    │    │         │                      │
│  dim_index_prices (Nifty50)  │    │  Claude Haiku                  │
└───────────────┬─────────────┘    │  (Candidate Digest)            │
                │                  └────────────────────────────────┘
                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         BACKEND (FastAPI)                            │
│                                                                      │
│  asyncpg  │  REST Endpoints  │  WebSocket Server  │  IST Timezone   │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       FRONTEND (React + Vite)                        │
│                                                                      │
│  ActiveCandidates.jsx  │  Portfolio.jsx  │  StockModal.jsx          │
│  StatsBar.jsx          │  WebSocket + REST Fallback                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Scoring System

Each stock is evaluated on a **100-point composite score** across four dimensions:

```
┌──────────────────────────────────────────────────────────┐
│                  STOCK SCORING MODEL                      │
├─────────────────────────────┬────────────────────────────┤
│  📊 Fundamental Score       │  55 points                 │
│     ├── ROE                 │                            │
│     ├── ROCE                │                            │
│     ├── Debt/Equity Ratio   │                            │
│     ├── Revenue Growth      │                            │
│     ├── Earnings Growth     │                            │
│     ├── Profit Margin       │                            │
│     └── Sector Premium      │                            │
├─────────────────────────────┼────────────────────────────┤
│  📈 Technical Score         │  25 points                 │
│     ├── MA-50 alignment     │                            │
│     ├── MA-200 alignment    │                            │
│     └── Price momentum      │                            │
├─────────────────────────────┼────────────────────────────┤
│  📦 Volume Score            │  10 points                 │
├─────────────────────────────┼────────────────────────────┤
│  💥 Breakout Bonus          │  +10 points                │
└─────────────────────────────┴────────────────────────────┘
```

### 🟢 Ideal Entry Zone Logic

A stock enters the **Active Candidates** list only when:

```python
# Entry Criteria
CMP is within 2% of max(MA-50, MA-200)   # Price near key moving average
AND fundamental_score >= 35               # Minimum fundamental quality

# Risk Management (auto-calculated)
stop_loss  = MA_200 × 0.97               # 3% below MA-200
target_1   = entry × 1.20               # +20% target
target_2   = entry × 1.35               # +35% target
```

---

## 🤖 AI Pipeline

The platform runs a dual-model AI pipeline every day:

```
Perplexity Sonar (Web Search)
        │
        │  Fetches latest news for each stock in portfolio
        │  + all active candidates
        ▼
Claude Sonnet 3.5
        │
        │  Generates Portfolio Newsletter
        │  ├── Uses fact_newsletter_log as RAG context
        │  ├── Avoids repeating recently sent insights
        │  └── Outputs structured market commentary
        ▼
Claude Haiku 3
        │
        │  Generates Candidate Digest
        │  ├── Uses fact_candidate_log as RAG context
        │  ├── Concise entry/exit signals per stock
        │  └── Highlights breakout candidates
        ▼
  Stored in TimescaleDB → Served via REST API
```

**RAG Design:** Both logs (`fact_newsletter_log`, `fact_candidate_log`) store historical AI outputs. Each new generation retrieves recent entries to prevent repetitive content and maintain continuity across days.

---

## 🖥️ Frontend Screens

### Active Candidates View
- Real-time WebSocket connection (REST fallback if socket fails)
- Displays ~35 stocks with score, sector, CMP, stop loss, targets
- Classification column showing market regime (BULL / BEAR / SIDEWAYS / VOLATILE)
- StatsBar: avg score + top sector computed from live data

### Portfolio View
- Add / Edit / Delete portfolio positions
- Live P&L computed from current market price
- Entry price, quantity, entry date per holding

### Stock Modal
- Full score breakdown: Fundamental / Technical / Breakout / Volume
- AI-generated news summary per stock
- Historical price chart overlay

---

## 🗄️ Database Schema (Key Tables)

```sql
-- Core stock universe
dim_stock_info       (ticker PK, company_name, sector, ...)
dim_portfolio        (ticker PK, entry_price, quantity, entry_date, ...)
dim_index_prices     (date, close)   -- Nifty 50 daily

-- Fact tables (TimescaleDB hypertables)
fact_ranking             (~296 rows/day, full universe scores)
fact_active_candidates   (~35 rows/day, entry-zone stocks)
fact_candidate_outcomes  (tracks actual outcome of each candidate)

-- AI RAG stores
fact_newsletter_log  (date, content, tokens)
fact_candidate_log   (date, ticker, content)
```

---

## ⚙️ Infrastructure & DevOps

```
Contabo VPS (Ubuntu)
│
├── Docker Compose
│   ├── nginx              (reverse proxy, SSL termination)
│   ├── fastapi-backend    (port 8000)
│   ├── react-frontend     (port 3000)
│   ├── timescaledb        (port 5432)
│   ├── airflow-webserver  (port 8080)
│   ├── airflow-scheduler
│   └── jupyterhub         (port 8888)
│
└── Watchtower             (auto-pulls new images from Docker Hub)
```

### CI/CD Pipeline
```
git push → GitHub Actions
              │
              ├── Run tests
              ├── Build Docker image
              ├── Push to Docker Hub (anoopm2801/*)
              └── Watchtower detects new image → auto-deploys on VPS
```

---

## 📁 Repository Structure

```
nse-stock-app/
├── backend/
│   ├── main.py                    # FastAPI app entry point
│   ├── routers/
│   │   ├── candidates.py          # Active candidates endpoints
│   │   ├── portfolio.py           # Portfolio CRUD
│   │   ├── ranking.py             # Full universe ranking
│   │   └── websocket.py           # WebSocket server
│   └── db/
│       └── connection.py          # asyncpg pool setup
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── ActiveCandidates.jsx
│   │   │   ├── Portfolio.jsx
│   │   │   ├── StockModal.jsx
│   │   │   └── StatsBar.jsx
│   │   └── App.jsx
│   └── vite.config.js
├── notebooks/
│   ├── NB_optimized_stock_fact_processor.ipynb
│   ├── NB_AI_Portfolio_Newsletter.ipynb
│   ├── NB_AI_Active_Candidates_Insight.ipynb
│   └── NB_Backtest_Fact_Processor_2.ipynb
├── dags/
│   ├── daily_fact_processor.py
│   └── ai_newsletter_dag.py
├── docker-compose.yml
├── nginx.conf
└── .github/
    └── workflows/
        └── deploy.yml
```

---

## 🗺️ Roadmap

- [x] Fundamental + Technical scoring engine
- [x] Active candidates with entry zone logic
- [x] AI newsletter pipeline (Perplexity + Claude)
- [x] React UI with WebSocket live updates
- [x] Portfolio tracker with live P&L
- [x] Full Docker + GitHub Actions CI/CD
- [ ] **Kafka + PySpark real-time streaming** (Angel One SmartAPI / Upstox API v2)
- [ ] Intraday price streaming (replacing yfinance polling)
- [ ] Backtesting dashboard for strategy validation
- [ ] Mobile-responsive PWA

---

## 🧰 Tech Stack Summary

| Layer | Technology |
|---|---|
| **Backend** | FastAPI, asyncpg, Python |
| **Database** | TimescaleDB (PostgreSQL), asyncpg |
| **Frontend** | React 18, Vite, WebSocket API |
| **Orchestration** | Apache Airflow (DockerOperator DAGs) |
| **Notebooks** | JupyterHub |
| **AI Models** | Claude Sonnet 3.5, Claude Haiku 3, Perplexity Sonar |
| **Infra** | Nginx, Docker Compose, Contabo VPS |
| **CI/CD** | GitHub Actions → Docker Hub → Watchtower |

---

<div align="center">

*Code is private. Architecture, design, and documentation are public.*
*Reach out if you want to discuss the system design or data engineering decisions.*

</div>