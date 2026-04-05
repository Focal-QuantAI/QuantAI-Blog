
# Indicators to Strategy Blueprint

The provided script is an excellent example of a visual "Smart Money Concepts" (SMC) toolkit. It identifies key price action zones like Order Blocks (OB), Fair Value Gaps (FVG), and Liquidity but provides no execution logic. To transform this `indicator` into a production-grade `strategy`, we must define a clear, non-discretionary ruleset based on these concepts.

The proposed strategy will be based on a core SMC principle: **enter on a pullback to a high-probability Point of Interest (POI) after a confirmed Break of Structure (BoS).**

---

### 1. Execution Triggers (Entry & Direction)

The visual signals will be translated into precise boolean conditions for automated execution. The strategy will wait for a market structure shift and then seek a limit entry on the subsequent retracement.

*   **Directional Bias:** The script's `lastState` variable, which tracks Breaks of Structure ('BoS') and Changes of Character ('CHoCH'), will serve as our primary directional filter. A `lastState` of 'up' means we are in a bullish market structure and will only look for Longs. A `lastState` of 'down' implies a bearish structure, and we will only look for Shorts.

*   **Long Entry Condition:**
    1.  **Confirmation:** A bullish Break of Structure must have occurred. We can capture this when `lastState` becomes 'up'.
    2.  **Point of Interest (POI) Identification:** After the BoS, the system will identify the most recent, unmitigated **Bullish Order Block** or **Bullish Fair Value Gap** created during the structural leg-up.
    3.  **Entry Trigger:** A `strategy.entry()` order with a `limit` price will be placed at the POI. The precise trigger is when the real-time `low` of a developing bar touches the upper boundary of the identified POI (e.g., `box.get_top(bullishOrderblock.block)` or `fvg.top`).

*   **Short Entry Condition:**
    1.  **Confirmation:** A bearish Break of Structure must have occurred (`lastState` becomes 'down').
    2.  **Point of Interest (POI) Identification:** The system identifies the most recent, unmitigated **Bearish Order Block** or **Bearish Fair Value Gap**.
    3.  **Entry Trigger:** A `strategy.entry()` order with a `limit` price is placed. The trigger is when the real-time `high` of a developing bar touches the lower boundary of the bearish POI (e.g., `box.get_bottom(bearishOrderblock.block)` or `fvg.bottom`).

*   **Execution Nuances:**
    *   **Execution Model:** This is a pullback strategy, making it ideal for **Limit Orders**. We are not chasing price; we are letting the price come to our pre-defined level. Therefore, execution must be based on "On Real-time Price Action" (using `calc_on_every_tick=true` in the `strategy` call is necessary for limit order fills intra-bar). We will place a limit order and wait for a fill. If a new, opposing BoS occurs before our order is filled, the entry order is canceled.
    *   **Signal Reversals:** A core rule of this framework is invalidation. If we are in a Long position and a bearish BoS occurs (`lastState` flips to 'down'), our bullish thesis is invalidated. The system will immediately close the long position using `strategy.close()`. It will then begin hunting for a new Short entry based on the new market structure.

### 2. Multi-Tiered Exit Logic

A professional exit strategy is not a single point but a dynamic process. We will combine volatility-based stops, multi-stage profit targets, and time-based rules.

*   **Initial Stop Loss:**
    *   The stop loss will be placed based on the price structure that validates the trade, not an arbitrary percentage.
    *   **For Longs:** The stop loss will be placed just below the swing low that preceded the Break of Structure, which is the low of the candle that forms the base of the Bullish Order Block. To avoid stop-hunting, we will place it at `bullishOrderblock.value - (atr * 0.5)`.
    *   **For Shorts:** The stop loss will be placed just above the swing high that preceded the bearish BoS, at `bearishOrderblock.value + (atr * 0.5)`.

*   **Take Profit/Trailing:**
    *   We will employ a two-stage profit-taking mechanism to secure gains while allowing for runners.
    *   **TP1 (Scaling Out):** The first take-profit target will be set at a fixed Risk-to-Reward ratio, for example, **2R**. If the risk on the trade is 100 points, TP1 will be 200 points from the entry. We will exit 50% of the position here.
    *   **TP2 (Trailing Stop):** After TP1 is hit, the stop loss for the remaining 50% of the position is moved to **Breakeven**. Simultaneously, a trailing stop is activated. A robust trailing mechanism would be to trail the stop below the most recent ZigZag low (`array.last(lowVal)`) for a long, or above the most recent ZigZag high (`array.last(highVal)`) for a short. This allows the trade to follow the market structure until it reverses.

*   **Time-Based Exits:**
    *   **End of Session:** For day trading strategies, an "End of Day" exit is critical. A rule will be implemented to close all open positions flat 30 minutes before the market close (e.g., at 15:30 EST for US Equities) to avoid overnight/weekend gap risk.
    *   **Stagnation Exit:** If a trade has been open for a predefined number of bars (e.g., `50` bars) and has not hit either the Stop Loss or TP1, it is considered "stagnant." The position will be closed at market to free up capital for higher-probability opportunities.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term viability. We will use a dynamic, risk-based model.

*   **Risk-Based Sizing:**
    *   The system will risk a fixed percentage of the total account equity on every single trade, for instance, **1%**.
    *   The position size is calculated *before* the entry order is placed, based on the distance between the entry price and the initial stop loss.
    *   **Formula:** `Position Size = (Account Equity * Risk Percentage) / |Entry Price - Stop Loss Price|`
    *   **Example:**
        *   Account Equity: $100,000
        *   Risk per Trade: 1% ($1,000)
        *   Entry Price (Limit): $500
        *   Stop Loss Price: $490
        *   Risk per Share: $10
        *   **Position Size:** $1,000 / $10 = **100 shares**
    *   This ensures that a losing trade always results in a predictable, fixed-percentage loss of capital.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** This is handled by our multi-tiered exit logic (TP1 and the trailing portion).
    *   **Pyramiding (Scaling In):** We will allow for one additional entry (pyramiding level 1). If the initial trade is in profit and a *new* Break of Structure occurs in the same direction, the system is permitted to take a new entry on the subsequent pullback to a new POI. The risk for this second position will also be 1% of the *current* account equity, and its SL/TP will be managed independently. The total number of open positions in one direction will be capped at 2 (`pyramiding = 2`).

### 4. Implementation Snippet (Pine Logic)

This pseudocode demonstrates how to wrap the indicator's logic within a `strategy` framework, focusing on the execution engine.

```pine
// @version=5
// 1. STRATEGY DECLARATION - Focus on Execution Reality
strategy(
     "SMC Execution Engine [UAlgo]", 
     overlay=true, 
     pyramiding=2, // Allow one scale-in entry
     initial_capital=100000,
     commission_type=strategy.commission.cash_per_order,
     commission_value=1.0, // $1 per order (entry/exit)
     slippage=2, // 2 ticks of slippage on market/stop orders
     calc_on_every_tick=true // Required for limit order fills
)

// --- INPUTS FOR STRATEGY ---
riskPercent = input.float(1.0, "Risk per Trade %", minval=0.1, maxval=5.0)
rr1 = input.float(2.0, "Take Profit 1 R:R")
stagnationBars = input.int(50, "Stagnation Exit (Bars)")
endOfSessionTime = input.string("1530", "End of Session (HHMM)")

// --- [ORIGINAL SCRIPT LOGIC FOR IDENTIFYING OBs, FVGs, BoS GOES HERE] ---
// We assume the arrays `bullishOrderblock`, `bearishOrderblock`, `lastState`, etc., are populated by the original script's logic.

// --- RISK MANAGEMENT & SIZING ---
riskEquity = (riskPercent / 100) * strategy.equity

// --- EXECUTION LOGIC ---
// Only process logic if we are not in a trade, to find a new entry
if strategy.position_size == 0
    // LONG ENTRY LOGIC
    if lastState == 'up' and array.size(bullishOrderblock) > 0
        // Get the most recent, unmitigated Order Block
        latestBullishOB = array.get(bullishOrderblock, array.size(bullishOrderblock) - 1)
        
        // Define entry, stop, and target
        entryPrice = box.get_top(latestBullishOB.block)
        stopPrice = latestBullishOB.value - (ta.atr(14) * 0.5) // Structural low + ATR buffer
        
        // Ensure there's a valid risk/reward
        if entryPrice > stopPrice
            riskPerUnit = entryPrice - stopPrice
            positionSize = riskEquity / riskPerUnit
            
            takeProfit1 = entryPrice + (riskPerUnit * rr1)
            
            // Place the limit order for the full size
            strategy.entry("Long Entry", strategy.long, qty=positionSize, limit=entryPrice)
            
            // Place the exit orders for TP1 (50%) and the final Stop Loss (100%)
            strategy.exit("Long TP1/SL", from_entry="Long Entry", qty_percent=50, profit=takeProfit1)
            strategy.exit("Long Final SL", from_entry="Long Entry", loss=stopPrice)

    // SHORT ENTRY LOGIC (mirror of long)
    if lastState == 'down' and array.size(bearishOrderblock) > 0
        latestBearishOB = array.get(bearishOrderblock, array.size(bearishOrderblock) - 1)
        entryPrice = box.get_bottom(latestBearishOB.block)
        stopPrice = latestBearishOB.value + (ta.atr(14) * 0.5)
        
        if entryPrice < stopPrice
            riskPerUnit = stopPrice - entryPrice
            positionSize = riskEquity / riskPerUnit
            takeProfit1 = entryPrice - (riskPerUnit * rr1)
            
            strategy.entry("Short Entry", strategy.short, qty=positionSize, limit=entryPrice)
            strategy.exit("Short TP1/SL", from_entry="Short Entry", qty_percent=50, profit=takeProfit1)
            strategy.exit("Short Final SL", from_entry="Short Entry", loss=stopPrice)

// --- IN-TRADE MANAGEMENT ---
// Move to Breakeven and activate trailing stop after TP1 is hit
// Note: Pine Script's native trailing is basic. A more robust implementation would track ZigZag pivots.
isInLong = strategy.position_size > 0
isInShort = strategy.position_size < 0
isHalfClosed = strategy.position_size == positionSize / 2 // Simplified check

if isHalfClosed and isInLong
    strategy.exit("Long BE/Trail", from_entry="Long Entry", loss=strategy.position_avg_price, trail_points=ta.atr(14) * 3)

// --- OVERRIDING EXITS (Reversals & Time) ---
// Structural Invalidation Exit
if (isInLong and lastState == 'down') or (isInShort and lastState == 'up')
    strategy.close_all(comment="Structural Invalidation")

// Stagnation Exit
if (time - strategy.opentrades.entry_time(0)) / 1000 > barmerge.lookahead_on * stagnationBars
    strategy.close_all(comment="Stagnation Exit")

// End of Session Exit
isEOD = (time("D") != time("D")[1]) and (hour(time, "America/New_York") * 100 + minute(time, "America/New_York") >= str.tonumber(endOfSessionTime))
if isEOD
    strategy.close_all(comment="End of Session")

```
    