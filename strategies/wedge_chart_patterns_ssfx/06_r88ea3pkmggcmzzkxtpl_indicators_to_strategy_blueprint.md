
# Indicators to Strategy Blueprint

The provided script is an excellent starting point for visual pattern recognition. However, to transition it from a chart `indicator` to a production-ready `strategy`, we must replace its visual alerts with a robust execution engine that manages entries, exits, and capital with precision.

The core logic—identifying wedge patterns via pivot points—will be preserved. Our focus is on building the automated trading machinery around it.

### 1. Execution Triggers (Entry & Direction)

The entry signal is generated when the price closes outside a confirmed wedge pattern with a candle body that meets a minimum size requirement. This is a "close-of-bar" signal, which is crucial for execution reality.

*   **Execution Timing:** The signal is confirmed *after* the breakout bar closes. Therefore, the entry order should be placed to execute at the **open of the next bar**. Attempting to enter intra-bar on a `close`-based signal introduces significant repaint risk and backtest/live divergence.

*   **Long Entry Condition:**
    ```
    (isFallingWedgeBreakout) AND (breakoutCandleBodyPercent >= MIN_BODY_PCT) AND (strategy.opentrades == 0)
    ```
    *   `isFallingWedgeBreakout`: `close` of the current bar is greater than the price of the upper trendline of a confirmed falling wedge.
    *   `strategy.opentrades == 0`: Ensures we are not already in a position.

*   **Short Entry Condition:**
    ```
    (isRisingWedgeBreakout) AND (breakoutCandleBodyPercent >= MIN_BODY_PCT) AND (strategy.opentrades == 0)
    ```
    *   `isRisingWedgeBreakout`: `close` of the current bar is less than the price of the lower trendline of a confirmed rising wedge.
    *   `strategy.opentrades == 0`: Ensures we are not already in a position.

*   **Signal Reversals:**
    For a robust system, a new, valid signal in the opposite direction should trigger an immediate exit of the current position and an entry into the new one. This is handled by first issuing a `strategy.close()` command for the existing trade ID, followed by a `strategy.entry()` for the new trade. This ensures the portfolio is always aligned with the most recent high-conviction signal. If pyramiding is disabled, a simple `strategy.entry()` call will automatically reverse the position.

### 2. Multi-Tiered Exit Logic

A static take-profit and stop-loss is insufficient for dynamic markets. A professional framework requires a multi-layered approach.

*   **Initial Stop Loss (Volatility-Based):**
    The script's "Previous Swing" option is a solid, structurally-based choice. We will enhance this by adding an ATR-based alternative, which is a market standard.
    *   **Logic:** The stop loss will be placed at the more conservative (further away) of either the previous swing high/low or `N` times the Average True Range (ATR) from the entry price.
    *   **Example (Long):** `stopLossPrice = min(recentSwingLow, entryPrice - (atr(14) * 2.0))`
    *   This prevents excessively tight stops on low-volatility breakouts while respecting market structure.

*   **Take Profit / Trailing Mechanism:**
    A single R:R target leaves profit on the table during strong trends. We will implement a two-stage exit strategy.
    1.  **Initial Target (TP1):** Set at a 1.5R multiple (`riskDistance * 1.5`). When this target is hit, **50% of the position is closed**.
    2.  **Breakeven & Trail (TP2):** Simultaneously, the stop loss for the remaining 50% of the position is moved to the entry price (breakeven).
    3.  **Trailing Stop Activation:** From this point, a trailing stop is activated. A Chandelier Exit is an excellent choice, trailing `X` * ATR from the highest high (for longs) or lowest low (for shorts) since the trade was entered. This allows the winning portion of the trade to run while protecting profits.

*   **Time-Based Exits:**
    Capital should not be tied up in stagnant trades.
    *   **End of Day (EOD):** For intraday strategies, all open positions must be squared off 5-10 minutes before the session close to avoid overnight risk. This is a non-negotiable rule for most day-trading systems.
    *   **Stagnation Exit:** If a trade has been open for `X` bars (e.g., 50) and has not yet reached the first take-profit target (TP1), it is closed. This frees up capital for higher-probability opportunities.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component of a professional strategy. We will move from a fixed contract size to a dynamic risk-based model.

*   **Risk-Based Sizing:**
    The strategy will risk a fixed percentage of account equity on every trade.
    *   **Rule:** The position size is calculated to ensure that if the initial stop loss is hit, the total loss will equal a predefined percentage (e.g., 1%) of the current `strategy.equity`.
    *   **Formula:**
        ```
        riskPerTradeUSD = strategy.equity * (riskPercent / 100)
        riskDistancePerShare = abs(entryPrice - stopLossPrice)
        positionSize = riskPerTradeUSD / riskDistancePerShare
        ```
    *   This automatically adjusts the size of each position based on the trade's specific volatility (the distance to the stop loss), ensuring consistent risk exposure.

*   **Pyramiding & Scaling:**
    *   **Scaling In (Pyramiding):** Adding to a winning position is an advanced technique. A safe rule would be: A second unit, equal to 50% of the initial size, can be added if the price moves 1R in our favor. The stop loss for the *entire* position (both units) must then be moved to the entry price of the *initial* trade to ensure the trade cannot become a loser. The `pyramiding` parameter in the `strategy` call must be set to a value greater than 1.
    *   **Scaling Out:** As defined in the exit logic, we will scale out by closing 50% of the position at TP1. This is managed via `strategy.exit` calls with a specified `qty_percent`.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the transformation from an `indicator` to a `strategy`, incorporating the professional execution logic. It replaces the visual plotting with order management commands.

```pine
// This work is licensed under Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International
// https://creativecommons.org/licenses/by-nc-sa/4.0/
// © SimplySafeFx QuantTrading

//@version=5
// 1. STRATEGY DECLARATION with realistic friction
strategy("Wedge Pattern Execution [PRO]",
     overlay=true,
     initial_capital=25000,
     commission_type=strategy.commission.percent,
     commission_value=0.04, // Example commission for futures/crypto
     slippage=2, // 2 ticks of slippage on market orders
     pyramiding=0, // Set to 0 to enforce one position at a time
     default_qty_type=strategy.fixed, // We will calculate size manually
     process_orders_on_close=true) // Ensures execution on next bar's open

// --- INPUTS ---
// [Keep original pattern detection inputs]
// ... (INPUT_PIVOT_LEFT, INPUT_PIVOT_RIGHT, etc.)

// New Risk Management Inputs
grp_risk = "Capital & Risk Management"
float RISK_PERCENT        = input.float(1.0, "Risk per Trade %", minval=0.1, maxval=5.0, step=0.1, group=grp_risk)
float ATR_SL_MULT         = input.float(2.0, "ATR Multiplier for Stop", minval=1.0, group=grp_risk)
float TP1_RR              = input.float(1.5, "TP1 Risk/Reward Ratio", minval=1.0, group=grp_risk)
int   STAGNATION_BARS     = input.int(75, "Max Bars in Trade (Stagnation)", minval=10, group=grp_risk)

// [Keep original pattern detection logic, types, and methods]
// ... (type Coordinate, type Wedge, pivot collection, etc.)

// --- EXECUTION LOGIC ---
// This block replaces the original "Trade logic" and "Visual signals" sections

// Signal evaluation (remains the same)
[risingSig, risingBody] = process_wedge(active_rising)
[fallingSig, fallingBody] = process_wedge(active_falling)

int signalDir = risingSig != 0 ? risingSig : fallingSig
bool longSignal  = signalDir == 1
bool shortSignal = signalDir == -1

// Calculate ATR for dynamic stops
float currentAtr = ta.atr(14)

// --- STRATEGY EXECUTION ---
if (strategy.opentrades == 0) // Only look for new entries if flat
    float entryPrice   = open // Entry is on the open of the bar AFTER the signal
    float slPrice      = na
    float tp1Price     = na
    float riskDistance = na

    if (longSignal[1]) // Signal occurred on the previous bar
        // Calculate Stop Loss based on structure and volatility
        recentSwingLow = ta.valuewhen(pl[1], low[1], 0)
        structuralSL = recentSwingLow
        volatilitySL = entryPrice - (currentAtr * ATR_SL_MULT)
        slPrice := math.min(structuralSL, volatilitySL) // Use the wider of the two stops

        // Calculate Risk and Position Size
        riskDistance := entryPrice - slPrice
        if (riskDistance > 0)
            tp1Price := entryPrice + (riskDistance * TP1_RR)
            riskPerTrade = (strategy.equity * RISK_PERCENT) / 100
            positionSize = riskPerTrade / riskDistance

            // Place Orders
            strategy.entry("Wedge L", strategy.long, qty=positionSize)
            // Set a multi-stage exit: 50% at TP1, remainder trails (managed below)
            strategy.exit("L TP1/SL", from_entry="Wedge L", qty_percent=50, profit=(tp1Price - entryPrice)/syminfo.mintick, loss=(entryPrice - slPrice)/syminfo.mintick)
            // This second exit order manages the remaining 50% with only a stop loss. The trailing logic will handle it.
            strategy.exit("L SL2", from_entry="Wedge L", loss=(entryPrice - slPrice)/syminfo.mintick)

    if (shortSignal[1]) // Signal occurred on the previous bar
        // Calculate Stop Loss
        recentSwingHigh = ta.valuewhen(ph[1], high[1], 0)
        structuralSL = recentSwingHigh
        volatilitySL = entryPrice + (currentAtr * ATR_SL_MULT)
        slPrice := math.max(structuralSL, volatilitySL)

        // Calculate Risk and Position Size
        riskDistance := slPrice - entryPrice
        if (riskDistance > 0)
            tp1Price := entryPrice - (riskDistance * TP1_RR)
            riskPerTrade = (strategy.equity * RISK_PERCENT) / 100
            positionSize = riskPerTrade / riskDistance

            // Place Orders
            strategy.entry("Wedge S", strategy.short, qty=positionSize)
            strategy.exit("S TP1/SL", from_entry="Wedge S", qty_percent=50, profit=(entryPrice - tp1Price)/syminfo.mintick, loss=(slPrice - entryPrice)/syminfo.mintick)
            strategy.exit("S SL2", from_entry="Wedge S", loss=(slPrice - entryPrice)/syminfo.mintick)

// --- POSITION MANAGEMENT ---
// Breakeven and Trailing Stop Logic
if (strategy.opentrades > 0)
    // Stagnation Exit
    if (bar_index - strategy.opentrades.entry_bar_index(0) > STAGNATION_BARS)
        strategy.close_all(comment="Stagnation Exit")

    // Check if TP1 was hit and move stop to Breakeven for the remaining position
    // Note: Pine Script's ability to dynamically check partial fills is limited.
    // A common proxy is to check if price has crossed the TP1 level.
    isLongTrade = strategy.position_size > 0
    tp1_price_level = isLongTrade ? strategy.opentrades.entry_price(0) + (strategy.opentrades.entry_price(0) - strategy.opentrades.stop_loss(0)) * TP1_RR : na
    
    if isLongTrade and high > tp1_price_level and strategy.opentrades.stop_loss(0) < strategy.opentrades.entry_price(0)
        // Move the stop loss of the second exit order ("L SL2") to breakeven
        strategy.exit("L SL2", from_entry="Wedge L", loss=(strategy.opentrades.entry_price(0) - strategy.opentrades.entry_price(0))/syminfo.mintick)
        // From here, a more complex trailing stop could be implemented by modifying this exit order on each bar.
```
    