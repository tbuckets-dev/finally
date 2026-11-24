# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving        │
│                      (Next.js export)           │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background tasks: market data, snapshots       │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no service orchestration needed in production |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

### Backend Startup Sequence

All initialization happens once, at startup, in FastAPI's lifespan handler (not lazily on first request):

1. Initialize the database: create tables and seed default data if the file is missing or empty
2. Load the **tracked ticker set** = watchlist tickers ∪ tickers with open positions
3. Start the market data source with the tracked set
4. Record an initial portfolio snapshot, then start the 30-second snapshot background task

On shutdown, stop the snapshot task and the market data source.

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── app/
│       ├── market/           # Market data subsystem (complete)
│       └── db/               # Schema definitions, seed data, init logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Convenience wrapper for running the app container
├── .env                      # Environment variables (gitignored)
├── .env.example              # Committed template for .env
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. The Python package is `app` (`backend/app/`). Internal structure is up to the Backend/Market Data agents.
- **`backend/app/db/`** contains schema SQL definitions and seed logic, importable as `app.db`.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (`docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false

# Optional: SQLite file location (see defaults below)
DB_PATH=
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests); no API key needed
- `DB_PATH` defaults to `/app/db/finally.db` in the container (set in the Dockerfile) and to `<project root>/db/finally.db` when running locally with `uv run`

### Loading

- **In Docker**: variables are passed with `docker run --env-file .env` (or `env_file:` in compose). There is no `.env` file inside the image.
- **Locally**: the backend loads the project-root `.env` with `python-dotenv` at startup (existing environment variables take precedence).
- `.env.example` with the variables above is committed; `.env` is gitignored.

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source. This subsystem is complete — see `planning/MARKET_DATA_SUMMARY.md`.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.); unknown tickers get a random seed price between $50 and $300
- Adding a ticker seeds the price cache immediately, so a new ticker has a price right away
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all tracked tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator
- A newly added ticker has no price until the next poll completes

### Tracked Tickers

The market data source tracks **watchlist tickers ∪ tickers with open positions**, so held positions can always be valued.

- Adding a ticker to the watchlist → `source.add_ticker()` (no-op if already tracked)
- Removing a ticker from the watchlist → `source.remove_ticker()` **only if** there is no open position in it
- Selling a position to zero → `source.remove_ticker()` **only if** the ticker is not on the watchlist
- Trading a ticker that is not tracked → `source.add_ticker()` first (see §8 trade rules)

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, timestamp, and **session open price** for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

**Session open price**: the first price the cache records for a ticker during this backend process's lifetime. It is the baseline for the watchlist's "change %" column (there is no real market open/previous close in the simulator). It resets when the backend restarts. *This requires a small addition to the completed market module: `PriceUpdate` gains `session_open` and `session_change_percent`, and `PriceCache.update()` carries `session_open` forward from the first update.*

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- The first message is `retry: 1000` (browser reconnects 1 second after a drop)
- The server checks the cache every ~500ms and sends an event **only when the cache has changed** (version-based change detection)
- **Each event contains all tracked tickers** as a single JSON object keyed by ticker:

```
data: {"AAPL": {...}, "GOOGL": {...}, ...}
```

Each ticker value:

```json
{
  "ticker": "AAPL",
  "price": 190.52,
  "previous_price": 190.41,
  "timestamp": 1764000000.123,
  "change": 0.11,
  "change_percent": 0.0578,
  "direction": "up",
  "session_open": 189.80,
  "session_change_percent": 0.3794
}
```

- `timestamp` is Unix seconds (float)
- `change` / `change_percent` are tick-over-tick (vs. `previous_price`) — use for the flash direction
- `session_change_percent` is vs. `session_open` — use for the watchlist "change %" column
- `direction` is `"up"`, `"down"`, or `"flat"`
- Percentages are expressed as percent values (0.38 means 0.38%), not fractions

---

## 7. Database

### SQLite, Initialized at Startup

At startup the backend checks the SQLite file at `DB_PATH`. If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

Every table except `users_profile` has a `user_id` column defaulting to `"default"` (in `users_profile`, the `id` column is the user key). This is hardcoded for now (single-user) but enables future multi-user support without schema migration. All queries filter on `user_id = 'default'`.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`
- The row is **deleted** when quantity falls below `0.0001` after a sell

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded once at startup, every 30 seconds by a background task (including when there are no positions — a flat cash line is fine), and immediately after each trade execution. All rows are kept.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — the `actions` object from the chat response, see §8; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

### Numeric Rules

- Quantities are rounded to 4 decimal places at the trade boundary; minimum trade quantity is `0.0001`
- Cash balance is rounded to 2 decimal places after every trade
- Prices and computed values (P&L, market value) are returned unrounded; the frontend formats them for display

---

## 8. API Endpoints

All endpoints are under `/api`, return JSON, and use ISO 8601 timestamps unless noted.

### Error Format

All errors use FastAPI's default shape:

```json
{"detail": "Insufficient cash: need $1,905.20, have $1,000.00"}
```

- `400` — business-rule failures (insufficient cash/shares, no price available, invalid ticker format)
- `404` — resource not found (e.g. removing a ticker not on the watchlist)
- `422` — request body fails schema validation (FastAPI/Pydantic default; `detail` is a list)

### Ticker Rules (apply everywhere a ticker is accepted, including LLM actions)

- Trimmed and uppercased before use
- Must match `^[A-Z.]{1,6}$`, otherwise `400`
- Simulator mode: any ticker that passes the format check is accepted
- Massive mode: adding a ticker that returns no data from the API is rejected with `400`

### Market Data

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates (format in §6) |

### Portfolio

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

**`GET /api/portfolio`** → `200`

```json
{
  "cash_balance": 8095.00,
  "total_value": 10012.40,
  "positions_value": 1917.40,
  "unrealized_pnl": 12.40,
  "unrealized_pnl_percent": 0.65,
  "positions": [
    {
      "ticker": "AAPL",
      "quantity": 10,
      "avg_cost": 190.50,
      "current_price": 191.74,
      "market_value": 1917.40,
      "unrealized_pnl": 12.40,
      "unrealized_pnl_percent": 0.65,
      "weight_percent": 19.15
    }
  ]
}
```

- `current_price` falls back to `avg_cost` if no cached price exists yet
- `unrealized_pnl_percent` = `(current_price - avg_cost) / avg_cost × 100`
- `weight_percent` = `market_value / total_value × 100` (used for heatmap sizing)

**`POST /api/portfolio/trade`**

Request:

```json
{"ticker": "AAPL", "side": "buy", "quantity": 10}
```

- `side` must be `"buy"` or `"sell"`; `quantity` must be > 0 (else `422`)
- Fill price is the current cached price
- If the ticker is not tracked, it is added to tracking (not to the watchlist). If there is still no cached price (Massive mode, before the next poll), returns `400` "Price not yet available for XYZ, try again in a moment"
- Buy: requires `cash_balance >= quantity × price`; updates `avg_cost` as a weighted average
- Sell: requires held quantity ≥ requested quantity; `avg_cost` is unchanged; row deleted if remaining < `0.0001`
- Records a trade row and a portfolio snapshot

Response `200`:

```json
{
  "trade": {
    "id": "uuid",
    "ticker": "AAPL",
    "side": "buy",
    "quantity": 10,
    "price": 190.52,
    "executed_at": "2026-11-23T14:30:00Z"
  },
  "cash_balance": 8094.80
}
```

The frontend refetches `GET /api/portfolio` after a successful trade.

**`GET /api/portfolio/history`** → `200`

```json
{
  "snapshots": [
    {"recorded_at": "2026-11-23T14:30:00Z", "total_value": 10000.00}
  ]
}
```

Returns at most the most recent 1,000 snapshots, oldest first.

### Watchlist

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

**`GET /api/watchlist`** → `200`

```json
{"tickers": ["AAPL", "GOOGL", "MSFT"]}
```

Ordered by `added_at` ascending. Prices are not included — they arrive via SSE within ~500ms; the UI shows a placeholder until then.

**`POST /api/watchlist`** with `{"ticker": "PYPL"}` → `201` `{"ticker": "PYPL"}`. Adding a ticker already on the watchlist returns `200` with the same body (idempotent).

**`DELETE /api/watchlist/{ticker}`** → `204`. `404` if the ticker isn't on the watchlist. Price tracking continues if there is an open position (§6).

### Chat

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |
| GET | `/api/chat` | Conversation history, for repopulating the chat panel on page load |

**`POST /api/chat`**

Request:

```json
{"message": "Buy 5 shares of PYPL and add it to my watchlist"}
```

Response `200` (always `200`, even if the LLM call fails — see §9):

```json
{
  "id": "uuid",
  "role": "assistant",
  "content": "Done — added PYPL to your watchlist and bought 5 shares at $68.20.",
  "created_at": "2026-11-23T14:30:02Z",
  "actions": {
    "watchlist_changes": [
      {"ticker": "PYPL", "action": "add", "status": "ok", "error": null}
    ],
    "trades": [
      {"ticker": "PYPL", "side": "buy", "quantity": 5, "status": "ok",
       "price": 68.20, "error": null}
    ]
  }
}
```

- Each action has `status` of `"ok"` or `"failed"`; failed actions carry an `error` string and `price: null`
- `actions` is `null` when the LLM requested no actions
- This `actions` object is exactly what is stored in `chat_messages.actions`

**`GET /api/chat`** → `200`

```json
{
  "messages": [
    {"id": "uuid", "role": "user", "content": "...", "actions": null,
     "created_at": "..."},
    {"id": "uuid", "role": "assistant", "content": "...", "actions": {...},
     "created_at": "..."}
  ]
}
```

Returns the most recent 100 messages, oldest first.

### System

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) → `200` `{"status": "ok"}` |

---

## 9. LLM Integration

When writing code to make calls to LLMs, use the `cerebras` skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Stores the user message in `chat_messages`
2. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
3. Loads the **last 20 messages** of conversation history from `chat_messages`
4. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
5. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the `cerebras` skill
6. Parses and validates the complete structured JSON response
7. Executes actions: **watchlist changes first, then trades**, each in the order listed (see Auto-Execution)
8. Stores the assistant message and the `actions` result object in `chat_messages`
9. Returns the response described in §8 (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute
- `watchlist_changes` (optional): Array of watchlist modifications; `action` is `"add"` or `"remove"`

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

Execution rules:
- Watchlist changes are applied before trades, so "add PYPL and buy 5" works in one turn
- Each action goes through exactly the same code path and validation as the equivalent REST endpoint (ticker rules, cash/share checks)
- Actions run sequentially and independently: a failed action does not stop the others, and later trades see the cash/position changes from earlier ones
- Failures are reported per action in the response (`status: "failed"`, `error`) so the UI can show them inline

The LLM's `message` is written before execution, so it may claim success for a trade that then fails. The frontend must render action results from `actions`, not infer them from the message text.

### Failure Handling

If the LLM call errors, times out (30-second limit), or returns JSON that fails schema validation:
- No actions are executed
- The endpoint still returns `200` with a friendly assistant message, e.g. "Sorry, I couldn't process that request. Please try again."
- That assistant message is stored in history like any other (with `actions: null`)
- The error is logged server-side

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic responses instead of calling OpenRouter. Mock responses go through the same parsing and execution path as real ones. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

Mock rules (case-insensitive match on the user message, first match wins):

| Message contains | Mock response |
|---|---|
| `buy` | `message: "Buying 1 share of AAPL."`, `trades: [{AAPL, buy, 1}]` |
| `sell` | `message: "Selling 1 share of AAPL."`, `trades: [{AAPL, sell, 1}]` |
| `add` | `message: "Adding PYPL to your watchlist."`, `watchlist_changes: [{PYPL, add}]` |
| `remove` | `message: "Removing PYPL from your watchlist."`, `watchlist_changes: [{PYPL, remove}]` |
| anything else | `message: "This is a mock response from FinAlly."`, no actions |

---

## 10. Frontend Design

### Layout

The frontend is a **single-page application on one route** (`/`) with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), change % (`session_change_percent` from SSE), and a sparkline mini-chart (accumulated from SSE since page load). Shows a placeholder (e.g. `—`) until the first price arrives.
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here. Defaults to the first watchlist ticker on load. Like sparklines, it only shows data accumulated from SSE since page load — there is no historical price endpoint.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by `weight_percent`, colored by `unrealized_pnl_percent`: linear scale from −5% (full red) through 0% (neutral gray) to +5% (full green), clamped beyond ±5%
- **P&L chart** — line chart showing total portfolio value over time, using `GET /api/portfolio/history`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill. Errors from the API (`detail`) are shown inline.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history (loaded from `GET /api/chat` on page load), loading indicator while waiting for LLM response. Trade executions and watchlist changes are rendered inline from the response's `actions` object, with failed actions shown distinctly alongside their error.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Data Flow

- **Prices**: one `EventSource` on `/api/stream/prices`; the latest price per ticker is held in client state and appended to per-ticker history arrays (for sparklines and the main chart). Cap each history array (e.g. last 1,000 points) to bound memory.
- **Live values**: the header total, positions table, and heatmap are computed on the frontend from the last `GET /api/portfolio` result (quantities, avg cost, cash) combined with live SSE prices. No polling.
- **Refresh triggers**: refetch `GET /api/portfolio` and `GET /api/portfolio/history` after a manual trade or a chat response containing actions; refetch `GET /api/watchlist` after watchlist changes (manual or via chat).
- **Connection status**: green when `EventSource` is open, yellow while reconnecting (`onerror` with `readyState === CONNECTING`), red when closed.

### Technical Notes

- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Charting:
  - **Lightweight Charts** (canvas) for the main price chart and the P&L chart
  - **Recharts `Treemap`** or a small custom SVG component for the heatmap (Lightweight Charts has no treemap)
  - Plain inline SVG polylines for sparklines — cheapest across 10+ instances
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - ENV DB_PATH=/app/db/finally.db
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000. The static export is mounted with `StaticFiles(directory="static", html=True)` at `/` **after** all `/api` routes are registered, so API routes take precedence. No SPA fallback is needed since the app is a single route.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### docker-compose.yml

A convenience wrapper equivalent to the `docker run` command above: builds the image, maps port 8000, mounts the `finally-data` volume at `/app/db`, and loads `.env` via `env_file:`. `docker compose up` is an alternative to the start scripts.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- *Optional nice-to-haves*: a `--build` flag to force a rebuild, opening the browser automatically

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

**Security note**: the app has no authentication. A publicly reachable deployment lets anyone trade and spend the OpenRouter key via `/api/chat`. Any public deployment needs at least basic auth or rate limiting in front of it.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface, session open price is recorded and carried forward
- Portfolio: trade execution logic, weighted avg cost, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss, selling to zero deletes the row, fractional rounding, no cached price)
- Tracked tickers: removing a held ticker from the watchlist keeps it tracked; selling to zero untracks a non-watchlist ticker
- LLM: structured output parsing handles all valid schemas, malformed responses return the fallback message with no actions, watchlist-before-trades ordering, partial failure across multiple trades, mock mode rules
- API routes: correct status codes, response shapes per §8, error format

**Frontend (Vitest + React Testing Library)** — limited to pure logic; E2E covers UI flows:
- Portfolio calculations from positions + live prices (market value, P&L, weights, total)
- Flash direction logic on price changes
- Heatmap color scale
- Number/currency/percent formatting

### E2E Tests (in `test/`)

**Infrastructure**: `test/docker-compose.test.yml` spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism, against a fresh database volume.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with colors matching the §10 scale, P&L chart has data points
- AI chat (mocked): send "buy", receive a response, trade execution appears inline and the position appears
- Chat history persists across a page reload
- SSE resilience: disconnect and verify reconnection
