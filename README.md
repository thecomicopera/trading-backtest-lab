# Trading Backtest Lab

A single-file dashboard for backtesting and comparing crypto and forex trading algorithms. No install, no build step, no API key, no dependencies — one HTML file with vanilla JS and Canvas charts.

## Quick start

1. Open `dashboard.html` in any modern browser (double-click it, or serve the folder with `python -m http.server` and visit `http://127.0.0.1:8000/dashboard.html`).
2. It auto-loads synthetic data and runs all 8 strategies on first paint, so you immediately see a working comparison.
3. Click **Load live (Binance)** for real price history, then **Run backtests**.

## Loading data

| Source | How | Notes |
|---|---|---|
| **Live crypto (Binance)** | Market = Crypto. Symbols: BTC/ETH/SOL/BNB/XRP/DOGE/ADA/LINK vs USDT, plus **PAXGUSDT / XAUTUSDT** (tokenized gold ≈ spot gold) and **EURUSDT** (EUR/USD proxy). Interval 1h/4h/1d, 200–3000 bars. | Public klines — no key, no account. Falls back to `data-api.binance.vision` (the primary `api.binance.com` is geo-blocked in some regions, e.g. US), then to synthetic data. |
| **Live forex (Kraken)** | Market = Forex. All six majors — **EUR/USD, GBP/USD, USD/JPY, USD/CHF, USD/CAD, AUD/USD** — plus AUD/JPY and EUR crosses (AUD, CAD, CHF, GBP, JPY). Interval 1h/4h/1d. | Kraken public OHLC — no key. Hard cap: **most recent 720 bars per interval** (1d ≈ 2 years, 4h ≈ 120 days, 1h ≈ 30 days); `since` cannot page into older history. Bar count auto-caps at 720 in forex mode. |
| **Synthetic** | Click *Synthetic data* | Regime-switching GBM with volatility clustering, random seed. Fully offline. For testing the tool, not for finding edges. |
| **CSV upload** | Click *Upload CSV* | Header row with `time/open/high/low/close/volume` (case-insensitive, comma/semicolon/tab). Only `close` is required — missing OHLC columns are reconstructed. Timestamps: ISO dates or unix seconds/ms. ≥50 rows. |

### Forex notes

- Kraken's forex pairs trade **24/7 on-venue** — weekend bars carry real (thinner) volume, unlike interbank 24/5 forex. The series is therefore annualized with the same √365/√2190/√8760 factors as crypto. Weekend liquidity is lower; treat weekend-heavy signals with suspicion.
- The forming (incomplete) last bar is dropped on load.
- For deeper forex history (daily ECB reference rates back to 1999, 30+ currencies), download from [Frankfurter](https://api.frankfurter.dev/v1/2020-01-01..2024-01-01?from=EUR&to=USD) and convert to the CSV format below — note it is close-only, so OHLC-dependent indicators (ATR, Donchian, stops) see reconstructed flat bars.

## Running backtests

- Each strategy row has an **enable checkbox** and **parameter inputs**. Defaults are the classic textbook values.
- **Run backtests** executes every enabled strategy on the loaded data with the current simulation settings.
- Click any row in the comparison table to inspect that strategy: price chart with entry/exit markers, drawdown curve, and full trade log.

### Strategies

| Strategy | Logic | Defaults |
|---|---|---|
| Buy & Hold | Benchmark, always long | — |
| SMA crossover | Fast SMA > slow SMA → long | 20 / 60 |
| EMA crossover | Fast EMA > slow EMA → long | 12 / 26 |
| RSI mean reversion | Buy RSI < oversold, sell RSI > overbought | 14 / 30 / 55 |
| Bollinger breakout | Close > upper band → long; close < mid → exit | 20 / 2.0 |
| MACD trend | MACD line > signal line → long | 12 / 26 / 9 |
| Donchian breakout | Close > N-bar prior high → long; < M-bar prior low → exit | 20 / 10 |
| Momentum (ROC) | N-bar ROC > threshold → long; ROC < 0 → exit | 30 / 5% |

### Simulation settings

- **Capital** — starting equity (default $10,000).
- **Fee / Slippage (bps per side)** — charged on every fill. Defaults 10 + 5 bps ≈ typical retail taker cost. Set both to 0 to see how much of an "edge" is just cost-free fantasy.
- **Stop-loss (%)** — 0 disables. While long, a bar whose low breaches entry × (1 − stop%) exits at the stop price minus slippage. Re-entry is blocked until the signal resets.
- **Out-of-sample (%)** — holds out the most recent bars. The comparison table then shows separate in-sample (IS) and out-of-sample (OOS) columns. **OOS is the honest column.**
- **Log equity** — log-scale the equity chart.

## Walk-forward validation

The panel below the sweep table re-optimizes the SMA crossover on rolling windows instead of fitting once:

1. Set **Train window (%)** (default 70) and **Step size (%)** (default 10).
2. Run backtests. Each window fits the best fast/slow pair from a 30-combo grid on its training slice, then reports the Sharpe on the immediately following test slice.
3. The headline number is the **average OOS Sharpe across windows** — if it collapses versus the full-sample Sharpe, the strategy was curve-fit.

## Parameter sweep

**SMA parameter sweep** runs 36 fast/slow combinations and ranks the top 12 by OOS Sharpe (or full-sample Sharpe if no split is set). A grid search mints lucky winners by chance alone — treat the top row as a hypothesis, not an edge, and deflate expectations for the number of trials (see López de Prado's deflated Sharpe ratio).

## Export

**Export results CSV** downloads the full comparison table (all metrics, IS/OOS columns when a split is set) for further analysis in a spreadsheet or Python.

## Execution model & caveats

- Signal computed on bar *t* close → position entered/exited at bar *t+1* **open**. No lookahead bias.
- Fees + slippage on every fill; stop fills assumed at the stop price (optimistic vs. real gap-through risk).
- Sharpe/Sortino annualized with √(bars/year): 365 for 1d, 2190 for 4h, 8760 for 1h. Risk-free rate 0.
- Long-only, 100% position sizing, single asset. No funding costs, liquidation modeling, partial fills, or maker rebates.
- Survivorship bias: you picked an instrument knowing it survived. **Backtest ≠ future performance.**

## Privacy

Everything runs locally in your browser. The only network requests are unauthenticated GETs to Binance's public klines endpoints and Kraken's public OHLC endpoint when you click *Load live*. No telemetry, no cookies, no accounts, no data upload — CSV files are parsed in-page and never leave your machine.
