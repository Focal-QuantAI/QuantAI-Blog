
# Indicators to Strategy Blueprint

The provided Pine Script is an `indicator` designed for visual analysis, identifying the highest and lowest price points within the user's visible chart window. Its core logic, relying on `chart.left_visible_bar_time`, is fundamentally incompatible with automated execution, as a server-side strategy has no concept of a "visible window."

To transform this into a professional algorithmic trading framework, we must first translate its *intent*—trading based on recent price extremes—into a quantifiable, non-visual rule. The most direct translation is to replace the "visible range" with a fixed "lookback period."

This new strategy will be a classic **breakout system**: it will enter long when price breaks above a recent high and short when it breaks below a recent low.

### 1. Execution Triggers (Entry & Direction)

The core of the strategy will be to identify the highest high and lowest low over a fixed `lookbackPeriod`. A breakout of this range will trigger an entry.

*   **Long Entry Condition:** The `close` of the current bar must be greater than the `highest high` over the preceding `lookbackPeriod` bars.
    *   `boolean long_condition = ta.crossover(close, highest(high, lookbackPeriod)[1])`
*   **Short Entry Condition:** The `close` of the current bar must be less than the `lowest low` over the preceding `lookbackPeriod` bars.
    *   `boolean short_condition = ta.crossunder(close, lowest(low, lookbackPeriod)[1])`

**Execution Nuances:**

*   **Execution Timing:** Orders will be executed **at the Close of the bar** where the condition is met (`process_orders_on_close=true`). This is a robust, non-repainting approach suitable for production. Attempting to execute on real-time price action for this type of breakout can lead to false signals and whipsaws as the bar's price fluctuates before closing.
*   **Signal Reversals:** The system is designed for immediate reversals. If the strategy is in a long position and a `short_condition` is triggered, the framework will automatically close the long position and initiate a new short position. This is inherent to using `strategy.entry` for both directions. The `[1]` offset in the high/low calculation is critical; it ensures we are comparing the current `close` to the range established by *previous* bars, preventing the entry bar from influencing its own signal.

### 2. Multi-Tiered Exit Logic

A professional strategy never enters a trade without a predefined exit plan. We will implement a multi-faceted exit logic to manage risk and secure profits.

*   **Initial Stop Loss:** The stop loss will be calculated dynamically based on market volatility using the Average True Range (ATR). This is superior to a fixed percentage, as it adapts to changing market conditions.
    *   **Long Position Stop:** `Entry Price - (ATR * Multiplier)`
    *   **Short Position Stop:** `Entry Price + (ATR * Multiplier)`
    *   The `Multiplier` (e.g., 2.5) will be a configurable parameter.

*   **Take Profit/Trailing:** We will implement a two-stage profit-taking mechanism.
    1.  **Initial Target (TP1):** A fixed Risk-to-Reward ratio target (e.g., 1.5R). When `TP1` is hit, we can scale out a portion of the position (e.g., 50%).
    2.  **Trailing Stop:** After `TP1` is hit, the stop loss for the remaining position is moved to breakeven. From this point, a trailing stop based on ATR or significant swing lows/highs will be activated to let the remainder of the position run while protecting profits.

*   **Time-Based Exits:** Capital should not be held hostage by stagnant trades.
    *   **End of Day (EOD) Exit:** For intraday strategies, a hard rule to close all open positions 5-10 minutes before the session close is mandatory to avoid overnight risk and funding charges.
    *   **Stagnation Exit:** If a trade has been open for `X` bars (e.g., 50 bars) without hitting either its stop loss or take profit, it will be closed. This frees up capital for more promising opportunities.

### 3. Capital Allocation & Risk Management

Position sizing is arguably the most critical component of a profitable system. We will use a dynamic, risk-based model.

*   **Risk-Based Sizing:** The core principle is to risk a fixed percentage of total account equity on every single trade, regardless of the trade's setup. For this example, we'll target risking **1% of equity per trade**.
    *   **Stop Loss Distance (in currency):** `stop_distance = abs(entry_price - stop_loss_price)`
    *   **Risk Amount (in currency):** `risk_amount = strategy.equity * risk_per_trade_pct`
    *   **Position Size (in contracts/shares):** `position_size = risk_amount / stop_distance`
    *   This calculation is performed *before* the entry order is sent, ensuring every trade has the same potential dollar risk.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, we will sell portions of the position at predefined profit targets. For example, sell 50% at 1.5R, and let the rest run on a trailing stop.
    *   **Pyramiding (Scaling In):** Adding to a winning position is an advanced technique. A rule could be: "If the position is profitable by at least 1R, and a *new* valid breakout signal occurs in the same direction, a new, smaller position (e.g., 50% of the initial size) can be added." The stop loss for the entire combined position must then be trailed up aggressively, for instance, to the breakeven point of the most recent entry, to ensure the overall trade risk does not increase.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the transformation from a visual `indicator` to a professional `strategy` incorporating the principles above.

```pine
//@version=5
// STEP 0: STRATEGY DECLARATION WITH REALISTIC PARAMETERS
strategy(
     "Professional Breakout Strategy", 
     overlay=true, 
     initial_capital=25000, 
     process_orders_on_close=true, // Execute on bar close for robustness
     commission_type=strategy.commission.percent, 
     commission_value=0.075, // Example commission
     slippage=2 // Example slippage in ticks
)

// SECTION 1: INPUTS & CONFIGURATION
string group_trade = "Trade Parameters"
lookbackPeriod = input.int(50, "Breakout Lookback Period", group=group_trade)
atrPeriod = input.int(14, "ATR Period", group=group_trade)
stopAtrMultiplier = input.float(2.5, "Stop Loss ATR Multiplier", group=group_trade)
rr_target = input.float(1.5, "Risk/Reward Target for TP1", group=group_trade)

string group_risk = "Risk Management"
equityRiskPercent = input.float(1.0, "Equity Risk % Per Trade", group=group_risk) / 100
useTimeExit = input.bool(true, "Use End-of-Day Exit?", group=group_risk)

// SECTION 2: CORE CALCULATIONS
// Translate the original script's "visible range" to a fixed lookback
float breakoutHigh = request.security(syminfo.tickerid, timeframe.period, high[1], lookback=lookbackPeriod)
float breakdownLow = request.security(syminfo.tickerid, timeframe.period, low[1], lookback=lookbackPeriod)
float atr = ta.atr(atrPeriod)

// SECTION 3: EXECUTION TRIGGERS & RISK MANAGEMENT
// Entry Conditions
bool longCondition = ta.crossover(close, breakoutHigh)
bool shortCondition = ta.crossunder(close, breakdownLow)

// Risk Sizing Logic
float stopDistance = atr * stopAtrMultiplier
float positionSize = (strategy.equity * equityRiskPercent) / stopDistance

// SECTION 4: ORDER EXECUTION
if (longCondition)
    // Calculate SL/TP prices *before* entry
    longStopPrice = close - stopDistance
    longTakePrice = close + (stopDistance * rr_target)
    // Execute Entry
    strategy.entry("Long", strategy.long, qty=positionSize)
    // Place Exit orders immediately
    strategy.exit("Long Exit", from_entry="Long", stop=longStopPrice, limit=longTakePrice)

if (shortCondition)
    // Calculate SL/TP prices *before* entry
    shortStopPrice = close + stopDistance
    shortTakePrice = close - (stopDistance * rr_target)
    // Execute Entry
    strategy.entry("Short", strategy.short, qty=positionSize)
    // Place Exit orders immediately
    strategy.exit("Short Exit", from_entry="Short", stop=shortStopPrice, limit=shortTakePrice)

// Time-Based Exit Logic
isLastSessionBar = (time_close(timeframe.period) - time) / (1000 * 60) <= 15 // Within 15 mins of session close
if useTimeExit and isLastSessionBar
    strategy.close_all(comment="EOD Exit")

// Plotting for visualization
plot(breakoutHigh, "Breakout Level", color.new(color.green, 0))
plot(breakdownLow, "Breakdown Level", color.new(color.red, 0))
```
