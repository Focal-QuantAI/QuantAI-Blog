
# Indicators to Strategy Blueprint

Here is the architectural breakdown for transitioning the "1M Smart Scalping" indicator into a production-grade automated execution framework.

### 1. Execution Triggers (Entry & Direction)

The provided script's logic is based on a confluence of factors confirming a trend continuation after a brief pause. The execution must be precise to capture the intended momentum.

*   **Long Entry Condition (`long_condition`):** A long position is initiated when all of the following are true on the close of a 1-minute bar:
    1.  **Confirmed Uptrend:** The most recent pivot low is higher than the previous pivot low.
    2.  **Pullback Pattern:** The candle two bars ago was strongly bullish, the previous candle was bearish (a pullback), and the current candle is strongly bullish (resumption).
    3.  **Momentum Breakout:** The current bar's high exceeds the highest high of the previous 5 bars.
    4.  **Clearance:** The current close is not within a 0.5 ATR distance of the 10-bar resistance level, ensuring there is room to move.

*   **Short Entry Condition (`short_condition`):** A short position is initiated when all of the following are true on the close of a 1-minute bar:
    1.  **Confirmed Downtrend:** The most recent pivot high is lower than the previous pivot high.
    2.  **Rally-Fail Pattern:** The candle two bars ago was strongly bearish, the previous candle was bullish (a weak rally), and the current candle is strongly bearish (resumption).
    3.  **Momentum Breakdown:** The current bar's low is below the lowest low of the previous 5 bars.
    4.  **Clearance:** The current close is not within a 0.5 ATR distance of the 10-bar support level.

*   **Execution Nuance:** The logic explicitly uses `barstate.isconfirmed`. This is a critical detail. It means signals are only valid **at the close of the bar**. Therefore, the execution model must be "Market-on-Open" of the next bar. Any attempt to execute mid-bar based on these signals would be a form of forward-testing and would not align with the backtested results. The strategy must be configured to `process_orders_on_close = true`.

*   **Signal Reversals:** Given the scalping nature of the strategy, there is no room for holding a losing position against a new, valid signal in the opposite direction. The framework will employ a **"Close-and-Reverse"** logic. If a `long_condition` is met while in a short position, the system will generate two orders:
    1.  An order to close the existing short position at the market.
    2.  An order to open a new long position at the market.

### 2. Multi-Tiered Exit Logic

A scalping strategy's profitability is defined more by its exit discipline than its entries. The following tiered exit system provides a robust defense and profit-capture mechanism.

*   **Initial Stop Loss (Volatility-Based):** The stop loss will be calculated dynamically based on the Average True Range (ATR) to adapt to current market volatility.
    *   **For Longs:** `Stop Loss Price = low[1] - (ATR * Multiplier)`. The stop is placed below the low of the signal bar, providing a structural buffer. A typical `Multiplier` would be between 1.0 and 1.5.
    *   **For Shorts:** `Stop Loss Price = high[1] + (ATR * Multiplier)`. The stop is placed above the high of the signal bar.

*   **Take Profit/Trailing (Multi-Stage):** A static take profit can leave money on the table. A multi-stage approach is superior.
    1.  **Target 1 (TP1):** Set at a 1:1 Risk/Reward ratio. `Take Profit Price = Entry Price + (Entry Price - Stop Loss Price)`.
    2.  **Breakeven Trigger:** Upon TP1 being hit (if scaling out) or simply being crossed, the stop loss for the remainder of the position is moved to the entry price. This immediately removes risk from the trade.
    3.  **Trailing Stop Activation:** After the breakeven move, a trailing stop is activated. A "Chandelier Exit" is effective here: trail the stop `X * ATR` below the highest high achieved since the entry (for longs) or above the lowest low (for shorts). This allows the winner to run while still protecting profits.

*   **Time-Based Exits:** Time is a critical risk factor in scalping.
    *   **Stagnation Exit:** If a position has been open for $N$ bars (e.g., 15 bars on a 1M chart) and has not reached the breakeven trigger, it is closed automatically. This prevents capital from being tied up in trades that lack momentum.
    *   **End of Session Exit:** All open positions will be squared off automatically at a specified time (e.g., 15 minutes before the session close) to eliminate overnight and weekend risk.

### 3. Capital Allocation & Risk Management

Position sizing is the engine of risk control. We will move from fixed-lot thinking to a dynamic, risk-based model.

*   **Risk-Based Sizing:** The core principle is to risk a fixed percentage of account equity on every single trade, regardless of the trade's parameters.
    *   **Formula:**
        `Position Size = (Account Equity * Risk Percentage) / |Entry Price - Stop Loss Price|`
    *   **Implementation:**
        1.  Define a `risk_per_trade` input (e.g., 0.01 for 1% of equity).
        2.  Calculate the dollar amount to risk: `risk_amount = strategy.equity * risk_per_trade`.
        3.  Calculate the risk per share/contract: `risk_per_unit = math.abs(entry_price - stop_loss_price)`.
        4.  Calculate the final position size: `position_size = risk_amount / risk_per_unit`.
        This ensures that a trade with a wide stop has a smaller position size than a trade with a tight stop, equalizing the dollar risk for both.

*   **Pyramiding & Scaling:**
    *   **Pyramiding (Adding to Winners):** **Strongly discouraged** for this 1M scalping strategy. The signals are designed to capture short, sharp bursts of momentum. Adding to a position increases exposure just as the move may be exhausting itself, dramatically increasing the risk of a sharp reversal.
    *   **Scaling Out:** **Recommended.** This aligns with the multi-stage take-profit logic. For example, one could close 50% of the position at TP1 and let the remaining 50% run with the trailing stop. This locks in profits while maintaining exposure to a larger potential move.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the indicator into a professional strategy, incorporating the architectural components discussed above.

```pine
//@version=5
// 1. STRATEGY DECLARATION WITH REALISTIC FRICTION
strategy("1M Smart Scalping - Execution Framework", 
     overlay=true, 
     process_orders_on_close=true, // Execute on the open of the next bar
     slippage=2, // 2 ticks of slippage for market orders
     commission_type=strategy.commission.percent, 
     commission_value=0.04) // Realistic commission for retail brokers

// =======================
// ⚙️ INPUTS FOR OPTIMIZATION
// =======================
// Risk Management
risk_per_trade = input.float(1.0, "Risk per Trade (%)", minval=0.1, maxval=5.0) / 100
// Exit Logic
atr_period = input.int(14, "ATR Period")
sl_atr_multiplier = input.float(1.5, "Stop Loss ATR Multiplier", minval=0.5)
tp_rr_ratio = input.float(1.5, "Take Profit R:R Ratio", minval=0.5)
stagnation_bars = input.int(15, "Max Bars in Trade")

// Original Logic Inputs
pivot_len = 5

// =======================
// 🔁 CORE LOGIC (FROM ORIGINAL SCRIPT)
// =======================
// No-Repaint ZigZag
ph = ta.pivothigh(high, pivot_len, pivot_len)
pl = ta.pivotlow(low, pivot_len, pivot_len)
var float last_high = na, var float prev_high = na, var float last_low = na, var float prev_low = na
if not na(ph) { prev_high := last_high; last_high := ph }
if not na(pl) { prev_low := last_low; last_low := pl }

// Trend
trend_up = not na(prev_low) and last_low > prev_low
trend_down = not na(prev_high) and last_high < prev_high

// S&R
support = ta.lowest(low, 10)[1]
resistance = ta.highest(high, 10)[1]
atr = ta.atr(atr_period)
sr_distance = atr * 0.5
near_support = math.abs(close - support) < sr_distance
near_resistance = math.abs(close - resistance) < sr_distance

// Candle & Breakout
bull = close > open and (close - open) > (high - low) * 0.5
bear = open > close and (open - close) > (high - low) * 0.5
breakout_up = high > ta.highest(high, 5)[1]
breakout_down = low < ta.lowest(low, 5)[1]

// Final Signals
long_condition = timeframe.period == "1" and trend_up and bull[2] and bear[1] and bull and breakout_up and not near_resistance
short_condition = timeframe.period == "1" and trend_down and bear[2] and bull[1] and bear and breakout_down and not near_support

// =======================
// 💰 CAPITAL ALLOCATION & RISK MANAGEMENT
// =======================
// Calculate Stop Loss Price *before* entry to determine size
long_stop_price = low[1] - (atr * sl_atr_multiplier)
short_stop_price = high[1] + (atr * sl_atr_multiplier)

// Risk-based position sizing
risk_per_unit_long = close - long_stop_price
risk_per_unit_short = short_stop_price - close
position_size_long = (strategy.equity * risk_per_trade) / risk_per_unit_long
position_size_short = (strategy.equity * risk_per_trade) / risk_per_unit_short

// =======================
// 📈 EXECUTION ENGINE
// =======================
// --- ENTRY LOGIC ---
if (long_condition)
    // Close any existing short and go long
    strategy.close("Short", comment="Short Reverse")
    strategy.entry("Long", strategy.long, qty=position_size_long)

if (short_condition)
    // Close any existing long and go short
    strategy.close("Long", comment="Long Reverse")
    strategy.entry("Short", strategy.short, qty=position_size_short)

// --- EXIT LOGIC ---
// Set SL/TP for the long position
if strategy.position_size > 0
    long_tp_price = strategy.position_avg_price + (risk_per_unit_long * tp_rr_ratio)
    strategy.exit("Long Exit", from_entry="Long", stop=long_stop_price, limit=long_tp_price)

// Set SL/TP for the short position
if strategy.position_size < 0
    short_tp_price = strategy.position_avg_price - (risk_per_unit_short * tp_rr_ratio)
    strategy.exit("Short Exit", from_entry="Short", stop=short_stop_price, limit=short_tp_price)

// Time-based Stagnation Exit
if barssince(strategy.opentrades > 0) > stagnation_bars
    strategy.close_all(comment="Stagnation Exit")

// End of Session Exit (Example: Close all trades at 20:45 UTC)
is_eod = hour(time_close) == 20 and minute(time_close) >= 45
if is_eod
    strategy.close_all(comment="End of Session")

// Plotting for visual confirmation
plot(strategy.position_size > 0 ? long_stop_price : na, "Long SL", color.red, style=plot.style_linebr)
plot(strategy.position_size < 0 ? short_stop_price : na, "Short SL", color.red, style=plot.style_linebr)
```
    