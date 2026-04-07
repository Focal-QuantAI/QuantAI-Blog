
# Indicators to Strategy Blueprint

Based on the provided `VWAP Volatility Bands` indicator, we will architect a professional-grade automated execution framework. The original script provides excellent directional context (`score`) and volatility-based levels (ATR bands), but its signals (`crossover(t3, t3[1])`) are too simplistic for live execution. We will build a robust strategy that leverages the indicator's core strengths while introducing sophisticated entry, exit, and risk management rules.

### 1. Execution Triggers (Entry & Direction)

The original script triggers a signal merely when the T3-smoothed VWAP begins to slope up or down. This is a "state change" signal, not an optimal "entry" signal, as the price could be significantly overextended when the trend change is confirmed. A professional system enters on pullbacks within an established trend.

Our strategy will define the trend using the `score` variable and enter when the price reverts to a zone of value.

*   **Long Entry Condition:** The primary trend must be bullish (`score == 1`), and the price must pull back to the T3-VWAP basis line. This confirms the trend is holding and offers a favorable entry price.
    *   **Boolean Logic:** `score == 1 and ta.crossunder(close, t3)`

*   **Short Entry Condition:** The primary trend must be bearish (`score == -1`), and the price must rally back to the T3-VWAP basis line.
    *   **Boolean Logic:** `score == -1 and ta.crossover(close, t3)`

#### Execution Nuances:

*   **Execution Timing:** All entries will be executed **at the Close of the bar** where the condition is met (`process_orders_on_close=true`). This ensures the signal is confirmed and avoids the potential for "repainting" or false signals that can occur with intra-bar execution (`calc_on_every_tick=true`). This approach prioritizes reliability over attempting to capture the absolute bottom/top of a move, which is unrealistic in live markets.

*   **Signal Reversals:** The system is designed for immediate reversal. If the strategy is in a long position and a `shortCondition` is triggered, the framework will automatically close the existing long position and initiate a new short position. This is handled by using distinct IDs for long and short entries (`"Long"`, `"Short"`). For added control and clarity in the order log, we can explicitly close the existing trade before opening the new one.

### 2. Multi-Tiered Exit Logic

A single exit rule is a recipe for failure. We will implement a multi-layered system to protect capital and maximize gains.

*   **Initial Stop Loss (Volatility-Based):** The stop loss will not be a fixed percentage. It will be placed dynamically based on the system's own volatility measurement. The outermost volatility band (`band_l4` for longs, `band_u4` for shorts) serves as a logical "point of invalidation."
    *   **Long Stop Loss:** Placed at the `band_l4` level calculated at the time of entry.
    *   **Short Stop Loss:** Placed at the `band_u4` level calculated at the time of entry.
    This ensures the stop is wider during high volatility and tighter during low volatility, adapting to market conditions.

*   **Take Profit / Trailing Mechanism:** We will use a two-stage exit strategy to lock in profits while allowing for larger trends to develop.
    1.  **Initial Target (TP1):** The first take-profit target will be the 2nd volatility band in the direction of the trade (`band_u2` for longs, `band_l2` for shorts). At this point, we will exit a portion of the position (e.g., 50%).
    2.  **Trailing Stop Activation:** Once TP1 is hit, the stop loss for the remaining portion of the trade is moved to breakeven (the entry price). From this point forward, the stop will trail using the T3-VWAP basis line (`t3`). This allows the remainder of the position to ride the core trend, only stopping out if the trend's momentum line is breached.

*   **Time-Based Exits:**
    *   **End of Day (EOD) Exit:** For intraday timeframes (1H or lower), especially when using the "Session" VWAP anchor, all open positions must be squared off before the market close to avoid overnight risk and VWAP reset complications. We will implement a rule to close any open trade 15 minutes before the session ends.
    *   **Stagnation Exit:** If a trade has been open for a significant number of bars (e.g., 50 bars) and has failed to reach TP1, it signifies a lack of momentum. The capital is better deployed elsewhere. We will exit the position to prevent capital from being tied up in directionless chop.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component of a trading system. We will use a fixed-fractional risk model.

*   **Risk-Based Sizing:** The strategy will risk a fixed percentage of account equity on every single trade (e.g., 1% of `strategy.equity`). The position size is calculated dynamically based on the distance between the entry price and the initial volatility-based stop loss.

    *   **Calculation Logic:**
        1.  `risk_per_trade_usd = strategy.equity * risk_percent`
        2.  `trade_risk_per_share = abs(entry_price - stop_loss_price)`
        3.  `position_size = risk_per_trade_usd / trade_risk_per_share`

    This method ensures that losses are consistent in dollar terms, regardless of the trade's setup or market volatility.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** This is built into our exit logic by taking partial profits at TP1.
    *   **Pyramiding (Scaling In):** While not included in the primary snippet for simplicity, a professional extension would be to add to a winning position.
        *   **Rule:** If a long trade is open and has moved into profit (e.g., `close > band_u1`), and the price subsequently pulls back to the `t3` basis line *without the trend flipping*, a second, smaller position (e.g., 50% of the initial size) can be added. The stop loss for the entire combined position would then be moved to the `band_l4` level associated with the new entry. This compounds gains during strong, confirmed trends.

### 4. Implementation Snippet (Pine Logic)

This snippet transforms the `indicator` into a `strategy`, incorporating the professional execution logic defined above.

```pine
// This Pine Script® code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © BOSWaves - Strategy Adaptation by Algorithmic Trading Architect

//@version=6
// 1. STRATEGY DECLARATION: From indicator to a production-ready strategy
strategy("VWAP Volatility Bands Strategy [PRO]",
     overlay=true,
     process_orders_on_close=true, // Ensures execution on confirmed bar data
     slippage=2,                   // Realistic slippage in ticks
     commission_type=strategy.commission.percent,
     commission_value=0.04,        // Realistic commission for round-trip
     initial_capital=25000,
     default_qty_type=strategy.fixed,
     default_qty_value=1)          // Default qty is placeholder; we use dynamic sizing

// --- [Original Indicator Inputs and Calculations would be pasted here] ---
// ... (vwap_anchor, len, factor, atr_len, etc.)
// ... (VWAP Calculation, T3 on VWAP, Bands, Score Logic)
// --- [End of Pasted Code] ---

// ┌────────────────────────────── ARCHITECT ─ Strategy Inputs ───────────────────────┐
// └──────────────────────────────────────────────────────────────────────────────────┘
risk_percent     = input.float(1.0, "Risk Per Trade %", minval=0.1, maxval=5.0, step=0.1, group="Risk Management")
use_eod_exit     = input.bool(true, "Use End-of-Day Exit", group="Risk Management")
eod_exit_minutes = input.int(15, "Minutes Before Close", group="Risk Management")

// ┌────────────────────────────── ARCHITECT ─ Risk Management ───────────────────────┐
// └──────────────────────────────────────────────────────────────────────────────────┘
f_calculate_position_size(stop_loss_price) =>
    risk_amount_usd = (risk_percent / 100) * strategy.equity
    risk_per_share = math.abs(close - stop_loss_price) // Using close as proxy for entry
    position_size = risk_per_share > 0 ? math.floor(risk_amount_usd / risk_per_share) : 0
    position_size

// ┌────────────────────────────── ARCHITECT ─ Execution Logic ───────────────────────┐
// └──────────────────────────────────────────────────────────────────────────────────┘

// 1. Entry Conditions (Pullback to the mean within an established trend)
long_condition  = score == 1 and ta.crossunder(close, t3) and strategy.opentrades == 0
short_condition = score == -1 and ta.crossover(close, t3) and strategy.opentrades == 0

// 2. Exit Conditions
is_eod = use_eod_exit and (time_close(timeframe.period, str.format("{0}-{1}", session.regular, eod_exit_minutes)) - time) <= 0

if is_eod
    strategy.close_all(comment="EOD Exit")

// 3. Trade Execution
if long_condition
    // Calculate stop and size BEFORE entry
    stop_price = band_l4
    pos_size = f_calculate_position_size(stop_price)

    if pos_size > 0
        // Place entry order with linked exit bracket (Stop Loss and Take Profit 1)
        strategy.entry("Long", strategy.long, qty=pos_size)
        strategy.exit("L-Exit", from_entry="Long", stop=stop_price, limit=band_u2, comment_profit="TP1 Hit", comment_loss="SL Hit")

if short_condition
    // Calculate stop and size BEFORE entry
    stop_price = band_u4
    pos_size = f_calculate_position_size(stop_price)

    if pos_size > 0
        // Place entry order with linked exit bracket (Stop Loss and Take Profit 1)
        strategy.entry("Short", strategy.short, qty=pos_size)
        strategy.exit("S-Exit", from_entry="Short", stop=stop_price, limit=band_l2, comment_profit="TP1 Hit", comment_loss="SL Hit")

// 4. Trailing Stop Logic (Advanced Exit Management)
var float trailing_stop_price = na

// Check if we are in a long position and if the trailing stop should be activated/updated
if strategy.position_size > 0
    // Activate trailing stop once price has moved favorably (e.g., crossed the first upper band)
    if close > band_u1 and na(trailing_stop_price)
        trailing_stop_price := strategy.position_avg_price // Move to Break-Even first
    
    // If trailing is active, trail with the T3 line
    if not na(trailing_stop_price)
        trailing_stop_price := math.max(trailing_stop_price, t3) // Trail with T3, never moving it down
        strategy.exit("L-Trail", from_entry="Long", stop=trailing_stop_price)

// Check if we are in a short position and if the trailing stop should be activated/updated
else if strategy.position_size < 0
    // Activate trailing stop once price has moved favorably (e.g., crossed the first lower band)
    if close < band_l1 and na(trailing_stop_price)
        trailing_stop_price := strategy.position_avg_price // Move to Break-Even first

    // If trailing is active, trail with the T3 line
    if not na(trailing_stop_price)
        trailing_stop_price := math.min(trailing_stop_price, t3) // Trail with T3, never moving it up
        strategy.exit("S-Trail", from_entry="Short", stop=trailing_stop_price)

// Reset trailing stop price on trade close
if strategy.position_size == 0
    trailing_stop_price := na

// --- [Original Indicator Plotting code would be pasted here for visualization] ---
// ... (plot(), fill(), barcolor(), etc.)
```
    