**AAPL Candlestick Pattern Backtest Strategy**

A technical analysis backtest that combines **Support/Resistance detection** with **candlestick reversal patterns** (Engulfing & Star formations) to generate buy/sell signals on Apple (AAPL) daily price data. 

**1. Dataset**

- **Ticker:** AAPL (Apple Inc.)
- **Timeframe:** Daily (1D)
- **Date Range:** March 17, 2025 – April 14, 2026
- **Total Candles:** 271
- **Source file:** `aapl_03152025_04152026.xlsx`


**2. Strategy Logic**

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

**2.1: ATR (Average True Range)**

A **14-period ATR** is calculated manually using the classic Wilder formula:

```
True Range (TR) = max(
    High - Low,
    |High - Previous Close|,
    |Low  - Previous Close|
)

ATR = 14-period rolling mean of TR
```

ATR is used throughout the strategy as a **dynamic, volatility-adjusted threshold** - for pattern size filters, proximity checks, and trade sizing (stop-loss & take-profit distances).


**2.2: Support & Resistance Detection**

Pivot-based S/R levels are identified using a **swing high / swing low** approach with a configurable lookback window:

- **n1 = 2** - candles before the pivot that must confirm direction
- **n2 = 2** - candles after the pivot that must confirm direction
- **backCandles = 30** - rolling window of past candles scanned for levels

**Support** (ss): A candle whose `low` is a local minimum - each preceding candle has a lower low than its successor, and each following candle has a higher low than its predecessor (strict V-shape bottom).

**Resistance** (rr): A candle whose `high` is a local maximum - an inverted V-shape.

For every signal bar, the strategy collects all active support prices and resistance prices from the prior backCandles window.


**2.3: Candlestick Pattern Detection**

Two reversal pattern types are evaluated, each returning 1 (bearish) or 2 (bullish):


**Engulfing Pattern**

Detects a two-candle reversal where the second candle's body completely "engulfs" the first.

- **Bearish (1):** Previous candle is bullish; current candle is bearish; current body engulfs prior body; gap-down open is within 0.2 × ATR
- **Bullish (2):** Previous candle is bearish; current candle is bullish; current body engulfs prior body; gap-up open is within 0.2 × ATR

Both candles must have a body size > 0.35 × ATR to filter out insignificant candles.

**Star Pattern (Shooting Star / Hammer)**

Evaluates shadow-to-body ratios to detect pin bar formations:

- **Bearish Shooting Star (1):** Upper shadow > 1.5 × body; lower shadow < 0.3× upper shadow
- **Bullish Hammer (2):** Lower shadow > 1.5 × body; upper shadow < 0.3 × lower shadow


**3. Signal Combination Logic**

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


**4. Execution Rules**

Trades are executed at market open the candle **after** the signal fires. No pyramiding — a new position is only opened when flat (not self.position).

- **Entry:** Market order (both directions)
- **Stop-Loss (Long):** Close - 1.0 × ATR
- **Stop-Loss (Short):** Close + 1.0 × ATR
- **Take-Profit (Long):** Close + 1.5 × ATR
- **Take-Profit (Short):** Close - 1.5 × ATR
- **Risk:Reward Ratio:** 1 : 1.5


**5. Performance Evaluation**
   
<img width="430" height="707" alt="image" src="https://github.com/user-attachments/assets/476c31f3-3d04-4753-852f-112d9990f282" /> <img width="421" height="355" alt="image" src="https://github.com/user-attachments/assets/d3709173-6806-4cec-973e-ecc4a86148ac" />


<img width="1706" height="659" alt="image" src="https://github.com/user-attachments/assets/aa4d8fa3-1ea7-4b59-ae11-90eb635a9919" />

**5.1. What Works**

* Positive Profit Factor (1.61): For every $1 lost, the strategy recovers $1.61 in gains. This is above the breakeven threshold of 1.0 and within a reasonable range for a trend-following reversal system
* 50% Win Rate: With a 1:1.5 Risk:Reward ratio, breakeven only requires around 40% win rate. Achieving 50% means the strategy has a positive edge per trade
* Near-zero beta (0.002): The strategy remains largely market-neutral, generating returns independently of AAPL's broad directional move

**5.2. What Still Needs Work**

* Very low signal frequency: Only 3 signals were generated across 271 trading days, resulting in just 2 executed trades. This is statistically too thin to draw reliable conclusions.
* Still significantly underperforms Buy & Hold: +1.00% vs +20.95% means the strategy is leaving most of the available return on the table. 
* Small sample size distorts all metrics: With only 2 trades, every metric is highly sensitive to the outcome of a single trade. Results cannot be considered statistically significant

**6. Conclusion**

**Overall Rating**: Underperform - Work in Progress

The strategy shows a conceptually valid idea executed too conservatively. 
Although it did turn profitable (+1.00%) and crossed several key thresholds: positive Profit Factor (1.61), positive Kelly Criterion (+0.18), and a 50% win rate, the overall verdict is still significantly underperformed. 

