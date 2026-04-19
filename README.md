# AAPL Candlestick Pattern Backtest Strategy

A technical analysis backtest that combines **Support/Resistance detection** with **candlestick reversal patterns** (Engulfing & Star formations) to generate buy/sell signals on Apple (AAPL) daily price data. Built with Python and the [`backtesting.py`](https://kernc.github.io/backtesting.py/) library.

---

## Table of Contents

- [Dataset](#dataset)
- [Strategy Logic](#strategy-logic)
- [Signal Generation Pipeline](#signal-generation-pipeline)
- [Execution Rules](#execution-rules)
- [Backtest Configuration](#backtest-configuration)
- [Performance Results](#performance-results)
- [Performance Evaluation](#performance-evaluation)
- [Improvement Suggestions](#improvement-suggestions)
- [Requirements](#requirements)
- [Usage](#usage)

---

## Dataset

- **Ticker:** AAPL (Apple Inc.)
- **Timeframe:** Daily (1D)
- **Date Range:** March 17, 2025 – April 14, 2026
- **Total Candles:** 271
- **Source file:** `aapl_03152025_04152026.xlsx`

Preprocessing removes zero-volume candles (non-trading days) and resets the index before any further computation.

---

## Strategy Logic

The strategy uses a **three-layer confirmation** approach before triggering a trade:

```
Layer 1: ATR (Volatility Baseline)
       ↓
Layer 2: Support & Resistance Level Detection
       ↓
Layer 3: Candlestick Pattern Confirmation (Engulfing or Star)
       ↓
   Trade Signal
```

### Layer 1 — ATR (Average True Range)

A **14-period ATR** is calculated manually using the classic Wilder formula:

```
True Range (TR) = max(
    High - Low,
    |High - Previous Close|,
    |Low  - Previous Close|
)

ATR = 14-period rolling mean of TR
```

ATR is used throughout the strategy as a **dynamic, volatility-adjusted threshold** — for pattern size filters, proximity checks, and trade sizing (stop-loss & take-profit distances).

---

### Layer 2 — Support & Resistance Detection

Pivot-based S/R levels are identified using a **swing high / swing low** approach with a configurable lookback window:

- **`n1 = 2`** — candles before the pivot that must confirm direction
- **`n2 = 2`** — candles after the pivot that must confirm direction
- **`backCandles = 30`** — rolling window of past candles scanned for levels

**Support** (`ss`): A candle whose `low` is a local minimum — each preceding candle has a lower low than its successor, and each following candle has a higher low than its predecessor (strict V-shape bottom).

**Resistance** (`rr`): A candle whose `high` is a local maximum — an inverted V-shape (Λ-shape top).

For every signal bar, the strategy collects all active support prices and resistance prices from the prior `backCandles` window.

---

### Layer 3 — Candlestick Pattern Detection

Two reversal pattern types are evaluated, each returning `1` (bearish) or `2` (bullish):

#### Engulfing Pattern

Detects a two-candle reversal where the second candle's body completely "engulfs" the first.

- **Bearish (1):** Previous candle is bullish; current candle is bearish; current body engulfs prior body; gap-down open is within `0.2 × ATR`
- **Bullish (2):** Previous candle is bearish; current candle is bullish; current body engulfs prior body; gap-up open is within `0.2 × ATR`

Both candles must have a body size `> 0.35 × ATR` to filter out insignificant candles.

#### Star Pattern (Shooting Star / Hammer)

Evaluates shadow-to-body ratios to detect pin bar formations:

- **Bearish Shooting Star (1):** Upper shadow `> 1.5×` body; lower shadow `< 0.3×` upper shadow
- **Bullish Hammer (2):** Lower shadow `> 1.5×` body; upper shadow `< 0.3×` lower shadow

---

### Signal Combination Logic

A signal is only generated when **both** the pattern layer and the S/R proximity check agree:

```python
# SELL signal (1)
if (isEngulfing(row) == 1 or isStar(row) == 1) and closeResistance(row, rr):
    signal[row] = 1

# BUY signal (2)
if (isEngulfing(row) == 2 or isStar(row) == 2) and closeSupport(row, ss):
    signal[row] = 2
```

**`closeResistance`**: The current bar's high must be within `1.0 × ATR` of the nearest resistance level, and the candle body must remain below the resistance.

**`closeSupport`**: The current bar's low must be within `1.5 × ATR` of the nearest support level, and the candle body must remain above the support.

> **Note:** The current implementation has a fallback condition (`or len(rr) >= 1`) that triggers a sell signal whenever any resistance level exists in the lookback window, regardless of proximity. This unintentionally loosens the resistance filter.

---

## Execution Rules

Trades are executed at market open the candle **after** the signal fires. No pyramiding — a new position is only opened when flat (`not self.position`).

- **Entry:** Market order (both directions)
- **Stop-Loss (Long):** `Close - 1.0 × ATR`
- **Stop-Loss (Short):** `Close + 1.0 × ATR`
- **Take-Profit (Long):** `Close + 1.5 × ATR`
- **Take-Profit (Short):** `Close - 1.5 × ATR`
- **Risk:Reward Ratio:** 1 : 1.5

---

## Backtest Configuration

```python
bt = Backtest(df, MyCandlesStrat, cash=10_000, commission=0.002)
```

- **Starting Capital:** $10,000
- **Commission:** 0.2% per trade (round-trip ~0.4%)
- **Timeframe:** Daily
- **Instrument:** AAPL

---

## Performance Results

- **Period:** 2025-03-17 → 2026-04-14 (393 days)
- **Final Equity:** $9,628.85
- **Net Return:** −3.71%
- **Buy & Hold Return:** +20.95%
- **Annualised Return:** −3.46%
- **CAGR:** −2.40%
- **Volatility (Ann.):** 5.39%
- **Sharpe Ratio:** −0.64
- **Sortino Ratio:** −0.77
- **Calmar Ratio:** −0.56
- **Alpha:** −3.64%
- **Beta:** −0.004
- **Max Drawdown:** −6.17%
- **Avg Drawdown:** −2.47%
- **Max Drawdown Duration:** 301 days
- **Market Exposure:** 12.92%
- **# Trades:** 4
- **Win Rate:** 25%
- **Best Trade:** +3.05%
- **Worst Trade:** −3.20%
- **Avg Trade:** −0.94%
- **Profit Factor:** 0.46
- **Expectancy:** −0.91%
- **SQN:** −0.69
- **Kelly Criterion:** −0.32

---

## Performance Evaluation

### What Works

- **Conservative drawdown control:** Max drawdown of −6.17% is well-contained relative to holding AAPL outright, which experienced significantly larger intra-year swings. The ATR-based stop-loss is principled and volatility-adaptive.
- **Disciplined R:R structure:** The 1:1.5 risk-to-reward ratio is theoretically sound — it only requires a ~40% win rate to break even. The setup is correct in theory.
- **Near-zero beta (−0.004):** The strategy is essentially market-neutral in terms of directional exposure, showing good isolation of the alpha signal from broad market moves.

### What Doesn't Work

- **Net loss vs Buy & Hold:** The strategy returned −3.71% while simply holding AAPL returned +20.95% — a gap of ~24.7 percentage points over the same period.
- **Critically low signal frequency:** Only 6 signals were generated across 271 trading days (~2.2% hit rate), resulting in just 4 executed trades. The strategy was in the market only 12.9% of the time.
- **Win rate below breakeven:** With a 1:1.5 R:R, a ~40% win rate is needed to be profitable. The strategy achieved only 25% — well below the breakeven threshold.
- **Negative Sharpe & Sortino:** Both risk-adjusted return metrics are negative, meaning the strategy generates losses on a risk-adjusted basis. Any risk-free alternative (T-bills, money market) would have outperformed.
- **Negative Kelly Criterion (−0.32):** A negative Kelly value means the mathematical expectancy of the edge is negative — position sizing models recommend zero allocation to this strategy in its current form.
- **Profit Factor of 0.46:** For every $1 of losses, only $0.46 is recovered in gains. A profitable system requires Profit Factor > 1.0; elite strategies typically exceed 1.5.
- **Loose resistance filter:** The `or len(rr) >= 1` fallback means a sell signal fires any time any resistance level has been detected in the past 30 candles — regardless of whether price is near it. This generates false bearish signals.
- **Lookahead risk in S/R detection:** The `n2` parameter requires future candles to confirm a pivot, meaning signal labels technically look ahead. While the actual trade fires after the window closes, extreme care is needed to avoid subtle data leakage.

---

## Improvement Suggestions

### 1. Fix the Loose Resistance Condition

The current fallback `or len(rr) >= 1` is too permissive. Replace it with strict proximity-only gating:

```python
# Before (buggy)
if (isEngulfing(row) == 1 or isStar(row) == 1) and (closeResistance(row, rr) or len(rr) >= 1):

# After (strict)
if (isEngulfing(row) == 1 or isStar(row) == 1) and closeResistance(row, rr):
```

This alone will reduce false sell signals and likely improve win rate significantly.

---

### 2. Add a Trend Filter

Reversal patterns at S/R are far more reliable when they align with the broader trend. Incorporate a directional filter so the strategy only trades in the trend direction:

```python
# Only take LONG trades when price is above the 50-day SMA
# Only take SHORT trades when price is below the 50-day SMA
df['SMA50'] = df['Close'].rolling(50).mean()

# In the signal logic:
if signal == 2 and close[row] > sma50[row]:  # long only in uptrend
    final_signal[row] = 2
if signal == 1 and close[row] < sma50[row]:  # short only in downtrend
    final_signal[row] = 1
```

---

### 3. Strengthen S/R Level Quality

Not all pivot points carry equal weight. Prioritize levels that have been **tested multiple times** (confluence zones):

```python
def cluster_levels(levels, tolerance):
    """Merge S/R levels within tolerance; keep only those tested 2+ times."""
    clustered = []
    for level in sorted(levels):
        if clustered and abs(level - clustered[-1][0]) <= tolerance:
            clustered[-1][1].append(level)
        else:
            clustered.append([level, [level]])
    return [np.mean(c[1]) for c in clustered if len(c[1]) >= 2]
```

Levels touched 2+ times are stronger barriers and produce higher-probability reversals.

---

### 4. Switch to Intraday or Higher-Frequency Data

With only 271 daily candles and strict pattern filters, the strategy produces just 6 signals in 13 months — far too few for statistical significance. Consider:

- **4-hour or 1-hour OHLCV data:** Same logic, ~6–20× more signals
- **Multiple tickers:** Apply the strategy across 10–20 large-cap stocks to diversify and increase trade frequency

A minimum of **30–50 trades** is needed before any performance metric carries statistical weight.

---

### 5. Optimize Parameters with Walk-Forward Testing

Replace the fixed `backCandles=30`, `n1=2`, `n2=2` with walk-forward optimization to avoid overfitting:

```python
stats, heatmap = bt.optimize(
    backCandles=range(15, 50, 5),
    n1=range(2, 5),
    n2=range(2, 5),
    maximize='Sharpe Ratio',
    method='grid',
    return_heatmap=True
)
```

Use **out-of-sample validation** (e.g., train on 2023–2024, test on 2025–2026) to confirm that optimized parameters generalise.

---

### 6. Implement Adaptive Position Sizing

Replace the implicit all-in sizing with a fixed-fractional model based on volatility:

```python
def next(self):
    risk_per_trade = 0.01 * self.equity       # risk 1% of equity per trade
    stop_distance = self.atr[-1]
    size = risk_per_trade / (stop_distance * self.data.Close[-1])
    size = max(1, int(size))

    if self.signal1[-1] == 2 and not self.position:
        self.buy(size=size, sl=..., tp=...)
```

This ensures no single losing trade threatens the account disproportionately.

---

### 7. Apply S/R Level Staleness Decay

Old support/resistance levels that formed many candles ago are less relevant than recent ones. Apply a decay weight:

```python
def weighted_proximity(level_candle_idx, current_idx, level_price, current_price, atr):
    age = current_idx - level_candle_idx
    decay = 1 / (1 + 0.05 * age)         # older levels get lower weight
    distance = abs(current_price - level_price) / atr
    return decay / (distance + 1e-6)      # higher score = stronger, more recent level
```

---

## Requirements

```
python >= 3.8
pandas
openpyxl
backtesting
bokeh
```

Install dependencies:

```bash
pip install pandas openpyxl backtesting bokeh
```

---

## Usage

1. Place `aapl_03152025_04152026.xlsx` in the project root.
2. Open `equity_backtest.ipynb` in Jupyter or Google Colab.
3. Run all cells sequentially.
4. The final cell renders an interactive Bokeh chart of the equity curve and trades.

```bash
jupyter notebook equity_backtest.ipynb
```

---

## Disclaimer

This project is for **educational and research purposes only**. Past backtest performance does not guarantee future results. This is not financial advice.
