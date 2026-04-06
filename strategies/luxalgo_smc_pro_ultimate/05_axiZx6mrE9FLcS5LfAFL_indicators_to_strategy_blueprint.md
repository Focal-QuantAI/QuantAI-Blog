
# Indicators to Strategy Blueprint

Here is the architectural breakdown for transforming the "LuxAlgo SMC Pro Ultimate" script into a production-grade execution framework.

---

The provided script, while labeled a `strategy`, operates more like a conceptual backtest. Its position sizing is naive, its exit logic is basic, and its order execution model doesn't reflect real-world market conditions. To elevate this to a professional automated system, we must overhaul its core execution, risk, and trade management engines.

### 1. Execution Triggers (Entry & Direction)

The core logic relies on a Market Structure Shift (MSS) confirmed by a confluence of optional filters. We will refine the execution of these triggers for production.

*   **Long Entry Condition:** A `bTrigger` becomes `true` when:
    1.  **Primary Signal:** A Market Structure Shift to the upside occurs (`mssL`). This happens when the price closes above a recently formed swing high or internal high.
    2.  **Confluence Filters (if enabled):**
        *   **Premium/Discount:** The price is in a "Discount" zone (below the 50% level of the `pdLookback` range).
        *   **Fair Value Gap (FVG):** A bullish FVG has formed on the current or previous bar.
        *   **Divergence:** A bullish RSI divergence is present.
        *   **Trend Filter:** The price is above the Bollinger Band basis line.

*   **Short Entry Condition:** A `sTrigger` becomes `true` when:
    1.  **Primary Signal:** A Market Structure Shift to the downside occurs (`mssS`). This happens when the price closes below a recently formed swing low or internal low.
    2.  **Confluence Filters (if enabled):**
        *   **Premium/Discount:** The price is in a "Premium" zone (above the 50% level of the `pdLookback` range).
        *   **Fair Value Gap (FVG):** A bearish FVG has formed on the current or previous bar.
        *   **Divergence:** A bearish RSI divergence is present.
        *   **Trend Filter:** The price is below the Bollinger Band basis line.

#### Execution Nuances

*   **Execution Timing:** The original script implicitly executes on the **open of the next bar**. This is unrealistic and introduces lookahead bias. A professional system must execute based on information available at the time of the decision.
    *   **Solution:** We will set `process_orders_on_close = true` in the `strategy` declaration. This ensures that if a signal (`bTrigger` or `sTrigger`) is confirmed at the close of Bar `X`, the market order is filled at the closing price of Bar `X`, providing a more realistic backtest. In a live environment, this translates to sending a market order the instant the bar closes.

*   **Signal Reversals:** The original script explicitly prevents entering a new trade while one is active (`strategy.position_size == 0`). This is too restrictive. A strong bearish signal should be able to close an existing long position and initiate a new short.
    *   **Solution:** We will implement logic to handle reversals. If a `sTrigger` occurs while `strategy.position_size > 0`, the system will first execute `strategy.close("Long")` and then immediately execute `strategy.entry("Short")`. This ensures a clean flip and accurate accounting.

### 2. Multi-Tiered Exit Logic

A single TP/SL system is brittle. A robust framework layers multiple exit conditions to adapt to market behavior.

*   **Initial Stop Loss:** The use of an ATR multiplier is a solid foundation. We will refine it to be placed relative to the structure that triggered the trade, not just the bar's low/high.
    *   **Longs:** The stop loss will be placed at `low[trigger_bar] - (atr * atrMult)`. This anchors the risk to the actual pivot or candle that generated the signal.
    *   **Shorts:** The stop loss will be placed at `high[trigger_bar] + (atr * atrMult)`.

*   **Take Profit / Trailing Mechanism:** We will replace the single TP and move-to-breakeven logic with a more dynamic, multi-stage approach.
    1.  **TP1 (Scaling Out):** At a 1.5R profit target, close **50%** of the position. This secures initial profit and de-risks the trade.
    2.  **Move to Breakeven:** *After* TP1 is hit, the stop loss for the remaining 50% of the position is moved to the entry price. This is a conservative approach.
    3.  **TP2 / Trailing Stop:** For the remaining position, we will not use a fixed second target. Instead, we will employ a more intelligent trailing stop to capture the majority of the trend. A **Chandelier Exit** is ideal here:
        *   **Long Trail:** The stop will trail at `ta.highest(high, 22) - (atr * atrMult)`. It trails below the highest high over the last `N` bars, giving the trend room to make pullbacks without stopping out prematurely.
        *   **Short Trail:** The stop will trail at `ta.lowest(low, 22) + (atr * atrMult)`.

*   **Time-Based Exits:** Capital should not be held hostage by stagnant trades.
    *   **Stagnation Exit:** If a position has been open for `X` bars (e.g., 50 bars) and has not yet hit TP1, exit the trade at market. This frees up capital for higher-probability opportunities.
    *   **End-of-Session Exit:** For intraday timeframes, an "End of Day" (EOD) exit is non-negotiable to manage overnight risk. The system will be configured to close any open position 15 minutes before the session close.

### 3. Capital Allocation & Risk Management

This is the most critical overhaul. The original script's `strategy.percent_of_equity` sizing is fundamentally flawed as it ignores the trade-specific risk (stop distance).

*   **Risk-Based Sizing:** We will implement a function to risk a fixed percentage of account equity on every single trade, regardless of the stop-loss distance.
    *   **Rule:** Risk exactly `1%` of `strategy.equity` per trade.
    *   **Calculation:**
        1.  `riskAmount = strategy.equity * riskPercent` (e.g., $10,000 * 0.01 = $100)
        2.  `riskPerUnit = abs(entry_price - stop_loss_price)`
        3.  `positionSize = riskAmount / riskPerUnit`
    *   This ensures that whether the stop is 10 points away or 100 points away, the maximum potential loss on the trade is always the same ($100 in this example).

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** This is handled by our multi-tiered exit logic (50% at TP1).
    *   **Pyramiding (Adding to Winners):** We will establish strict rules for adding to a position. This is an advanced feature and should be used with caution.
        *   **Condition:** A new position can only be added if the original trade is in profit by at least 1.5R (i.e., TP1 has been hit and the stop is at breakeven).
        *   **Signal:** A new, valid entry signal (`bTrigger` or `sTrigger`) must occur on a pullback (e.g., a new FVG is formed and tested).
        *   **Risk:** The new position will be sized to risk `0.5%` of current equity. The total risk exposure across all open positions on a single asset should not exceed the initial max risk (e.g., 1.5%).

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the transition to a professional `strategy` call, incorporating the principles of realistic friction, risk-based sizing, and multi-stage exits.

```pine
// This Pine Script® code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
//@version=5

// --- 1. STRATEGY DECLARATION: PRODUCTION-GRADE ---
strategy("SMC Pro - Execution Framework", 
     overlay=true, 
     process_orders_on_close=true, // Execute on bar close for realism
     initial_capital=10000,
     commission_type=strategy.commission.percent,
     commission_value=0.075, // Realistic broker commission
     slippage=2) // Realistic slippage in ticks

// --- 2. RISK MANAGEMENT INPUTS ---
riskPercent = input.float(1.0, "Risk Per Trade %", minval=0.1, maxval=5.0, step=0.1) / 100
allowReversals = input.bool(true, "Allow Position Reversals?")
stagnationBars = input.int(50, "Max Bars in Trade Before Exit")

// --- [Original Script's Core Calculations: mssL, mssS, bTrigger, sTrigger, etc.] ---
// ... (Assume all the indicator logic from the original script is here)

// --- 3. DYNAMIC POSITION SIZING & EXECUTION ---
var float entryPrice = na
var float stopLossPrice = na
var int entryBar = na

// Calculate Position Size based on fixed-fractional risk
riskAmount = strategy.equity * riskPercent
riskPerUnit = bTrigger ? (close - (low - atr * atrMult)) : sTrigger ? ((high + atr * atrMult) - close) : na
positionSize = riskPerUnit > 0 ? riskAmount / (riskPerUnit * syminfo.pointvalue) : 0

// --- ENTRY LOGIC ---
// Handle Long Entry
if (bTrigger)
    if (strategy.position_size < 0 and allowReversals)
        strategy.close("Short", comment="Reversal to Long")
    if (strategy.position_size == 0)
        entryPrice := close
        stopLossPrice := low - (atr * atrMult)
        entryBar := bar_index
        strategy.entry("Long", strategy.long, qty=positionSize, comment="SMC Long Entry")

// Handle Short Entry
if (sTrigger)
    if (strategy.position_size > 0 and allowReversals)
        strategy.close("Long", comment="Reversal to Short")
    if (strategy.position_size == 0)
        entryPrice := close
        stopLossPrice := high + (atr * atrMult)
        entryBar := bar_index
        strategy.entry("Short", strategy.short, qty=positionSize, comment="SMC Short Entry")

// --- 4. MULTI-TIERED EXIT LOGIC ---
if (strategy.position_size != 0)
    // A. Define TP and Trail levels
    tp1Price = strategy.position_size > 0 ? entryPrice + (entryPrice - stopLossPrice) * tp1RR : entryPrice - (stopLossPrice - entryPrice) * tp1RR
    trailStopPrice = strategy.position_size > 0 ? ta.highest(high, 22) - (atr * atrMult) : ta.lowest(low, 22) + (atr * atrMult)

    // B. Execute Exits with unique IDs for clarity
    // TP1: Scale out 50%
    strategy.exit("TP1", from_entry=strategy.position_size > 0 ? "Long" : "Short", qty_percent=50, limit=tp1Price)
    
    // After TP1, move SL to Breakeven for the rest
    isTp1Hit = strategy.closedtrades.exit_comment(strategy.closedtrades - 1) == "TP1"
    breakevenStop = isTp1Hit ? entryPrice : stopLossPrice

    // Trailing Stop for the remaining position
    finalStopPrice = strategy.position_size > 0 ? math.max(breakevenStop, trailStopPrice) : math.min(breakevenStop, trailStopPrice)
    strategy.exit("TrailSL", from_entry=strategy.position_size > 0 ? "Long" : "Short", stop=finalStopPrice)

    // C. Time-Based Stagnation Exit
    if (bar_index - entryBar > stagnationBars and not isTp1Hit)
        strategy.close(strategy.position_size > 0 ? "Long" : "Short", comment="Stagnation Exit")

// Note: An End-of-Session exit would require session time checks, e.g., `time_close('1500-1545')`
```
    