
# Indicators to Strategy Blueprint

The provided "Smart Levels Pro" script is an excellent visualization tool, identifying critical price levels based on previous daily, weekly, monthly, and session data. However, as an `indicator`, it generates no trades. To transform this into a production-ready execution framework, we must define a concrete trading hypothesis based on these levels and then build a robust engine to manage entries, exits, and risk.

The chosen strategy will be a **"Break and Retest"** of key levels. This is a classic pattern that aims to filter out false breakouts by waiting for the market to confirm the level has flipped from resistance to support (for longs) or support to resistance (for shorts).

### 1. Execution Triggers (Entry & Direction)

The core premise is to trade the first confirmed retest of the Previous Day's High (PDH) or Previous Day's Low (PDL). These levels are often the most relevant for intraday price action.

*   **Long Entry Condition:**
    1.  The market must first establish a clear breakout above the Previous Day's High (`pd_high`).
    2.  Price must then pull back, with a bar's `low` dipping below `pd_high`. This signifies the "retest."
    3.  The retest bar must then close *above* `pd_high`, confirming the level is now acting as support.
    *   **Boolean Logic:** `low[1] < pd_high and close[1] > pd_high and strategy.position_size == 0`
        *   This condition identifies a bar that dipped below the level but closed above it, signaling a successful hold. We check `strategy.position_size == 0` to ensure we are not already in a trade.

*   **Short Entry Condition:**
    1.  The market must first break down below the Previous Day's Low (`pd_low`).
    2.  Price must then rally back, with a bar's `high` pushing above `pd_low`. This is the retest.
    3.  The retest bar must close *below* `pd_low`, confirming the level is now acting as resistance.
    *   **Boolean Logic:** `high[1] > pd_low and close[1] < pd_low and strategy.position_size == 0`

#### Execution Nuances:

*   **Execution Timing:** All entry signals will be executed **on the close of the signal bar**. This is represented by the logic `close[1]` and `high[1]`, which evaluates the completed previous bar before making a decision on the open of the current bar. This approach prevents the "repainting" and whipsaw common with real-time (`calc_on_every_tick=true`) execution, providing more reliable backtest results that better reflect live performance.
*   **Signal Reversals:** The system is designed to be "flat" before entering a new trade (`strategy.position_size == 0`). If a long trade is active and a valid short signal appears, the system will not automatically flip. A dedicated `strategy.close()` call would be required based on the opposing signal if a "stop and reverse" functionality is desired. For robustness, we will initially manage each trade independently from entry to exit.

### 2. Multi-Tiered Exit Logic

A professional exit strategy is not a single rule but a combination of logic to protect capital and maximize gains.

*   **Initial Stop Loss (Volatility-Based):**
    *   The stop loss will be placed based on a multiple of the Average True Range (ATR). This adapts the stop distance to the market's current volatility.
    *   **Long Stop Loss:** `entry_price - (ATR_Multiplier * ATR_value)`. A more aggressive, structure-based stop could be placed just below the `low` of the signal bar (`low[1]`), as this is the structural point that confirmed the retest.
    *   **Short Stop Loss:** `entry_price + (ATR_Multiplier * ATR_value)`. Similarly, a structural stop could be placed just above the `high` of the signal bar (`high[1]`).

*   **Take Profit / Trailing (Multi-Stage):**
    *   The script already provides perfect candidates for take-profit targets: the higher timeframe levels.
    *   **TP1 (Scaling Out):** The first take-profit target could be the **Previous Week's High (PWH)** for a long trade or the **Previous Week's Low (PWL)** for a short trade. At this point, we can exit a portion of the position (e.g., 50%) to lock in profits.
    *   **Trailing Stop (Profit Protection):** After TP1 is hit, the stop loss for the remaining position will be moved to the entry price (breakeven). From there, a trailing stop can be activated, trailing the price by a multiple of ATR to capture the remainder of the trend.

*   **Time-Based Exits:**
    *   **End of Day (EOD):** As this is an intraday strategy, all open positions must be squared off before the session close to avoid overnight risk. A rule will be implemented to close any open trade at a specified time (e.g., 15 minutes before market close).
    *   **Stagnation Exit:** If a trade has been open for a significant number of bars (e.g., 25 bars on a 15-minute chart) without hitting either the stop loss or the first take-profit, it can be considered "stagnant." An exit rule will close the position to free up capital for more promising opportunities.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term survival. We will implement a dynamic, risk-based sizing model.

*   **Risk-Based Sizing:**
    *   The strategy will risk a fixed percentage of the account equity on every single trade (e.g., 1%).
    *   **Logic:**
        1.  Define `riskPercent` (e.g., 1.0 for 1%).
        2.  On a trade signal, calculate the distance in points/pips from the entry price to the initial stop-loss price. This is the `tradeRiskPerUnit`.
        3.  Calculate the total dollar amount to risk: `equityToRisk = strategy.equity * (riskPercent / 100)`.
        4.  Calculate the position size: `positionSize = equityToRisk / (tradeRiskPerUnit * syminfo.pointvalue)`.
    *   This ensures that a trade with a wider stop (due to higher volatility) will have a smaller position size, and a trade with a tighter stop will have a larger one, equalizing the dollar risk for every entry.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, the system will scale out of winning trades at predefined levels (PWH/PWL). The `strategy.exit` command with the `qty_percent` parameter is ideal for this.
    *   **Pyramiding (Scaling In):** While not in the initial implementation for simplicity, a pyramiding rule could be added. For example, if a long trade is entered at PDH and is profitable, a second, smaller position could be added if the price then breaks and retests the **London Session High**. The stop for the entire combined position would need to be trailed up aggressively to manage the increased exposure.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the transformation from an `indicator` to a `strategy`, incorporating the professional execution logic defined above. It omits the visual components (dashboard, labels) to focus purely on the trading engine.

```pine
//@version=5
// --- STRATEGY DECLARATION ---
strategy("Smart Levels Pro - Execution Engine",
     overlay=true,
     initial_capital=100000,
     commission_type=strategy.commission.percent,
     commission_value=0.04, // Example for futures/forex broker
     slippage=2, // 2 ticks of slippage
     pyramiding=0, // No pyramiding in this version
     default_qty_type=strategy.fixed,
     process_orders_on_close=true) // Ensures execution on bar close

// --- INPUTS FOR STRATEGY ---
grp_risk = "Risk Management"
riskPercent    = input.float(1.0, "Risk per Trade (%)", minval=0.1, maxval=5, step=0.1, group=grp_risk)
atrLength      = input.int(14, "ATR Length", group=grp_risk)
atrMultiplier  = input.float(2.0, "ATR Stop Multiplier", group=grp_risk)
useEodExit     = input.bool(true, "Use End-of-Day Exit?", group=grp_risk)
eodHour        = input.int(20, "EOD Exit Hour (GMT)", minval=0, maxval=23, group=grp_risk)
eodMinute      = input.int(45, "EOD Exit Minute (GMT)", minval=0, maxval=59, group=grp_risk)

// --- LEVEL CALCULATION (Simplified from original script) ---
is_new_day   = dayofmonth != dayofmonth[1]
is_new_week  = weekofyear != weekofyear[1]

var float pd_high = na, var float pd_low = na
var float pw_high = na, var float pw_low = na

[d_high, d_low] = request.security(syminfo.tickerid, "D", [high[1], low[1]], lookahead=barmerge.lookahead_off)
if is_new_day
    pd_high := d_high
    pd_low  := d_low

[w_high, w_low] = request.security(syminfo.tickerid, "W", [high[1], low[1]], lookahead=barmerge.lookahead_off)
if is_new_week
    pw_high := w_high
    pw_low  := w_low

// --- CORE STRATEGY LOGIC ---
atrValue = ta.atr(atrLength)

// 1. ENTRY TRIGGERS (Break & Retest Logic)
longEntryCondition = low[1] < pd_high and close[1] > pd_high and strategy.position_size == 0 and not na(pd_high)
shortEntryCondition = high[1] > pd_low and close[1] < pd_low and strategy.position_size == 0 and not na(pd_low)

// 2. RISK MANAGEMENT & POSITION SIZING
var float stopLossPrice = na
var float takeProfitPrice = na

if (longEntryCondition)
    stopLossPrice   := close - (atrValue * atrMultiplier)
    takeProfitPrice := pw_high // Target the previous week's high
    
    // Calculate position size based on 1% equity risk
    riskPerUnit     = (close - stopLossPrice) * syminfo.pointvalue
    equityToRisk    = strategy.equity * (riskPercent / 100)
    positionSize    = equityToRisk / riskPerUnit
    
    strategy.entry("Long", strategy.long, qty=positionSize)
    // Place a multi-stage exit order: 50% at TP1, rest managed by SL
    strategy.exit("Exit Long TP1/SL", from_entry="Long", qty_percent=50, limit=takeProfitPrice, stop=stopLossPrice)
    // This second exit order manages the stop for the remaining 50%
    strategy.exit("Exit Long SL2", from_entry="Long", stop=stopLossPrice)

if (shortEntryCondition)
    stopLossPrice   := close + (atrValue * atrMultiplier)
    takeProfitPrice := pw_low // Target the previous week's low

    riskPerUnit     = (stopLossPrice - close) * syminfo.pointvalue
    equityToRisk    = strategy.equity * (riskPercent / 100)
    positionSize    = equityToRisk / riskPerUnit

    strategy.entry("Short", strategy.short, qty=positionSize)
    strategy.exit("Exit Short TP1/SL", from_entry="Short", qty_percent=50, limit=takeProfitPrice, stop=stopLossPrice)
    strategy.exit("Exit Short SL2", from_entry="Short", stop=stopLossPrice)

// 3. TIME-BASED EXIT
isEod = useEodExit and (hour(time, "GMT") == eodHour) and (minute(time, "GMT") >= eodMinute)
if (isEod)
    strategy.close_all(comment="EOD Exit")

// --- PLOTTING FOR VISUALIZATION ---
plot(pd_high, "PDH", color.new(color.blue, 0), style=plot.style_linebr)
plot(pd_low, "PDL", color.new(color.red, 0), style=plot.style_linebr)
plot(pw_high, "PWH", color.new(color.orange, 40), style=plot.style_circles)
plot(pw_low, "PWL", color.new(color.purple, 40), style=plot.style_circles)
```
    