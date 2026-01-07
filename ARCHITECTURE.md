# Architecture

This document describes the code flow and user flow for the ChatGPT Micro-Cap Experiment trading system.

## System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        User Interface Layer                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │ ProcessPortfolio │  │ trading_script   │  │ Generate Graph   │   │
│  │      .py         │  │    (CLI)         │  │      .py         │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
└───────────┼─────────────────────┼─────────────────────┼─────────────┘
            │                     │                     │
            ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Core Trading Engine                           │
│                      (trading_script.py)                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │
│  │ Data Access │  │  Portfolio  │  │   Trade     │  │ Reporting  │  │
│  │   Layer     │  │ Operations  │  │  Logging    │  │ & Metrics  │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └─────┬──────┘  │
└─────────┼────────────────┼────────────────┼───────────────┼─────────┘
          │                │                │               │
          ▼                ▼                ▼               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Data Layer                                   │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────────┐│
│  │ Yahoo Finance │  │    Stooq      │  │      CSV Files            ││
│  │   (Primary)   │  │  (Fallback)   │  │ Daily Updates, Trade Log  ││
│  └───────────────┘  └───────────────┘  └───────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

---

## Code Flow

### 1. Entry Points

The system has multiple entry points depending on use case:

```
Entry Points
├── trading_script.py --data-dir <DIR>    # Direct CLI with custom directory
├── Scripts and CSV Files/ProcessPortfolio.py    # Author's portfolio
├── Start Your Own/Process Portfolio.py          # User's own portfolio
└── simple_automation.py                         # LLM-automated trading
```

### 2. Data Fetching Pipeline

```
download_price_data(ticker)
         │
         ▼
┌─────────────────────┐
│ 1. Yahoo Finance    │──── Success ────▶ Return FetchResult
│    (yfinance)       │
└─────────┬───────────┘
          │ Failure
          ▼
┌─────────────────────┐
│ 2. Stooq PDR        │──── Success ────▶ Return FetchResult
│  (pandas-datareader)│
└─────────┬───────────┘
          │ Failure
          ▼
┌─────────────────────┐
│ 3. Stooq CSV        │──── Success ────▶ Return FetchResult
│   (Direct HTTP)     │
└─────────┬───────────┘
          │ Failure
          ▼
┌─────────────────────┐
│ 4. Proxy Indices    │──── Success ────▶ Return FetchResult
│ (^GSPC→SPY, etc.)   │
└─────────┬───────────┘
          │ Failure
          ▼
     Return Empty DataFrame
```

**Key Functions:**
- `download_price_data()` - Main accessor with fallback chain
- `_yahoo_download()` - yfinance wrapper with error suppression
- `_stooq_download()` - pandas-datareader Stooq source
- `_stooq_csv_download()` - Direct CSV download from Stooq
- `_normalize_ohlcv()` - Standardizes output to [Open, High, Low, Close, Adj Close, Volume]

### 3. Portfolio Processing Flow

```
main()
  │
  ├──▶ set_data_dir(path)           # Configure CSV locations
  │
  ├──▶ load_latest_portfolio_state()
  │         │
  │         ├── Read Daily Updates.csv
  │         ├── Extract latest non-TOTAL rows
  │         ├── Get cash from latest TOTAL row
  │         └── Return (portfolio_df, cash)
  │
  ├──▶ process_portfolio(portfolio, cash, interactive)
  │         │
  │         ├── Interactive Trade Entry (if interactive=True)
  │         │     ├── 'b' - Manual Buy (MOO or Limit)
  │         │     ├── 's' - Manual Sell (MOO or Limit)
  │         │     └── 'u' - Update Stop Loss
  │         │
  │         ├── Daily Pricing Loop (for each holding)
  │         │     ├── Fetch current price data
  │         │     ├── Check stop-loss trigger (Low <= stop_loss)
  │         │     │     ├── If triggered: Execute sell at Open or Stop price
  │         │     │     └── Call log_sell() to record trade
  │         │     └── Calculate current value and PnL
  │         │
  │         ├── Generate TOTAL row
  │         └── Append to Daily Updates.csv
  │
  └──▶ daily_results(portfolio, cash)
            │
            ├── Fetch prices for holdings + benchmarks
            ├── Calculate metrics (Sharpe, Sortino, Beta, Alpha)
            ├── Display formatted results
            └── Print LLM prompt instructions
```

### 4. Trade Logging Flow

```
Trade Execution
      │
      ├──▶ log_sell()                    # Automated stop-loss sells
      │         ├── Record to Trade Log.csv
      │         └── Remove ticker from portfolio_df
      │
      ├──▶ log_manual_buy()              # User-initiated buys
      │         ├── Validate price vs OHLC range
      │         ├── Determine execution price (Open or Limit)
      │         ├── Record to Trade Log.csv
      │         └── Add/update position in portfolio_df
      │
      └──▶ log_manual_sell()             # User-initiated sells
                ├── Validate shares owned
                ├── Determine execution price (Open or Limit)
                ├── Record to Trade Log.csv
                └── Update/remove position in portfolio_df
```

### 5. Metrics Calculation Flow

```
daily_results()
      │
      ├── Portfolio Returns
      │     └── equity_series.pct_change()
      │
      ├── Risk Metrics
      │     ├── Max Drawdown = min(equity / cummax - 1)
      │     ├── Sharpe = (return - rf) / std
      │     └── Sortino = (return - rf) / downside_std
      │
      └── CAPM Analysis
            ├── Download S&P 500 data
            ├── Align dates with portfolio
            ├── Calculate excess returns
            └── Linear regression for Beta, Alpha, R²
```

### 6. Visualization Flow (Generate Graph.py)

```
main()
  │
  ├──▶ load_portfolio_details()
  │         └── Read TOTAL rows from Daily Updates.csv
  │
  ├──▶ download_sp500()
  │         ├── Fetch ^GSPC via yfinance
  │         ├── Align to portfolio dates (forward-fill)
  │         └── Normalize to starting equity
  │
  └──▶ plot_comparison()
            ├── Plot portfolio line
            ├── Plot S&P 500 line (dashed)
            ├── Annotate percentage returns
            └── Save/display chart
```

---

## User Flow

### Daily Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DAILY TRADING WORKFLOW                          │
└─────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐
    │  Market      │
    │  Closes      │
    │  (4:00 PM)   │
    └──────┬───────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────┐
    │  Step 1: Run Trading Script                          │
    │  $ python trading_script.py --data-dir "Start Your   │
    │    Own"                                              │
    └──────┬───────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────┐
    │  Step 2: Review Current Portfolio                    │
    │  - Script displays holdings, prices, and P&L         │
    │  - Stop-losses are automatically checked/executed    │
    └──────┬───────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────┐
    │  Step 3: Enter Manual Trades (Optional)              │
    │  - 'b' for buy (MOO or Limit)                        │
    │  - 's' for sell (MOO or Limit)                       │
    │  - 'u' to update stop-loss                           │
    │  - Enter to continue                                 │
    └──────┬───────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────┐
    │  Step 4: Review Daily Results                        │
    │  - Price & volume table                              │
    │  - Risk metrics (Sharpe, Sortino, Drawdown)          │
    │  - CAPM analysis (Beta, Alpha)                       │
    │  - Liquidity warnings                                │
    └──────┬───────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────┐
    │  Step 5: Copy Output to ChatGPT                      │
    │  - Paste the terminal output into ChatGPT            │
    │  - Receive trading recommendations                   │
    └──────┬───────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────┐
    │  Step 6: Execute Recommendations (Next Day)          │
    │  - Run script again after next market close          │
    │  - Enter any buy/sell orders from ChatGPT            │
    └──────────────────────────────────────────────────────┘
```

### Initial Setup Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     INITIAL SETUP WORKFLOW                          │
└─────────────────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────┐
    │  Step 1: Clone Repository                            │
    │  $ git clone <repo-url>                              │
    │  $ cd ChatGPT-Micro-Cap-Experiment                   │
    └──────┬───────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────┐
    │  Step 2: Install Dependencies                        │
    │  $ python -m venv venv                               │
    │  $ source venv/bin/activate                          │
    │  $ pip install -r requirements.txt                   │
    └──────┬───────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────┐
    │  Step 3: Run First Time                              │
    │  $ python trading_script.py --data-dir "Start Your   │
    │    Own"                                              │
    │  - Enter starting cash amount when prompted          │
    └──────┬───────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────┐
    │  Step 4: Set Up Initial Portfolio                    │
    │  - Enter 'b' to buy initial positions                │
    │  - Set stop-losses for risk management               │
    │  - Press Enter when done                             │
    └──────────────────────────────────────────────────────┘
```

### Analysis and Reporting Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ANALYSIS WORKFLOW                               │
└─────────────────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────┐
    │  Generate Performance Graph                          │
    │  $ python "Start Your Own/Generate Graph.py"         │
    │                                                      │
    │  Options:                                            │
    │  --start-date YYYY-MM-DD                             │
    │  --end-date YYYY-MM-DD                               │
    │  --start-equity 100                                  │
    │  --output chart.png                                  │
    └──────┬───────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────┐
    │  Output:                                             │
    │  - Line chart: Portfolio vs S&P 500                  │
    │  - Percentage annotations                            │
    │  - Both series normalized to starting equity         │
    └──────────────────────────────────────────────────────┘
```

---

## Data Files

### CSV Schema

**Daily Updates.csv** - Portfolio snapshots
```
Date, Ticker, Shares, Buy Price, Cost Basis, Stop Loss,
Current Price, Total Value, PnL, Action, Cash Balance, Total Equity
```

**Trade Log.csv** - Trade execution history
```
Date, Ticker, Shares Bought, Buy Price, Cost Basis, PnL,
Reason, Shares Sold, Sell Price
```

### File Locations

```
Repository Root
├── trading_script.py          # Core engine (imports from here)
├── Daily Updates.csv          # Default location (if no --data-dir)
├── Trade Log.csv
│
├── Scripts and CSV Files/     # Author's live portfolio
│   ├── Daily Updates.csv
│   ├── Trade Log.csv
│   └── ProcessPortfolio.py    # Wrapper script
│
└── Start Your Own/            # User's personal portfolio
    ├── Daily Updates.csv
    ├── Trade Log.csv
    ├── Process Portfolio.py   # Wrapper script
    └── Generate Graph.py      # Visualization
```

---

## Key Design Decisions

1. **Multi-source Data Fallback**: Yahoo Finance → Stooq → CSV → Proxy indices ensures data reliability for micro-cap stocks.

2. **Weekend Handling**: Automatic detection of weekends with fallback to Friday's data prevents script failures.

3. **Interactive Mode**: Allows manual trade entry while maintaining automated stop-loss execution.

4. **CSV Persistence**: Simple, transparent storage format that's human-readable and version-controllable.

5. **LLM Integration**: Output formatted specifically for pasting into ChatGPT for trading recommendations.

6. **Lookahead Bias Prevention**: Orders generated after market close must be executed on the following trading day.
