
# Indicators to Strategy Blueprint

This Elliott Wave analysis script is a sophisticated piece of pattern recognition software, but its current form as an `indicator` makes it a tool for discretionary analysis, not automated execution. To bridge the gap from chart signals to live order fills, we must re-architect it into a robust `strategy` with a professional-grade execution engine.

The core challenge is transforming its "repaint-on-last-bar" logic into a causal, bar-by-bar decision framework that can be realistically backtested and deployed.

### 1. Execution Triggers (Entry & Direction)

The original script identifies several potential signals. We will codify the highest-probability setups—the Wave 3 entry and the Wave C reversal—into precise, non-repainting execution triggers.

*   **Core Principle:** All calculations for pattern validation and signal generation must be completed on the close of the current bar (`barstate.isconfirmed`). The original script's reliance on `barstate.islast` is for display purposes and is unsuitable for a trading strategy. We will execute orders on the open of the *next* bar after a signal is confirmed.

*   **Long Entry Condition (`longCondition`):** A long trade is triggered if *either* of the following is true on the bar's close:
    1.  **Wave 3 Entry:** A potential Wave 2 pullback is confirmed (`detectWave3Entry` is true), the underlying pattern context has a confidence score above a defined threshold (e.g., `confirmedCtx.confidence > 0.50`), and the identified trend is bullish (`not lastP.isHigh`).
    2.  **Wave C Reversal:** A corrective pattern (Zigzag or Flat) is confirmed (`confirmedCtx.confirmedPhase == "corrective"`), the correction was bearish (`not confirmedCtx.isBullish`), and the final pivot (`lastP`) is a confirmed low.

*   **Short Entry Condition (`shortCondition`):** A short trade is triggered if *either* of the following is true on the bar's close:
    1.  **Wave 3 Entry:** A potential Wave 2 bounce is confirmed (`detectWave3Entry` is true), pattern confidence is sufficient (`confirmedCtx.confidence > 0.50`), and the identified trend is bearish (`lastP.isHigh`).
    2.  **Wave C Reversal:** A corrective pattern is confirmed, the correction was bullish (`confirmedCtx.isBullish`), and the final pivot (`lastP`) is a confirmed high.

*   **Signal Reversals:** The strategy will be configured with `strategy(..., pyramiding=0)` to ensure it holds only one position at a time in a given direction. When a `shortCondition` is met while a long trade is active, the `strategy.entry()` command for the short trade will automatically close the existing long position before opening the new short. This provides a built-in mechanism for handling reversals.

### 2. Multi-Tiered Exit Logic

A professional strategy never relies on a single exit condition. We will implement a multi-layered exit framework to manage risk, secure profits, and adapt to market behavior.

*   **Initial Stop Loss (Volatility-Based):** The stop loss is not a generic percentage. It is placed at a structurally significant price level, adjusted for volatility.
    *   **Calculation:** The stop is placed below the pivot that confirms the entry setup. For a Wave 3 long entry, this is the Wave 2 low. For a Wave C reversal long, it's the Wave C low.
    *   **Formula:** `stopLossPrice = pivot_low_price - (ATR[14] * 1.5)`. The 1.5x ATR multiplier provides a buffer against volatility-induced stop-hunts. The corresponding level for shorts is `pivot_high_price + (ATR[14] * 1.5)`.
    *   This stop is calculated *before* the entry order is placed to determine position size.

*   **Take Profit / Trailing Stop (Dynamic & Multi-Stage):**
    1.  **Take Profit 1 (TP1):** The first target is derived from the script's own forecast logic, typically the 1.0x Fibonacci extension of the initial impulse wave (Wave A or Wave 1). Upon reaching TP1, we will **scale out** of a portion of the position (e.g., 50%).
    2.  **Breakeven Stop:** Immediately after TP1 is hit, the stop loss for the remaining position is moved to the entry price. This makes the rest of the trade "risk-free."
    3.  **Trailing Stop (Structural):** For the remaining position, the stop will now trail dynamically. The most logical trailing mechanism for an Elliott Wave strategy is to trail the most recently confirmed structural pivot.
        *   For a long trade, once a Wave 4 low is identified and confirmed, the stop loss is moved to just below that low (`wave_4_low - 0.5 * ATR`). This allows the trade to breathe during the final Wave 5 advance while protecting the majority of the gains.
    4.  **Final Exit Signal:** The original script's `wave5ExitSignal` will be used as a hard exit trigger for any remaining portion of the trade, signifying the likely completion of the entire impulse sequence.

*   **Time-Based Exits:**
    *   **End of Day (EOD):** For strategies running on intraday timeframes (e.g., H1 or lower), an EOD exit rule is critical. A function will trigger `strategy.close_all()` 15 minutes before the session close to avoid overnight risk.
    *   **Stagnation Exit:** If a position has been open for a significant number of bars (e.g., `100` bars) without hitting either its stop loss or its first take-profit target, it will be closed. This "time stop" prevents capital from being tied up in non-performing trades.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical factor in long-term viability. We will move from the default fixed-quantity contracts to a dynamic, risk-based model.

*   **Risk-Based Sizing:** The strategy will risk a fixed percentage of account equity on every single trade, regardless of the trade setup.
    *   **Inputs:** `Account Equity`, `Risk per Trade %` (e.g., 1.5%), `Entry Price`, `Stop Loss Price`.
    *   **Logic:**
        1.  On a signal bar, calculate the `stopLossPrice` as defined above.
        2.  Calculate the `riskDistance` in dollars: `abs(entryPrice - stopLossPrice)`.
        3.  Calculate the `dollarRisk`: `strategy.equity * (risk_per_trade / 100)`.
        4.  Calculate **Position Size**: `positionSize = dollarRisk / riskDistance`.
    *   This calculation ensures that a trade with a wider stop will have a smaller position size, and a trade with a tighter stop will have a larger one, but the maximum potential loss for both is identical.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, the strategy will scale out at predefined profit targets. This is the primary method for realizing profits.
    *   **Pyramiding (Adding to Winners):** While powerful, pyramiding adds significant complexity. For the initial production version, this will be **disabled** (`pyramiding = 0`). A future iteration could include a rule to add a smaller position (e.g., 0.5R) upon the breakout of the Wave 1 high, but only if the initial position is already in profit.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the `indicator` into a `strategy`, focusing on the execution engine. The complex wave detection functions are assumed to be present and are called by this new logic. The `barstate.islast` wrappers around the core logic have been removed.

```pine
// This Pine Script™ code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © Elliott Wave Advanced v3 - Execution Framework

//@version=6
// 1. STRATEGY DECLARATION - Professional Grade
strategy("Elliott Wave Execution Framework", "EW Exec", overlay=true,
     pyramiding=0, // One trade at a time
     initial_capital=100000,
     default_qty_type=strategy.fixed, // We will calculate size manually
     default_qty_value=1, // Placeholder, size is dynamic
     commission_type=strategy.commission.percent,
     commission_value=0.075, // Realistic commission for crypto/futures
     slippage=2) // Realistic slippage in ticks

// ============================================================================
// INPUTS - STRATEGY CONTROLS
// ============================================================================
riskMgmtGroup = "Risk Management"
riskPercent = input.float(1.5, "Risk per Trade (%)", minval=0.1, maxval=5.0, step=0.1, group=riskMgmtGroup)
atrLen = input.int(14, "ATR Length (for Stops)", group=riskMgmtGroup)
atrStopMultiplier = input.float(1.5, "ATR Stop Multiplier", group=riskMgmtGroup)
confidenceThreshold = input.float(0.50, "Min Pattern Confidence", minval=0.35, maxval=1.0, group=riskMgmtGroup)
tp1Extension = input.float(1.0, "Take Profit 1 (Fib Extension)", group=riskMgmtGroup)

// [ ... All original Elliott Wave functions and types (Pivot, ElliottWave, etc.) go here ... ]
// CRITICAL: Remove all `barstate.islast` wrappers from core calculation functions.
// The pivot detection, pattern matching, and context updates must run on every bar.

// ============================================================================
// MAIN EXECUTION LOGIC (runs on every bar)
// ============================================================================

// --- Recalculate pivots and patterns on every bar ---
// This part is now outside of any `barstate.islast` block.
// (The original pivot detection logic is assumed to be here)
// ...
// Reset context for fresh detection on each bar's new info
confirmedCtx.confirmedPattern := "none"
confirmedCtx.confidence := 0.0
WavePattern bestPattern = drawValidatedWaveLabels(primaryPivots, secondaryPivots) // This function now just returns the pattern, not draws it
// ...

// --- Signal Generation ---
var bool longCondition = false
var bool shortCondition = false

if array.size(primaryPivots) >= 3
    Pivot lastP = array.get(primaryPivots, array.size(primaryPivots) - 1)
    bool isW3Setup = detectWave3Entry(primaryPivots)
    bool hasConfidence = confirmedCtx.confidence >= confidenceThreshold

    // Long Conditions
    isW3Long = isW3Setup and hasConfidence and not lastP.isHigh
    isCReversalLong = signalWCReversal and confirmedCtx.confirmedPhase == "corrective" and not confirmedCtx.isBullish and not lastP.isHigh
    longCondition := isW3Long or isCReversalLong

    // Short Conditions
    isW3Short = isW3Setup and hasConfidence and lastP.isHigh
    isCReversalShort = signalWCReversal and confirmedCtx.confirmedPhase == "corrective" and confirmedCtx.isBullish and lastP.isHigh
    shortCondition := isW3Short or isCReversalShort

// --- Execution ---
if strategy.opentrades == 0 // Only look for new entries if flat
    float atrValue = ta.atr(atrLen)
    float stopLossLevel = na
    float takeProfitLevel = na
    float entryPrice = ta.valuewhen(longCondition or shortCondition, close, 0)

    if longCondition
        // Stop Loss is below the Wave 2 or Wave C low
        Pivot setupPivot = array.get(primaryPivots, array.size(primaryPivots) - 1)
        stopLossLevel := setupPivot.price - (atrValue * atrStopMultiplier)
        
        // Take Profit based on Wave 1 extension
        Pivot w1_start = array.get(primaryPivots, array.size(primaryPivots) - 3)
        Pivot w1_end = array.get(primaryPivots, array.size(primaryPivots) - 2)
        float w1_len = w1_end.price - w1_start.price
        takeProfitLevel := setupPivot.price + (w1_len * tp1Extension)

        // Risk-based Position Sizing
        float riskDistance = entryPrice - stopLossLevel
        float dollarRisk = strategy.equity * (riskPercent / 100)
        float positionSize = dollarRisk / riskDistance
        
        strategy.entry("EW_Long", strategy.long, qty=positionSize)
        strategy.exit("Exit_L", "EW_Long", stop=stopLossLevel, limit=takeProfitLevel, qty_percent=50) // Exit 50% at SL or TP1
        strategy.exit("Exit_L_BE", "EW_Long", stop=stopLossLevel) // Manages the other 50%

    if shortCondition
        // Stop Loss is above the Wave 2 or Wave C high
        Pivot setupPivot = array.get(primaryPivots, array.size(primaryPivots) - 1)
        stopLossLevel := setupPivot.price + (atrValue * atrStopMultiplier)

        // Take Profit based on Wave 1 extension
        Pivot w1_start = array.get(primaryPivots, array.size(primaryPivots) - 3)
        Pivot w1_end = array.get(primaryPivots, array.size(primaryPivots) - 2)
        float w1_len = w1_start.price - w1_end.price
        takeProfitLevel := setupPivot.price - (w1_len * tp1Extension)

        // Risk-based Position Sizing
        float riskDistance = stopLossLevel - entryPrice
        float dollarRisk = strategy.equity * (riskPercent / 100)
        float positionSize = dollarRisk / riskDistance

        strategy.entry("EW_Short", strategy.short, qty=positionSize)
        strategy.exit("Exit_S", "EW_Short", stop=stopLossLevel, limit=takeProfitLevel, qty_percent=50)
        strategy.exit("Exit_S_BE", "EW_Short", stop=stopLossLevel)

// --- Dynamic Trade Management ---
// Move stop to breakeven after TP1 is hit
if strategy.opentrades > 0 and strategy.opentrades[1] > strategy.opentrades
    if strategy.position_avg_price != 0
        // A partial exit just occurred, move stop for remaining position to breakeven
        if strategy.position_is_long
            strategy.exit("Exit_L_BE", "EW_Long", stop=strategy.position_avg_price)
        if strategy.position_is_short
            strategy.exit("Exit_S_BE", "EW_Short", stop=strategy.position_avg_price)

// Final Wave 5 Exit Signal
bool wave5ExitSignal = signalW5Exit and confirmedCtx.confirmedPattern == "impulse" // Simplified from original
if wave5ExitSignal
    strategy.close_all(comment="Wave 5 Exit")

// --- Drawing and UI (can remain in barstate.islast for performance) ---
if barstate.islast
    // All original drawing functions (drawWavePattern, drawForecastProjection, infoPanel, etc.) go here
    // They will use the `bestPattern` and `confirmedCtx` calculated on the final bar.
```
    