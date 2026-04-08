
# Indicators to Strategy Blueprint

The provided script is a sophisticated visualization tool for Volume Profile analysis, offering context rather than explicit trading signals. To transform this into a production-ready automated strategy, we must first define a clear, testable hypothesis based on its outputs.

A classic and robust approach is to build a **mean-reversion strategy** around the key Value Area (VA) levels. The core assumption is that the price will tend to revert to the area of highest liquidity (the Value Area) after extending beyond its boundaries (VAH and VAL).

### 1. Execution Triggers (Entry & Direction)

The strategy will identify when the price has moved outside the previous session's value area and then shows signs of returning. This "failed auction" is our primary entry trigger. We will use the primary Higher Timeframe (HTF #1) for our levels, but this logic can be extended to any of the configured timeframes.

*   **Long Entry Condition:**
    1.  The price must first trade *below* the previous HTF session's Value Area Low (VAL).
    2.  The entry signal is triggered when the price subsequently closes *back above* that same VAL. This confirms the rejection of lower prices and the start of a potential reversion back into the value area.

*   **Short Entry Condition:**
    1.  The price must first trade *above* the previous HTF session's Value Area High (VAH).
    2.  The entry signal is triggered when the price subsequently closes *back below* that same VAH. This confirms the rejection of higher prices and a potential move back towards value.

*   **Execution Nuances:**
    *   **Execute at "Close" of the bar:** This is non-negotiable for this strategy. The confirmation signal *is* the close back inside the value area. Attempting to execute on real-time price action (e.g., as soon as the price crosses the level intra-bar) would result in numerous false signals from wicks and should be avoided. The strategy must wait for the bar to complete to validate the signal.
    *   **Signal Reversals:** The system must be able to "flip" its position. If the strategy is in a long position and a valid short entry signal occurs, the framework should automatically close the long position and initiate the new short position. This is handled in Pine Script by using the same `id` for both `strategy.entry` calls for long and short trades.

### 2. Multi-Tiered Exit Logic

A static stop or target is insufficient. A professional exit framework must adapt to market conditions and manage the trade dynamically.

*   **Initial Stop Loss:**
    *   **Volatility-Based Calculation:** The stop loss will be placed based on the Average True Range (ATR). Upon a long entry (crossing back above VAL), the stop loss will be set at `Entry Price - (ATR_Multiplier * ATR)`. A typical `ATR_Multiplier` is between 1.5 and 2.5.
    *   **Logic:** This ensures the stop is placed outside the normal "noise" of the market. For a long entry, a logical alternative is to place the stop below the low of the candle that initiated the move below the VAL, but an ATR-based stop is more systematic and easier to automate.

*   **Take Profit / Trailing Mechanism:**
    *   **Target 1 (Scaling Out):** The first logical target for a mean-reversion trade is the **Point of Control (POC)** of the same session. Upon reaching the POC, the strategy will sell a portion of the position (e.g., 50%).
    *   **Target 2 (Final Target):** The final target is the opposite side of the value area. For a long trade, this is the **Value Area High (VAH)**.
    *   **Dynamic Trailing:** Once Target 1 (POC) is hit and the position is partially closed, the stop loss on the remaining portion should be moved to **breakeven**. From there, a trailing stop can be activated, such as an ATR-based trail or a Chandelier Exit, to protect profits while allowing the trade to reach its final target.

*   **Time-Based Exits:**
    *   **End of Session:** If a trade is still open as the current HTF session is about to close, the position should be squared. This prevents holding a mean-reversion trade into the uncertainty of a new session's value area formation.
    *   **Stagnation Exit:** If a trade has been open for a specified number of bars (e.g., `MaxBarsInTrade = 50`) without hitting either a stop loss or a take-profit level, it should be closed. This frees up capital from non-performing trades.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term survival. We will implement a fixed-fractional risk model.

*   **Risk-Based Sizing:**
    The strategy will risk a fixed percentage of account equity on every single trade.
    1.  Define `RiskPerTrade` (e.g., `0.01` for 1% of equity).
    2.  Calculate `TradeRiskInCurrency` = `strategy.equity * RiskPerTrade`.
    3.  Calculate `StopLossDistanceInPoints` = `abs(EntryPrice - StopLossPrice)`.
    4.  Calculate `PositionSize` = `TradeRiskInCurrency / (StopLossDistanceInPoints * syminfo.pointvalue)`.
    This calculation must be performed *before* each entry, ensuring that a wider stop results in a smaller position size and vice-versa, keeping the dollar risk constant.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, the strategy will scale out by default: 50% at the POC and the remaining 50% at the VAH/VAL.
    *   **Pyramiding (Scaling In):** This is an advanced feature and should be disabled by default. If enabled, rules must be strict. For example: A new position can only be added if the original trade is in profit by at least 1R (one times the initial risk) *and* a new low-risk entry forms, such as a pullback to a fast-moving average that has now crossed into the value area. The total risk of all combined positions should not exceed a predefined maximum.

### 4. Implementation Snippet (Pine Logic)

The provided script is an `indicator` that draws objects on the last bar. A significant refactoring is required to make it a `strategy` that can access historical VAH, VAL, and POC values. The `getHTFvals` function needs to be modified to store these key levels for each completed HTF period, likely in an array or map.

Assuming this refactoring is done and we have access to `htf1_VAH`, `htf1_VAL`, and `htf1_POC` as historical series, the strategy implementation would look like this:

```pine
// This is a conceptual snippet assuming the source script has been refactored
// to provide historical VAH, VAL, and POC data for each HTF session.

//@version=5
strategy("VP Mean Reversion Strategy", 
     overlay=true, 
     pyramiding=0, // No pyramiding by default
     initial_capital=100000, 
     default_qty_type=strategy.fixed, 
     commission_type=strategy.commission.cash_per_order, 
     commission_value=4, // Example: $4 per order
     slippage=2) // Example: 2 ticks of slippage

// --- Strategy Inputs ---
riskPercent     = input.float(1.0, "Risk per Trade %", minval=0.1, maxval=10) / 100
atrLength       = input.int(14, "ATR Length")
atrMultiplier   = input.float(2.0, "ATR Stop Multiplier")
maxBarsInTrade  = input.int(50, "Max Bars in Trade (Stagnation Exit)")

// --- Data Retrieval (Requires Refactoring Original Script) ---
// These functions would need to be created to return historical values, not just draw them.
[htf1_VAH, htf1_VAL, htf1_POC] = getHistoricalVPLevels(htf1) 

// --- Core Calculations ---
atrValue = ta.atr(atrLength)

// We trade against the PREVIOUS session's value area.
prev_VAH = htf1_VAH[1]
prev_VAL = htf1_VAL[1]
prev_POC = htf1_POC[1]

// --- Entry Conditions ---
longEntryCondition = ta.crossunder(low, prev_VAL) and ta.crossover(close, prev_VAL)
shortEntryCondition = ta.crossover(high, prev_VAH) and ta.crossunder(close, prev_VAH)

// --- Risk Management & Sizing ---
longStopPrice = close - (atrValue * atrMultiplier)
shortStopPrice = close + (atrValue * atrMultiplier)

riskCapital = strategy.equity * riskPercent
longPositionSize = riskCapital / ((close - longStopPrice) * syminfo.pointvalue)
shortPositionSize = riskCapital / ((shortStopPrice - close) * syminfo.pointvalue)

// --- Time-Based Exit ---
isNewSession = timeframe.change(htf1)
stagnationExit = barssince(strategy.opentrades > 0) > maxBarsInTrade

if (stagnationExit or isNewSession)
    strategy.close_all(comment="Time/Stagnation Exit")

// --- Execution Logic ---
if (longEntryCondition and strategy.opentrades == 0)
    strategy.entry("VP_Long", strategy.long, qty=longPositionSize)
    // Multi-tier exit order
    strategy.exit("Exit Long", from_entry="VP_Long", stop=longStopPrice, limit=prev_POC, qty_perc=50)
    strategy.exit("Exit Long TP2", from_entry="VP_Long", limit=prev_VAH) // Exit remaining at VAH

if (shortEntryCondition and strategy.opentrades == 0)
    strategy.entry("VP_Short", strategy.short, qty=shortPositionSize)
    // Multi-tier exit order
    strategy.exit("Exit Short", from_entry="VP_Short", stop=shortStopPrice, limit=prev_POC, qty_perc=50)
    strategy.exit("Exit Short TP2", from_entry="VP_Short", limit=prev_VAL) // Exit remaining at VAL

// --- Plotting for Verification ---
plot(prev_VAH, "VAH", color.red, style=plot.style_stepline)
plot(prev_VAL, "VAL", color.green, style=plot.style_stepline)
plot(prev_POC, "POC", color.blue, style=plot.style_cross)
```
    