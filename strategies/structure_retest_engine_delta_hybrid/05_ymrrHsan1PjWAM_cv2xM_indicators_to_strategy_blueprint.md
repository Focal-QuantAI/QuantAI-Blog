
# Indicators to Strategy Blueprint

Here is the transformation of the "Structure Retest Engine Delta Hybrid" indicator into a production-ready algorithmic trading framework.

### 1. Execution Triggers (Entry & Direction)

The core of this strategy is a "break and retest" of a market structure level, defined as a recent swing high or low. An entry is only considered after a "Change of Character" (CHoCH) establishes a new potential trend direction.

*   **Long Entry Condition (`bullEntry`):**
    1.  A **Bullish CHoCH** must occur: Price must break and close above a previously confirmed swing high (`lastPH`) while the prior trend state was bearish (`trendState == -1`).
    2.  This break establishes the `activeLevel` (the price of the broken swing high) and the bullish `activeDir`.
    3.  Price must first move away from the level, confirming the breakout (`hasLeftLevel = true`).
    4.  Price must then return to the retest zone, defined as `activeLevel +/- (ATR * sensitivity)`.
    5.  The user-defined `confirmMode` condition on the entry bar must be met:
        *   **Touch:** The bar touches the zone and closes in the direction of the trend.
        *   **Close Outside Level (COL):** The bar touches the zone and then closes decisively back *outside* the zone on the trend side.
        *   **Engulfing:** The bar forms a bullish engulfing pattern while touching the zone.
    6.  The trade is only triggered if the level has not been previously "consumed" (unless `allowReEntry` is enabled).

*   **Short Entry Condition (`bearEntry`):**
    1.  A **Bearish CHoCH** must occur: Price must break and close below a previously confirmed swing low (`lastPL`) while the prior trend state was bullish (`trendState == 1`).
    2.  This break establishes the `activeLevel` (the price of the broken swing low) and the bearish `activeDir`.
    3.  Price must first move away from the level (`hasLeftLevel = true`).
    4.  Price must then return to the retest zone.
    5.  The `confirmMode` condition (Bearish Touch, COL, or Engulfing) must be met.
    6.  The trade is only triggered if the level has not been previously "consumed."

#### Execution Nuances & Signal Reversals

*   **Execution Timing:** All conditions are evaluated at the **Close of the bar**. The original script uses `alert.freq_once_per_bar_close`, and for robust backtesting and live execution, this is the most reliable method. Attempting intra-bar execution (`calc_on_every_tick=true`) with this logic would introduce significant repainting risk and execution uncertainty, especially with "Touch" or "Engulfing" confirmations that can flicker mid-candle. Therefore, we will use `process_orders_on_close = true` in the strategy, ensuring orders are generated only once a candle's final state is known.

*   **Signal Reversals:** The system is designed to follow one primary structure at a time. If the strategy is in a long position and a Bearish CHoCH occurs, the premise for the long trade is invalidated.
    *   **Logic:** A new CHoCH in the opposite direction of an open position serves as an immediate **exit signal**. The strategy will close the existing trade on the same bar the reversal structure is confirmed. It will then look for a new entry in the direction of the new CHoCH on subsequent bars.

### 2. Multi-Tiered Exit Logic

The original indicator lacks any exit logic. A professional framework requires a dynamic and multi-faceted exit strategy to manage risk and secure profits.

*   **Initial Stop Loss (Volatility-Based):**
    The stop loss will be placed based on market structure and volatility (ATR), not an arbitrary percentage.
    *   **Calculation:** The stop loss is placed a multiple of the 14-period ATR beyond the structural level that was broken.
    *   **For Longs:** `Stop Loss Price = activeLevel - (ATR * StopAtrMultiplier)`
    *   **For Shorts:** `Stop Loss Price = activeLevel + (ATR * StopAtrMultiplier)`
    *   A `StopAtrMultiplier` input (e.g., default of 2.0) gives the trader control over how much room the trade has. This anchors the risk to the specific market structure being traded.

*   **Take Profit / Trailing Mechanism:**
    We will implement a two-stage take-profit mechanism to secure initial gains while allowing for larger wins.
    1.  **Target 1 (Initial Profit):** A fixed Risk-to-Reward (R:R) target. For example, at 1.5R, we exit 50% of the position. The distance from entry to stop loss defines "1R".
    2.  **Target 2 (Trailing Stop):** After Target 1 is hit, the stop loss for the remaining 50% of the position is moved to breakeven. From this point, a trailing stop (e.g., a Chandelier Exit trailing `X` ATRs from the highest high/lowest low of the trade) can be used to let the remainder of the position run. For simplicity in the initial strategy implementation, we can let the second half run until the stop is hit or a time-based exit is triggered.

*   **Time-Based Exits:**
    Trades should not be held indefinitely if they are not performing.
    1.  **End of Day (EOD):** For intraday timeframes, all open positions will be squared off at the end of the trading session to eliminate overnight risk. This can be implemented by checking if `dayofweek` changes.
    2.  **Trade Stagnation:** If a position has been open for a specified number of bars (e.g., 100 bars) and has not hit either its take profit or stop loss, it is considered "dead money." The strategy will exit the position to free up capital for new opportunities.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term viability. We will move from the default fixed-quantity contract model to a dynamic risk-based model.

*   **Risk-Based Sizing:**
    The strategy will risk a fixed percentage of account equity on every single trade, ensuring that a string of losses does not lead to catastrophic drawdowns.
    *   **Inputs:**
        *   `accountEquity`: The total value of the trading account (`strategy.equity`).
        *   `riskPercent`: A user-defined percentage of equity to risk per trade (e.g., 1.0%).
    *   **Calculation:**
        1.  `riskAmountPerTrade = accountEquity * (riskPercent / 100)`
        2.  `riskDistanceInPrice = abs(entryPrice - stopLossPrice)`
        3.  `positionSize = riskAmountPerTrade / riskDistanceInPrice`
    *   This calculation is performed for every new entry, ensuring each trade has an equal dollar-risk impact on the account, regardless of the volatility or setup.

*   **Pyramiding & Scaling:**
    *   **Scaling In (Pyramiding):** The script's `allowReEntry` input provides a natural mechanism for pyramiding. If enabled, and a second valid entry signal appears while the first trade is open and in profit, a new, smaller position (e.g., 50% of the initial risk) can be added. A crucial rule: never add to a position that is currently at a loss.
    *   **Scaling Out:** As defined in the exit logic, the strategy will automatically scale out by exiting a portion of the position at the first take-profit target. This is handled via `strategy.exit` using the `qty_percent` parameter.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion from an `indicator` to a `strategy`, incorporating the professional-grade components discussed above. It focuses on the new logic and assumes the core signal generation (`bullEntry`, `bearEntry`, `activeLevel`, etc.) from the original script remains intact.

```pine
// This Pine Script® code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0.
//@version=5

// 1. STRATEGY DECLARATION with realistic friction and execution settings
strategy("Structure Retest Engine - Strategy", 
     overlay=true, 
     process_orders_on_close=true, // Execute on bar close for reliability
     commission_type=strategy.commission.percent, 
     commission_value=0.075, // Example commission
     slippage=2) // Example slippage in ticks

// ... [Keep all original inputs from the indicator] ...

// 2. NEW INPUTS FOR RISK AND EXIT MANAGEMENT
GRP_RISK = "Risk & Exit Management"
riskPercent    = input.float(1.0, "Risk Per Trade (%)", minval=0.1, maxval=10, step=0.1, group=GRP_RISK)
stopAtrMult    = input.float(2.0, "Stop Loss ATR Multiplier", minval=0.5, group=GRP_RISK)
tp1Rr          = input.float(1.5, "Take Profit 1 R:R", minval=0.5, group=GRP_RISK)
useTimeExit    = input.bool(true, "Use Time-Based Exit?", group=GRP_RISK)
maxBarsInTrade = input.int(100, "Max Bars in Trade", minval=10, group=GRP_RISK)

// ... [Keep all original logic for pivot detection, delta, state, and entry conditions] ...
// Assume 'bullEntry', 'bearEntry', 'activeLevel', 'activeDir', and 'atr' are calculated as before.

// 3. POSITION SIZING AND EXECUTION LOGIC
var float stopLossPrice = na
var float takeProfit1Price = na

// --- Bullish Entry Execution ---
if (bullEntry and strategy.position_size == 0)
    // Calculate Stop Loss and Take Profit levels AT THE TIME OF ENTRY
    stopLossPrice := activeLevel - (atr * stopAtrMult)
    entryPrice := close
    riskDistance := entryPrice - stopLossPrice
    takeProfit1Price := entryPrice + (riskDistance * tp1Rr)

    // Calculate Position Size based on risk
    riskAmount = strategy.equity * (riskPercent / 100)
    positionSize = riskAmount / riskDistance

    // Execute Orders
    strategy.entry("Long", strategy.long, qty=positionSize, comment="Bull Entry")
    // Set a two-stage exit: 50% at TP1, remainder managed by the second exit call
    strategy.exit("TP1/SL", from_entry="Long", qty_percent=50, limit=takeProfit1Price, stop=stopLossPrice)
    strategy.exit("SL2", from_entry="Long", stop=stopLossPrice) // Manages the stop for the second half

// --- Bearish Entry Execution ---
if (bearEntry and strategy.position_size == 0)
    // Calculate Stop Loss and Take Profit levels
    stopLossPrice := activeLevel + (atr * stopAtrMult)
    entryPrice := close
    riskDistance := stopLossPrice - entryPrice
    takeProfit1Price := entryPrice - (riskDistance * tp1Rr)

    // Calculate Position Size
    riskAmount = strategy.equity * (riskPercent / 100)
    positionSize = riskAmount / riskDistance

    // Execute Orders
    strategy.entry("Short", strategy.short, qty=positionSize, comment="Bear Entry")
    strategy.exit("TP1/SL", from_entry="Short", qty_percent=50, limit=takeProfit1Price, stop=stopLossPrice)
    strategy.exit("SL2", from_entry="Short", stop=stopLossPrice)

// 4. TRADE MANAGEMENT LOGIC (REVERSALS AND TIME EXITS)

// --- Reversal Exit: Close position if a CHoCH occurs in the opposite direction ---
isNewBearishChoch = bearishBreak and trendState[1] == 1
isNewBullishChoch = bullishBreak and trendState[1] == -1

if (strategy.position_size > 0 and isNewBearishChoch)
    strategy.close("Long", comment="Exit: Bearish CHoCH")

if (strategy.position_size < 0 and isNewBullishChoch)
    strategy.close("Short", comment="Exit: Bullish CHoCH")

// --- Time-Based Exit: Close if trade is open for too long ---
if (useTimeExit and strategy.position_size != 0)
    barsSinceEntry = bar_index - strategy.opentrades.entry_bar_index(0)
    if (barsSinceEntry > maxBarsInTrade)
        strategy.close_all(comment="Exit: Time Stop")

// --- EOD Exit: Close all positions at the end of the day (for intraday charts) ---
isNewDay = dayofweek != dayofweek[1]
if (isNewDay and timeframe.isintraday)
    strategy.close_all(comment="Exit: EOD")

// ... [Plotting logic can be adapted to show trades on chart using plotshape for strategy.opentrades, etc.] ...
```
    