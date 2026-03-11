# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**jy_algo** is an early-stage algorithmic trading/analysis project focused on Bitcoin market dynamics, geopolitical risk factors, and macroeconomic correlation analysis. The repository currently contains research documentation but no implemented code yet.

## Repository State

This is a newly initialized repository with no build system, dependencies, or code structure established yet. The primary file is `btc-iran-session.md`, a research document covering:

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
