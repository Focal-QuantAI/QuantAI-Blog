
# Indicators to Strategy Blueprint

Here is the architectural blueprint for transforming the "Adaptive Regime Filter + Divergence (AER-VN)" indicator into a production-ready automated execution framework.

### 1. Execution Triggers (Entry & Direction)

The core of this strategy is to use the high-probability divergence signals as entry triggers, but only when confirmed by the adaptive regime filter. This avoids taking reversal signals in strongly trending markets where they are likely to fail.

*   **Long Entry Condition (`longCondition`):**
    1.  A **Regular Bullish Divergence** (`regBull`) is detected. This signals potential trend exhaustion to the downside and is a primary reversal signal.
    2.  **AND** the market regime is **NOT** in a confirmed `isDowntrend`. This acts as a crucial filter, preventing entries into a "falling knife" scenario. We are willing to enter during consolidation or chop, but not against a strong, established downtrend.

*   **Short Entry Condition (`shortCondition`):**
    1.  A **Regular Bearish Divergence** (`regBear`) is detected. This signals potential trend exhaustion to the upside.
    2.  **AND** the market regime is **NOT** in a confirmed `isUptrend`. This filter prevents shorting a powerful bull market that is merely pausing.

*   **Execution Nuances:**
    *   **Execute on Bar Close:** The divergence signals (`regBull`, `regBear`) are only confirmed after a swing point is established and the subsequent bar closes. The indicator's logic confirms the swing using `close[1]`, meaning the signal is valid only at the `close` of the current bar. Therefore, the strategy must execute on bar close (`process_orders_on_close=true`). Attempting to execute on real-time price action would lead to trades based on unconfirmed, repainting signals.
    *   **Signal Reversals:** The system must be able to flip positions. If the strategy is in a long position and a valid `shortCondition` occurs, the framework will issue a command to close the long position and immediately initiate a new short position. This is handled by explicitly closing the existing trade ID before opening the new one, ensuring clean order management.

### 2. Multi-Tiered Exit Logic

A static exit strategy is a recipe for failure. A professional system requires a dynamic, multi-stage approach to manage the trade from entry to exit.

*   **Initial Stop Loss (Volatility-Based & Structural):**
    *   The stop loss is not based on a fixed percentage but is placed at a structurally significant level, invalidated by volatility.
    *   **For Longs:** The stop is placed below the swing low that formed the bullish divergence, with an added buffer based on the Average True Range (ATR).
        *   `Stop Loss Price = swing_low_price - (ATR_Stop_Multiplier * ATR)`
    *   **For Shorts:** The stop is placed above the swing high that formed the bearish divergence, plus an ATR-based buffer.
        *   `Stop Loss Price = swing_high_price + (ATR_Stop_Multiplier * ATR)`
    *   This method places the stop at a point where, if breached, the original premise for the trade (the divergence) is logically invalidated.

*   **Take Profit / Trailing Mechanism:**
    *   **Stage 1: Initial Profit Target (Scaling Out):** We will not wait for a full trend reversal to take all profits. The first target is based on a Risk/Reward multiple.
        *   **Logic:** Exit 50% of the position when the price reaches a 2:1 reward-to-risk ratio (e.g., `entry_price + 2 * initial_risk_per_share`). This secures profit, reduces the position's net risk, and allows the remainder to run.
    *   **Stage 2: Trailing Stop (Regime-Aware):** After the first profit target is hit, the stop loss on the remaining 50% of the position is moved to breakeven. Subsequently, a dynamic trailing stop is activated.
        *   **Logic:** The trade is allowed to continue as long as the regime remains favorable (`isTrending` is true). If the regime filter detects that the trend has ended (`isTrending` becomes false), the remainder of the position is closed. This uses the indicator's own logic to signal that the favorable market conditions for the trade have ceased.

*   **Time-Based Exits:**
    *   **Stagnation Exit:** If a trade has been open for a specified number of bars (e.g., 50 bars) and has not hit either its stop loss or its first take-profit target, it is closed. This prevents capital from being tied up in non-performing trades.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical factor in determining long-term survival and profitability. We will employ a precise, risk-based model.

*   **Risk-Based Sizing:**
    *   The core principle is to risk a fixed percentage of total account equity on every single trade, regardless of the trade setup.
    *   **Formula:**
        `Position Size = (Trade_Capital * Risk_Per_Trade_%) / abs(Entry_Price - Stop_Loss_Price)`
    *   **Example:** With a $100,000 account and a 1% risk rule, we risk $1,000 per trade. If the distance from our entry to our structural stop loss is $5.00 per share, the position size would be `$1000 / $5.00 = 200 shares`. This calculation is performed dynamically for every new trade signal.

*   **Pyramiding & Scaling:**
    *   **Scaling In (Pyramiding):** While not implemented in the initial version for simplicity and safety, a future enhancement would be to use the "Hidden Divergence" signals (`hidBull`, `hidBear`) as triggers to add to an existing, profitable position. For example, if in a profitable long trade, a `hidBull` signal appears while the regime is still `isUptrend`, a new, smaller position could be added with its own discrete risk parameters. For now, the strategy will enforce a maximum of one open position at a time (`pyramiding=0`).
    *   **Scaling Out:** As defined in the exit logic, the strategy is built to scale out of trades, securing profits at logical milestones rather than relying on an all-or-nothing exit.

### 4. Implementation Snippet (Pine Logic)

This pseudocode and Pine Script snippet demonstrates the transition from the `indicator` to a `strategy`, incorporating the professional-grade logic defined above.

```pine
// ========== STRATEGY TRANSFORMATION BLOCK ==========
//@version=5
// Note: The original script was v6, but strategy features are more mature in v5.
// This snippet is illustrative and would be merged with the original indicator code.

// 1. STRATEGY DECLARATION
strategy("AER-VN Execution Engine [KEYALGOS]", 
     overlay=true, 
     process_orders_on_close=true,
     pyramiding=0, // One trade at a time
     default_qty_type=strategy.fixed, // We will calculate size manually
     initial_capital=100000,
     commission_type=strategy.commission.percent,
     commission_value=0.05, // 0.05% per order
     slippage=2 // 2 ticks of slippage
     )

// 2. RISK MANAGEMENT INPUTS
riskPercent = input.float(1.0, "Risk per Trade (%)", minval=0.1, maxval=5.0, step=0.1)
atrStopMultiplier = input.float(1.5, "ATR Stop Multiplier", minval=0.5, maxval=5.0)
rrRatio = input.float(2.0, "Take Profit R:R Ratio", minval=1.0)
maxBarsInTrade = input.int(50, "Max Bars in Stagnant Trade", minval=10)

// Assume all indicator logic (ER, Divergence, Regimes) from the original script exists above this line.
// We need to capture the swing point prices for our stops.
var float swingLowPriceForStop = na
var float swingHighPriceForStop = na

if swingHighConfirm
    swingHighPriceForStop := close[1]
if swingLowConfirm
    swingLowPriceForStop := close[1]

// 3. EXECUTION TRIGGERS (ENTRY & DIRECTION)
longCondition = regBull and not isDowntrend
shortCondition = regBear and not isUptrend

// 4. POSITION SIZING & STOP CALCULATION
tradeCapital = strategy.equity
riskPerTrade = (riskPercent / 100) * tradeCapital

// Long Trade Calculations
longStopPrice = swingLowPriceForStop - (ta.atr(atrLength) * atrStopMultiplier)
longRiskAmount = close - longStopPrice
longPositionSize = longRiskAmount > 0 ? riskPerTrade / longRiskAmount : 0
longTakeProfitPrice = close + (longRiskAmount * rrRatio)

// Short Trade Calculations
shortStopPrice = swingHighPriceForStop + (ta.atr(atrLength) * atrStopMultiplier)
shortRiskAmount = shortStopPrice - close
shortPositionSize = shortRiskAmount > 0 ? riskPerTrade / shortRiskAmount : 0
shortTakeProfitPrice = close - (shortRiskAmount * rrRatio)

// 5. ORDER EXECUTION LOGIC
// --- ENTRY LOGIC ---
if (strategy.position_size == 0) // Only enter if flat
    if (longCondition)
        strategy.entry("Long", strategy.long, qty=longPositionSize)
        // Set up exits for the "Long" trade
        strategy.exit("Long TP/SL", from_entry="Long", qty_percent=50, profit=longTakeProfitPrice, loss=longStopPrice)
        
    if (shortCondition)
        strategy.entry("Short", strategy.short, qty=shortPositionSize)
        // Set up exits for the "Short" trade
        strategy.exit("Short TP/SL", from_entry="Short", qty_percent=50, profit=shortTakeProfitPrice, loss=shortStopPrice)

// --- DYNAMIC EXIT LOGIC (TIME & REGIME BASED) ---
// Stagnation Exit
isStagnant = (bar_index - strategy.opentrades.entry_bar_index(0)) > maxBarsInTrade
if isStagnant
    strategy.close_all(comment="Stagnation Exit")

// Regime-based Trailing Stop for the second half of the position
isInProfitLong = strategy.position_size > 0 and close > strategy.opentrades.entry_price(0)
isInProfitShort = strategy.position_size < 0 and close < strategy.opentrades.entry_price(0)

// If in a profitable long trade and the trend dies, exit.
if isInProfitLong and not isTrending
    strategy.close("Long", comment="Regime Exit Long")

// If in a profitable short trade and the trend dies, exit.
if isInProfitShort and not isTrending
    strategy.close("Short", comment="Regime Exit Short")

```
    