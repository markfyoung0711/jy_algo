# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**jy_algo** is the market data collection repository for the jy trading system. Its sole responsibility is capturing, validating, and storing raw market data across multiple feeds. Trading logic, analysis, and correlation models live in a separate repository.

**Scope of this repo:**
- Feed producers (crypto, VIX, breakeven, GPR, Coinglass, Glassnode)
- Symbol reference data ETL
- Raw → staged → warehouse zone pipeline
- Validation subsystem
- Message bus (Redis Streams) and Parquet storage

**Out of scope:** backtesting, order execution, correlation models, dashboards, trading strategies.

## Design Documentation

Full pipeline architecture, ETL design decisions, message schema, and phased implementation plan: [`docs/architecture.md`](docs/architecture.md)

## Commands

**Python**
```bash
source .venv/bin/activate       # activate virtualenv
uv pip install -e ".[dev]"      # install/sync dependencies
pytest                          # run tests
ruff check .                    # lint
```

**Rust**
```bash
source ~/.cargo/env             # make cargo available (new shells)
cd rs && cargo build            # build
cd rs && cargo test             # run tests
cd rs && cargo run              # run binary
```

## Project Structure

```
jy_algo/
├── pyproject.toml      # Python project config + deps
├── rs/                 # Rust crate (jy-algo-rs)
│   ├── Cargo.toml
│   └── src/main.rs
├── .env                # API keys (not committed)
└── .env.example        # API key template
```

Python virtualenv is at `.venv/`. The Rust toolchain is managed via rustup (`~/.cargo/`).

## Repository State

The primary research reference is `btc-iran-session.md`, covering:

- BTC price correlation with geopolitical events (Iran conflict timeline)
- BTC vs. US 5Y Bond yield correlation
- Sentiment data sources ranked by quality
- How to quantify key indices: VIX, 5Y Breakeven, GPR Index, Coinglass, Glassnode

## Domain Knowledge

Key concepts documented in `btc-iran-session.md`:

- **"Springboard pattern"**: BTC historically rallies ~62% within 2 months of geopolitical conflict resolution
- **Three mechanisms for BTC selloffs during crises**: liquidity crises (margin call hedging), oil/inflation fears keeping rates high, risk-on reclassification
- **High-quality sentiment sources**: Coinglass (funding rates/liquidations), Glassnode (on-chain), AAII survey, GPR Index
- **Key correlation indices**: VIX (CBOE), 5Y Breakeven Rate (FRED T5YIE), GPR Index (geopolitical risk)
