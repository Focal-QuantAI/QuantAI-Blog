
# Indicators to Strategy Blueprint

Based on the provided `indicator` script, here is the architectural blueprint for its transformation into a production-ready, automated execution framework.

### 1. Execution Triggers (Entry & Direction)

The core logic is a three-bar reversal pattern confirmed on the close of the third bar. This "end-of-bar" confirmation is crucial for avoiding false signals that may flicker intra-bar.

*   **Long Entry Condition:** A trade is triggered when all the following are true:
    1.  The current bar is the 3rd bar in a 15-minute block on a 5-minute chart.
    2.  The first bar of the pattern was a Red candle (Close < Open).
    3.  The second bar was a Green candle (Close > Open).
    4.  The third (current) bar is a Green candle (Close > Open).
    5.  The close of the current bar is **greater than the high** of the first bar in the pattern.

*   **Short Entry Condition:** A trade is triggered when all the following are true:
    1.  The current bar is the 3rd bar in a 15-minute block on a 5-minute chart.
    2.  The first bar of the pattern was a Green candle (Close > Open).
    3.  The second bar was a Red candle (Close < Open).
    4.  The third (current) bar is a Red candle (Close < Open).
    5.  The close of the current bar is **less than the low** of the first bar in the pattern.

*   **Execution Nuance:** Orders must be executed **at the open of the bar following the signal bar**. The logic (`close > high[2]`) is only confirmed at the exact moment the signal bar closes. Attempting to execute intra-bar would constitute a different, more aggressive strategy prone to "whipsaw" entries. The system will place a market order to be filled at the open of the next candle.

*   **Signal Reversals:** The system must be "always-in" or handle reversals gracefully. If the strategy is in a Long position and a valid Short Entry Condition is met, the existing Long position must be closed immediately, and the new Short position must be opened. This is a "stop and reverse" (SAR) mechanism. For precise control, this is best handled as two distinct orders: `strategy.close()` followed by `strategy.entry()`.

### 2. Multi-Tiered Exit Logic

A single stop-loss is insufficient for a professional system. We will implement a multi-layered exit strategy to manage the trade lifecycle.

*   **Initial Stop Loss (Structure-Based):**
    *   The script's original stop-loss logic is sound, as it's based on the volatility and structure of the entry pattern itself.
    *   **For Longs:** The stop loss is placed at the lowest low of the three-bar entry pattern (`blkLow`).
    *   **For Shorts:** The stop loss is placed at the highest high of the three-bar entry pattern (`blkHigh`).
    This is superior to a fixed percentage as it adapts to the specific market conditions at the time of entry.

*   **Take Profit / Trailing Mechanism:**
    We will employ a two-stage profit-taking and trailing stop model to lock in gains while allowing for runners.
    1.  **Target 1 (Scaling Out):** A primary take-profit order is placed at a 1.5R multiple, where 'R' is the initial risk (distance from entry to stop loss). Upon hitting this target, 50% of the position is closed.
    2.  **Trailing Stop Activation:** Once Target 1 is hit, the stop loss on the remaining 50% of the position is moved to the entry price (breakeven).
    3.  **Dynamic Trailing:** For the remaining position, an ATR-based trailing stop is activated. The stop will trail the highest high (for longs) or lowest low (for shorts) achieved since the breakeven move, by a distance of 2x the 14-period ATR. This allows the profitable portion of the trade to run while protecting against significant reversals.

*   **Time-Based Exits:**
    Capital preservation requires time-based invalidation rules.
    1.  **End of Day (EOD) Exit:** As an intraday strategy, all open positions must be squared off automatically before the session close to eliminate overnight risk. We will define an EOD exit time (e.g., 15 minutes before market close) at which `strategy.close_all()` is triggered.
    2.  **Stagnation Exit:** If a trade has been open for more than 15 bars (75 minutes on a 5-min chart) and has not yet reached Target 1, it will be closed. This prevents capital from being tied up in non-performing, "dead" trades.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term viability. We will move from arbitrary quantities to a fixed-fractional risk model.

*   **Risk-Based Sizing:**
    The strategy will risk a fixed percentage of account equity on every single trade.
    *   **Rule:** Risk a maximum of 1% of `strategy.equity` per trade.
    *   **Calculation (for a Long trade):**
        1.  `riskPerUnit = entry_price - stop_loss_price`
        2.  `equityRisk = strategy.equity * 0.01`
        3.  `positionSize = equityRisk / riskPerUnit`
    The calculated `positionSize` will be used for the `strategy.entry` quantity, ensuring uniform risk across all trades regardless of the stop-loss distance in price points.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, the strategy will scale out by closing 50% of the position at the first take-profit target. This is a core feature.
    *   **Pyramiding (Scaling In):** Pyramiding will be **disabled** for this initial version (`pyramiding = 0`). The three-bar pattern is a specific reversal/breakout signal, and adding to the position based on different criteria would complicate the core logic. A robust pyramiding module would require its own distinct entry signals (e.g., breakout of a subsequent consolidation) and risk management rules, which can be added in a future version.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the `indicator` into a `strategy` incorporating the architectural principles outlined above.

```pine
//@version=5
// TRANSFORMATION: From Indicator to Production Strategy
strategy(
    "Intraday 3-Bar Reversal Strategy", 
    overlay=true, 
    initial_capital=25000, 
    commission_type=strategy.commission.percent, 
    commission_value=0.04, // Realistic commission for futures/CFDs
    slippage=2, // 2 ticks of slippage on market orders
    pyramiding=0, // No pyramiding in this version
    process_orders_on_close=true // Ensures execution on next bar's open
)

// === Inputs for Strategy Control ===
riskPercent = input.float(1.0, "Risk per Trade %", minval=0.1, maxval=5.0)
tp1RR = input.float(1.5, "Take Profit 1 R:R", minval=0.5)
atrLength = input.int(14, "ATR Length for Trailing Stop")
atrMultiplier = input.float(2.0, "ATR Multiplier for Trailing Stop")
stagnationBars = input.int(15, "Max Bars in Trade Before Exit")
eodHour = input.int(15, "End of Day Exit Hour (24h format)")
eodMinute = input.int(45, "End of Day Exit Minute")

// === Core Signal Logic (from original script) ===
is5min = timeframe.period == '5'
blockStartMs = math.floor(time / (15 * 60 * 1000)) * 15 * 60 * 1000
idxInBlk = math.round((time - blockStartMs) / (timeframe.multiplier * 60 * 1000)) + 1
blockComplete = is5min and idxInBlk % 3 == 0

c0r = close[2] < open[2]
c1g = close[1] > open[1]
c2g = close > open
buySignal = blockComplete and c0r and c1g and c2g and close > high[2]

c0g = close[2] > open[2]
c1r = close[1] < open[1]
c2r = close < open
sellSignal = blockComplete and c0g and c1r and c2r and close < low[2]

// === Position Sizing & Risk Management ===
longStopPrice = low[2]
shortStopPrice = high[2]
atrValue = ta.atr(atrLength)

riskAmount = (riskPercent / 100) * strategy.equity
longPositionSize = math.max(1, math.floor(riskAmount / (close - longStopPrice)))
shortPositionSize = math.max(1, math.floor(riskAmount / (shortStopPrice - close)))

// === Exit Logic Variables ===
longTp1Price = strategy.position_avg_price + (strategy.position_avg_price - longStopPrice) * tp1RR
shortTp1Price = strategy.position_avg_price - (shortStopPrice - strategy.position_avg_price) * tp1RR

// Trailing Stop Logic
var float trail_stop_price = na
if strategy.position_size > 0 // Long position
    if close > longTp1Price[1] // After TP1 is hit, activate trail
        new_trail_stop = high - atrValue * atrMultiplier
        trail_stop_price := na(trail_stop_price) ? strategy.position_avg_price : math.max(trail_stop_price, new_trail_stop, strategy.position_avg_price)
else if strategy.position_size < 0 // Short position
    if close < shortTp1Price[1] // After TP1 is hit, activate trail
        new_trail_stop = low + atrValue * atrMultiplier
        trail_stop_price := na(trail_stop_price) ? strategy.position_avg_price : math.min(trail_stop_price, new_trail_stop, strategy.position_avg_price)
else
    trail_stop_price := na // Reset when flat

// === Execution Engine ===
isFlat = strategy.position_size == 0
isInLong = strategy.position_size > 0
isInShort = strategy.position_size < 0

// 1. Entry Logic
if isFlat
    if buySignal
        strategy.entry("Long", strategy.long, qty=longPositionSize)
        strategy.exit("Long Exit", from_entry="Long", stop=longStopPrice, limit=longTp1Price, qty_percent=50)
    if sellSignal
        strategy.entry("Short", strategy.short, qty=shortPositionSize)
        strategy.exit("Short Exit", from_entry="Short", stop=shortStopPrice, limit=shortTp1Price, qty_percent=50)

// 2. Trailing Stop Management
if not na(trail_stop_price)
    strategy.exit("Trailing Stop", stop=trail_stop_price)

// 3. Time-Based Exits
isEOD = hour == eodHour and minute >= eodMinute
stagnationExit = bar_index - strategy.opentrades.entry_bar_index(0) >= stagnationBars and na(trail_stop_price) // Only if TP1 not hit

if isEOD or stagnationExit
    strategy.close_all(comment="Time-based Exit")
    trail_stop_price := na // Reset trail on close

// 4. Stop and Reverse (SAR) Logic
if isInLong and sellSignal
    strategy.close("Long", comment="SAR Close")
    strategy.entry("Short", strategy.short, qty=shortPositionSize)
    strategy.exit("Short Exit", from_entry="Short", stop=shortStopPrice, limit=shortTp1Price, qty_percent=50)

if isInShort and buySignal
    strategy.close("Short", comment="SAR Close")
    strategy.entry("Long", strategy.long, qty=longPositionSize)
    strategy.exit("Long Exit", from_entry="Long", stop=longStopPrice, limit=longTp1Price, qty_percent=50)

```
