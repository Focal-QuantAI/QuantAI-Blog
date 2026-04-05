
# Indicators to Strategy Blueprint

The provided "QuantFlow: Precision Momentum Oscillator" is an excellent foundation, offering nuanced signals like predictive sparks and divergences. However, as an `indicator`, it only provides visual cues. To transform this into a production-ready automated execution framework, we must build a robust engine around its core logic, focusing on order management, risk, and real-world market friction.

### 1. Execution Triggers (Entry & Direction)

The raw oscillator provides several potential signals. For a robust system, we will focus on the most definitive one—the zero-line cross—as our primary entry trigger. The "sparks" and "divergences" can be used as filters or for more aggressive, alternative strategies, but the zero-cross represents the clearest shift in momentum control.

*   **Long Entry Condition:** A long position is initiated when the `momOsc` crosses *above* the zero line. This signifies a confirmed shift from bearish to bullish momentum control.
    ```pinescript
    // Primary Bullish Entry Trigger
    bool longCondition = ta.crossover(momOsc, 0)
    ```
*   **Short Entry Condition:** A short position is initiated when the `momOsc` crosses *below* the zero line. This signifies a confirmed shift from bullish to bearish momentum control.
    ```pinescript
    // Primary Bearish Entry Trigger
    bool shortCondition = ta.crossunder(momOsc, 0)
    ```

*   **Execution Nuance:** These conditions are evaluated at the **close of the bar**. A `ta.crossover` is only true on the historical bar where the event occurred. Therefore, to avoid repainting and ensure realistic execution, orders must be placed to be filled on the **open of the next bar**. The strategy will execute `strategy.entry()` on the bar the signal occurs, with the order being filled at the market open of the subsequent bar.

*   **Signal Reversals:** The framework must handle an open position when an opposing signal occurs. If the system is in a long position and the `shortCondition` becomes true, the strategy will automatically close the existing long position and initiate a new short position. This "stop and reverse" (SAR) behavior is native to Pine Script's `strategy.entry()` function and ensures the system is always aligned with the latest primary signal.

### 2. Multi-Tiered Exit Logic

A professional strategy never relies on a single exit condition. We will implement a multi-faceted exit system to manage risk, protect profits, and prevent capital from being tied up in stagnant trades.

*   **Initial Stop Loss (Volatility-Based):** We will discard arbitrary percentage-based stops in favor of a stop loss calculated from market volatility using the Average True Range (ATR). This adapts the risk distance to the asset's current behavior.
    *   **Logic:** For a long entry, the stop loss is placed at `Entry Price - (ATR Multiplier * ATR)`. For a short, it's `Entry Price + (ATR Multiplier * ATR)`. A typical multiplier is between 1.5 and 3.
    *   **Implementation:** We will calculate the 14-period ATR and use it to define the stop loss distance in points/pips from the entry price.

*   **Take Profit / Trailing Mechanism (Multi-Stage):** We will implement a two-stage take-profit system to lock in gains while allowing for further upside.
    *   **Stage 1 (Initial Target):** A fixed Risk-to-Reward (RR) target. For example, at a 1.5R profit level (`Entry Price + 1.5 * Stop Loss Distance`), we will close 50% of the position.
    *   **Stage 2 (Trailing Stop):** After Stage 1 is hit, the stop loss for the remaining 50% of the position is moved to breakeven. From this point, a trailing stop is activated. A common method is an ATR-based trailing stop that follows the price at a distance of `N * ATR` from the highest high (for longs) or lowest low (for shorts) since the trade was initiated. For simplicity in this framework, we will let the remaining portion run until an opposite entry signal occurs or a time-based exit is triggered.

*   **Time-Based Exits:**
    *   **End of Day (EOD):** For intraday strategies (e.g., on timeframes H1 or lower), it is critical to have a hard exit rule to square all positions before the market close. This avoids unpredictable overnight gaps and funding fees. We will implement a rule to close any open position in the last hour of the trading session.
    *   **Trade Stagnation:** If a trade has been open for a specified number of bars (e.g., 50 bars) and has not hit either its stop loss or its initial take-profit target, it is considered "dead money." The strategy will exit the position to free up capital for more promising opportunities.

### 3. Capital Allocation & Risk Management

Position sizing is arguably the most critical component of a successful algorithmic strategy. We will implement a dynamic, risk-based sizing model.

*   **Risk-Based Sizing:** The strategy will risk a fixed percentage of the total account equity on every single trade, regardless of the trade's parameters.
    *   **Logic:**
        1.  Define a `riskPercent` (e.g., 1.0 for 1% of equity).
        2.  Calculate the `riskAmount` in currency: `strategy.equity * (riskPercent / 100)`.
        3.  Calculate the `stopLossDistance` in currency per share/contract: `abs(entry_price - stop_loss_price)`.
        4.  Calculate the final **Position Size**: `riskAmount / stopLossDistance`.
    *   **Implementation:** This calculation will be performed on every entry signal to determine the exact quantity for the `strategy.entry()` order. This ensures uniform risk exposure across all trades.

*   **Pyramiding & Scaling:**
    *   **Scaling In (Pyramiding):** This initial framework will not include pyramiding to maintain simplicity and control risk. Advanced versions could add logic to add to a position if, for example, the `momOsc` pulls back towards zero from an extreme, holds, and then accelerates again (a positive `velocity` reading after a dip). This would require sophisticated logic for managing the blended cost basis and adjusting the overall position's stop loss.
    *   **Scaling Out:** As defined in the exit logic, the strategy will scale out by closing a portion of the position at the first take-profit target. This is handled via a `strategy.exit()` or `strategy.close()` call with a specified `qty_percent`.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the transformation of the `indicator` into a `strategy`, incorporating the architectural components discussed above.

```pine
//@version=5
// --- STRATEGY DECLARATION ---
// Note: overlay=true to see trades on the main chart.
// Slippage and commission are CRITICAL for realistic backtests.
strategy("QuantFlow Execution Engine", 
     overlay=true, 
     pyramiding=0, // No pyramiding in this version
     initial_capital=10000, 
     default_qty_type=strategy.calculated, // We will calculate position size
     commission_type=strategy.commission.percent,
     commission_value=0.04, // Realistic commission for crypto/stocks
     slippage=2) // Slippage in ticks

// --- STRATEGY INPUTS ---
riskPercent  = input.float(1.0, "Risk per Trade %", minval=0.1, maxval=10)
atrPeriod    = input.int(14, "ATR Period for Stop Loss")
atrMultiplier= input.float(2.0, "ATR Multiplier for Stop Loss")
rrRatio      = input.float(1.5, "Risk/Reward Ratio for TP1")
exitBars     = input.int(50, "Exit After X Bars of Stagnation")

// --- [PASTE THE ORIGINAL SCRIPT'S LOGIC HERE] ---
// ... from "Constants" down to "Momentum Divergence Logic" ...
// We only need the core calculations, not the plots or table.

// --- Core Calculations (from original script) ---
// [Assuming all the necessary logic to calculate 'momOsc' is here]
// Example placeholder for momOsc
momOsc = ta.ema(close, 12) - ta.ema(close, 26) 
// --- END OF PASTED LOGIC ---


// --- 1. EXECUTION TRIGGERS ---
longCondition = ta.crossover(momOsc, 0)
shortCondition = ta.crossunder(momOsc, 0)

// --- 2. & 3. RISK & POSITION SIZING ---
atrValue = ta.atr(atrPeriod)
stopLossDistance = atrValue * atrMultiplier
riskAmount = (riskPercent / 100) * strategy.equity
positionSize = riskAmount / stopLossDistance

// --- STRATEGY EXECUTION LOGIC ---
// Only trade if we have a valid ATR value
if (atrValue > 0)
    // --- LONG ENTRY ---
    if (longCondition)
        // Close any open short position and go long
        strategy.entry("Long", strategy.long, qty=positionSize, comment="Long Entry")
        
        // Set the bracket orders (Stop Loss and Take Profit 1)
        longStopPrice = strategy.opentrades.entry_price(0) - stopLossDistance
        longTakeProfitPrice = strategy.opentrades.entry_price(0) + (stopLossDistance * rrRatio)
        
        // Exit 50% at TP1, the rest is managed by other exit rules
        strategy.exit("Long TP1/SL", from_entry="Long", qty_percent=50, profit=longTakeProfitPrice, loss=longStopPrice)

    // --- SHORT ENTRY ---
    if (shortCondition)
        // Close any open long position and go short
        strategy.entry("Short", strategy.short, qty=positionSize, comment="Short Entry")

        // Set the bracket orders (Stop Loss and Take Profit 1)
        shortStopPrice = strategy.opentrades.entry_price(0) + stopLossDistance
        shortTakeProfitPrice = strategy.opentrades.entry_price(0) - (stopLossDistance * rrRatio)

        // Exit 50% at TP1, the rest is managed by other exit rules
        strategy.exit("Short TP1/SL", from_entry="Short", qty_percent=50, profit=shortTakeProfitPrice, loss=shortStopPrice)


// --- MULTI-TIERED EXIT LOGIC (CONTINUED) ---
// Time-Based Stagnation Exit: Close position if open for too long
if (barssince(strategy.opentrades > 0) > exitBars)
    strategy.close_all(comment="Stagnation Exit")

// End of Day Exit: Close all positions on the last bar of the day (for intraday)
isEod = timeframe.isintraday and (time_close("D") - time < 60000 * 60) // Last hour
if isEod
    strategy.close_all(comment="EOD Exit")

// --- VISUALS for Strategy ---
// Plot stop loss levels for open trades
plot(strategy.opentrades > 0 ? strategy.opentrades.entry_price(0) - stopLossDistance : na, "Long SL", color.red, style=plot.style_linebr)
plot(strategy.opentrades > 0 ? strategy.opentrades.entry_price(0) + stopLossDistance : na, "Short SL", color.red, style=plot.style_linebr)
```
    