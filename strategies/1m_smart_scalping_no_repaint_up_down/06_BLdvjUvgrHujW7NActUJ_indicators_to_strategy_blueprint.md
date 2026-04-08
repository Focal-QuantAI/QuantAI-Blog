
# Indicators to Strategy Blueprint

Here is the transformation of the "1M Smart Scalping" indicator into a production-ready algorithmic trading framework.

### 1. Execution Triggers (Entry & Direction)

The provided script's logic is sound but requires translation into explicit order commands. The core signals are generated from a confluence of trend, a specific 3-bar momentum pattern, a breakout, and avoidance of immediate support/resistance.

*   **Long Entry Condition:** A long position will be initiated when `up_signal` becomes `true`. This requires all of the following conditions to be met on the most recently closed 1-minute bar:
    1.  **Trend:** The established trend is up (current pivot low is higher than the previous pivot low).
    2.  **Candle Pattern:** A specific three-bar sequence has occurred: a strong bullish candle two bars ago, a bearish retracement candle one bar ago, and a strong bullish candle on the current signal bar.
    3.  **Confirmation:** The signal bar is also a breakout above the high of the last 5 bars.
    4.  **Filter:** The closing price is not within an ATR-based proximity to the 10-bar resistance level.

*   **Short Entry Condition:** A short position will be initiated when `down_signal` becomes `true`. This is the mirror logic:
    1.  **Trend:** The established trend is down (current pivot high is lower than the previous pivot high).
    2.  **Candle Pattern:** A three-bar sequence of strong bearish, bullish retracement, and strong bearish candles.
    3.  **Confirmation:** The signal bar is a breakout below the low of the last 5 bars.
    4.  **Filter:** The closing price is not within an ATR-based proximity to the 10-bar support level.

*   **Execution Nuance:** The script correctly uses `barstate.isconfirmed`. This dictates that all logic is evaluated and orders are sent **at the close of the bar**. This is the only reliable way to execute a strategy based on non-repainting historical data (`[1]`, `[2]`) and ensures that backtest results align with potential live performance. Attempting to execute this logic intra-bar would lead to significant repainting and unreliable signals.

*   **Signal Reversals:** For a scalping strategy, immediate position flips are critical. If the system is in a long position and a `down_signal` occurs, the framework must execute two orders in sequence:
    1.  `strategy.close("Long")`: An order to close the existing long position at the market.
    2.  `strategy.entry("Short", strategy.short)`: An order to open a new short position.
    This ensures risk and position size are recalculated for the new trade direction, rather than simply reversing the existing position.

### 2. Multi-Tiered Exit Logic

The original script contains no exit logic, which is the most critical component for profitability and risk control. A professional framework must incorporate a layered approach.

*   **Initial Stop Loss (Volatility-Based):** An arbitrary percentage or point-based stop is inadequate for a 1-minute chart where volatility can change drastically. The stop loss will be calculated dynamically using the Average True Range (ATR).
    *   **Long Stop Loss:** `entry_price_low - (ATR * 1.5)`. The stop is placed 1.5x the current 14-period ATR value below the low of the signal candle. This places it outside the immediate noise zone.
    *   **Short Stop Loss:** `entry_price_high + (ATR * 1.5)`. The stop is placed 1.5x ATR above the high of the signal candle.

*   **Take Profit / Trailing Mechanism:** A static take-profit can cut winning trades short. A dynamic trailing stop is superior for scalping.
    1.  **Initial Profit Target (TP1):** Set an initial take-profit at a 1.5:1 Risk/Reward Ratio. For example, if the distance from entry to the initial stop loss is 10 points, TP1 is set at `entry_price + 15` points.
    2.  **Trailing Stop Activation:** Once price hits TP1, the initial stop loss is cancelled and a dynamic trailing stop is activated. A robust method is a **Chandelier Exit**:
        *   **Trailing Long:** The stop is trailed at `highest(high, X) - (ATR * Y)`, where `X` is the number of bars since the trade entry and `Y` is an ATR multiplier (e.g., 2.0). The stop only moves up, never down.
        *   **Trailing Short:** The stop is trailed at `lowest(low, X) + (ATR * Y)`.

*   **Time-Based Exits:** Scalping positions should not be held indefinitely.
    *   **Stagnation Exit:** If a position has been open for more than 20 bars and has not hit either the stop loss or the initial take profit, it is considered a "dead trade." The position will be closed at the market to free up capital and reduce exposure to random events.
    *   **End of Day (EOD) Exit:** All open positions will be squared off 15 minutes before the session close to avoid overnight risk, funding charges, and gap risk on the next day's open.

### 3. Capital Allocation & Risk Management

Position sizing is not an afterthought; it is a core component of the strategy's architecture.

*   **Risk-Based Sizing:** The strategy will risk a fixed percentage of account equity on every trade, standardizing risk regardless of the trade's specific stop-loss distance.
    *   **Formula:**
        `Risk_Amount = Account_Equity * Risk_Per_Trade_Percent`
        `Trade_Risk_Per_Share = abs(Entry_Price - Stop_Loss_Price)`
        `Position_Size = Risk_Amount / Trade_Risk_Per_Share`
    *   **Example:** With a $10,000 account and a 1% risk setting, the risk per trade is $100. If the distance from entry to the ATR-based stop loss is $0.50, the position size would be `$100 / $0.50 = 200` shares.

*   **Pyramiding & Scaling:**
    *   **Pyramiding (Scaling In):** **Not recommended** for this 1-minute scalping strategy. The holding period is too short, and the logic is designed for a single, precise entry point. Adding to the position (pyramiding) introduces significant complexity, increases the average entry price, and magnifies risk in a fast-moving environment.
    *   **Scaling Out:** This is a viable alternative to the single TP/Trailing Stop model. The position could be exited in stages:
        *   Exit 50% of the position at TP1 (e.g., 1.5R).
        *   Move the stop loss for the remaining 50% to breakeven.
        *   Trail the stop for the remainder using the Chandelier Exit described above to capture a larger move.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion from an `indicator` to a `strategy` incorporating the professional-grade execution logic.

```pine
//@version=5
// 1. STRATEGY DECLARATION - From Indicator to a realistic execution engine
strategy("1M Pro Scalping Framework", 
     overlay=true, 
     pyramiding=0, // No pyramiding allowed
     initial_capital=10000, 
     default_qty_type=strategy.cash, // We will calculate size manually
     commission_type=strategy.commission.cash_per_order,
     commission_value=1.00, // Realistic per-order commission
     slippage=2) // 2 ticks of slippage for 1M timeframe

// =======================
// ⚙️ INPUTS & CORE LOGIC (Unchanged from original script)
// =======================
pivot_len = input.int(5, "Pivot Length")
atr_period = input.int(14, "ATR Period")
atr_sl_multiplier = input.float(1.5, "ATR Stop Loss Multiplier")
atr_sr_multiplier = input.float(0.5, "ATR S/R Filter Multiplier")
stagnation_bars = input.int(20, "Max Bars in Trade")
risk_percent = input.float(1.0, "Risk % Per Trade")

// Original Logic
ph = ta.pivothigh(high, pivot_len, pivot_len)
pl = ta.pivotlow(low, pivot_len, pivot_len)
var float last_high = na, var float prev_high = na, var float last_low = na, var float prev_low = na
if not na(ph) { prev_high := last_high; last_high := ph }
if not na(pl) { prev_low := last_low; last_low := pl }
trend_up = not na(prev_low) and last_low > prev_low
trend_down = not na(prev_high) and last_high < prev_high
support = ta.lowest(low, 10)[1]
resistance = ta.highest(high, 10)[1]
atr = ta.atr(atr_period)
sr_distance = atr * atr_sr_multiplier
near_support = math.abs(close - support) < sr_distance
near_resistance = math.abs(close - resistance) < sr_distance
bull = close > open and (close - open) > (high - low) * 0.5
bear = open > close and (open - close) > (high - low) * 0.5
breakout_up = high > ta.highest(high, 5)[1]
breakout_down = low < ta.lowest(low, 5)[1]

// Final Signals (barstate.isconfirmed is implicit in strategy execution on bar close)
up_signal = timeframe.period == "1" and trend_up and bull[2] and bear[1] and bull and breakout_up and not near_resistance
down_signal = timeframe.period == "1" and trend_down and bear[2] and bull[1] and bear and breakout_down and not near_support

// =======================
// 2. RISK & EXIT CALCULATION
// =======================
// Function to calculate position size based on risk
f_getPosSize(risk_capital, price_risk) =>
    price_risk > 0 ? math.floor(risk_capital / price_risk) : 0

// Calculate Stop Loss and Position Size for potential trades
long_stop_price = low - (atr * atr_sl_multiplier)
short_stop_price = high + (atr * atr_sl_multiplier)
long_tp_price = close + (close - long_stop_price) * 1.5 // 1.5R Take Profit
short_tp_price = close - (short_stop_price - close) * 1.5 // 1.5R Take Profit

risk_capital_per_trade = (strategy.equity * risk_percent) / 100
long_pos_size = f_getPosSize(risk_capital_per_trade, close - long_stop_price)
short_pos_size = f_getPosSize(risk_capital_per_trade, short_stop_price - close)

// Time-based Exit Conditions
is_eod = hour(time_close) == 15 and minute(time_close) >= 45 // Example for US Equities (close at 16:00)
is_stagnant = barssince(strategy.opentrades > 0) > stagnation_bars

// =======================
// 3. EXECUTION ENGINE
// =======================
// --- ENTRY LOGIC ---
if (up_signal)
    // Handle reversal: close short before entering long
    if (strategy.position_size < 0)
        strategy.close("Short", comment="Short Reversal")
    // Enter new long position
    strategy.entry("Long", strategy.long, qty=long_pos_size)
    // Place SL/TP bracket order for the new long position
    strategy.exit("Long Exit", from_entry="Long", stop=long_stop_price, limit=long_tp_price)

if (down_signal)
    // Handle reversal: close long before entering short
    if (strategy.position_size > 0)
        strategy.close("Long", comment="Long Reversal")
    // Enter new short position
    strategy.entry("Short", strategy.short, qty=short_pos_size)
    // Place SL/TP bracket order for the new short position
    strategy.exit("Short Exit", from_entry="Short", stop=short_stop_price, limit=short_tp_price)

// --- TIME-BASED EXIT LOGIC ---
if (is_eod or is_stagnant)
    strategy.close_all(comment = is_eod ? "EOD Close" : "Stagnation Exit")

// =======================
// 4. VISUALIZATION (Optional)
// =======================
bgcolor(strategy.position_size > 0 ? color.new(color.blue, 90) : strategy.position_size < 0 ? color.new(color.purple, 90) : na)
plot(strategy.position_size > 0 ? long_stop_price : na, "SL", color.red, style=plot.style_linebr)
plot(strategy.position_size > 0 ? long_tp_price : na, "TP", color.green, style=plot.style_linebr)
plot(strategy.position_size < 0 ? short_stop_price : na, "SL", color.red, style=plot.style_linebr)
plot(strategy.position_size < 0 ? short_tp_price : na, "TP", color.green, style=plot.style_linebr)
```
    