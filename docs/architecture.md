# Data Pipeline Architecture

This is a classic ETL (Extract, Transform, Load) pipeline. The design decisions here matter because the raw archive is the foundation everything else is derived from — get the capture layer right first.

---

## Data Zones

Data moves forward through three zones. It never moves backward. Each zone transition is gated by a validation pass.

```
[APIs / Files]
      │
      ▼
┌──────────────────────────────────────────────────────┐
│  raw/<capturename>/{YYYYMMDD}/{epoch_s}.<ext>    │  Exact bytes from source.
│  (append-only, never modified)                       │  Native format: .json, .csv,
│                                                      │  .bin, .pb, .xml — whatever
│                                                      │  the source delivers. No conversion.
└───────────────────┬──────────────────────────────────┘
                    │
          field/format validation
          (per-feed validator: parse raw bytes,
           check types, ranges, required fields,
           timestamp sanity)
                    │
            pass ──┘──── fail → validation/<capturename>/failures/
                    │
                    ▼  convert to Parquet here
┌─────────────────────────────────────────────┐
│  staged/<capturename>/{YYYYMMDD}/       │  Parquet. First structured form.
│  {epoch_s}.parquet                          │  Schema enforced. Safe to process.
└───────────────────┬─────────────────────────┘
                    │
          business rule validation
          (cross-feed consistency, outlier flagging,
           timestamp alignment, unit sanity,
           duplicate detection)
                    │
            pass ──┘──── fail → validation/<capturename>/rejected/
                    │
                    ▼  load into Iceberg
┌─────────────────────────────────────────────┐
│  warehouse/<capturename>/               │  Apache Iceberg tables.
│  (Iceberg table, Parquet + snapshots)       │  Bi-temporal: snapshot = transaction time,
│                                             │  valid_from/valid_to = valid time.
│                                             │  Normalized, z-scored, outlier-tagged.
└─────────────────────────────────────────────┘
      │
      ▼
[Consumed by trading/analysis repo]
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
raw/
  {capturename}/
    {YYYYMMDD}/
      {unix_epoch_s}.{ext}     # native format — exact bytes from source
```

The file extension reflects the actual format delivered by the source:

| Source | Format | Example path |
|---|---|---|
| CBOE VIX | JSON | `raw/cboe_vix/20260311/1741651201.json` |
| FRED breakeven | JSON | `raw/fred_breakeven/20260311/1741694400.json` |
| GPR Index | CSV | `raw/gpr_index/20260301/1741651200.csv` |
| Coinglass funding rates | JSON | `raw/coinglass_funding_rates/20260311/1741651260.json` |
| Coinglass open interest | JSON | `raw/coinglass_open_interest/20260311/1741651260.json` |
| Binance OHLCV | Binary | `raw/binance_btcusdt_ohlcv/20260311/1741651260.bin` |

### Redis Streams as the Capture Buffer

Producers publish to `market:{capturename}` streams. Consumer groups allow the Parquet writer and DuckDB writer to consume independently without coupling. Redis `MAXLEN ~604800` (≈7 days at 1/sec) bounds memory. `noeviction` policy is critical — Redis must not silently drop messages.

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
| **raw/** | Exact bytes from source | Native format (`raw/{capturename}/{YYYYMMDD}/{unix_epoch_s}.{ext}`) | Append-only, never modified |
| **staged/** | Passed field/format validation | Parquet (`staged/{capturename}/{YYYYMMDD}/{unix_epoch_s}.parquet`) | Written by validator, never modified |
| **warehouse/** | Passed all business rule validation | Apache Iceberg (`warehouse/{capturename}/`) | Bi-temporal, normalized, z-scored |

All three zones use the same Parquet file convention and can be queried by any compatible tool (DuckDB, Pandas, Polars, Spark, etc.). The database backend for the warehouse is TBD — Parquet is the portable interchange format regardless of what sits on top.

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
│          Redis Streams  (market:{capturename})             │
│          Consumer groups: raw-writer, norm-writer        │
│          MAXLEN ~604800 (7-day rolling window)           │
└──────────┬───────────────────────────┬───────────────────┘
           │ XREADGROUP                │ XREADGROUP
           ▼                           ▼
  ┌─────────────────┐        ┌──────────────────────┐
  │  parquet_writer │        │  normalizer +        │
  │  → raw/         │        │  duckdb_writer       │
  │                 │        │  → warehouse/        │
  │  raw/{capturename}/    │        │  jy_algo.duckdb      │
  │  {YYYYMMDD}/    │        │                      │
  │  {epoch_s}.parq │        │                      │
  └─────────────────┘        └──────────────────────┘
```

---

## Module Structure

```
jy_algo/
├── pyproject.toml              # redis>=5, pyarrow, duckdb, pydantic-settings
├── docker-compose.yml          # Redis only (redis:7-alpine, appendonly yes, noeviction)
├── Makefile
├── raw/                        # gitignored — zone 1: immutable captures
├── staged/                     # gitignored — zone 2: field-validated
├── warehouse/                  # gitignored — zone 3: business-rule validated, normalized
├── validation/                 # gitignored — validation reports and failure records
├── reference/                  # gitignored — symbol reference snapshots
├── redis-data/                 # gitignored — Redis AOF persistence
├── rs/                         # Rust crate — Phase 3 L2 order book normalizer
└── jy_algo/
    ├── core/
    │   ├── bus.py              # Redis Streams: async publish() / consume()
    │   ├── schema.py           # MarketMessage + NormalizedRecord + PyArrow schemas
    │   └── settings.py         # pydantic-settings reading .env
    ├── feeds/
    │   ├── base.py             # BaseFeed ABC: retry loop, exponential backoff, publish()
    │   ├── base_rest.py        # RestFeed(BaseFeed): poll interval, HTTP session, rate limit handling
    │   ├── base_websocket.py   # WebSocketFeed(BaseFeed): connection lifecycle, reconnect, heartbeat
    │   ├── base_file.py        # FileFeed(BaseFeed): schedule check, HTTP GET, file hash dedup
    │   │
    │   ├── crypto_feed.py      # CryptoFeed(WebSocketFeed): ccxt.pro WS, BTC 1-min OHLCV
    │   ├── vix_feed.py         # VixFeed(RestFeed): CBOE CDN, 1-sec poll, market-hours guard
    │   ├── breakeven_feed.py   # BreakevenFeed(RestFeed): FRED T5YIE, daily post-close, vintage dates
    │   ├── gpr_feed.py         # GprFeed(FileFeed): policyuncertainty.com CSV, monthly, hash dedup
    │   ├── coinglass_feed.py   # CoinglassFeed(RestFeed): 1-min poll, all exchanges, no pre-aggregation
    │   └── glassnode_feed.py   # GlassnodeFeed(RestFeed): on-demand, full metric catalogue, 429 backoff
    ├── writers/
    │   └── parquet_writer.py   # raw zone writer (consumer of Redis Streams)
    ├── validators/
    │   ├── base.py             # BaseValidator ABC, ValidationReport, ValidationFailure
    │   ├── crypto_validator.py
    │   ├── vix_validator.py
    │   ├── breakeven_validator.py
    │   ├── gpr_validator.py
    │   ├── coinglass_validator.py
    │   └── glassnode_validator.py
    ├── reference/
    │   ├── symbol_registry.py  # SymbolRegistry, SymbolRecord — canonical cross-exchange map
    │   └── exchange_adapters/
    │       ├── base.py         # normalize ccxt market dict → SymbolRecord
    │       ├── binance.py      # Binance-specific quirks
    │       ├── okx.py
    │       ├── bybit.py
    │       └── bitmex.py
    └── scripts/
        ├── run_feed.py         # python -m jy_algo.scripts.run_feed --feed vix
        ├── backfill.py         # one-time historical load per feed
        ├── validate.py         # python -m jy_algo.scripts.validate --feed vix --date 20260311
        ├── refresh_symbols.py  # python -m jy_algo.scripts.refresh_symbols --exchange all
        └── query.py            # python -m jy_algo.scripts.query --feed vix --date 20260311
```

Directory layout:

```
raw/
  {capturename}/{YYYYMMDD}/{unix_epoch_s}.parquet    # zone 1: immutable captures

staged/
  {capturename}/{YYYYMMDD}/{unix_epoch_s}.parquet    # zone 2: passed field/format validation

warehouse/
  {capturename}/{YYYYMMDD}/                          # zone 3: passed all validations, normalized

validation/
  {capturename}/{YYYYMMDD}/report.parquet            # validation run results
  {capturename}/failures/{YYYYMMDD}/                 # records that failed field/format (raw→staged)
  {capturename}/rejected/{YYYYMMDD}/                 # records that failed business rules (staged→warehouse)

reference/
  symbols/{exchange}/{YYYYMMDD}.parquet       # symbol snapshots per exchange
  symbols/canonical_map.parquet               # latest cross-exchange map
```

---

## Downloader Class Hierarchy

Each feed is its own independent downloader class. They share infrastructure via inheritance but diverge freely — a feed can override any method without affecting others. The hierarchy has three levels:

```
BaseFeed (ABC)
├── RestFeed
│   ├── VixFeed
│   ├── BreakevenFeed
│   ├── CoinglassFeed
│   └── GlassnodeFeed
├── WebSocketFeed
│   └── CryptoFeed
└── FileFeed
    └── GprFeed
```

### `BaseFeed` — shared contract
Owns the run loop, retry logic with exponential backoff, Redis publish, structured error logging, and the `MarketMessage` envelope. Every feed gets these for free. **Feeds are responsible only for raw capture — no validation logic lives here.**

```python
class BaseFeed(ABC):
    async def run(self) -> None: ...         # entry point; calls fetch() in a retry loop
    async def publish(self, msg) -> None: ...# XADD to Redis Streams
    @abstractmethod
    async def fetch(self) -> list[MarketMessage]: ...  # each feed implements this
    def _backoff(self, attempt: int) -> float: ...     # 5s base, 5min cap
```

### `RestFeed(BaseFeed)` — polling HTTP sources
Adds: configurable `poll_interval`, shared `aiohttp.ClientSession`, rate limit handling (respects `Retry-After` headers), and a `should_poll()` hook for time-window guards (e.g. market hours).

```python
class RestFeed(BaseFeed):
    poll_interval: int           # seconds between fetches
    async def should_poll(self) -> bool: ... # override to gate by time/calendar
    async def get(self, url, **params): ...  # shared session with timeout + retry
```

**Per-feed specializations:**
- `VixFeed` — overrides `should_poll()` to return `False` outside 9:30am–4pm ET on trading days
- `BreakevenFeed` — overrides `should_poll()` for once-daily post-close cadence; `fetch()` preserves FRED vintage dates in payload
- `CoinglassFeed` — `fetch()` returns one `MarketMessage` per exchange per metric (no pre-aggregation); handles Coinglass pagination
- `GlassnodeFeed` — `fetch()` iterates the full metric catalogue; overrides `_backoff()` to honor `Retry-After` on 429

### `WebSocketFeed(BaseFeed)` — persistent connection sources
Adds: connection lifecycle (connect/disconnect/reconnect), heartbeat, message buffering during reconnect, and backpressure handling.

```python
class WebSocketFeed(BaseFeed):
    async def connect(self) -> None: ...
    async def on_message(self, raw) -> list[MarketMessage]: ...  # parse ws frame
    async def on_disconnect(self) -> None: ...  # triggers reconnect with backoff
```

**Per-feed specializations:**
- `CryptoFeed` — uses ccxt.pro's async WebSocket abstraction; subscribes to multiple symbols and timeframes; buffers incomplete OHLCV bars until the bar closes

### `FileFeed(BaseFeed)` — scheduled file downloads
Adds: schedule checking (run at most once per period), HTTP GET of a remote file, SHA-256 hash deduplication (don't reprocess a file that hasn't changed), and raw file storage alongside the parsed message.

```python
class FileFeed(BaseFeed):
    check_interval: int          # how often to check for a new file (seconds)
    async def is_new(self, url) -> bool: ...   # HEAD request + hash comparison
    async def download(self, url) -> bytes: ...
```

**Per-feed specializations:**
- `GprFeed` — overrides `check_interval` to daily (the monthly CSV rarely changes mid-month); `fetch()` parses CSV, emits one `MarketMessage` per month row; stores raw CSV bytes in payload for archive

---

## Validation Subsystem

Validation is **orthogonal to capture**. It is a separate service that reads from raw data locations and produces validation reports. It has no coupling to the feed classes — it can be run independently, pointed at any `data/{capturename}/{YYYYMMDD}/` path, and re-run at any time without touching the capture pipeline.

```
raw/{capturename}/{YYYYMMDD}/      ← written by feeds (zone 1)
         │
         │  pointed at by
         ▼
  validators/{feed}.py      ← per-feed validator, runs independently
         │
         ├── pass → staged/{capturename}/{YYYYMMDD}/
         └── fail → validation/{capturename}/failures/{YYYYMMDD}/

staged/{capturename}/{YYYYMMDD}/   ← zone 2; validator runs again for business rules
         │
         ├── pass → warehouse/{capturename}/{YYYYMMDD}/
         └── fail → validation/{capturename}/rejected/{YYYYMMDD}/

validation/{capturename}/{YYYYMMDD}/report.parquet  ← one row per file checked
```

### Class Design

```python
class BaseValidator(ABC):
    feed: str

    def run(self, path: str) -> ValidationReport: ...
    # reads all parquet files under path, calls check() on each record

    @abstractmethod
    def check(self, record: dict) -> list[ValidationFailure]: ...
    # returns empty list if valid; list of failures otherwise
```

Each feed has its own validator that knows what fields to check:

```python
class CryptoValidator(BaseValidator):
    # low <= open, close, high; volume >= 0; ts within expected interval; no nulls

class VixValidator(BaseValidator):
    # value in range 5–150; ts is today; reject stale (ts unchanged from prior record)

class BreakevenValidator(BaseValidator):
    # value in range −2% to 10%; vintage_date present and parseable; series_id == "T5YIE"

class GprValidator(BaseValidator):
    # expected columns present; value > 0; period is a valid month; no duplicate months in one file

class CoinglassValidator(BaseValidator):
    # funding rate in range −1% to 1% per 8h; exchange in known set; open_interest >= 0

class GlassnodeValidator(BaseValidator):
    # metric name matches request; timestamp aligns to declared resolution; null handling per metric
```

### `ValidationReport` and `ValidationFailure`

```python
class ValidationFailure(TypedDict):
    field:    str       # which field failed
    rule:     str       # human-readable rule that was violated
    value:    Any       # the actual value that failed
    severity: str       # "error" | "warning"

class ValidationReport(TypedDict):
    feed:          str
    path:          str          # the data path that was validated
    run_ts:        int          # when validation ran (unix ms)
    total:         int          # records checked
    valid:         int
    invalid:       int
    failures:      list[ValidationFailure]
```

Reports are written to `validation/{capturename}/{YYYYMMDD}/report.parquet` and queryable alongside zone data.

### Module Location

```
jy_algo/
└── validators/
    ├── base.py              # BaseValidator ABC, ValidationReport, ValidationFailure
    ├── crypto_validator.py
    ├── vix_validator.py
    ├── breakeven_validator.py
    ├── gpr_validator.py
    ├── coinglass_validator.py
    └── glassnode_validator.py
```

Run independently:
```bash
python -m jy_algo.scripts.validate --feed vix --date 20260311
python -m jy_algo.scripts.validate --feed coinglass --date 20260311 --path raw/coinglass/20260311
```

---

## Symbol Reference Data Subsystem

Crypto exchanges each use different naming conventions for the same instrument (`BTC/USDT` on Binance, `BTC-USDT` on OKX, `XBTUSD` on BitMEX). The symbol reference subsystem downloads and maintains a canonical symbol map per exchange, used by `CryptoFeed` and `CoinglassFeed` to resolve instruments at startup and during subscription changes.

This is **reference data**, not time-series data — it has a different lifecycle (refresh daily or on-demand, not streamed) and lives in its own storage separate from the capture pipeline.

### Storage

```
data/
  reference/
    symbols/
      {exchange}/
        {YYYYMMDD}.parquet     # full symbol list as of that date
      canonical_map.parquet    # latest resolved cross-exchange symbol map
```

### Class Design

```python
class SymbolRegistry:
    """
    Downloads and caches the full instrument list per exchange.
    Feeds call registry.resolve(canonical, exchange) to get the
    exchange-native symbol string before subscribing.
    """

    async def refresh(self, exchange: str) -> None: ...
    # fetches exchange.load_markets() via ccxt, writes to reference/symbols/{exchange}/{date}.parquet

    def resolve(self, canonical: str, exchange: str) -> str: ...
    # e.g. resolve("BTC/USDT", "okx") → "BTC-USDT"
    # raises UnknownSymbolError if not found

    def list_exchanges(self) -> list[str]: ...
    def list_symbols(self, exchange: str) -> list[SymbolRecord]: ...
```

```python
class SymbolRecord(TypedDict):
    canonical:     str    # normalized form: "BTC/USDT"
    exchange:      str    # "binance" | "okx" | "bybit" | "bitmex" | ...
    native:        str    # exchange's own symbol string
    base:          str    # "BTC"
    quote:         str    # "USDT"
    active:        bool   # whether the exchange currently lists this instrument
    type:          str    # "spot" | "future" | "perp" | "option"
    settle:        str | None   # settlement currency for derivatives
    expiry:        str | None   # ISO date for dated futures; None for perps
    fetched_date:  str    # YYYYMMDD when this record was downloaded
```

### Refresh Strategy

- **At startup**: `CryptoFeed` and `CoinglassFeed` call `registry.refresh(exchange)` if the cached file is older than 24 hours, then call `registry.resolve()` to build their subscription list.
- **Daily**: a scheduled `scripts/refresh_symbols.py` run refreshes all configured exchanges and writes a new dated snapshot. Historical snapshots are preserved — instruments get listed and delisted, and you may need to know what was tradeable on a given date for backtesting.
- **On-demand**: `python -m jy_algo.scripts.refresh_symbols --exchange binance`

### Module Location

```
jy_algo/
├── reference/
│   ├── __init__.py
│   ├── symbol_registry.py     # SymbolRegistry, SymbolRecord
│   └── exchange_adapters/
│       ├── base.py            # BaseExchangeAdapter: normalize ccxt market dict → SymbolRecord
│       ├── binance.py         # BinanceAdapter: handle Binance-specific quirks
│       ├── okx.py
│       ├── bybit.py
│       └── bitmex.py
└── scripts/
    └── refresh_symbols.py     # python -m jy_algo.scripts.refresh_symbols --exchange all
```

Exchange adapters exist because ccxt normalizes most fields but each exchange has quirks in how it represents perpetuals, quanto contracts, and coin-margined instruments that need per-exchange handling.

---

## SID Registry (Security Master)

SID is a globally unique, monotonically increasing integer assigned to every instrument the system has ever seen — across all instrument types and all sources. SID 813 might be IBM equity; SID 814 might be a US 30Y Treasury. The pool is undivided: there are no separate sequences per instrument type.

A SID is permanent and never reused. If an instrument is delisted or superseded, its SID remains in the registry with `active = false` and optionally `superseded_by` pointing to the new SID.

### Instrument Types

```
equity      # IBM, AAPL — primary source: Bloomberg
bond        # US 30Y Treasury, corporate bonds
currency    # USD, EUR, GBP — FX pairs
crypto      # BTC, ETH — primary source: exchange APIs
future      # ES, NQ, CL — CME product specs
option      # equity and index options
```

### SID Registry Schema

```
reference/sids/registry.parquet   ← append-only; new rows only
```

```
sid:              int         # monotonically increasing, globally unique, never reused
created_at:       timestamp   # when this SID was allocated in our system
creation_source:  string      # "bloomberg" | "ccxt:binance" | "cme_product" | "manual"
creation_capture: string      # capturename that triggered SID creation
instrument_type:  string      # equity | bond | currency | crypto | future | option
primary_exchange: string      # NYSE | NASDAQ | CME | BINANCE | OKX | etc.
ticker:           string      # human-readable symbol at time of creation
name:             string      # full instrument name
currency:         string      # denomination (USD, USDT, BTC)
isin:             string|null
cusip:            string|null
figi:             string|null
exchange_native:  string      # exchange's own symbol string at time of creation
active:           bool        # false if delisted or superseded
superseded_by:    int|null    # SID of replacement instrument if applicable
```

### SID Creation Flow

New instruments enter the registry through a dedicated **SID creation capture** — a specific capture event (not a market data feed) that detects a previously unseen instrument and allocates the next SID.

```
Creation capture detects new instrument
        │
        ▼
Check registry: does a SID already exist for this instrument?
  (match on: instrument_type + primary_exchange + ticker or ISIN/CUSIP/FIGI)
        │
   no match
        │
        ▼
Allocate next SID (MAX(sid) + 1, serialized write)
Write new row to reference/sids/registry.parquet
        │
        ▼
SID is now available for all captures to reference
```

Deduplication on write is critical — two simultaneous creation captures for the same instrument must not allocate two SIDs. The writer serializes on a lock file or uses an atomic append mechanism.

### Sources of New SIDs by Instrument Type

| Type | Primary creation source | Trigger |
|---|---|---|
| Equity | Bloomberg feed | New listing, IPO |
| Bond | Bloomberg / FRED | New issuance |
| Currency | Manual / config | Rarely changes |
| Crypto | Exchange API (`ccxt load_markets()`) | New token listed on exchange |
| Future | CME product spec download | New contract listed |
| Option | Exchange API | New expiry listed |

### Corporate Actions and Crypto Events

Some corporate actions require a **new SID** because the resulting instrument is economically distinct:

| Event | Action |
|---|---|
| Stock split (e.g. 2:1) | Same SID; adjustment factor recorded in `reference/corporate_actions/` |
| Ticker change (e.g. FB → META) | Same SID; new ticker row in history |
| Merger/acquisition (target delisted) | Target SID: `active=false, superseded_by=acquirer_SID` |
| Reverse merger (new entity) | New SID allocated |
| Crypto token redenomination (e.g. LUNA → LUNA2) | Old SID: `active=false, superseded_by=new_SID`; new SID allocated |
| Crypto chain split / hard fork | New SID for each resulting token |
| Token burn/relaunch | New SID |

Corporate actions that do **not** create a new SID still generate a capture event recorded in `reference/corporate_actions/{YYYYMMDD}/{epoch_s}.json` with the SID, action type, effective date, and adjustment details.

---

## Capture Configuration

Each capturename has a configuration file at `config/captures/{capturename}.yaml` that defines its full behavior. Adding a new capture requires only a config file and a class stub — nothing else in the pipeline changes.

```yaml
# config/captures/cboe_vix.yaml
capturename:      cboe_vix
method:           rest_poll
url:              https://cdn.cboe.com/api/global/delayed_quotes/charts/historical/_VIX.json
poll_interval_s:  1
format:           json

timezone:         America/New_York
trading_calendar: NYSE               # crypto_247 | NYSE | CME | null (always open)
market_hours:     ["09:30", "16:00"] # local tz; null = 24h
expected_absences: [weekend, us_holiday]
absence_policy:   expected           # expected | alert | retry
```

```yaml
# config/captures/coinglass_funding_rates.yaml
capturename:      coinglass_funding_rates
method:           rest_poll
url:              https://open-api.coinglass.com/public/v2/funding
poll_interval_s:  60
format:           json

timezone:         UTC
trading_calendar: crypto_247
market_hours:     null
expected_absences: []
absence_policy:   alert
```

**`absence_policy` values:**
- `expected` — no alert if data is absent on a non-trading day
- `alert` — flag even expected absences (e.g. confirm GPR was actually published)
- `retry` — keep retrying until data arrives (e.g. FRED sometimes posts late)

Trading calendars are maintained in `config/calendars/` as lists of holidays per calendar name. Crypto captures use `crypto_247` which has no holidays.

---

## Override Ledger

When a vendor delivers incorrect data that they will never correct, overrides can be declared and auto-applied whenever the ETL pipeline is rerun. Overrides are stored as an append-only Parquet file — never mutated, new corrections are new rows.

**Location:** `reference/overrides/overrides.parquet`

**Schema (tall format — one row per field correction):**

```
sid           string    # Symbol ID of the affected instrument
capturename   string    # which capture source
valid_from    date      # start of affected period (valid time, inclusive)
valid_to      date      # end of affected period (valid time, inclusive)
field_name    string    # which field to replace
bad_value     string    # original vendor value (preserved for audit)
good_value    string    # corrected value to apply
reason        string    # human explanation
created_at    timestamp # when this override was entered
```

Multiple field corrections for the same SID/date range are multiple rows. The ETL join at staged → warehouse:

```sql
SELECT s.*, o.field_name, o.good_value, o.bad_value
FROM staged/{capturename} s
LEFT JOIN reference/overrides/overrides.parquet o
  ON  s.sid        = o.sid
  AND s.capturename = o.capturename
  AND s.ts BETWEEN o.valid_from AND o.valid_to
```

Warehouse records carry `is_overridden: bool` and retain `bad_value` alongside the corrected `value` for full auditability.

**GUI:** A Streamlit app (`tools/override_gui.py`) provides search by capturename + date, view existing overrides for a SID, and append new corrections. See Task #5.

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
