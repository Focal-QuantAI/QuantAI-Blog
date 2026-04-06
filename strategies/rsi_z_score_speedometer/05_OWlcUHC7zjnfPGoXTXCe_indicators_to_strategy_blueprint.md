
# Indicators to Strategy Blueprint

The provided script is a sophisticated visualization tool, displaying RSI and Z-Score values as speedometers. While excellent for discretionary analysis, it contains no execution logic. To transform this into a production-ready automated framework, we must extract the core metrics—RSI for momentum and Z-Score for mean-reversion—and build a robust trading engine around them.

The proposed strategy will be a **Mean-Reversion system with Momentum Confirmation**. It will aim to enter trades when the price is statistically overextended (identified by the Z-Score) and shows initial signs of reverting (confirmed by the RSI).

### 1. Execution Triggers (Entry & Direction)

The core premise is to fade extreme price moves. We will use the Z-Score to identify a statistical anomaly and the RSI to confirm that momentum is beginning to shift, providing a trigger for entry.

*   **Long Entry Condition:** A `Long` position is initiated when the price is statistically oversold and momentum begins to turn bullish.
    *   `z_raw < Z_Score_Threshold_Lower`: The price has dropped significantly below its recent mean (e.g., more than 2 standard deviations).
    *   `ta.crossover(rsi_metric, RSI_Confirm_Lower)`: The RSI, having been in an oversold state, crosses back up over a confirmation level (e.g., 30), indicating that selling pressure is waning and buying interest is emerging.

*   **Short Entry Condition:** A `Short` position is initiated when the price is statistically overbought and momentum begins to turn bearish.
    *   `z_raw > Z_Score_Threshold_Upper`: The price has risen significantly above its recent mean (e.g., more than 2 standard deviations).
    *   `ta.crossunder(rsi_metric, RSI_Confirm_Upper)`: The RSI, having been in an overbought state, crosses back down below a confirmation level (e.g., 70), indicating that buying pressure is exhausted.

#### Execution Nuances

*   **Execution Timing:** All entry and exit conditions will be evaluated **at the close of the bar** (`process_orders_on_close = true`). This is a standard professional practice that prevents signal flickering and ensures backtest results are reproducible and realistic. An order generated on the close of Bar A is filled at the open of Bar B.
*   **Signal Reversals:** The system is designed to be "always in" or to flip positions. If the strategy is in a `Long` position and a valid `Short Entry Condition` occurs, the framework will immediately close the existing long position and initiate a new short position. This is handled by issuing a new `strategy.entry` command for the opposite direction, which automatically reverses the current market position.

### 2. Multi-Tiered Exit Logic

A static exit strategy is a primary cause of failure. A professional system must adapt to market volatility and protect profits dynamically.

*   **Initial Stop Loss (Volatility-Based):** The initial stop loss will not be a fixed percentage but will be calculated dynamically using the **Average True Range (ATR)**.
    *   **Logic:** Upon entry, the stop loss is placed at a multiple of the ATR away from the entry price.
    *   **Long Stop Price:** `entry_price - (ATR_Multiplier * atr_value)`
    *   **Short Stop Price:** `entry_price + (ATR_Multiplier * atr_value)`
    *   A typical `ATR_Multiplier` is between 2 and 3, providing enough room to absorb market noise without taking on excessive risk.

*   **Take Profit & Trailing Stop (Hybrid Model):** We will implement a two-stage profit-taking mechanism to secure gains while allowing for larger wins.
    *   **Stage 1 (Initial Target & Risk Reduction):** A primary take-profit target (TP1) is set at a specific Risk/Reward ratio, for instance, 1.5 times the initial risk (`1.5R`). When TP1 is hit, a portion of the position (e.g., 50%) is closed. Simultaneously, the stop loss on the remaining position is moved to the **breakeven** point. This makes the rest of the trade risk-free.
    *   **Stage 2 (Dynamic Trailing):** The remaining portion of the trade is managed with an **ATR Trailing Stop**. The stop will trail the price by a set ATR multiple (e.g., 2.5 * ATR), but it will only move in the direction of the trade. This allows the system to capture a significant portion of a sustained reversal move.

*   **Time-Based Exits:**
    *   **End of Day (EOD) Exit:** For intraday strategies (on timeframes H1 or lower), it is critical to include a hard exit time to avoid overnight/weekend risk. The strategy will be programmed to close any open positions a few minutes before the session close (e.g., 16:50 EST for US Equities).
    *   **Stagnation Exit:** Capital should not be tied up in unproductive trades. An exit will be triggered if a position has been open for a specified number of bars (`Max_Bars_In_Trade`) without hitting either its stop loss or its primary take-profit target.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component of a trading system's long-term viability. We will abandon fixed-lot sizing in favor of a precise risk-based model.

*   **Risk-Based Sizing:** The strategy will risk a fixed percentage of the total account equity on every single trade.
    *   **Logic:**
        1.  Define a risk percentage (e.g., 1% of `strategy.equity`).
        2.  Calculate the stop-loss distance in currency terms (`entry_price - stop_loss_price`).
        3.  The position size is then calculated to ensure that if the stop loss is hit, the loss incurred is exactly the predefined risk amount.
    *   **Formula:** `Position Size = (Equity * Risk %) / (Stop Distance in Points * Value per Point)`
    *   This method normalizes risk across all trades, regardless of the asset's volatility or price.

*   **Pyramiding & Scaling:**
    *   **Pyramiding (Scaling In):** **Not recommended** for this strategy. Mean-reversion strategies are inherently contrarian. Adding to a position as it moves in your favor (away from the mean) contradicts the core thesis that the price will revert. Pyramiding is best suited for trend-following systems.
    *   **Scaling Out:** **Integral** to the exit logic. As described in the exit strategy, we will scale out by closing 50% of the position at the first take-profit target (TP1). This improves the strategy's win rate and smooths the equity curve.

### 4. Implementation Snippet (Pine Logic)

The following code block demonstrates the transformation of the visual indicator into a complete strategy, incorporating the professional-grade components discussed. All visual drawing functions have been removed to focus purely on the execution engine.

```pine
//@version=5
// --- STRATEGY DECLARATION ---
strategy(
     "RSI & Z-Score Mean Reversion Engine", 
     overlay=true, 
     process_orders_on_close=true,
     initial_capital=100000,
     default_qty_type=strategy.fixed, // We will calculate quantity manually
     commission_type=strategy.commission.percent,
     commission_value=0.04, // Realistic commission for retail brokers
     slippage=2 // 2 ticks of slippage per order
     )

// --- STRATEGY INPUTS ---
// Risk Management
risk_percent = input.float(1.0, title="Risk Per Trade %", minval=0.1, maxval=5.0, step=0.1)
// Z-Score Parameters
z_len   = input.int(20, title="Z-Score Length", minval=2)
z_entry_threshold = input.float(2.0, title="Z-Score Entry Threshold (±)", minval=1.0, step=0.1)
// RSI Parameters
rsi_len = input.int(14, title="RSI Length")
rsi_confirm_upper = input.int(70, title="RSI Bearish Confirmation Level")
rsi_confirm_lower = input.int(30, title="RSI Bullish Confirmation Level")
// Exit Parameters
atr_len = input.int(14, title="ATR Length for Stops")
atr_stop_multiplier = input.float(2.5, title="ATR Stop Loss Multiplier")
rr_tp1 = input.float(1.5, title="Take Profit 1 Risk/Reward Ratio")
max_bars_in_trade = input.int(100, title="Max Bars in Stagnant Trade")

// --- INDICATOR CALCULATIONS ---
// RSI Metric
rsi_metric = ta.rsi(close, rsi_len)
// Z-Score Metric
z_mean = ta.sma(close, z_len)
z_std  = ta.stdev(close, z_len)
z_raw  = z_std != 0.0 ? (close - z_mean) / z_std : 0.0
// Volatility
atr_val = ta.atr(atr_len)

// --- ENTRY CONDITIONS ---
longCondition = z_raw < -z_entry_threshold and ta.crossover(rsi_metric, rsi_confirm_lower)
shortCondition = z_raw > z_entry_threshold and ta.crossunder(rsi_metric, rsi_confirm_upper)

// --- RISK MANAGEMENT & POSITION SIZING ---
stop_distance_points = atr_val * atr_stop_multiplier
risk_per_trade_currency = (risk_percent / 100) * strategy.equity
position_size = risk_per_trade_currency / (stop_distance_points * syminfo.pointvalue)

// --- EXECUTION LOGIC ---
// Store stop and target levels in variables to manage exits
var float long_stop_price = na
var float long_tp1_price = na
var float short_stop_price = na
var float short_tp1_price = na

// Long Entry Logic
if (longCondition and strategy.position_size == 0)
    long_stop_price := close - stop_distance_points
    long_tp1_price := close + (stop_distance_points * rr_tp1)
    strategy.entry("Long", strategy.long, qty=position_size)
    // Place initial SL and TP1 orders for 50% of the position
    strategy.exit("Long TP1/SL", from_entry="Long", qty_percent=50, profit=long_tp1_price, loss=long_stop_price)

// Short Entry Logic
if (shortCondition and strategy.position_size == 0)
    short_stop_price := close + stop_distance_points
    short_tp1_price := close - (stop_distance_points * rr_tp1)
    strategy.entry("Short", strategy.short, qty=position_size)
    // Place initial SL and TP1 orders for 50% of the position
    strategy.exit("Short TP1/SL", from_entry="Short", qty_percent=50, profit=short_tp1_price, loss=short_stop_price)

// --- EXIT MANAGEMENT ---
// Breakeven & Trailing Stop Logic
var bool long_tp1_hit = false
var bool short_tp1_hit = false

if strategy.position_size > 0
    // Check if TP1 was hit on the current bar
    if not long_tp1_hit and high >= long_tp1_price
        long_tp1_hit := true
    // If TP1 has been hit, manage the remainder with a trailing stop (move to breakeven first)
    if long_tp1_hit
        breakeven_price = strategy.opentrades.entry_price(0)
        trailing_stop_price = math.max(breakeven_price, high - stop_distance_points) // ATR Trail
        strategy.exit("Long Trail", from_entry="Long", loss=trailing_stop_price)
    // Stagnation Exit
    if barssince(longCondition) > max_bars_in_trade
        strategy.close("Long", comment="Stagnation Exit")

if strategy.position_size < 0
    // Check if TP1 was hit on the current bar
    if not short_tp1_hit and low <= short_tp1_price
        short_tp1_hit := true
    // If TP1 has been hit, manage the remainder with a trailing stop (move to breakeven first)
    if short_tp1_hit
        breakeven_price = strategy.opentrades.entry_price(0)
        trailing_stop_price = math.min(breakeven_price, low + stop_distance_points) // ATR Trail
        strategy.exit("Short Trail", from_entry="Short", loss=trailing_stop_price)
    // Stagnation Exit
    if barssince(shortCondition) > max_bars_in_trade
        strategy.close("Short", comment="Stagnation Exit")

// Reset TP hit flags when flat
if strategy.position_size == 0
    long_tp1_hit := false
    short_tp1_hit := false

// EOD Exit (Example for a specific time)
is_eod_exit = time_close("1D") > 0 and hour == 15 and minute >= 50 // Example: Exit at 15:50
if is_eod_exit
    strategy.close_all(comment="EOD Exit")
```
    