
# Indicators to Strategy Blueprint

Based on the "Dynamic Swing VWAP" indicator, here is the architectural blueprint for a production-ready, automated execution framework. The core logic of the indicator, which identifies trend direction by anchoring a VWAP to significant swing points, provides a robust foundation for a trend-following or mean-reversion strategy.

### 1. Execution Triggers (Entry & Direction)

The original script's alert conditions provide the ideal, non-repainting trigger for our strategy. The signal is generated on the confirmation of a new swing pivot that re-anchors the VWAP.

*   **Core Logic:** An entry signal is valid only when the market structure shifts, confirmed by a new swing high or low that meets the `onlyLLHHInput` criteria (if enabled). This ensures we are trading based on significant structural changes, not minor fluctuations.

*   **Precise Boolean Conditions:**
    *   **Long Entry (`goLong`):** A trade is initiated when the VWAP anchor flips from a swing high to a swing low, signaling a potential bottom and the start of a bullish trend.
        ```pinescript
        // A confirmed swing pivot has just occurred
        bool dirFlipped = dir != dir[1]
        // The pivot qualifies as an anchor point (respecting the LL/HH filter)
        bool isAnchorPoint = dirFlipped and (not onlyLLHHInput or lastSwingStr == "LL" or lastSwingStr == "HH" or lastSwingStr == "IL" or lastSwingStr == "IH")
        // The new anchor is a swing low, indicating a bullish direction
        bool goLong = isAnchorPoint and dir == 1
        ```
    *   **Short Entry (`goShort`):** A trade is initiated when the VWAP anchor flips from a swing low to a swing high, signaling a potential top and the start of a bearish trend.
        ```pinescript
        // Re-use the same boolean logic from the Long condition
        bool goShort = isAnchorPoint and dir == -1
        ```

*   **Execution Nuance:**
    *   **Execute at "Close" of the Bar:** The swing detection logic (`dir != dir[1]`) relies on comparing the current bar's state to the previous one. The signal is only confirmed and stable *after the bar closes*. Therefore, all entry orders must be placed to execute on the open of the next bar, or using `strategy.close()` which simulates execution at the current bar's close. Attempting real-time execution would lead to false signals and repainting issues.

*   **Signal Reversals:**
    *   The system is inherently a reversal strategy. When a `goLong` signal occurs while a short position is active, the framework must first close the existing short position and then open the new long. This is handled by explicitly calling `strategy.close()` before `strategy.entry()`. This provides deterministic control over the position state, preventing the engine from mismanaging a "flip."

### 2. Multi-Tiered Exit Logic

A static stop-loss or take-profit is insufficient for a dynamic system. The exit logic must adapt to market volatility and trade progression.

*   **Initial Stop Loss (Volatility-Based):**
    *   **Calculation:** The stop loss will be placed at a multiple of the Average True Range (ATR) from the entry price. This ensures the stop is wider in volatile markets and tighter in calm ones. The script already calculates `atr`, which we will leverage.
    *   **Logic:**
        *   For a **Long** position: `StopLossPrice = entry_price - (atr_at_entry * atrMultiplierSL)`
        *   For a **Short** position: `StopLossPrice = entry_price + (atr_at_entry * atrMultiplierSL)`
    *   `atrMultiplierSL` should be a user-defined input (e.g., default `2.0`).

*   **Take Profit/Trailing (Dynamic & Context-Aware):**
    *   **Option A: ATR Trailing Stop:** Once the trade is profitable by a certain amount (e.g., 1x the initial risk), a trailing stop is activated. The stop will trail the price at a distance of `atr_at_entry * atrMultiplierTrail`. This allows the trade to capture the majority of a strong trend.
    *   **Option B: VWAP Band Exit (Recommended):** The indicator's built-in deviation bands are the perfect, context-aware take-profit targets.
        *   **Long Exit:** Close the position when `high` crosses above the upper VWAP band. This signifies that the price has reached a point of potential overextension relative to its dynamic mean.
        *   **Short Exit:** Close the position when `low` crosses below the lower VWAP band.
    *   **Multi-Stage Take Profit:** For advanced systems, combine approaches. Exit 50% of the position at the VWAP band touch and move the stop loss on the remaining 50% to breakeven. Let the remainder run with an ATR trailing stop to capture outsized moves.

*   **Time-Based Exits:**
    *   **End of Day (EOD) Squaring:** For intraday strategies, it's critical to avoid overnight risk. A hard exit rule should be implemented to close any open position a few minutes before the session close.
        *   **Logic:** `strategy.close("long_id", when = time_close("D"))`
    *   **Stagnation Exit:** A trade that goes nowhere ties up capital. If a position has been open for `X` bars (e.g., `100`) and has not achieved a profit of at least `0.5R` (half the initial risk), it should be closed. This frees up capital for more promising opportunities.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term viability. We will move from fixed-lot sizes to a dynamic risk-based model.

*   **Risk-Based Sizing:**
    *   **Rule:** The strategy will risk a fixed percentage of the total account equity on every single trade. A typical value is 1% (`riskPercent = 1.0`).
    *   **Mathematical Logic:**
        1.  Define `riskPercent` (e.g., `1.0` for 1%).
        2.  Calculate `equityRisk = (strategy.equity * riskPercent) / 100`.
        3.  Calculate the stop-loss distance in price points: `stopDistance = abs(entry_price - stop_loss_price)`.
        4.  Calculate the position size: `positionSize = equityRisk / stopDistance`.
    *   This calculation must be performed *before* each `strategy.entry` call to ensure every trade is sized correctly based on its specific stop-loss distance and the current account equity.

*   **Pyramiding & Scaling:**
    *   **Pyramiding (Scaling In):** The indicator's `lastSwingStr` provides perfect opportunities to add to a winning position.
        *   **Rule:** If a **Long** position is open and profitable, and the indicator prints a "HL" (Higher Low) swing, a second, smaller position (e.g., 50% of the initial size) can be added. This new entry gets its own ATR-based stop loss.
        *   **Rule:** If a **Short** position is open and profitable, and the indicator prints a "LH" (Lower High) swing, a second position can be added.
        *   This method adds exposure only when the market structure confirms the existing trend.
    *   **Scaling Out:** As defined in the Multi-Tiered Exit Logic, scaling out at predefined targets (like the VWAP bands) is a core part of the risk management strategy. It reduces risk by realizing partial profits while allowing a portion of the trade to continue.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the transformation from an `indicator` to a `strategy`, incorporating the professional execution logic.

```pine
//@version=5
// ══════════════════════════════════════════════════════════
// DYNAMIC SWING VWAP [EXECUTION FRAMEWORK]
// ══════════════════════════════════════════════════════════
strategy(
    title               = "DS-VWAP Execution",
    shorttitle          = "DS-VWAP Strat",
    overlay             = true,
    pyramiding          = 0, // We will control pyramiding manually
    initial_capital     = 100000,
    default_qty_type    = strategy.fixed,
    commission_type     = strategy.commission.percent,
    commission_value    = 0.075, // Realistic broker commission
    slippage            = 2 // Realistic slippage in ticks
)

// ... [Paste the entire original indicator code here, from GRP_MAIN to the end of core logic] ...
// We need all the indicator's calculations like 'dir', 'wasAnchor', 'currentVwap', 'atr', etc.

// ══════════════════════════════════════
// STRATEGY CONFIGURATION
// ══════════════════════════════════════
GRP_STRAT = "📈 Strategy Settings"
riskPercent     = input.float(1.0, "Risk per Trade %", minval=0.1, maxval=5.0, step=0.1, group=GRP_STRAT)
atrMultiplierSL = input.float(2.0, "ATR Stop Loss Multiplier", minval=0.5, step=0.1, group=GRP_STRAT)
useBandExit     = input.bool(true, "Use VWAP Bands for Take Profit", group=GRP_STRAT)
stagnationBars  = input.int(100, "Exit After X Bars of Stagnation", minval=10, group=GRP_STRAT)
useEodExit      = input.bool(true, "Enable End-of-Day Exit", group=GRP_STRAT)

// ══════════════════════════════════════
// EXECUTION LOGIC
// ══════════════════════════════════════

// 1. --- ENTRY TRIGGERS ---
bool dirFlipped    = dir != dir[1]
bool isAnchorPoint = dirFlipped and (not onlyLLHHInput or lastSwingStr == "LL" or lastSwingStr == "HH" or lastSwingStr == "IL" or lastSwingStr == "IH")
bool goLong        = isAnchorPoint and dir == 1
bool goShort       = isAnchorPoint and dir == -1

// 2. --- POSITION SIZING & RISK MANAGEMENT ---
float atrValue = ta.atr(14) // Use a standard 14-period ATR for stop calculation
float stopDistance = atrValue * atrMultiplierSL
float riskAmount = (strategy.equity * riskPercent) / 100
float positionSize = riskAmount / (stopDistance * syminfo.pointvalue)

// 3. --- EXIT CONDITIONS ---
// VWAP Band Exit Logic
float upperBand = currentVwap + (ewmaAtr * bandMultInput)
float lowerBand = currentVwap - (ewmaAtr * bandMultInput)
bool exitLongSignal = useBandExit and high > upperBand
bool exitShortSignal = useBandExit and low < lowerBand

// Time-based Exits
bool eodExitSignal = useEodExit and (time_close("D") - time) < (60000 * 5) // Exit 5 mins before session close
bool stagnationExitSignal = barssince(strategy.opentrades > 0) > stagnationBars

// 4. --- ORDER EXECUTION ---
if (goLong)
    // First, close any existing short position to handle reversals
    if strategy.position_size < 0
        strategy.close("Short", comment="Reversal to Long")
    
    // Then, enter the new long position with SL and TP
    strategy.entry("Long", strategy.long, qty=positionSize)
    // Set the initial Stop Loss
    strategy.exit("Long SL/TP", from_entry="Long", loss=stopDistance * syminfo.pointvalue)

if (goShort)
    // First, close any existing long position
    if strategy.position_size > 0
        strategy.close("Long", comment="Reversal to Short")

    // Then, enter the new short position
    strategy.entry("Short", strategy.short, qty=positionSize)
    // Set the initial Stop Loss
    strategy.exit("Short SL/TP", from_entry="Short", loss=stopDistance * syminfo.pointvalue)

// --- HANDLE DYNAMIC & TIME-BASED EXITS ---
if strategy.position_size > 0 and (exitLongSignal or eodExitSignal or stagnationExitSignal)
    strategy.close("Long", comment=exitLongSignal ? "TP Band" : eodExitSignal ? "EOD" : "Stagnation")

if strategy.position_size < 0 and (exitShortSignal or eodExitSignal or stagnationExitSignal)
    strategy.close("Short", comment=exitShortSignal ? "TP Band" : eodExitSignal ? "EOD" : "Stagnation")

```
    