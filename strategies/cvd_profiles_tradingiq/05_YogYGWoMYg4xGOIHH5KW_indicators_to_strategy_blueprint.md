
# Indicators to Strategy Blueprint

Based on the provided `CVD Profiles` indicator, which is a sophisticated contextual tool for analyzing order flow, we can architect a breakout confirmation strategy. The indicator itself provides no explicit signals; it offers a map of market activity. Our strategy will interpret this map to generate high-probability entry and exit orders.

The core thesis is to trade breakouts from the established session Value Area (VA), but only when confirmed by a surge in Cumulative Volume Delta (CVD), indicating that strong, directional order flow is driving the move.

### 1. Execution Triggers (Entry & Direction)

The strategy will operate on the close of each bar to ensure all profile data for that bar is final and not subject to repainting. This is a critical consideration for strategies using volume profile or CVD data, which are cumulative by nature.

**Key Data Points to Extract:**
*   `VAH`: The Value Area High for the current session.
*   `VAL`: The Value Area Low for the current session.
*   `cvdDelta`: The bar-over-bar change in Cumulative Volume Delta. This is our order flow confirmation metric.

**Boolean Entry Conditions:**

A dynamic threshold will be used for `cvdDelta` to filter out noise. We will only consider a delta surge significant if it exceeds a multiple of its recent average.

```
// Threshold Calculation
cvdDeltaSma = ta.sma(math.abs(cvdDelta), 20)
cvdThreshold = cvdDeltaSma * 1.5

// Long Entry Condition
longCondition = close > VAH and cvdDelta > cvdThreshold

// Short Entry Condition
shortCondition = close < VAL and cvdDelta < -cvdThreshold
```

*   **Long Entry:** The price must close **above** the session's Value Area High, AND the `cvdDelta` for that bar must be positive and significantly larger than its recent average. This confirms that aggressive buyers are participating in the breakout.
*   **Short Entry:** The price must close **below** the session's Value Area Low, AND the `cvdDelta` for that bar must be negative and its absolute value significantly larger than its recent average. This confirms aggressive sellers are driving the breakdown.

**Execution Nuances:**

*   **Execution Timing:** All entry decisions will be made at the bar's close. The `strategy()` declaration will include `process_orders_on_close = true`. Attempting to execute this logic intra-bar would lead to flawed backtests and live performance, as the VAH, VAL, and `cvdDelta` values are only finalized once the bar completes.
*   **Signal Reversals:** The system will be designed to handle reversals. If the strategy is in a long position and a valid `shortCondition` triggers, the system will first close the existing long position and then immediately enter a new short position on the same bar. This is achieved by using `strategy.close()` followed by `strategy.entry()`.

### 2. Multi-Tiered Exit Logic

A static stop-loss or take-profit is insufficient for a professional system. Our exit logic will be dynamic, adapting to market volatility and trade progression.

*   **Initial Stop Loss (Volatility-Based):**
    *   The stop loss will be calculated using the Average True Range (ATR). Upon entry, the initial stop will be placed at a multiple (e.g., 2x) of the 14-period ATR away from the entry price.
    *   *Alternative (Structure-Based):* For a more sophisticated approach, the stop for a long breakout above VAH could be placed at the session's Point of Control (POC). This places the stop at a logical point of high liquidity, giving the trade a structural "line in the sand."
    *   **Calculation:**
        *   `longStopPrice = entry_price - (ta.atr(14) * 2)`
        *   `shortStopPrice = entry_price + (ta.atr(14) * 2)`

*   **Take Profit/Trailing (Multi-Stage):**
    1.  **Target 1 (Scaling Out):** A primary take-profit target will be set at a 1.5:1 Risk/Reward ratio. If the initial risk (distance from entry to stop) is $50, the first take-profit will be at Entry + $75. At this point, 50% of the position will be closed.
    2.  **Trailing Stop Activation:** Once Target 1 is hit, the stop loss on the remaining 50% of the position is moved to the original entry price (breakeven).
    3.  **Dynamic Trailing:** For the remaining position, an ATR-based trailing stop is activated. The stop will trail the highest price (for longs) or lowest price (for shorts) by a distance of 2.5x ATR, allowing the winning portion of the trade to run while protecting profits.

*   **Time-Based Exits:**
    *   **End of Day (EOD):** As this is a session-based strategy, all open positions must be automatically closed 15 minutes before the market close to avoid overnight risk and settlement issues.
    *   **Stagnation Exit:** If a trade has been open for a specified number of bars (e.g., 25 bars) and has failed to reach the first take-profit target, it will be closed automatically. This frees up capital from non-performing trades.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component of risk management. We will use a fixed-fractional model to ensure consistent risk exposure on every trade.

*   **Risk-Based Sizing:**
    *   The strategy will risk a fixed percentage (e.g., 1%) of the total account equity on each trade. The size of the position is determined by the distance to the initial stop loss.
    *   **Formula:**
        `Risk_Amount_Per_Trade = Account_Equity * Risk_Percentage`
        `Stop_Loss_Distance_in_Dollars = abs(Entry_Price - Stop_Loss_Price) * Ticks_Per_Point * Tick_Value`
        `Position_Size (Contracts/Shares) = Risk_Amount_Per_Trade / Stop_Loss_Distance_in_Dollars`
    *   This formula ensures that whether the stop is wide (low volatility) or tight (high volatility), the maximum potential loss on the trade remains a constant fraction of the account.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, the system is designed to scale out of positions at pre-defined profit milestones. This is the primary method of managing a winning trade.
    *   **Pyramiding (Scaling In):** Adding to a winning position will be disabled by default to maintain risk simplicity. If enabled, a strict rule would apply: A new position (pyramid) can only be added if the price has moved at least one full Risk/Reward unit (e.g., 1.5x the initial risk) in favor of the trade, AND a new, valid entry signal (e.g., another CVD confirmation bar) occurs. The stop for the entire combined position would then be moved to the breakeven point of the *last* entry.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates how to wrap the core logic from the `CVD Profiles` indicator into a professional `strategy()` framework. It focuses on the execution engine, not the complex drawing elements of the original script.

```pine
//@version=5
// --- STRATEGY DECLARATION ---
strategy("CVD Breakout Architect", 
     overlay=true, 
     process_orders_on_close=true,
     initial_capital=100000,
     default_qty_type=strategy.fixed,
     commission_type=strategy.commission.percent,
     commission_value=0.07, // Example: 0.07%
     slippage=2 // Example: 2 ticks of slippage
     )

// --- INPUTS ---
riskPercent = input.float(1.0, "Risk per Trade %", minval=0.1, maxval=10) / 100
atrPeriod = input.int(14, "ATR Period")
slMultiplier = input.float(2.0, "Stop Loss ATR Multiplier")
tp1RR = input.float(1.5, "Take Profit 1 R:R")
cvdThresholdMultiplier = input.float(1.5, "CVD Delta Threshold Multiplier")
eodCloseMinutes = input.int(15, "Minutes Before EOD to Close")

// --- CORE DATA EXTRACTION (Simplified from original script) ---
// NOTE: This requires refactoring the original script to return these values.
// For this example, we assume functions `get_vah()`, `get_val()`, and `get_cvd_delta()` exist.
var float VAH = na
var float VAL = na
var float cvdDelta = na

// [VAH, VAL, cvdDelta] = getSignalData() // Placeholder for refactored indicator logic

// --- RISK & SIZING CALCULATION ---
atrValue = ta.atr(atrPeriod)
stopLossDistance = atrValue * slMultiplier
riskAmount = strategy.equity * riskPercent
positionSize = riskAmount / (stopLossDistance * syminfo.pointvalue)

// --- ENTRY CONDITIONS ---
cvdDeltaSma = ta.sma(math.abs(cvdDelta), 20)
cvdThreshold = cvdDeltaSma * cvdThresholdMultiplier

longCondition = not na(VAH) and close > VAH and cvdDelta > cvdThreshold
shortCondition = not na(VAL) and close < VAL and cvdDelta < -cvdThreshold

// --- EXIT CONDITIONS ---
isEOD = (hour(time_close) == 15 and minute(time_close) >= (60 - eodCloseMinutes)) // Example for US Equities

// --- EXECUTION LOGIC ---
if (strategy.position_size == 0) // Only look for new entries if flat
    if (longCondition)
        longStopPrice = close - stopLossDistance
        longTakeProfitPrice = close + (stopLossDistance * tp1RR)
        strategy.entry("Long", strategy.long, qty=positionSize)
        strategy.exit("Long Exit", from_entry="Long", stop=longStopPrice, limit=longTakeProfitPrice)

    if (shortCondition)
        shortStopPrice = close + stopLossDistance
        shortTakeProfitPrice = close - (stopLossDistance * tp1RR)
        strategy.entry("Short", strategy.short, qty=positionSize)
        strategy.exit("Short Exit", from_entry="Short", stop=shortStopPrice, limit=shortTakeProfitPrice)

// EOD Exit
if (isEOD)
    strategy.close_all(comment="EOD Close")

```
    