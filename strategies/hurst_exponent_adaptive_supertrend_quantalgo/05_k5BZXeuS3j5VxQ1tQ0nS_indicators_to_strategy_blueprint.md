
# Indicators to Strategy Blueprint

Here is the transformation of the Hurst Exponent Adaptive Supertrend indicator into a production-ready algorithmic trading framework.

### 1. Execution Triggers (Entry & Direction)

The core of the entry logic is the trend flip, identified by the `turnedBullish` and `turnedBearish` boolean variables. We will translate these signals into concrete order execution commands.

*   **Long Entry Condition:** A long position is initiated when the trend flips from bearish to bullish.
    *   `boolean condition`: `turnedBullish` which is defined as `trend == 1 and trend[1] == -1`.

*   **Short Entry Condition:** A short position is initiated when the trend flips from bullish to bearish.
    *   `boolean condition`: `turnedBearish` which is defined as `trend == -1 and trend[1] == 1`.

*   **Execution Nuance (Close vs. Real-time):**
    The indicator's calculations, including the Hurst Exponent, Kalman Smoother, and final trend state, are based on the `close` of the historical bar. Attempting to execute intra-bar based on these signals would lead to repainting and unreliable performance, as the final `close` value is unknown until the bar is complete.

    **Execution Mandate:** All entry and exit decisions must be calculated on the bar's close. The resulting orders (market or limit) should be placed for execution at the **open of the next bar**. This ensures that the signal is confirmed and non-repainting. In Pine Script, this is achieved by setting `process_orders_on_close=true` in the `strategy()` declaration.

*   **Signal Reversals:**
    The system is inherently a reversal strategy. When a `turnedBearish` signal occurs, it implies an existing long position (or no position) should be flipped to short. A professional execution model handles this explicitly:
    1.  First, an order is sent to `close` the existing long position (e.g., `strategy.close("Long")`).
    2.  Immediately after, an order is sent to `entry` a new short position (e.g., `strategy.entry("Short", strategy.short)`).

    This two-step logic provides clearer state management and order tracking than relying on a single `strategy.entry` call to auto-reverse the position.

### 2. Multi-Tiered Exit Logic

A simple reversal system is brittle. A robust framework requires a layered defense of profits and capital.

*   **Initial Stop Loss (Volatility-Based):**
    The indicator itself provides the perfect initial stop loss. Instead of an arbitrary percentage, the stop is placed at the indicator's trend line value *at the moment of entry*.
    *   **For a Long Entry:** The initial stop loss is placed at the `upBand` value on the signal bar.
    *   **For a Short Entry:** The initial stop loss is placed at the `dnBand` value on the signal bar.
    This dynamically adjusts the risk based on the market's recent volatility and trend structure as interpreted by the Hurst-adapted ATR.

*   **Take Profit / Trailing Mechanism:**
    The primary exit mechanism will be a dynamic trailing stop that also leverages the indicator's core component.
    *   **Trailing Stop Logic:** The `trendLine` variable (`upBand` for longs, `dnBand` for shorts) naturally ratchets in the direction of the trend. This value will serve as a sophisticated, volatility-adaptive trailing stop.
        *   For a long position, the stop will be trailed upwards along the `upBand`.
        *   For a short position, the stop will be trailed downwards along the `dnBand`.
    *   **Multi-Stage Take Profit (Optional Scaling):** For strategies requiring partial profit-taking, we can define targets based on ATR multiples from the entry price.
        *   **TP1:** `entry_price + (atr_on_entry * 2)`
        *   **TP2:** `entry_price + (atr_on_entry * 4)`
        At TP1, the strategy could exit 50% of the position, moving the stop loss for the remainder to breakeven. The rest of the position would then be managed by the primary `trendLine` trailing stop.

*   **Time-Based Exits:**
    Unproductive trades tie up capital and introduce overnight risk.
    *   **End of Day (EOD) Exit:** For intraday timeframes (e.g., 1H and below), a non-negotiable rule will be to square all open positions 15-30 minutes before the session close to avoid overnight/weekend gap risk.
    *   **Stagnation Exit:** If a position has been open for `N` bars (e.g., 50) and has not achieved a new equity high (i.e., the trade is moving sideways), it can be closed to free up capital for more promising opportunities.

### 3. Capital Allocation & Risk Management

Position sizing is not an afterthought; it is central to survival and profitability. We will move from fixed-lot trading to a dynamic risk-based model.

*   **Risk-Based Sizing:**
    The strategy will risk a fixed percentage of account equity on every trade. The position size is calculated dynamically based on the distance from the entry price to the initial stop loss.
    *   **Inputs:**
        *   `AccountEquity`: The total current value of the trading account (`strategy.equity`).
        *   `RiskPercent`: A user-defined percentage (e.g., 1.0 for 1% risk per trade).
        *   `RiskPerUnit`: The dollar risk per share/contract, calculated as `abs(entry_price - stop_loss_price)`.
    *   **Formula:**
        `PositionSize = (AccountEquity * (RiskPercent / 100)) / RiskPerUnit`
    *   **Implementation:** This calculation will be performed on the signal bar to determine the quantity for the `strategy.entry` order on the next bar.

*   **Pyramiding & Scaling:**
    Adding to winning positions can significantly enhance returns, but requires strict rules.
    *   **Pyramiding Condition:** A new unit can be added to an existing position only if:
        1.  The initial position is profitable by at least `1.5 * ATR`.
        2.  The `trendLine` has ratcheted up (for longs) or down (for shorts), providing a new, higher-probability support/resistance level.
        3.  The total risk of all open units does not exceed a master risk cap (e.g., 2.5% of equity).
    *   **Scaling Out:** As defined in the take-profit section, the strategy can be configured to sell portions of a position at predefined profit milestones, securing gains while allowing the remainder to capture further trend movement.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the `indicator` into a `strategy` incorporating the professional execution logic described above.

```pine
//@version=5
// --- STRATEGY DECLARATION ---
strategy("Hurst Adaptive Supertrend [Execution Framework]",
     overlay=true,
     process_orders_on_close=true, // IMPORTANT: Execute on next bar's open
     initial_capital=100000,
     commission_type=strategy.commission.percent,
     commission_value=0.075, // Realistic commission (0.075%)
     slippage=2, // Realistic slippage in ticks
     pyramiding=0, // Set to > 0 to enable pyramiding rules
     default_qty_type=strategy.percent_of_equity, // Fallback sizing
     default_qty_value=10)

// --- RISK MANAGEMENT INPUTS ---
riskPercent = input.float(1.0, "Risk per Trade %", minval=0.1, maxval=10, step=0.1, group="Risk Management")
useEodExit  = input.bool(true, "Use End-of-Day Exit?", group="Risk Management")
eodTime     = input.session("1530-1600", "EOD Exit Time", group="Risk Management")

// --- PASTE ORIGINAL SCRIPT'S CALCULATION LOGIC HERE ---
// (Inputs, Hurst Exponent, Kalman Smoother, Trend Band, Trend States)
// ... [Original code from line 21 to 141] ...
// For brevity, we assume the original calculations for 'trend', 'turnedBullish', 'turnedBearish', and 'trendLine' are present.

// --- EXECUTION LOGIC ---

// Determine if the EOD exit condition is met
bool isEod = time(timeframe.period, eodTime) and not time(timeframe.period, eodTime)[1]

// Calculate Position Size based on Risk
stopLossLevel = trendLine[1] // Stop is the trendline of the signal bar
riskPerUnit   = math.abs(close - stopLossLevel)
riskAmount    = (riskPercent / 100) * strategy.equity
positionSize  = riskAmount / riskPerUnit

// --- ENTRY CONDITIONS ---
if (turnedBullish)
    // Close any existing short position before entering long
    strategy.close("Short", comment="Close Short for Reverse")
    // Enter new long position with calculated size
    strategy.entry("Long", strategy.long, qty=positionSize, comment="Enter Long")

if (turnedBearish)
    // Close any existing long position before entering short
    strategy.close("Long", comment="Close Long for Reverse")
    // Enter new short position with calculated size
    strategy.entry("Short", strategy.short, qty=positionSize, comment="Enter Short")

// --- EXIT CONDITIONS ---

// 1. Trailing Stop Loss based on the indicator's trendline
if (strategy.position_size > 0)
    strategy.exit("Exit Long", from_entry="Long", stop=trendLine)

if (strategy.position_size < 0)
    strategy.exit("Exit Short", from_entry="Short", stop=trendLine)

// 2. End-of-Day Exit
if (useEodExit and isEod)
    strategy.close_all(comment="EOD Exit")

// --- VISUALIZATION (Unchanged from original) ---
plot(trendLine, color=trendCol, linewidth=4, style=plot.style_line, title="Hurst Exponent Adaptive Supertrend")
```
    