# MT5 Trading Platform
![alt text](image-1.png)

How the system works (the short tour)
Data flows top-down through 6 tiers, with two distinct paths shown by the arrow colors in the diagram:
Live path (green) — what fills your chart in real time:

MT5 EA on your Windows VM streams EURUSD ticks + closed M1 candles over TCP.
mt5-bridge translates TCP → AMQP and drops messages onto RabbitMQ exchanges.
backend-api's MarketDataConsumer picks them up:

aggregates ticks into forming bars in memory,
writes closed bars into ClickHouse async (1000-row batches or 500 ms flush),
broadcasts a market_candles update to every WS subscriber that matches the sessionId/symbols/timeframes filter.


The frontend chart (subscribed to /ws/market) renders the forming bar every second and replaces it with the closed bar at minute boundary.

Historical path (purple, dashed) — what fills the chart on page refresh:

Frontend hits GET /api/market-data/candles?symbol=EURUSD&timeframe=M1&limit=200, or sends a subscribe over WS.
DomainService.ListMarketCandles queries ClickHouse first (FINAL on the ReplacingMergeTree), falls back to Postgres.
Snapshot returns up to N bars, chart auto-fits, then live path takes over.

Order/session path (grey) — placing trades:

Frontend POST /api/sessions/{id}/orders → backend creates pending order in Postgres → publishes confirmations.APPROVED to RabbitMQ.
oms-rms consumes, validates against risk rules, publishes orders.validated.
backend-api consumes the validated event, publishes SEND_ORDER toward mt5-bridge, which executes on MT5.
Fill confirmation comes back through the same chain in reverse → WS update to frontend.

Two changes I just applied to address the chart flicker you saw in the recording:

candlePreviewPublishInterval: 250 ms → 1 s. Quarters the chart re-render rate.
publishLive now skips broadcasts when OHLC + volume haven't moved since the last push. On a stalled price, no update goes out at all.



A real-time trading platform that connects MetaTrader 5 to a TradingView-style web frontend. Built with Go, ClickHouse, PostgreSQL, and RabbitMQ.

> **Status:** active development · single-symbol streaming working end-to-end · multi-symbol scaling in progress.

---

## What it does

- **Streams live OHLCV** from MT5 to a web chart at sub-second latency.
- **Persists candles** to ClickHouse for fast pan/zoom across long history (months/years).
- **Routes orders** from web → backend → MT5 with risk validation and an async OMS.
- **Supports copy-trading** between master and slave sessions (SL/TP modify, close-on-close, configurable lot sizing).
- **Per-trader scoping** of positions, P&L, and websocket events via JWT roles.

If you want to build a custom trading dashboard on top of an MT5 broker account, this is the kind of plumbing you need.

---

## Architecture

```
┌────────────────────────────────────────────────────────┐
│  MetaTrader 5 + Expert Advisor (.mq5)                  │
│  Sends ticks + closed candles over TCP                 │
└──────────────────────┬─────────────────────────────────┘
                       │ JSON / TCP :5556
                       ▼
┌────────────────────────────────────────────────────────┐
│  mt5-bridge  (Go)                                      │
│  TCP listener → AMQP publisher                         │
└──────────────────────┬─────────────────────────────────┘
                       │ AMQP
                       ▼
┌────────────────────────────────────────────────────────┐
│  RabbitMQ                                              │
│  Topic exchanges: e.market.data, e.trades.orders,      │
│                   e.trades.confirmations, e.system.*   │
└─────┬──────────────────────────────────────────┬───────┘
      │                                          │
      ▼                                          ▼
┌──────────────────────────┐         ┌─────────────────────┐
│  backend-api  (Go)       │         │  oms-rms  (Go)      │
│  HTTP + WebSocket        │  ◀────▶ │  order validation   │
│  market-data consumer    │  AMQP   │  risk rules         │
│  in-memory aggregator    │         │  trailing stop      │
│  ClickHouse writer       │         └─────────────────────┘
└─────┬──────────────────┬─┘
      │ async batch     │ JSONB
      ▼                  ▼
┌──────────────┐   ┌─────────────────┐
│  ClickHouse  │   │  PostgreSQL     │
│  candles     │   │  users          │
│  (Replacing- │   │  sessions       │
│   MergeTree) │   │  orders         │
│              │   │  audit_logs     │
└──────────────┘   └─────────────────┘
      │                  │
      ▼                  ▼
┌────────────────────────────────────────────────────────┐
│  Frontend  (Next.js / React)                           │
│  TradingView-style chart · watchlist · order ticket    │
└────────────────────────────────────────────────────────┘
```

---

## Services

| Service             | Language | Purpose                                                                    |
| ------------------- | -------- | -------------------------------------------------------------------------- |
| `backend-api`       | Go       | HTTP API + WebSocket, JWT auth, market-data consumer, ClickHouse writer    |
| `oms-rms`           | Go       | Async order validation, risk rules, trailing stop, copy-trading sync       |
| `mt5-bridge`        | Go       | Bridge between MT5 EA (TCP) and RabbitMQ                                   |
| `market-data-bridge`| Go       | Optional dedicated tick/candle pipe when EA splits ports                   |
| `rabbitmq`          | -        | Message broker                                                             |
| `postgres`          | -        | Operational data (users, sessions, orders, audit logs)                     |
| `clickhouse`        | -        | OHLCV chart store, primary read path for candles                           |

The frontend lives in a separate repo.

---

## Tech stack

- **Language:** Go 1.22+
- **HTTP router:** Chi
- **Auth:** JWT (HS256)
- **Message broker:** RabbitMQ 3.x
- **Operational DB:** PostgreSQL 16
- **Time-series / chart store:** ClickHouse 24.x (ReplacingMergeTree, partitioned by month)
- **MT5 integration:** MQL5 Expert Advisor + custom TCP framing
- **Container runtime:** Docker Compose
- **CI/CD:** GitHub Actions (validates → builds → pushes images to GHCR → self-hosted runner deploys)

---

## Quick start (local)

```bash
git clone <this repo>
cd <repo>
cp .env.example .env
docker compose up -d --build
```

Services:
- `backend-api` — http://localhost:8081
  - REST: `/api/*`
  - WebSocket: `/ws`, `/ws/market`
- `rabbitmq` management — http://localhost:15672 (guest/guest)
- `pgadmin` — http://localhost:5050
- `clickhouse` HTTP — http://localhost:8123

Health check:
```bash
curl http://localhost:8081/health
```

---

## REST endpoints (selected)

```
POST /auth/login
GET  /api/me
GET  /api/sessions
POST /api/sessions/{id}/orders
GET  /api/sessions/{id}/orders/history
POST /api/sessions/{id}/orders/break-even
POST /api/sessions/{id}/copy-links
GET  /api/market-data/candles?symbol=EURUSD&timeframe=M1&limit=200
GET  /api/market-data/symbols
GET  /api/portfolio/metrics
GET  /api/economic-events
```

Full list: `backend-api/ENDPOINTS.md`.

---

## WebSocket

Connect with a JWT and subscribe to a symbol/timeframe:

```
ws://localhost:8081/ws/market?token=<JWT>&sessionId=<S>&symbols=EURUSD&timeframes=M1
```

Message shape:
```json
{
  "type": "table_update",
  "table": "market_candles",
  "data": {
    "symbol": "EURUSD",
    "timeframe": "M1",
    "timestamp": 1709848200000,
    "open": 1.07210,
    "high": 1.07225,
    "low":  1.07208,
    "close":1.07219,
    "volume": 47
  }
}
```

Tables: `market_candles`, `market_ticks`, `market_candles_history`, `sessions`, `orders`, `portfolio`.

---

## ClickHouse schema

Single table; ReplacingMergeTree handles duplicates from EA + aggregator emitting the same close.

```sql
CREATE TABLE candles (
  broker_id   LowCardinality(String),
  symbol      LowCardinality(String),
  timeframe   LowCardinality(String),
  ts          DateTime64(3, 'UTC'),
  open Float64, high Float64, low Float64, close Float64, volume Float64,
  source      LowCardinality(String) DEFAULT '',
  inserted_at DateTime DEFAULT now()
)
ENGINE = ReplacingMergeTree(inserted_at)
PARTITION BY toYYYYMM(ts)
ORDER BY (broker_id, symbol, timeframe, ts);
```

---

## Environment variables (backend-api)

```bash
PORT=8081
POSTGRES_DSN=postgres://user:pass@postgres:5432/db?sslmode=disable
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
CLICKHOUSE_DSN=clickhouse://user:pass@clickhouse:9000/db?dial_timeout=5s

JWT_SECRET=<32+ char secret>
JWT_EXPIRES_IN_MINUTES=0          # 0 = no expiry, revoke via token_version
ADMIN_USERNAME=admin
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=<10+ char password>

MARKET_DATA_PREFETCH=200
CHART_DATA_DISABLED=false
```

---

## Project layout

```
.
├── backend-api/         # Main HTTP/WS API (Go)
│   ├── cmd/server/      # entrypoint
│   ├── internal/
│   │   ├── http/        # routes, handlers, middleware
│   │   ├── services/    # domain logic, ClickHouse writer
│   │   ├── messaging/   # RabbitMQ client, market-data consumer
│   │   ├── ws/          # event hub, realtime manager
│   │   ├── repository/  # PostgreSQL persistence
│   │   ├── clickhouse/  # CH driver wrapper, schema init
│   │   ├── postgres/    # DB bootstrap, schema migrations
│   │   ├── auth/        # JWT
│   │   └── models/      # domain types
│   └── go.mod
├── oms-rms/             # Order/risk worker (Go)
├── mt5-bridge/          # MT5 TCP ↔ RabbitMQ bridge (Go)
├── market-data-bridge/  # Dedicated market-data bridge (Go)
├── docker/clickhouse/   # ClickHouse config + init SQL
├── DOCUMENTATION/       # Architecture & ops docs
├── .github/workflows/   # CI/CD
└── docker-compose.yml
```

---

## Performance notes

- Live forming-bar broadcast cadence: **1 s** (configurable). Skips emission when OHLC+volume hasn't moved.
- ClickHouse writes are **batched async** (1000 rows or 500 ms flush window).
- WebSocket fan-out runs through a **bounded worker pool** (no unbounded goroutine spawn per update).
- DB updates for session metrics use a **single batch transaction** per refresh tick (avoids N+1).
- Symbol metadata cached **in-memory** with RWMutex; updated every 3-5s from MT5.

For a more detailed bottleneck audit see `PERFORMANCE_BOTTLENECKS.md`.

---

## Roadmap

- [ ] Multi-symbol streaming from EA (currently single-symbol mode)
- [ ] Tick batching at the bridge layer (50ms flush) to cut AMQP overhead
- [ ] Redis cache for symbol metadata + JWT token-version (multi-replica scaling)
- [ ] ClickHouse materialized views for higher-TF aggregation (replace in-process aggregator)
- [ ] Frontend mobile layout

---

## Contributing

Issues and PRs welcome. The codebase is opinionated — read `DOCUMENTATION/00_SYSTEM_OVERVIEW.md` and `CODEBASE_MAP.md` before opening a structural change.

Style:
- Go formatting: `gofmt` + `goimports`
- Error wrapping: `fmt.Errorf("context: %w", err)`
- All HTTP handlers must use the response helpers, not `http.Error`
- All chart writes must go through `services.ClickHouseCandleService`

---

## License

MIT — see `LICENSE`.
