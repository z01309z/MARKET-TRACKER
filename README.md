# Market Tracker 

A production-grade PySide6 desktop application that streams live cryptocurrency,
foreign-exchange, and global equity data from multiple independent providers with
automatic failover and parallel multi-source aggregation, strict validation, and an exact one-second UI refresh cycle —
all on a custom glassmorphism UI over a user-supplied background image.

**No fabricated prices. No single-provider dependency. No silent failures.**

<img width="1919" height="1031" alt="Screenshot 2026-11-14 063054" src="https://github.com/user-attachments/assets/4e69fc28-8eb5-4b38-98ef-4cb6b87f91e7" />

<img width="1919" height="1027" alt="Screenshot 2026-11-14 063109" src="https://github.com/user-attachments/assets/bd2f664c-f788-4155-8b22-5b8111548492" />

<img width="1919" height="1031" alt="Screenshot 2026-11-14 063118" src="https://github.com/user-attachments/assets/5e1de6af-2348-4b88-a5df-9f3611f2b15c" />

<img width="1919" height="1031" alt="Screenshot 2026-11-14 063132" src="https://github.com/user-attachments/assets/4941a06b-b500-4057-9613-0f5f57d42a94" />

---

## Table of Contents

1. [Features](#features)
2. [Architecture](#architecture)
3. [Data Providers & Limitations](#data-providers--limitations)
4. [Installation](#installation)
5. [Configuration](#configuration)
6. [Running](#running)
7. [Testing](#testing)
8. [How the 1-Second Cycle Works](#how-the-1-second-cycle-works)
9. [Reliability Design](#reliability-design)
10. [Packaging as a Windows Executable](#packaging-as-a-windows-executable)
11. [Project Structure](#project-structure)
12. [Known Limitations](#known-limitations)

---

## Features

### Market coverage
- **CRYPTO page** — top coins **dynamically ranked by market capitalization**
  (discovered live, not hardcoded). The ten user-specified assets
  (BTC, ETH, USDT, BNB, SOL, XRP, USDC, DOGE, ADA, TRX) are always tracked
  as a floor set. Columns: rank, asset, live USD price, 24h %, 24h volume,
  market cap, contributing source count, freshness badge.
- **FOREX page** — 37 pairs: the majors (EUR/USD, GBP/USD, USD/JPY, USD/CHF,
  USD/CAD, AUD/USD, NZD/USD, USD/CNY, USD/JOD), key crosses (EUR/GBP,
  EUR/JPY, EUR/CHF, GBP/JPY, AUD/JPY, CHF/JPY, CAD/JPY, EUR/AUD), Asian &
  emerging (SGD, HKD, TRY, MXN, SAR, AED, EGP, KRW, INR, RUB, BRL, ZAR)
  and Scandinavian/Eastern European (SEK, NOK, DKK, PLN, CZK, HUF, EUR/TRY,
  EUR/SEK). Columns: rate, bid/ask where available, daily change, full
  currency names, source count, freshness badge.
- **STOCKS page** — curated board of 25 global blue chips across US
  (AAPL, MSFT, GOOGL, AMZN, NVDA, META, TSLA, BRK-B, JPM, V, XOM, KO, WMT,
  JNJ, DIS), Asia (Tencent, Alibaba, Toyota, Sony, Samsung) and Europe
  (ASML, SAP, Siemens, Shell, Nestlé). Columns: ticker, company, last price
  (listing currency), day %, volume, currency, source count, freshness
  badge. Closed venues are honestly labelled `DELAYED`.
- **ALL MARKETS tab** (new in v8) — all three markets stacked on one
  scrollable page, each with its own live table and search.
- **Detachable market windows** (new in v8) — "Open Crypto window", "Open
  Forex window" and "Open Stocks window" buttons spawn independent
  top-level windows that update live; keep all three open side by side
  with the main window.
- **Parallel multi-source accuracy** (new in v8) — every market polls all
  of its providers on every cycle and the aggregator takes the median
  price, tracks cross-source disagreement per symbol, and merges the best
  bid/ask across sources. Google Finance joined the stocks chain in v8.
- **Per-symbol backoff** (new in v8) — a symbol that persistently fails on
  one venue (for example `WMT:NYSE` on Google Finance, or coins not listed
  on Coinbase) is backed off after repeated failures and retried
  occasionally instead of producing a failed request every cycle. The
  status bar shows failures as `failed/total` with rate-based coloring,
  so a couple of known per-symbol gaps no longer look like an outage.

### Data infrastructure
- **7 independent providers** across 2 chains with automatic failover.
- **Circuit breakers** per provider (closed → open → half-open → closed).
- **Retries with exponential backoff + jitter**, honouring `Retry-After`.
- **HTTP 429 / 418 awareness** including a local Binance request-weight
  budget so the app self-throttles before the exchange bans the IP.
- **Validation pipeline**: price plausibility, crossed/wide bid-ask spread
  rejection, timestamp freshness (with extended grace for documented-delay
  sources), 24h-change outlier flagging, cross-source discrepancy detection.
- **Consensus pricing**: per-symbol median across providers (robust to one
  bad feed), best bid/ask composite, disagreement % surfaced in the UI.
- **SQLite persistence** (SQLAlchemy): full quote history + last-known-good
  snapshots restored on startup.
- **Graceful degradation**: when every provider fails, the last validated
  price is shown with a `STALE`/`OFFLINE` badge and its original source
  timestamp — never zero, never "live"-looking.

### UI
- Complete dark **glassmorphism redesign** over the user-supplied background
  image (charcoal + steel-blue): the image is painted edge-to-edge and
  cover-scaled with the window; all panels are semi-transparent glass.
- CRYPTO (blue) / FOREX (amber) / STOCKS (cyan) navigation pills, sortable
  columns, live search.
- Green/red movement colours, price-change flash animation.
- Per-row freshness badges: `● LIVE`, `● RT`, `◐ DELAYED`, `⚠ STALE`,
  `✕ OFFLINE`.
- Status panel: active crypto/forex provider, backup availability, network
  state, websocket state, last sync time, failed-request counter.
- Loading toasts, manual refresh (F5), search focus (Ctrl+F), 1 Hz clock.

---

## Architecture

```
┌───────────────────────────── Qt main thread ─────────────────────────────┐
│  MainWindow / MarketPage / StatusPanel          (render only, no IO)     │
└───────────────▲─────────────────────────────────────────────────────────┘
                │ EventManager (Qt signals, queued cross-thread)
┌───────────────┴───────────────────────────── pipeline thread ────────────┐
│  Scheduler (monotonic, fixed-grid, overlap-protected)                    │
│   ├─ discovery@300s   market-cap / volume board re-rank                  │
│   ├─ crypto_rest@30s  failover chain refresh (reconciles WS gaps)        │
│   ├─ forex_rest@30s   failover chain refresh                             │
│   ├─ stocks_rest@20s  equities failover chain refresh                    │
│   ├─ aggregator@1s    consensus build → UI + SQLite + status             │
│   └─ housekeeping@60s cache eviction, DB prune, sleep detection          │
│                                                                          │
│  WebSocketManager (Binance combined miniTicker stream, auto-reconnect)   │
│  ValidationService → QuoteCache → PriceAggregator → Database             │
│  CircuitBreakers / RetryManager / ProviderHealthMonitor                  │
└──────────────────────────────────────────────────────────────────────────┘
```

Layer responsibilities are strictly separated: **providers** only translate
raw HTTP/WS payloads into `Quote` models; **services** validate, cache, and
aggregate; **core** schedules and bridges; **ui** renders. The GUI thread
never performs network IO, and the pipeline never touches widgets.

---

## Data Providers & Limitations

Every provider works **without an API key**. Honest limitations:

### Crypto

| Provider | Role | Limitations |
|---|---|---|
| **Binance** | Primary quotes + **WebSocket live stream** + discovery fallback | Spot USDT pairs only; no market cap/rank; REST weight limit 1200/min and **IP bans (HTTP 418) if exceeded** — the app self-throttles to 600/min; 24h window is rolling UTC; unavailable in some jurisdictions. |
| **CoinGecko** | Primary discovery (true market-cap ranking, cap, rank, volume) | Public tier ~5–15 req/min, **hard IP throttling (shared NAT IPs get 429 for minutes at a time)**; responses cached server-side 45–60 s, so no sub-minute ticks; aggregated volume across exchanges. |
| **Kraken** | Backup quotes (bid/ask, volume) | Public ticker has **no event timestamp** (receipt time used); ~1 req/s sustained; occasional `EService: Busy`. |
| **Coinbase** | Backup quotes (bid/ask) | One HTTP request per product (slower for boards); volume in base currency; no market cap. |

### Forex

| Provider | Role | Limitations |
|---|---|---|
| **Yahoo Finance** | Primary rates + daily change | **Unofficial endpoint**, no SLA, may change; aggregated quotes frequently 15–20 min delayed; weekend quotes freeze at Friday close; heavy scripted use can be throttled. |
| **open.er-api.com** | Backup rates | **Daily** reference refresh (~00:05 UTC); no intraday movement; no bid/ask; USD-base only (crosses via inversion lose precision). |
| **Frankfurter (ECB)** | Backup rates | ECB reference rates published **once per business day ~16:00 CET**; no weekend/holiday updates; EUR-base inversion; reference rates, not tradable prices. |

### Stocks

| Provider | Role | Limitations |
|---|---|---|
| **Yahoo Finance (query1)** | Primary global equities | Unofficial chart endpoint (no SLA); one request per symbol; US venues near real-time, non-US venues and closed markets delayed 15-20 min; prices in listing currency (HKD/JPY/EUR/GBP/CHF/KRW), not USD. |
| **Yahoo Finance (query2)** | Hot backup | Independent front-end mirror of the same data; protects against per-host throttling, not Yahoo-wide outages. |
| **Google Finance** (new in v8) | Third parallel source | Public quote-page scrape (no official API); one request per symbol; covers NASDAQ/NYSE/HKG/TYO/KRX/AMS/ETR/LON/SWX; intraday where the venue is open, last close otherwise; prices in listing currency. `WMT` currently returns a data error from Google itself; Yahoo covers it. |

### Cross-cutting truths

- **FX market hours**: the interbank market closes Friday ~21:00 UTC and
  reopens Sunday ~21:00 UTC. During that window every provider returns
  Friday's close; the app shows it as `DELAYED`/`STALE` with the real
  source timestamp instead of pretending it is live.
- **No provider guarantees sub-second publication.** A 1-second UI cycle
  re-renders what the sources last published; it cannot conjure ticks that
  do not exist. Source timestamps are always displayed.
- Crypto "market cap" only exists on the CoinGecko path; when discovery
  falls back to Binance volume ranking, rows show volume-based rank and the
  cap column shows `—`.

---

## Installation

Requirements: **Python 3.12+**

```bash
python -m venv .venv
# Windows:
.venv\Scripts\activate
# Linux/macOS:
source .venv/bin/activate

pip install -e .
# (or: pip install -r market_tracker/requirements.txt)

# for development / running the tests:
pip install -e ".[test]"
```

Linux servers/headless boxes additionally need Qt's runtime libraries
(Debian/Ubuntu):

```bash
sudo apt install libegl1 libgl1 libxkbcommon0 \
                 libdbus-1-3 libfontconfig1
```

---

## Configuration

Everything works with zero configuration. Optional overrides:

**Environment variables**

| Variable | Default | Meaning |
|---|---|---|
| `MT_COINGECKO_KEY` | *(empty)* | CoinGecko demo key — raises rate limits |
| `MT_TWELVEDATA_KEY` | *(empty)* | reserved for the optional Twelve Data forex provider |
| `MT_CRYPTO_TOP_N` | `25` | size of the dynamically discovered universe |
| `MT_LOG_LEVEL` | `INFO` | `DEBUG`/`INFO`/`WARNING`/`ERROR` |
| `MT_DATA_DIR` | `<package>/data` | DB + logs location |
| `MT_DB_PATH` | `<data>/market_tracker.db` | SQLite file |
| `MT_CONFIG` | `<package>/config.json` | JSON overrides file |

**`config.json`** (optional, next to the package):

```json
{
  "network":   { "request_timeout": 8.0, "max_retries": 3 },
  "breaker":   { "failure_threshold": 4, "recovery_timeout": 30.0 },
  "validation":{ "cross_source_tolerance_pct": 2.0, "outlier_pct_change": 25.0 },
  "scheduler": { "crypto_rest_interval": 30.0, "forex_rest_interval": 30.0 }
}
```

---

## Running

**Windows, fastest way**: double-click `run.bat` in the repository root
(it enters `market_tracker/` for you).

```bash
python -m market_tracker.main
# or, from inside market_tracker/:
python main.py
# or run a UI module directly (also works after the launch fix):
python market_tracker/ui/main_window.py
```

First launch creates `market_tracker/data/market_tracker.db` and
`market_tracker/data/market_tracker.log`.

**Diagnostics tools**

```bash
# live smoke-test every provider independently
python market_tracker/tools/provider_smoke_test.py

# long-running stability test (no UI): cadence, drift, WS, DB growth
python market_tracker/tools/soak_test.py 300
```

---

## Testing

From the **repository root** (recommended — this is what CI runs):

```bash
pip install -e ".[test]"
pytest
```

Or the classic way from any directory:

```bash
python -m pytest market_tracker/tests/ -v
```

> **VS Code / Pylance note** — if you see
> `Import "pytest" could not be resolved (reportMissingImports)`, it means
> the editor is not using the virtual environment where you installed the
> dependencies. Fix: `Ctrl+Shift+P` → *Python: Select Interpreter* → pick
> the `.venv` you created in Installation (or run
> `pip install pytest pytest-asyncio` into the interpreter Pylance shows in
> the status bar). The tests themselves never import the UI or Qt, so the
> suite also runs headless on CI.

The suite (88 tests) covers:

- `test_providers.py` — payload parsing for all 9 providers (incl. the
  Yahoo stocks pair and its query2 mirror), error mapping,
  **failover chains** (primary→backup→all-fail), simulated HTTP 429,
  simulated downtime, circuit-breaker opening under repeated failure.
- `test_validation.py` — zero/negative/absurd prices, crossed and absurd
  bid-ask spreads, staleness windows, delayed-source grace (weekend ECB),
  outlier flagging, cross-source discrepancy detection, batch separation.
- `test_recovery.py` — retry/backoff/jitter/Retry-After, auth fail-fast,
  full circuit-breaker lifecycle (incl. half-open probe failure), cache TTL
  eviction, multi-provider cache semantics, DB history + snapshot round-trip
  + pruning, aggregator median robustness, best bid/ask, stale/offline
  degradation with last-known-good prices.
- `test_scheduler.py` — 1 Hz cadence accuracy, **no drift accumulation**,
  overlap protection (slow job never re-entered), job exception isolation,
  error backoff growth, catch-up suppression after simulated system sleep.

Live verification performed during development (documented in the delivery
notes): every provider hit independently, real CoinGecko 429 → automatic
Binance discovery failover, real Binance 418 weight-ban → app now
self-throttles, websocket reconnect logic, 300 s soak with 0 ms drift and
0 % failed requests.

---

## How the 1-Second Cycle Works

1. `Scheduler` runs on the pipeline thread using `time.monotonic()` with
   **fixed-grid scheduling**: job *n* fires at `t0 + n·interval`, so slow
   cycles never accumulate drift (measured: 0.0 ms drift over 300 s).
2. The aggregator job runs every 1 s: it rebuilds the consensus view from
   the cache (where websocket ticks land within milliseconds of arrival)
   and emits it toward the UI via a queued Qt signal.
3. The Qt main thread only renders; a 1 s `PreciseTimer` heartbeat updates
   the clock/"as-of" labels.
4. REST providers poll at their own, polite cadence (30 s) — **the UI
   refreshes every second even when the source updates slower**, and the
   row badge always shows which is which (`LIVE` = websocket sub-second,
   `RT` = REST seconds-level, `DELAYED` = documented delay).
5. Overlap protection: a job still running when its next slot is due is
   skipped, never queued twice.

No prices are ever synthesized between source updates.

---

## Reliability Design

| Failure | Behaviour |
|---|---|
| Timeout / connection refused / DNS | typed exception → retry with backoff+jitter → next provider in chain |
| HTTP 429 | `RateLimitError`, `Retry-After` honoured, breaker failure counted |
| Binance weight overflow (418 risk) | local weight budget (600/min) skips Binance before the exchange does |
| HTTP 401/403 | `AuthError` — **not retried**, chain moves on immediately |
| Invalid JSON / missing fields | `MalformedDataError`, entry skipped, batch continues |
| Repeated provider failure | circuit **opens**, calls suppressed 30 s, half-open probe, auto-close |
| All providers fail | last validated cache values shown, `STALE`/`OFFLINE` badge + original timestamp; toast notification |
| WebSocket drop | exponential-backoff reconnect with jitter, state surfaced in status panel |
| System sleep / network change | catch-up suppression (no request burst); snapshot restore on restart |
| Database error | one-shot self-heal (engine rebuild); persistence degrades, UI unaffected |
| Any unexpected job error | isolated to the job, logged with category, error backoff |

Structured logs (console + rotating file) carry timestamp, level, provider,
category, state, and retry attempt.

---

## Packaging as a Windows Executable

**1. Install PyInstaller** (in the project venv):

```bat
pip install pyinstaller
```

**2. Build with the included spec** (from the `market_tracker/` folder):

```bat
pyinstaller --clean MarketTracker.spec
```

or the equivalent one-liner:

```bat
pyinstaller --noconfirm --windowed --name MarketTracker ^
    --collect-all PySide6 ^
    --hidden-import market_tracker.core.application ^
    --paths . main.py
```

**3. Output**: `dist\MarketTracker\` (folder app — keeps the SQLite DB and
logs writable next to the exe) or a single `dist\MarketTracker.exe` if you
add `--onefile` (then set `MT_DATA_DIR` to a writable path like
`%LOCALAPPDATA%\MarketTracker`, because a onefile bundle's install dir may
be read-only).

**4. Runtime notes**
- `market_tracker/data/` (DB + logs) is created on first run beside the
  executable; ship `config.json` in the same folder for custom settings.
- PySide6 needs `--collect-all PySide6` for the platform plugins.
- Windows Defender SmartScreen may prompt on unsigned executables —
  sign the binary for distribution (`signtool sign /fd SHA256 ...`).
- To slim the binary, prune unused Qt modules (WebEngine etc.) with
  `--exclude-module PySide6.QtWebEngineCore` (already done in the spec).

---

## Project Structure

```
market_tracker/
├── main.py                     
├── config.py                 
├── models.py                   
├── requirements.txt
├── assets/
│   └── background.jpg        
├── core/
│   ├── application.py       
│   ├── scheduler.py          
│   ├── event_manager.py     
│   └── data_manager.py    
├── providers/
│   ├── base_provider.py      
│   ├── crypto_providers.py     
│   ├── forex_providers.py      
│   ├── stock_providers.py     
│   ├── google_finance.py     
│   ├── discovery.py           
│   └── websocket_manager.py    
├── infrastructure/
│   ├── cache.py               
│   ├── database.py             
│   ├── connection_manager.py
│   ├── retry_manager.py        
│   ├── circuit_breaker.py     
│   └── health_monitor.py      
├── services/
│   ├── crypto_service.py       
│   ├── forex_service.py    
│   ├── stocks_service.py    
│   ├── validation_service.py  
│   └── price_aggregator.py 
├── ui/
│   ├── main_window.py        
│   ├── all_markets_page.py   
│   ├── market_window.py     
│   ├── market_page.py         
│   ├── crypto_forex_pages.py 
│   ├── components.py       
│   └── theme.py               
├── utils/
│   ├── logger.py              
│   ├── exceptions.py          
│   └── validators.py         
├── tools/
│   ├── provider_smoke_test.py 
│   └── soak_test.py            
└── tests/
    ├── conftest.py            
    ├── test_providers.py
    ├── test_validation.py
    ├── test_recovery.py
    └── test_scheduler.py
```

---

## Known Limitations

- **Accuracy is bounded by the sources.** Cross-provider disagreement is
  surfaced (median + spread %), never silently resolved; nothing here can
  make a 15-minute-delayed Yahoo quote real-time.
- **CoinGecko public API is IP-throttled hard**; on shared IPs the app will
  spend most of its time on the Binance volume-ranked discovery path, which
  ranks by volume instead of market cap (cap/rank columns degrade to `—`).
- **Binance availability varies by jurisdiction**; if both Binance and
  CoinGecko are unreachable, crypto discovery falls back to the curated
  10-coin floor set.
- The **Binance websocket** covers USDT spot pairs; forex and stocks have
  no keyless websocket, so those rows are at best `RT` (REST seconds-level)
  or `DELAYED`.
- **Stock prices are in the listing currency** (Tencent in HKD, Toyota in
  JPY, ASML in EUR…) — the CCY column states each one; no FX conversion is
  applied because converting would just couple the row to a second feed's
  accuracy.
- **Non-US venues** (HKG/JPX/KSC/AMS/GER/LSE/SWX) publish delayed quotes on
  Yahoo's public endpoints; the DATA column says `◐ DELAYED` for them.
- **JOD** (Jordanian Dinar) is a pegged currency; all providers will show
  ~0.709 with essentially zero daily movement — that is the market, not a
  bug.
- The app is a **monitor**, not a trading terminal: no orders, no
  authentication with exchanges, no portfolio tracking.

**Made By z01309z**

License

Copyright © 2026 z01309z. All rights reserved.

This project is developed and owned by z01309z.

Unauthorized copying, modification, redistribution, or commercial use of this project is prohibited without prior permission from the author.

Developer: z01309z
