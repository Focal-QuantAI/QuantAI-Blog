
# Indicators to Strategy Blueprint

Based on the provided "Scalper Pro 3 Min Gold" indicator, here is the architectural blueprint for its transformation into a production-grade automated execution framework.

### 1. Execution Triggers (Entry & Direction)

The core logic of the indicator is a breakout from a consolidation zone, confirmed by market structure. We will translate these visual cues into precise, machine-executable commands.

*   **Long Entry Condition (`bullishBreakout`):** A long position is initiated when ALL of the following conditions are met on a single bar:
    1.  The bar closes above the most recent significant pivot high (`lastPivotHigh`).
    2.  The low of the breakout bar remains above the most recent significant pivot low (`lastPivotLow`), confirming a higher-low structure.
    3.  The breakout bar is bullish (close > open).
    4.  The market was in a state of consolidation prior to the breakout, defined as the range of the previous `consLength` bars being less than or equal to `atrMult` times the 14-period ATR.
    5.  The system is not in a "cooldown" period; at least `cooldownBars` have passed since the last trade entry.

*   **Short Entry Condition (`bearishBreakout`):** A short position is initiated when ALL of the following conditions are met on a single bar:
    1.  The bar closes below the most recent significant pivot low (`lastPivotLow`).
    2.  The high of the breakout bar remains below the most recent significant pivot high (`lastPivotHigh`), confirming a lower-high structure.
    3.  The breakout bar is bearish (close < open).
    4.  The market meets the same consolidation criteria as the long entry.
    5.  The system is not in a "cooldown" period.

#### Execution Nuances

*   **Execution Timing:** The logic is based on `close`, `high`, and `low` of a completed bar. Therefore, orders must be executed **at the close of the signal bar** or, more realistically, **at the open of the next bar**. This "On Bar Close" approach prevents acting on false breakouts that fail to hold into the close (long wicks). For a scalping strategy, this is a crucial filter to reduce noise. Real-time execution based on price crossing a level intra-bar would introduce significant risk of "whipsaw" entries.

*   **Signal Reversals:** The built-in `cooldownBars` filter explicitly **prevents** immediate signal reversals. If the system is in a long position and a valid `bearishBreakout` signal occurs before the cooldown period ends, the signal is ignored. In a production environment, this is a clear rule: we do not flip positions. The current trade must be resolved (via Stop Loss, Take Profit, or other exit logic) and the cooldown must expire before a new trade, in either direction, can be considered.

### 2. Multi-Tiered Exit Logic

The original script's fixed Risk:Reward target is a good starting point but lacks adaptability. A professional system requires a more dynamic and layered exit strategy to manage risk and capitalize on momentum.

*   **Initial Stop Loss (Volatility-Based):**
    *   The script's method of placing the stop loss at the opposing pivot (`lastPivotLow` for longs, `lastPivotHigh` for shorts) is excellent. It is a structure-based, dynamic stop that adapts to recent market volatility. We will retain this as the initial stop placement mechanism.
    *   **Calculation:** For a long entry, `StopLossPrice = lastPivotLow`. For a short entry, `StopLossPrice = lastPivotHigh`. This value is calculated and fixed *at the moment of entry*.

*   **Take Profit / Trailing Mechanism (Dynamic & Multi-Stage):**
    *   **Stage 1 - Initial Target (Profit Securing):** Instead of a single target, we will implement a two-stage exit. The first target will be set at a 1.0 or 1.5 Risk:Reward ratio.
        *   `TargetPrice1 = EntryPrice + (RiskDistance * 1.5)`
        *   Upon reaching `TargetPrice1`, the strategy will close 50% of the position.
    *   **Stage 2 - Trailing Stop (Profit Maximization):** After Stage 1 is hit, the stop loss on the remaining 50% of the position is moved to breakeven. From this point, a dynamic trailing stop is activated. A robust choice is an **ATR Trailing Stop**.
        *   **Logic:** For a long position, the stop will trail at `high - (ATR * N)`, where `N` is a multiplier (e.g., 2.5). The stop only moves up, never down. This allows the remainder of the trade to capture a larger trend move while protecting profits.

*   **Time-Based Exits (Capital Preservation):**
    *   **End of Day (EOD) Exit:** As a scalping strategy on Gold, holding positions overnight introduces unnecessary gap risk. A hard rule will be implemented to close any open position at a specified time, for example, 15 minutes before the session close (e.g., 16:45 EST).
    *   **Stagnation Exit:** If a trade has been open for a specified number of bars (e.g., `25` bars) and has not hit either the initial stop loss or the first take-profit target, it will be closed. This prevents capital from being tied up in trades that lack momentum.

### 3. Capital Allocation & Risk Management

This is the most critical component in transitioning from a signal generator to a trading business. We will implement a fixed-fractional position sizing model.

*   **Risk-Based Sizing:** The strategy will risk a fixed percentage of account equity on every single trade, ensuring symmetrical risk exposure regardless of the trade's volatility.
    *   **Inputs:**
        *   `accountEquity`: The total current value of the trading account (`strategy.equity`).
        *   `riskPerTradePercent`: A user-defined input, e.g., `1.0` for 1% risk per trade.
    *   **Calculation Logic:**
        1.  **Determine Dollar Risk:** `dollarRisk = accountEquity * (riskPerTradePercent / 100)`.
        2.  **Determine Risk Per Contract/Unit:** `riskPerUnit = abs(entryPrice - stopLossPrice)`. For futures, this would be multiplied by the contract's point value.
        3.  **Calculate Position Size:** `positionSize = dollarRisk / riskPerUnit`. The result is then rounded down to the nearest tradable lot size.

*   **Pyramiding & Scaling:**
    *   **Scaling In (Pyramiding):** For this specific scalping logic, pyramiding (adding to a winning position) is **not recommended**. It adds significant complexity and risk to a short-term strategy. The model will be "one entry, one position."
    *   **Scaling Out:** As defined in the Multi-Tiered Exit Logic, scaling out is a core component. The strategy will exit 50% at the first profit target and trail the rest.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the `indicator` into a `strategy` incorporating the professional-grade features discussed above.

```pine
//@version=5
// STRATEGY CONVERSION: From Indicator to Execution Engine
strategy("Scalper Pro 3 Min Gold [ARCHITECT]", 
     overlay=true, 
     pyramiding=0, // No pyramiding allowed
     initial_capital=10000,
     default_qty_type=strategy.calculated, // Position size determined by our logic
     commission_type=strategy.commission.cash_per_order,
     commission_value=2.5, // Example: $2.50 per order (entry/exit)
     slippage=2) // Example: 2 ticks of slippage per order

// --- INPUTS ---
// Risk Management
riskPercent = input.float(1.0, title="Risk Per Trade (%)", minval=0.1, maxval=5.0)
// Original Logic Inputs
targetMult = input.float(1.5, title="Target 1 Multiplier (R:R)")
pivotLength = input.int(3, title="Structure Pivot Length")
// Filters
useConsolidation = input.bool(true, title="Require Consolidation Before Breakout", group="Filters")
consLength = input.int(10, title="Consolidation Length (Bars)", group="Filters")
atrMult = input.float(3.0, title="Max Range (ATR Multiplier)", group="Filters")
useCooldown = input.bool(true, title="Enable Cooldown Between Trades", group="Filters")
cooldownBars = input.int(30, title="Cooldown Length (Bars)", group="Filters")
// Exit Logic
useEodExit = input.bool(true, "Use End of Day Exit?", group="Exits")
eodTime = input.session("1645-1700", "EOD Exit Time", group="Exits") // Closes positions 15 mins before market close

// --- CORE LOGIC (Unchanged from original) ---
var float lastPivotHigh = na
var float lastPivotLow = na
var int lastTradeEntryBar = na

ph = ta.pivothigh(high, pivotLength, pivotLength)
pl = ta.pivotlow(low, pivotLength, pivotLength)
if not na(ph)
    lastPivotHigh := ph
if not na(pl)
    lastPivotLow := pl

pastHigh = ta.highest(high[1], consLength)
pastLow = ta.lowest(low[1], consLength)
currentAtr = ta.atr(14)
isConsolidating = useConsolidation ? (pastHigh - pastLow <= (currentAtr * atrMult)) : true
canEnter = useCooldown ? (na(lastTradeEntryBar) or (bar_index - lastTradeEntryBar >= cooldownBars)) : true

bullishBreakout = close > open and ta.crossover(close, lastPivotHigh) and low > lastPivotLow and isConsolidating and canEnter
bearishBreakout = close < open and ta.crossunder(close, lastPivotLow) and high < lastPivotHigh and isConsolidating and canEnter

// --- EXECUTION ENGINE ---

// Check if we are in a position
bool inTrade = strategy.position_size != 0
bool isFlat = strategy.position_size == 0
bool isTimeForEodExit = useEodExit and time(timeframe.period, eodTime)

// Position Sizing Calculation
float calculatePositionSize(float stopPrice) =>
    float riskPerUnit = math.abs(close - stopPrice)
    if riskPerUnit == 0
        0 // Avoid division by zero
    else
        float dollarRisk = (riskPercent / 100.0) * strategy.equity
        dollarRisk / riskPerUnit

// --- ENTRY LOGIC ---
if (isFlat)
    // Long Entry
    if (bullishBreakout)
        float stopPrice = lastPivotLow
        float targetPrice = close + (math.abs(close - stopPrice) * targetMult)
        float qty = calculatePositionSize(stopPrice)
        
        if qty > 0
            strategy.entry("Long", strategy.long, qty=qty)
            strategy.exit("Long Exit", from_entry="Long", stop=stopPrice, limit=targetPrice)
            lastTradeEntryBar := bar_index

    // Short Entry
    if (bearishBreakout)
        float stopPrice = lastPivotHigh
        float targetPrice = close - (math.abs(close - stopPrice) * targetMult)
        float qty = calculatePositionSize(stopPrice)

        if qty > 0
            strategy.entry("Short", strategy.short, qty=qty)
            strategy.exit("Short Exit", from_entry="Short", stop=stopPrice, limit=targetPrice)
            lastTradeEntryBar := bar_index

// --- EXIT LOGIC ---
// End of Day Exit
if (inTrade and isTimeForEodExit)
    strategy.close_all(comment="EOD Exit")

// NOTE: The multi-stage exit with trailing ATR requires more complex state management
// and is best implemented using `var` variables to track the state of the trade (e.g., if TP1 was hit).
// The `strategy.exit` call above represents the initial bracket order for Stop Loss and the FIRST Take Profit.
// A more advanced implementation would cancel this and submit a new trailing stop order once TP1 is filled.
```
    