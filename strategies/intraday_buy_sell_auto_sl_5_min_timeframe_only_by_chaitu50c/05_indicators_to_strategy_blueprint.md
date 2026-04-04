
# Indicators to Strategy Blueprint

Based on the provided `indicator` script, here is the architectural blueprint for its transformation into a production-ready, automated execution framework. The original logic, a 3-bar reversal pattern on a 5-minute chart, serves as the core signal generator. We will build a professional execution and risk management layer around it.

### 1. Execution Triggers (Entry & Direction)

The core of the entry logic is a three-candle pattern that must complete at the end of a 15-minute block (i.e., on the third 5-minute candle).

*   **Long Entry Condition:** A Long position is initiated when all the following conditions are met on the close of a 5-minute bar:
    1.  The bar is the third in a 15-minute sequence (e.g., at xx:15, xx:30, xx:45, xx:00).
    2.  The pattern of the last three bars is: **Red Candle, Green Candle, Green Candle**.
    3.  The `close` of the current bar is **greater than the `high`** of the first bar in the three-candle sequence (the Red Candle).

*   **Short Entry Condition:** A Short position is initiated when all the following conditions are met on the close of a 5-minute bar:
    1.  The bar is the third in a 15-minute sequence.
    2.  The pattern of the last three bars is: **Green Candle, Red Candle, Red Candle**.
    3.  The `close` of the current bar is **less than the `low`** of the first bar in the three-candle sequence (the Green Candle).

#### Execution Nuances:

*   **Execution Timing:** The strategy must execute **at the close of the bar**. The conditions `close > high[2]` and `close < low[2]` can only be confirmed once the bar has fully formed. Therefore, the strategy should be configured with `process_orders_on_close = true`. This ensures that orders are calculated and sent to the broker engine after the bar's data is complete, mimicking a market-on-close order.

*   **Signal Reversals:** This is a reversal system. If the strategy is in a Long position and a valid `sellSignal` occurs, the system must immediately flatten the Long position and initiate a new Short position. A professional framework handles this in a single logical step to minimize latency and potential slippage between closing and opening. The `strategy.entry()` command in Pine Script handles this reversal automatically if called for an opposing trade while a position is open. We will explicitly close any existing position before entering a new one for clarity and control.

### 2. Multi-Tiered Exit Logic

A single, static stop-loss is insufficient for a robust system. We will implement a multi-layered exit strategy to manage risk and secure profits dynamically.

*   **Initial Stop Loss (Volatility-Based):**
    *   The original script's stop-loss logic is sound and based on the volatility of the signal formation. We will adopt it.
    *   **For Longs:** The stop loss is placed at the **lowest `low`** of the three-candle signal pattern.
    *   **For Shorts:** The stop loss is placed at the **highest `high`** of the three-candle signal pattern.
    *   This stop order will be placed *immediately* upon entry as part of a bracket order.

*   **Take Profit / Trailing Mechanism (Multi-Stage):**
    *   **Target 1 (Scaling Out):** A primary take-profit target will be set at a fixed Risk/Reward ratio. A common starting point is **1.5R**, where 'R' is the initial risk (distance from entry to stop loss). Upon reaching this target, **50% of the position will be closed**.
    *   **Target 2 (Trailing Stop):** After Target 1 is hit, the stop loss on the remaining 50% of the position is moved to breakeven. From this point, a **trailing stop** will be activated to protect profits while allowing the trade to continue running. A suitable trailing mechanism would be an **ATR (Average True Range) based trail**. For example, the stop will trail the price by a multiple of the 14-period ATR (e.g., `TrailOffset = 2 * ta.atr(14)`).

*   **Time-Based Exits:**
    *   **End of Day (EOD) Close:** As this is an intraday strategy, all open positions must be squared off before the market close to avoid overnight risk. A hard rule will be implemented to **flatten all positions 15 minutes before the session ends**.
    *   **Stagnation Exit:** If a trade has been open for a specified number of bars (e.g., 25 bars, or ~2 hours) and has not hit either the initial stop loss or the first take-profit target, it will be closed. This frees up capital from non-performing trades.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for transitioning from a theoretical model to a real-world trading system. We will use a fixed-fractional risk model.

*   **Risk-Based Sizing:** The strategy will risk a fixed percentage of account equity on every trade (e.g., 1%). The position size is calculated dynamically based on the distance to the initial stop loss.

    *   **Formula:**
        1.  `Risk_Per_Trade_USD = strategy.equity * Risk_Percent`
        2.  `Trade_Risk_Per_Share = abs(Entry_Price - Stop_Loss_Price)`
        3.  `Position_Size = floor(Risk_Per_Trade_USD / Trade_Risk_Per_Share)`

    *   This ensures that a losing trade, if hit at the initial stop, will result in a predictable, fixed-percentage loss of the total account equity, regardless of the trade's volatility.

*   **Pyramiding & Scaling:**
    *   **Pyramiding (Scaling In):** This strategy's logic is based on reversals, not trend continuation. Therefore, adding to a winning position (pyramiding) is **not recommended** and will be disabled (`pyramiding = 0`). An entry signal is also an exit signal for the previous trend, making scaling-in counter-intuitive to the core logic.
    *   **Scaling Out:** As defined in the exit logic, the strategy will **scale out** at the first take-profit target, selling 50% of the initial position. This is managed via the `strategy.exit()` command using quantity-based parameters.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the `indicator` into a `strategy`, incorporating the professional-grade components discussed above.

```pine
//@version=5
// 1. STRATEGY DECLARATION with realistic execution parameters
strategy(
     'Production-Grade Intraday Reversal', 
     overlay=true, 
     process_orders_on_close=true, // Execute on bar close
     initial_capital=25000,
     commission_type=strategy.commission.cash_per_order,
     commission_value=1.0, // Example: $1 per order
     slippage=2, // Example: 2 ticks of slippage
     pyramiding=0 // Disable adding to positions
)

// === INPUTS ===
riskPercent = input.float(1.0, "Risk per Trade (%)", minval=0.1, maxval=5.0)
rr1 = input.float(1.5, "Take Profit 1 R:R", minval=0.5)
atrLength = input.int(14, "ATR Length for Trailing Stop")
atrMultiplier = input.float(2.0, "ATR Multiplier for Trailing Stop")
eodHour = input.int(15, "End of Day Hour (24h format)")
eodMinute = input.int(45, "End of Day Minute")

// === CORE SIGNAL LOGIC (from original script) ===
is5min = timeframe.period == '5'
blockComplete = is5min and bar_index % 3 == 0 // Simplified block logic for strategy context

c0g = close[2] > open[2]
c1r = close[1] < open[1]
c2r = close < open
c0r = close[2] < open[2]
c1g = close[1] > open[1]
c2g = close > open

buySignal = blockComplete and c0r and c1g and c2g and close > high[2] and strategy.position_size == 0
sellSignal = blockComplete and c0g and c1r and c2r and close < low[2] and strategy.position_size == 0

// === RISK & EXIT CALCULATION ===
// Stop Loss Levels
longStopPrice = ta.lowest(low, 3)
shortStopPrice = ta.highest(high, 3)

// Position Sizing Function
getPosSize(riskPerc, stopLossDistance) =>
    equity = strategy.equity
    riskAmount = equity * (riskPerc / 100)
    size = riskAmount / stopLossDistance
    math.max(1, size) // Ensure at least 1 share/contract

// Take Profit Levels
longRisk = close - longStopPrice
shortRisk = shortStopPrice - close
longTp1 = close + (longRisk * rr1)
shortTp1 = close - (shortRisk * rr1)

// === EXECUTION ENGINE ===
if buySignal
    // 3. Capital Allocation: Calculate position size
    posSize = getPosSize(riskPercent, longRisk)
    
    // 2. Execution Trigger: Enter long position
    strategy.entry("Long", strategy.long, qty=posSize)
    
    // 2. Multi-Tiered Exit: Place bracket order for TP1 and initial SL
    strategy.exit("Long Exit", from_entry="Long", qty_percent=50, profit=longTp1) // TP1 for 50%
    strategy.exit("Long Stop", from_entry="Long", loss=longStopPrice) // SL for 100%

if sellSignal
    // 3. Capital Allocation: Calculate position size
    posSize = getPosSize(riskPercent, shortRisk)
    
    // 2. Execution Trigger: Enter short position
    strategy.entry("Short", strategy.short, qty=posSize)
    
    // 2. Multi-Tiered Exit: Place bracket order for TP1 and initial SL
    strategy.exit("Short Exit", from_entry="Short", qty_percent=50, profit=shortTp1) // TP1 for 50%
    strategy.exit("Short Stop", from_entry="Short", loss=shortStopPrice) // SL for 100%

// === DYNAMIC & TIME-BASED EXIT MANAGEMENT ===
// ATR Trailing Stop for remaining position
var float trail_price = na
if strategy.position_size > 0 and strategy.opentrades == 1 // Position has been partially closed
    if high > trail_price + 0.0001 // Move trail up
        trail_price := math.max(trail_price, close - (ta.atr(atrLength) * atrMultiplier))
    strategy.exit("Long Trail", "Long", trail_price=trail_price, trail_offset=0)
else if strategy.position_size > 0
    trail_price := 0.0 // Reset on new trade

if strategy.position_size < 0 and strategy.opentrades == 1
    if low < trail_price - 0.0001 // Move trail down
        trail_price := math.min(trail_price, close + (ta.atr(atrLength) * atrMultiplier))
    strategy.exit("Short Trail", "Short", trail_price=trail_price, trail_offset=0)
else if strategy.position_size < 0
    trail_price := 999999.9 // Reset on new trade

// End of Day Exit
isEod = hour == eodHour and minute >= eodMinute
if isEod
    strategy.close_all(comment="EOD Close")
    trail_price := na
```
