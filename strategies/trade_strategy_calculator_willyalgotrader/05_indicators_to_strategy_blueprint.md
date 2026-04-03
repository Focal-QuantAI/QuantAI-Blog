
# Indicators to Strategy Blueprint

The provided script is a powerful manual trade calculator, functioning as a dashboard rather than an automated trading system. To transform this `indicator` into a production-ready `strategy`, we must replace its manual inputs with algorithmic triggers and build a robust execution engine around its risk calculation core.

We will introduce a simple, common logic—a dual moving average (MA) crossover—to generate entry signals. This will serve as the foundation upon which we build the professional execution framework.

### 1. Execution Triggers (Entry & Direction)

The core challenge is to replace the manual `directionInput` and `entryManualInput` with dynamic, market-driven signals. We will use a fast and slow moving average crossover for this purpose.

*   **Long Entry Condition:** A long position is initiated when the `fastMA` crosses above the `slowMA`.
    *   `longCondition = ta.crossover(fastMA, slowMA)`
*   **Short Entry Condition:** A short position is initiated when the `fastMA` crosses below the `slowMA`.
    *   `shortCondition = ta.crossunder(fastMA, slowMA)`

#### Execution Nuances

*   **Execution Timing:** For strategies based on standard indicators like moving averages, execution **at the "Close" of the bar** is paramount. This ensures the signal is confirmed and prevents the strategy from firing multiple orders based on intra-bar price fluctuations (whipsawing), which is a common cause of backtest overfitting. This is achieved in Pine Script with `process_orders_on_close = true` within the `strategy()` declaration. Real-time execution (`calc_on_every_tick = true`) should be reserved for strategies specifically designed for tick-level scalping, which this is not.

*   **Signal Reversals:** The system must handle a new signal in the opposite direction while a trade is already open. The most robust method is to explicitly close the existing position before entering the new one. This provides clear accounting for the exit reason (e.g., "Reversal Close") and ensures precise control over the new entry.
    *   **Logic:** If `longCondition` becomes true while `strategy.position_size < 0` (a short is open), the strategy will first execute `strategy.close()` for the short position, then execute `strategy.entry()` for the new long position.

### 2. Multi-Tiered Exit Logic

A static take-profit is insufficient for professional trading. We will implement a dynamic, multi-stage exit system that leverages the excellent multi-TP framework from the original script but enhances it with volatility-based stops and time constraints.

*   **Initial Stop Loss (Volatility-Based):**
    Instead of a fixed percentage (`slPctInput`), the stop loss will be calculated using the Average True Range (ATR). This adapts the stop distance to current market volatility, placing it wider in volatile conditions and tighter in calm markets.
    *   **Calculation:** `stopLossDistance = ta.atr(14) * atrMultiplier` (where `atrMultiplier` is a new input, e.g., 2.0).
    *   **Long SL Price:** `entryPrice - stopLossDistance`
    *   **Short SL Price:** `entryPrice + stopLossDistance`

*   **Take Profit / Trailing Mechanism:**
    We will adopt the multi-TP structure from the calculator but implement it as a scaling-out mechanism. The final portion of the position will be managed by a trailing stop to capture runaway trends.
    1.  **TP1 (e.g., 1R):** Close a portion of the position (e.g., 50%) at a profit target calculated from the ATR-based risk. `tp1Price = entryPrice + (stopLossDistance * rrTp1Input)`.
    2.  **Move to Breakeven:** Once TP1 is hit, the stop loss for the remaining position is moved to the entry price to eliminate further risk.
    3.  **TP2 / Trailing Stop:** The remaining portion of the trade can either target a second fixed R:R level (`rrTp2Input`) or, more effectively, be managed with a trailing stop. The trailing stop would activate after TP1 is hit, trailing the price by a multiple of ATR (e.g., `1.5 * ta.atr(14)`). This allows the trade to capture a large, unexpected move.

*   **Time-Based Exits:**
    Capital must be protected from both time decay and opportunity cost.
    *   **End of Day (EOD) Exit:** For intraday strategies, all open positions should be squared off before the session close to avoid overnight risk. This can be implemented by checking if the trading day has changed.
    *   **Stagnation Exit:** If a trade has been open for a specified number of bars (e.g., 50 bars) without hitting either its stop loss or a take-profit target, it should be closed. This frees up capital from non-performing trades.

### 3. Capital Allocation & Risk Management

The calculator's core strength is its position sizing logic. We will integrate this directly into the strategy's execution flow.

*   **Risk-Based Sizing:**
    The strategy will calculate the position size for every trade to risk a fixed percentage of the current account equity. This is the cornerstone of professional risk management.
    *   **Step 1: Define Risk Amount in USD:** `riskAmountUSD = strategy.equity * (riskPctInput / 100.0)`
    *   **Step 2: Define Risk Per Share/Unit:** This is the dollar distance from the entry price to the volatility-based stop loss price. `riskPerUnit = abs(entryPrice - stopLossPrice)`
    *   **Step 3: Calculate Position Size (Quantity):** `positionQuantity = riskAmountUSD / riskPerUnit`
    *   This calculation must be performed *immediately before* the `strategy.entry()` call on every new signal.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** This is handled by our multi-tiered exit logic. We will use `strategy.exit` with specified `qty_percent` parameters (e.g., `qty_percent = 50` for TP1) to close portions of the position at different price levels.
    *   **Pyramiding (Scaling In):** Adding to a winning position is an advanced technique. Clear rules must be established to prevent "revenge adding" to a losing trade.
        *   **Rule 1:** Set `strategy(pyramiding = X)` where X is the maximum number of additional entries allowed (e.g., 2).
        *   **Rule 2:** A pyramid entry is only allowed if the initial position is in profit by at least 1R (one risk unit).
        *   **Rule 3:** The stop loss for the *entire* combined position must be moved to a point that ensures the total trade cannot result in a loss greater than the initial 1R risk. Often, this means moving the stop for the whole position to the breakeven point of the initial entry.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the transformation from the manual calculator to an automated strategy, incorporating the principles above. It replaces the manual inputs with MA crossover logic and integrates the risk and exit management into the `strategy` framework.

```pine
// This Pine Script™ code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
//@version=5

// 1. STRATEGY DECLARATION - From Indicator to a full-fledged Strategy
strategy(
    "Automated Execution Framework [WillyAlgoTrader]",
    overlay = true,
    initial_capital = 10000, // Corresponds to 'depositInput'
    process_orders_on_close = true, // Ensures execution on confirmed signals
    commission_type = strategy.commission.percent,
    commission_value = 0.04, // Corresponds to 'commissionInput'
    slippage = 5 // Represents slippage in ticks, a more realistic measure
)

// 2. INPUTS - Algorithmic triggers and dynamic risk
// --- Entry Logic Inputs
fastMA_len = input.int(20, "Fast MA Length", group="Execution Triggers")
slowMA_len = input.int(50, "Slow MA Length", group="Execution Triggers")

// --- Risk & Exit Logic Inputs (Adapted from original script)
riskPctInput = input.float(1.0, "Risk per Trade (%)", minval=0.1, step=0.1, group="Risk Management")
atrLength = input.int(14, "ATR Length for SL", group="Risk Management")
atrMultiplierSL = input.float(2.0, "ATR Multiplier for Stop Loss", group="Risk Management")
rrTp1 = input.float(1.0, "R:R for TP1", group="Risk Management")
rrTp2 = input.float(2.0, "R:R for TP2", group="Risk Management")
exitQtyPctTp1 = input.int(50, "Volume to Close at TP1 (%)", group="Risk Management")
maxBarsInTrade = input.int(100, "Max Bars in Trade (Stagnation Exit)", group="Risk Management")

// 3. CORE LOGIC - Indicators and Conditions
fastMA = ta.ema(close, fastMA_len)
slowMA = ta.ema(close, slowMA_len)
atrValue = ta.atr(atrLength)

plot(fastMA, "Fast MA", color.aqua)
plot(slowMA, "Slow MA", color.orange)

longCondition = ta.crossover(fastMA, slowMA)
shortCondition = ta.crossunder(fastMA, slowMA)

// 4. EXECUTION ENGINE - Sizing and Order Management
if (strategy.position_size == 0) // Only calculate for new entries
    // --- Risk-Based Position Sizing ---
    riskAmountUSD = strategy.equity * (riskPctInput / 100.0)
    stopLossDistance = atrValue * atrMultiplierSL
    positionQuantity = riskAmountUSD / stopLossDistance

    // --- Multi-Tiered Exit Price Calculation ---
    tp1Distance = stopLossDistance * rrTp1
    tp2Distance = stopLossDistance * rrTp2

    // --- Entry and Exit Order Placement ---
    if (longCondition)
        // Entry
        strategy.entry("Long", strategy.long, qty = positionQuantity)
        // Multi-Tiered Exit: One command for SL and multiple TPs
        strategy.exit("Long Exit", from_entry="Long", 
             stop = close - stopLossDistance, 
             limit = close + tp1Distance, qty_percent = exitQtyPctTp1, comment="TP1 Hit")
        strategy.exit("Long Exit TP2", from_entry="Long", 
             stop = close - stopLossDistance, 
             limit = close + tp2Distance, comment="TP2 Hit / Final Exit")

    if (shortCondition)
        // Entry
        strategy.entry("Short", strategy.short, qty = positionQuantity)
        // Multi-Tiered Exit
        strategy.exit("Short Exit", from_entry="Short", 
             stop = close + stopLossDistance, 
             limit = close - tp1Distance, qty_percent = exitQtyPctTp1, comment="TP1 Hit")
        strategy.exit("Short Exit TP2", from_entry="Short", 
             stop = close + stopLossDistance, 
             limit = close - tp2Distance, comment="TP2 Hit / Final Exit")

// --- Time-Based Exit Logic ---
if (bar_index - strategy.opentrades.entry_bar_index(0) > maxBarsInTrade)
    strategy.close_all(comment = "Stagnation Exit")

// --- Signal Reversal Logic ---
if (strategy.position_size > 0 and shortCondition)
    strategy.close("Long", comment = "Reversal to Short")

if (strategy.position_size < 0 and longCondition)
    strategy.close("Short", comment = "Reversal to Long")

```
