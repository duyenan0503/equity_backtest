# AAPL Candlestick Pattern Backtest Strategy

A technical analysis backtest that combines **Support/Resistance detection** with **candlestick reversal patterns** (Engulfing & Star formations) to generate buy/sell signals on Apple (AAPL) daily price data. 

## Dataset

- **Ticker:** AAPL (Apple Inc.)
- **Timeframe:** Daily (1D)
- **Date Range:** March 17, 2025 – April 14, 2026
- **Total Candles:** 271
- **Source file:** `aapl_03152025_04152026.xlsx`


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

### Layer 1: ATR (Average True Range)

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


### Layer 2: Support & Resistance Detection

Pivot-based S/R levels are identified using a **swing high / swing low** approach with a configurable lookback window:

- **n1 = 2** - candles before the pivot that must confirm direction
- **n2 = 2** - candles after the pivot that must confirm direction
- **backCandles = 30** - rolling window of past candles scanned for levels

**Support** (ss): A candle whose `low` is a local minimum - each preceding candle has a lower low than its successor, and each following candle has a higher low than its predecessor (strict V-shape bottom).

**Resistance** (rr): A candle whose `high` is a local maximum - an inverted V-shape (Λ-shape top).

For every signal bar, the strategy collects all active support prices and resistance prices from the prior `backCandles` window.


### Layer 3: Candlestick Pattern Detection

Two reversal pattern types are evaluated, each returning 1 (bearish) or 2 (bullish):

#### Engulfing Pattern

Detects a two-candle reversal where the second candle's body completely "engulfs" the first.

- **Bearish (1):** Previous candle is bullish; current candle is bearish; current body engulfs prior body; gap-down open is within 0.2 × ATR
- **Bullish (2):** Previous candle is bearish; current candle is bullish; current body engulfs prior body; gap-up open is within 0.2 × ATR

Both candles must have a body size > 0.35 × ATR to filter out insignificant candles.

#### Star Pattern (Shooting Star / Hammer)

Evaluates shadow-to-body ratios to detect pin bar formations:

- **Bearish Shooting Star (1):** Upper shadow > 1.5 × body; lower shadow < 0.3× upper shadow
- **Bullish Hammer (2):** Lower shadow > 1.5 × body; upper shadow < 0.3 × lower shadow


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

**closeResistance**: The current bar's high must be within 1.0 × ATR of the nearest resistance level, and the candle body must remain below the resistance.

**`closeSupport`**: The current bar's low must be within 1.5 × ATR of the nearest support level, and the candle body must remain above the support.

> **Note:** The current implementation has a fallback condition (or len(rr) >= 1) that triggers a sell signal whenever any resistance level exists in the lookback window, regardless of proximity. This unintentionally loosens the resistance filter.


## Execution Rules

Trades are executed at market open the candle **after** the signal fires. No pyramiding — a new position is only opened when flat (`not self.position`).

- **Entry:** Market order (both directions)
- **Stop-Loss (Long):** Close - 1.0 × ATR
- **Stop-Loss (Short):** Close + 1.0 × ATR
- **Take-Profit (Long):** Close + 1.5 × ATR
- **Take-Profit (Short):** Close - 1.5 × ATR
- **Risk:Reward Ratio:** 1 : 1.5


## Performance Evaluation
<img width="365" height="799" alt="image" src="https://github.com/user-attachments/assets/31bf0fa3-6f03-4ced-9fdb-edede31b5047" />

<img width="1531" height="562" alt="image" src="https://github.com/user-attachments/assets/e36f7c74-a1ac-4040-97f9-05141ec5b13f" />


### What Works

- **Conservative drawdown control:** Max drawdown of −6.17% is well-contained relative to holding AAPL outright, which experienced significantly larger intra-year swings. The ATR-based stop-loss is principled and volatility-adaptive.
- **Disciplined R:R structure:** The 1:1.5 risk-to-reward ratio is theoretically sound - it only requires a around 40% win rate to break even. The setup is correct in theory.
- **Near-zero beta (−0.004):** The strategy is essentially market-neutral in terms of directional exposure, showing good isolation of the alpha signal from broad market moves.

### What Doesn't Work

- **Net loss vs Buy & Hold:** The strategy returned −3.71% while simply holding AAPL returned +20.95% — a gap of ~24.7 percentage points over the same period.
- **Critically low signal frequency:** Only 6 signals were generated across 271 trading days (~2.2% hit rate), resulting in just 4 executed trades. The strategy was in the market only 12.9% of the time.
- **Win rate below breakeven:** With a 1:1.5 R:R, a ~40% win rate is needed to be profitable. The strategy achieved only 25% - well below the breakeven threshold.
- **Negative Sharpe & Sortino:** Both risk-adjusted return metrics are negative, meaning the strategy generates losses on a risk-adjusted basis. Any risk-free alternative (T-bills, money market) would have outperformed.
- **Negative Kelly Criterion (−0.32):** A negative Kelly value means the mathematical expectancy of the edge is negative — position sizing models recommend zero allocation to this strategy in its current form.
- **Profit Factor of 0.46:** For every $1 of losses, only $0.46 is recovered in gains. A profitable system requires Profit Factor > 1.0; elite strategies typically exceed 1.5.
- **Loose resistance filter:** The `or len(rr) >= 1` fallback means a sell signal fires any time any resistance level has been detected in the past 30 candles — regardless of whether price is near it. This generates false bearish signals.
- **Lookahead risk in S/R detection:** The `n2` parameter requires future candles to confirm a pivot, meaning signal labels technically look ahead. While the actual trade fires after the window closes, extreme care is needed to avoid subtle data leakage.

## Disclaimer

This project is for **educational and research purposes only**. Past backtest performance does not guarantee future results. This is not financial advice.
