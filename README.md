# XAUBot AI

**AI-powered XAUUSD (Gold) trading bot** with XGBoost ML, Smart Money Concepts (SMC), and HMM regime detection for MetaTrader 5.

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![MetaTrader 5](https://img.shields.io/badge/broker-MetaTrader%205-orange.svg)](https://www.metatrader5.com/)

---

## Features

| Feature | Description |
|---------|-------------|
| **XGBoost ML Model** | 37-feature model predicting BUY/SELL/HOLD with calibrated confidence |
| **Smart Money Concepts** | Order Blocks, Fair Value Gaps, Break of Structure, Change of Character |
| **HMM Regime Detection** | 3-state Hidden Markov Model classifying trending/ranging/volatile markets |
| **Dynamic Risk Management** | ATR-based stop loss, Kelly criterion sizing, daily loss limits |
| **Session-Aware Trading** | Optimized for Sydney, London, and New York sessions |
| **Auto-Retraining** | Models automatically retrain when market conditions shift |
| **Telegram Alerts** | Real-time trade notifications and daily summaries |
| **Web Dashboard** | Next.js monitoring interface for live tracking |

## Architecture

```
                          ┌─────────────────┐
                          │   MetaTrader 5   │
                          │   (XAUUSD M15)   │
                          └────────┬─────────┘
                                   │ OHLCV
                          ┌────────▼─────────┐
                          │   Data Pipeline   │
                          │  (Polars Engine)  │
                          └────────┬─────────┘
                                   │
                 ┌─────────────────┼─────────────────┐
                 │                 │                  │
        ┌────────▼───────┐ ┌──────▼───────┐ ┌───────▼──────┐
        │  SMC Analyzer   │ │  Feature Eng  │ │ HMM Regime   │
        │  (OB/FVG/BOS)  │ │ (37 features) │ │  Detector    │
        └────────┬───────┘ └──────┬───────┘ └───────┬──────┘
                 │                │                  │
                 └─────────────────┼─────────────────┘
                                   │
                          ┌────────▼─────────┐
                          │  XGBoost Model    │
                          │  (Signal + Conf)  │
                          └────────┬─────────┘
                                   │
                 ┌─────────────────┼─────────────────┐
                 │                 │                  │
        ┌────────▼───────┐ ┌──────▼───────┐ ┌───────▼──────┐
        │  11 Entry       │ │  Risk Engine  │ │  Position    │
        │  Filters        │ │  (ATR + Kelly)│ │  Manager     │
        └────────┬───────┘ └──────┬───────┘ └───────┬──────┘
                 │                │                  │
                 └────────────────┼──────────────────┘
                                  │
                         ┌────────▼─────────┐
                         │  Trade Execution   │
                         │  (MT5 + Logging)   │
                         └───────────────────┘
```

## Project Structure

```
xaubot-ai/
├── main_live.py              # Main async trading orchestrator
├── train_models.py           # Model training script
├── src/                      # Core modules
│   ├── config.py             #   Trading configuration & capital modes
│   ├── mt5_connector.py      #   MetaTrader 5 connection layer
│   ├── smc_polars.py         #   Smart Money Concepts analyzer
│   ├── ml_model.py           #   XGBoost trading model
│   ├── feature_eng.py        #   Feature engineering (37 features)
│   ├── regime_detector.py    #   HMM market regime detection
│   ├── risk_engine.py        #   Risk calculations & validation
│   ├── smart_risk_manager.py #   Dynamic risk management
│   ├── session_filter.py     #   Session filter (Sydney/London/NY)
│   ├── position_manager.py   #   Open position management
│   ├── dynamic_confidence.py #   Adaptive confidence thresholds
│   ├── auto_trainer.py       #   Auto-retraining pipeline
│   ├── news_agent.py         #   Economic news filtering
│   ├── telegram_notifier.py  #   Telegram alerts
│   ├── trade_logger.py       #   Trade logging to DB
│   └── utils.py              #   Utility functions
├── backtests/                # Backtesting
│   ├── backtest_live_sync.py #   Main backtest (synced with live)
│   └── archive/              #   Historical versions
├── scripts/                  # Utility scripts
│   ├── check_market.py       #   Quick SMC market analysis
│   ├── check_positions.py    #   View open positions
│   ├── check_status.py       #   Account status check
│   ├── close_positions.py    #   Emergency close all
│   ├── modify_tp.py          #   Modify take-profit levels
│   └── get_trade_history.py  #   Pull trade history
├── tests/                    # Tests
├── models/                   # Trained models (.pkl)
├── data/                     # Market data & trade logs
├── docs/                     # Documentation
│   ├── arsitektur-ai/        #   Architecture docs (23 components)
│   └── research/             #   Research & analysis
├── web-dashboard/            # Next.js monitoring dashboard
└── docker/                   # Docker configuration
```

## Backtest Results (Jan 2025 - Feb 2026)

| Metric | Value |
|--------|-------|
| Total Trades | 654 |
| Win Rate | 63.9% |
| Net P/L | $4,189.52 |
| Profit Factor | 2.64 |
| Max Drawdown | 2.2% |
| Sharpe Ratio | 4.83 |

## Installation

### Prerequisites

- Python 3.11+
- MetaTrader 5 terminal (Windows)
- PostgreSQL (optional, for trade logging)

### Setup

```bash
# Clone the repository
git clone https://github.com/GifariKemal/xaubot-ai.git
cd xaubot-ai

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with your MT5 credentials and Telegram token
```

### Configuration

Key settings in `.env`:

```env
# MetaTrader 5
MT5_LOGIN=your_login
MT5_PASSWORD=your_password
MT5_SERVER=your_server
MT5_PATH=C:/Program Files/MetaTrader 5/terminal64.exe

# Telegram Notifications
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Trading
CAPITAL=5000
SYMBOL=XAUUSD
```

### Run

```bash
# Train models first
python train_models.py

# Start the bot
python main_live.py

# Run backtest
python backtests/backtest_live_sync.py --tune
```

## Risk Management

| Protection | Details |
|-----------|---------|
| **ATR-Based Stop Loss** | Minimum 1.5x ATR distance |
| **Broker-Level SL** | Emergency SL set at broker level |
| **Position Sizing** | Kelly criterion with capital mode scaling |
| **Daily Loss Limit** | 5% of capital per day |
| **Total Loss Limit** | 10% of capital |
| **Position Limit** | Max 2 concurrent positions |
| **Time-Based Exit** | Max 6 hours per trade |
| **Session Filter** | Only trades during active sessions |
| **Spread Filter** | Rejects trades during high spread |
| **Cooldown** | Minimum time between trades |

## Tech Stack

- **Polars** — High-performance data engine (not Pandas)
- **XGBoost** — Gradient boosted ML model
- **hmmlearn** — Hidden Markov Model for regime detection
- **MetaTrader5** — Broker connection API
- **asyncio** — Async event loop for low-latency execution
- **loguru** — Structured logging
- **PostgreSQL** — Trade database
- **Next.js** — Web dashboard

## Disclaimer

> This software is for **educational and research purposes only**. Trading foreign exchange (Forex) and commodities on margin carries a high level of risk and may not be suitable for all investors. Past performance is not indicative of future results. You could lose some or all of your investment. **Use at your own risk.**

## License

[MIT License](LICENSE) - Copyright (c) 2025-2026 Gifari Kemal
