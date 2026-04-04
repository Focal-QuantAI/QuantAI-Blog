
    # Indicators to Strategy Blueprint

    Based on the provided `indicator` script, here is the architectural blueprint for its transformation into a production-ready, automated execution framework.

### 1. Execution Triggers (Entry & Direction)

The original script's logic is based on a three-bar pattern that must complete at the end of a 15-minute block on a 5-minute chart. This confirmation is based on the closing price of the third bar.

*   **Long Entry Condition:** A `buySignal` is triggered if and only if all the following conditions are met on the close of a 5-minute bar:
    1.  The bar is the third and final bar of a 15-minute quarter-hour block (e.g., at 9:45, 10:00, 10:15).
    2.  The pattern of the last three bars is: **Red** (`close[2] < open[2]`), **Green** (`close[1] > open[1]`), **Green** (`close > open`).
    3.  The closing price of the current bar (`close`) breaks above the high of the first bar in the pattern (`high[2]`).

*   **Short Entry Condition:** A `sellSignal` is triggered if and only if all the following conditions are met on the close of a 5-minute bar:
    1.  The bar is the third and final bar of a 15-minute quarter-hour block.
    2.  The pattern of the last three bars is: **Green** (`close[2] > open[2]`), **Red** (`close[1] < open[1]`), **Red** (`close < open`).
    3.  The closing price of the current bar (`close`) breaks below the low of the first bar in the pattern (`low[2]`).

#### Execution Nuances

*   **Execution Timing:** The strategy must execute **at the close of the bar** where the signal is confirmed. The conditions `close > high[2]` and `close < low[2]` are only known once the bar has finished forming. Attempting to execute this logic intra-bar would lead to significant repaint and deviation from the backtest. The strategy declaration must include `process_orders_on_close = true`.
*   **Signal Reversals:** The system must be configured to handle reversals immediately. If the strategy is currently in a Long position and a `sellSignal` is generated, the system will execute a single order to close the long position and simultaneously open a new short position. This is managed by using consistent IDs for `strategy.entry` calls, which automatically handles the position flip.

### 2. Multi-Tiered Exit Logic

The original script's "auto SL" is a static stop based on the pattern's extremity. A professional framework requires a more dynamic and comprehensive exit plan.

*   **Initial Stop Loss:**
    *   The initial stop loss is calculated based on the volatility of the signal pattern itself, which is a robust method.
    *   **For Longs:** The stop loss is placed at the lowest low of the three-bar signal pattern (`blkLow`).
    *   **For Shorts:** The stop loss is placed at the highest high of the three-bar signal pattern (`blkHigh`).
    *   This order is placed immediately upon entry.

*   **Take Profit / Trailing Mechanism:**
    *   **Stage 1: Initial Target (Scaling Out):** A primary take-profit target will be set based on a Risk/Reward ratio. For example, a 1.5R target, where `R` is the distance from the entry price to the initial stop loss.
        *   `longTakeProfit1 = strategy.position_avg_price + 1.5 * (strategy.position_avg_price - stopLossPrice)`
        *   At this level, the strategy will close a portion of the position (e.g., 50%).
    *   **Stage 2: Trailing Stop (Profit Protection):** After the first take-profit target is hit, the stop loss for the remaining position is moved to breakeven. Subsequently, a dynamic trailing stop is activated. A Chandelier Exit is an excellent choice:
        *   **Logic:** The trailing stop follows the highest high (for longs) or lowest low (for shorts) since the trade was initiated, minus a multiple of the Average True Range (ATR).
        *   `trailingStop = highest(high, bars_since_entry) - (atr(14) * 3)`
        *   This allows the remainder of the position to capture a larger trend while aggressively protecting accumulated profits.

*   **Time-Based Exits:**
    *   **End of Day (EOD) Squaring:** As this is an intraday strategy, all open positions must be closed before the market close to avoid overnight risk. A rule will be implemented to flatten all positions 15 minutes before the session ends (e.g., at 15:45 EST for US Equities).
    *   **Trade Stagnation:** If a trade has been open for a specified number of bars (e.g., 20 bars, or 100 minutes) without hitting either the stop loss or a take-profit level, it will be closed. This frees up capital from non-performing trades.

### 3. Capital Allocation & Risk Management

Position sizing will be dynamic, ensuring that each trade risks a consistent, predefined percentage of total account equity.

*   **Risk-Based Sizing:**
    *   The core rule: **Risk a fixed percentage (e.g., 1%) of account equity on every trade.**
    *   **Calculation Steps:**
        1.  Define `riskPercent` (e.g., 1.0 for 1%).
        2.  Calculate `equity` (`strategy.equity`).
        3.  Calculate `capitalToRisk = equity * (riskPercent / 100)`.
        4.  On a signal, determine the `initialRiskPerShare` = `abs(entry_price - stop_loss_price)`. The entry price is the `close` of the signal bar.
        5.  Calculate **Position Size**: `positionSize = capitalToRisk / initialRiskPerShare`.
    *   This ensures that a trade with a wider stop (higher volatility) will have a smaller position size than a trade with a tighter stop, equalizing the dollar risk of each.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, the strategy will scale out by closing portions of the position at predefined take-profit levels.
    *   **Pyramiding (Scaling In):** Adding to a winning position will be disallowed in the initial version to maintain simplicity and control risk. The logic `if (strategy.position_size == 0)` will precede entry conditions to ensure only one position is taken per directional bias. Advanced versions could allow pyramiding only after the initial position's stop has been moved to breakeven and a new, valid 3-bar signal occurs.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the `indicator` into a `strategy` incorporating the architectural principles above.

```pine
//@version=5
// Transformed from Indicator to a full Strategy Framework
strategy(
     "Pro Execution: 3-Bar Reversal", 
     overlay=true, 
     process_orders_on_close=true, // CRITICAL: Execute on bar close to match logic
     initial_capital=100000,
     commission_type=strategy.commission.percent,
     commission_value=0.075, // Realistic commission
     slippage=2 // Realistic slippage in ticks
)

// --- Risk Management Inputs ---
riskPercent = input.float(1.0, "Risk per Trade %", minval=0.1, maxval=5.0)
rrRatio = input.float(2.0, "Take Profit Risk/Reward Ratio", minval=0.5)
useEodExit = input.bool(true, "Use End-of-Day Exit?")
eodExitTime = input.int(1545, "EOD Exit Time (HHMM)")

// === Original 3-Candle Reversal Logic ===
is5min = timeframe.period == "5"
isNewDay = ta.change(time("D"))

// Simplified block logic for clarity
isThirdBarOf15Min = minute % 15 == 10 and is5min // e.g., 9:10, 9:25, 9:40 -> bar closes at 9:15, 9:30, 9:45

c0g = close[2] > open[2]
c1r = close[1] < open[1]
c2r = close < open
c0r = close[2] < open[2]
c1g = close[1] > open[1]
c2g = close > open

buySignal = isThirdBarOf15Min and c0r and c1g and c2g and close > high[2]
sellSignal = isThirdBarOf15Min and c0g and c1r and c2r and close < low[2]

// === Execution & Risk Calculation ===
if (strategy.position_size == 0) // Only enter if flat
    // --- Long Entry ---
    if (buySignal)
        longStopPrice = ta.lowest(low, 3)
        initialRiskPerShare = close - longStopPrice
        
        if (initialRiskPerShare > 0)
            capitalToRisk = strategy.equity * (riskPercent / 100)
            positionSize = capitalToRisk / initialRiskPerShare
            longTakeProfitPrice = close + (initialRiskPerShare * rrRatio)
            
            strategy.entry("Long", strategy.long, qty=positionSize)
            strategy.exit("Exit Long", from_entry="Long", stop=longStopPrice, limit=longTakeProfitPrice)

    // --- Short Entry ---
    if (sellSignal)
        shortStopPrice = ta.highest(high, 3)
        initialRiskPerShare = shortStopPrice - close

        if (initialRiskPerShare > 0)
            capitalToRisk = strategy.equity * (riskPercent / 100)
            positionSize = capitalToRisk / initialRiskPerShare
            shortTakeProfitPrice = close - (initialRiskPerShare * rrRatio)

            strategy.entry("Short", strategy.short, qty=positionSize)
            strategy.exit("Exit Short", from_entry="Short", stop=shortStopPrice, limit=shortTakeProfitPrice)

// === Time-Based Exit Logic ===
isTradingSession = time(timeframe.period, "0930-1600")
isEod = isTradingSession and (hour * 100 + minute >= eodExitTime)

if (useEodExit and isEod and strategy.position_size != 0)
    strategy.close_all(comment="EOD Exit")

// --- Visuals for confirmation ---
plot(strategy.position_size != 0 ? strategy.opentrades.entry_price(strategy.opentrades - 1) : na, "Entry Price", color.blue, style=plot.style_linebr)
```
    