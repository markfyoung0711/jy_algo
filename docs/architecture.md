# Data Pipeline Architecture

This is a classic ETL (Extract, Transform, Load) pipeline. The design decisions here matter because the raw archive is the foundation everything else is derived from — get the capture layer right first.

---

## The Three Stages

```
[APIs / Files]
      │
      ▼
[Raw Capture]  ──────────────────────────────────► [Raw Archive]
 Poll on schedule                                   (immutable,
 Store full response                                append-only)
 Tag with capture time
      │
      ▼
[Normalization]
 Align timestamps
 Handle missing values (tag, don't silently fill)
 Convert units — keep native AND z-scored
 Flag outliers (3σ from 90-day rolling mean)
 Attach source metadata
      │
      ▼
[Normalized Store]  ──────────────────────────────► [Analysis]
 One row per metric per day                          Correlation
 z-scored copy alongside native units                Backtesting
                                                     Dashboards
```

---

## Stage 1: Raw Capture (Extract)

**The raw archive is sacred.** Every API response is stored exactly as received — no transformation, no interpretation. If normalization logic has a bug or you want a new derived metric, you re-derive the normalized store from the raw archive. Never modify raw captures.

### Per-Feed Capture Details

| Feed | Source | Frequency | Raw content to store |
|---|---|---|---|
| **Crypto** | ccxt.pro WebSocket (Binance) | 1-min OHLCV | Full tick or OHLCV bar + exchange + timestamp |
| **VIX** | CBOE CDN (`cdn.cboe.com`) | Every second (market hours only) | Value, bid/ask, change%, source URL; only during 9:30am–4pm ET |
| **5Y Breakeven** | FRED API (`T5YIE`) | Daily after market close | Full JSON response including vintage dates (revision history — keep these for backtesting) |
| **GPR Index** | policyuncertainty.com CSV | Monthly | Raw file + capture timestamp; no streaming API exists |
| **Coinglass** | REST API | Every minute | Full JSON: per-exchange funding rates, open interest, liquidation volumes (1h/4h/24h windows) — do not aggregate, keep all exchange granularity |
| **Glassnode** | REST API | On-demand (hourly or daily) | Full JSON per metric: SOPR, MVRV, exchange net position change; updates ~every 10 min as blocks confirm |

### Raw Storage Path Convention

```
data/
  {feed_name}/
    {YYYYMMDD}/
      {unix_ms}.parquet        # one file per capture event
```

Examples:
```
data/vix/20260311/1741651201000.parquet
data/coinglass/20260311/1741651260000.parquet
data/gpr/202603/20260301.parquet           # monthly: YYYYMM directory
data/breakeven/20260311/1741694400000.parquet
```

### Redis Streams as the Capture Buffer

Producers publish to `market:{feed_name}` streams. Consumer groups allow the Parquet writer and DuckDB writer to consume independently without coupling. Redis `MAXLEN ~604800` (≈7 days at 1/sec) bounds memory. `noeviction` policy is critical — Redis must not silently drop messages.

```
market:crypto      market:vix      market:breakeven
market:gpr         market:coinglass   market:glassnode
market:dlq         ← dead letter for failed writes
```

**VIX note:** Only poll during US market hours (9:30am–4pm ET on trading days). No point running the feed at 2am.

---

## Stage 2: Normalization (Transform)

### Canonical Message Schema (raw layer)

```python
class MarketMessage(TypedDict):
    feed:        str   # "crypto" | "vix" | "breakeven" | "gpr" | "coinglass" | "glassnode"
    symbol:      str   # "BTC/USDT" | "^VIX" | "T5YIE" | "GPR" | "BTC_funding_rate"
    ts:          int   # unix ms UTC — event time (what period this represents)
    ts_ingested: int   # unix ms UTC — when your system captured it
    value:       float # primary numeric value in native units
    payload:     dict  # feed-specific fields (OHLCV bars, per-exchange breakdowns, etc.)
    source:      str   # "ccxt:binance" | "cboe" | "fred" | "coinglass" | "glassnode"
```

### Normalized Series Schema

```python
# One row per metric per timestamp, after alignment and cleaning
class NormalizedRecord(TypedDict):
    ts:              datetime  # UTC, aligned to common backbone (daily UTC close)
    feed:            str
    symbol:          str
    value_native:    float     # original units (VIX points, %, index, bps, ratio)
    value_zscore:    float     # z-scored vs 90-day rolling window
    unit:            str       # "vix_pts" | "pct" | "index" | "bps_per_8h" | "ratio"
    is_outlier:      bool      # True if |zscore| > 3
    is_filled:       bool      # True if value was forward-filled (synthetic, not observed)
    source:          str
    capture_ts:      datetime  # when your system fetched it
    data_version:    str | None  # FRED vintage date or equivalent if source supports it
```

### Key Normalization Challenges

**Timestamp alignment** is the hardest problem. VIX has second-level granularity during market hours. FRED gives daily values. GPR gives monthly values. Coinglass gives rolling windows. Glassnode anchors to UTC midnight daily.

Common denominator: **daily UTC close**. For intraday sources (VIX, Coinglass), use the last reading of the day. For monthly sources (GPR), forward-fill across all days in the month.

**Missing value policy** — never silently fill gaps:
- Forward-fill (carry last known value) → set `is_filled = True`
- Mark null and let analysis handle it
- Tag the gap so you know it's synthetic

**Unit normalization** — all sources are on different scales:
| Feed | Native unit | Example value |
|---|---|---|
| VIX | Volatility index pts | 18.5 |
| 5Y Breakeven | % inflation rate | 2.34 |
| GPR | Index (100 = historical avg) | 142 |
| Coinglass funding | Basis points per 8h | 1.2 bps |
| Glassnode MVRV | Ratio | 2.1 |

Store both native units and z-scored. Never overwrite native units.

**Outlier flagging** — tag records where value is >3σ from 90-day rolling mean with `is_outlier = True`. Never remove them from the store. Include or exclude deliberately in analysis.

---

## Stage 3: Storage (Load)

Two separate stores — never mix them:

| Store | Purpose | Format | Mutation policy |
|---|---|---|---|
| **Raw archive** | Faithful capture of every API response | Parquet (`data/{feed}/{YYYYMMDD}/{ts_ms}.parquet`) | Append-only, never modified |
| **Normalized store** | Clean, aligned, analysis-ready time series | DuckDB + Parquet (`data/normalized/{YYYYMMDD}/`) | Rebuilt from raw on schema changes |

DuckDB queries both stores in-place via `read_parquet('data/vix/20260311/*.parquet')` — no import step needed.

For production scale: TimescaleDB as a third sink via a second consumer group on Redis Streams.

---

## System Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    PRODUCER LAYER                        │
│  crypto_feed  vix_feed  breakeven_feed  gpr_feed         │
│  coinglass_feed  glassnode_feed                          │
│  (each runs as independent process/container)            │
└──────────────────────────┬───────────────────────────────┘
                           │ XADD
                           ▼
┌──────────────────────────────────────────────────────────┐
│          Redis Streams  (market:{feed_name})             │
│          Consumer groups: raw-writer, norm-writer        │
│          MAXLEN ~604800 (7-day rolling window)           │
└──────────┬───────────────────────────┬───────────────────┘
           │ XREADGROUP                │ XREADGROUP
           ▼                           ▼
  ┌─────────────────┐        ┌──────────────────────┐
  │  parquet_writer │        │  normalizer +        │
  │  (raw archive)  │        │  duckdb_writer       │
  │                 │        │                      │
  │  data/{feed}/   │        │  data/normalized/    │
  │  {YYYYMMDD}/    │        │  jy_algo.duckdb      │
  │  {ts_ms}.parquet│        │                      │
  └─────────────────┘        └──────────────────────┘
```

---

## Module Structure

```
jy_algo/
├── pyproject.toml              # redis>=5, pyarrow, duckdb, pydantic-settings
├── docker-compose.yml          # Redis only (redis:7-alpine, appendonly yes, noeviction)
├── Makefile
├── data/                       # gitignored
│   ├── {feed}/{YYYYMMDD}/      # raw parquet by capture event
│   ├── normalized/             # normalized parquet
│   └── redis/                  # Redis AOF persistence
├── rs/                         # Rust crate — Phase 3 L2 order book normalizer
└── jy_algo/
    ├── core/
    │   ├── bus.py              # Redis Streams: async publish() / consume()
    │   ├── schema.py           # MarketMessage + NormalizedRecord + PyArrow schemas
    │   └── settings.py         # pydantic-settings reading .env
    ├── feeds/
    │   ├── base.py             # BaseFeed ABC: retry loop, exponential backoff
    │   ├── crypto_feed.py      # ccxt.pro WebSocket, BTC 1-min OHLCV
    │   ├── vix_feed.py         # CBOE CDN, 1-sec poll, market hours only
    │   ├── breakeven_feed.py   # FRED T5YIE, daily post-close, store vintage dates
    │   ├── gpr_feed.py         # policyuncertainty.com CSV, monthly HTTP GET
    │   ├── coinglass_feed.py   # REST 1-min, all exchanges, no pre-aggregation
    │   └── glassnode_feed.py   # REST on-demand, full metric catalogue, 429 backoff
    ├── writers/
    │   ├── parquet_writer.py   # raw archive consumer
    │   └── duckdb_writer.py    # normalized store consumer
    └── scripts/
        ├── run_feed.py         # python -m jy_algo.scripts.run_feed --feed vix
        ├── backfill.py         # one-time historical load per feed
        └── query.py            # python -m jy_algo.scripts.query --feed vix --date 20260311
```

---

## Phased Implementation

### Phase 1 — Foundation + Crypto + VIX (Weeks 1–2)
1. `core/bus.py`, `core/schema.py`, `core/settings.py`
2. `docker-compose.yml` (Redis only)
3. `feeds/base.py` with retry/backoff loop
4. `feeds/crypto_feed.py` — ccxt.pro WebSocket
5. `feeds/vix_feed.py` — CBOE CDN, market-hours guard
6. `writers/parquet_writer.py` — raw archive
7. Integration test: feed → Redis → Parquet → DuckDB read-back

### Phase 2 — Remaining Feeds + Normalization (Weeks 3–4)
1. `feeds/breakeven_feed.py`, `feeds/gpr_feed.py`, `feeds/coinglass_feed.py`, `feeds/glassnode_feed.py`
2. `writers/duckdb_writer.py` with normalization logic (timestamp alignment, z-score, outlier flagging)
3. `scripts/backfill.py` — historical load (FRED years back, GPR to 1985, Glassnode history)
4. `scripts/query.py` — cross-feed SQL via DuckDB

### Phase 3 — Operationalization (Weeks 5–6)
1. Health check HTTP endpoints on each feed
2. Dead letter stream (`market:dlq`) with alerting
3. `Makefile` targets: `make ingest-all`, `make query feed=vix date=20260311`
4. Rust tick normalizer for L2 order book data (if L2 feeds added)
5. Optional: TimescaleDB writer as third consumer group

---

## Design Decisions to Make Before Building

**History backfill window** — FRED and Glassnode have years of history via API. GPR has monthly data to 1985. Coinglass free tier has limited history. Decide on a backfill window and run `scripts/backfill.py` before starting live capture.

**Failure handling** — distinguish "source unavailable" (retry with backoff) from "bad request" (log and skip). A failed poll must never leave a silent gap — log it explicitly with `feed`, `ts`, and `error`.

**Coinglass WebSocket** — paid tier supports WebSocket push for near-realtime liquidation data. If liquidation latency matters, replace REST polling with a persistent WebSocket connection in Phase 3.

**Correlation cadence** — with GPR monthly and VIX by-the-second, "correlation" requires a common time axis. Standard approach: align to daily, compute rolling correlations over 30-, 60-, and 90-day windows, and track how they shift around events (e.g., Iran strikes in April 2024).

---

## API Notes

| Feed | Free Tier | Key Constraint |
|---|---|---|
| Crypto (Binance) | Unlimited via WebSocket | Use ccxt.pro — avoids REST rate limits entirely |
| VIX (CBOE CDN) | Unlimited | Delayed quote; real-time requires ~$50/mo subscription |
| Breakeven (FRED) | 120 req/min | Free API key; daily poll is trivial |
| GPR | No API | Monthly HTTP GET of published CSV file |
| Coinglass | Free: funding + OI | API key required (free registration); WebSocket on paid tier |
| Glassnode | Free: ~30 metrics, daily only | Paid (~$29/mo) for hourly + full catalogue; design for graceful 403 degradation |
