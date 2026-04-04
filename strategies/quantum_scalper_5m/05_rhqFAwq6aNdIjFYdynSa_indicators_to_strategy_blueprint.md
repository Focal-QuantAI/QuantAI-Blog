
# Indicators to Strategy Blueprint

Here is the architectural breakdown for transforming the "Quantum Scalper 5M" indicator into a production-ready automated execution framework.

### 1. Execution Triggers (Entry & Direction)

The core of the entry logic is a three-factor confluence: a retest of a Support/Resistance level, a Bollinger Band, and the "Quantum Trend Flow" channel.

*   **Long Entry Condition:** A `longCondition` is met when:
    1.  The bar's `low` simultaneously touches or penetrates the fixed Support level (`srLow_fixed`), the lower Bollinger Band (`bb_lower`), and the lower Quantum Channel (`qtf_lower`).
    2.  The bar's `close` price recovers to finish *above* all three of these same levels.
    3.  The optional higher timeframe (MTF) trend filter confirms a bullish environment (`mtf_check_long`).
    4.  The strategy is currently flat or in a short position (to allow for reversals).

    ```pinescript
    // Precise Boolean Logic for Long Entry
    bool longTouch = low <= srLow_fixed and low <= bb_lower and low <= qtf_lower
    bool longReject = close > srLow_fixed and close > bb_lower and close > qtf_lower
    bool longCondition = longTouch and longReject and mtf_check_long
    ```

*   **Short Entry Condition:** A `shortCondition` is met when:
    1.  The bar's `high` simultaneously touches or extends beyond the fixed Resistance level (`srHigh_fixed`), the upper Bollinger Band (`bb_upper`), and the upper Quantum Channel (`qtf_upper`).
    2.  The bar's `close` price rejects the highs to finish *below* all three of these same levels.
    3.  The optional MTF trend filter confirms a bearish environment (`mtf_check_short`).
    4.  The strategy is currently flat or in a long position.

    ```pinescript
    // Precise Boolean Logic for Short Entry
    bool shortTouch = high >= srHigh_fixed and high >= bb_upper and high >= qtf_upper
    bool shortReject = close < srHigh_fixed and close < bb_upper and close < qtf_upper
    bool shortCondition = shortTouch and shortReject and mtf_check_short
    ```

*   **Execution Nuances:**
    *   **Execute at "Close" of the bar.** The signal's structure is a "touch and reject" pattern that is only confirmed *after* the bar has completed. Attempting to execute on real-time price action (`calc_on_every_tick=true`) is highly susceptible to repainting; a wick can form and then retract, invalidating the signal before the bar closes. For reliability and to prevent false signals, orders should be placed only on the confirmation of a bar's close. The entry price will be the open of the next bar.

*   **Signal Reversals:**
    *   The system must be designed to handle immediate reversals. If the strategy is in a long position and a `shortCondition` is triggered, the framework should execute a single transaction that closes the long position and simultaneously opens a new short position. In Pine Script, using the same `id` for `strategy.entry` (e.g., `strategy.entry("QS_Trade", strategy.long)`) automatically handles this by reversing the existing position.

### 2. Multi-Tiered Exit Logic

Moving beyond the script's fixed-percentage visualizer, a professional exit framework must be dynamic and multi-faceted.

*   **Initial Stop Loss (Volatility-Based):**
    *   The stop loss will not be a fixed percentage. It will be calculated based on the market's recent volatility using the Average True Range (ATR).
    *   **Calculation (Long):** `Stop Loss Price = Entry Price - (ATR(14) * StopMultiplier)`. A `StopMultiplier` of 1.5 to 2.5 is standard.
    *   **Calculation (Short):** `Stop Loss Price = Entry Price + (ATR(14) * StopMultiplier)`.
    *   *Alternative:* A structural stop could be placed just below the `low` of the signal bar for a long entry (or above the `high` for a short), plus a small buffer. This anchors the stop to the specific price action that generated the signal.

*   **Take Profit / Trailing Mechanism:**
    *   A static take profit is inflexible. We will implement a two-stage dynamic exit.
    *   **Stage 1: Initial Target & Breakeven.** The first take-profit target will be set at a 1.5:1 Risk/Reward ratio. For example, if the distance to the stop loss is $50, the first target is $75 from the entry. Upon hitting this target, 50% of the position is closed. Simultaneously, the stop loss on the remaining 50% is moved to the entry price (breakeven).
    *   **Stage 2: Trailing Stop.** The remaining position is managed with a trailing stop to capture the majority of a trend. The stop will trail behind the price using the `qtf_basis` (the 20-period EMA from the Quantum Channel). If price closes below this basis (for a long) or above it (for a short), the remainder of the position is exited.

*   **Time-Based Exits:**
    *   **End of Day (EOD) Exit:** As a scalping/intraday strategy, all open positions must be squared before the market close to eliminate overnight risk. A time-based rule will close any open position 15 minutes before the end of the trading session.
    *   **Stagnation Exit:** If a trade has been open for a specified number of bars (e.g., 75 bars on a 5-minute chart) and has not hit either the initial take profit or the stop loss, it will be closed. This prevents capital from being tied up in non-performing trades.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term viability. We will use a fixed-fractional risk model.

*   **Risk-Based Sizing:**
    *   The strategy will risk a fixed percentage of the total account equity on every single trade (e.g., 1% of equity). The position size is then derived from this risk amount and the distance to the initial stop loss.
    *   **Formula:**
        `StopLossDistance = abs(EntryPrice - StopLossPrice)`
        `RiskAmount = AccountEquity * RiskPercent`
        `PositionSize = RiskAmount / StopLossDistance`
    *   This ensures that a losing trade always results in a predictable, fixed-percentage loss of capital, regardless of the trade's specific entry/stop parameters.

*   **Pyramiding & Scaling:**
    *   **Pyramiding (Scaling In):** The base logic of the "Quantum Scalper" is a reversal/rejection strategy, which is not naturally suited for pyramiding. Adding to a position would require a separate set of rules (e.g., buying on a successful retest of the `qtf_basis` after an initial entry). For this framework, pyramiding will be **disabled** to maintain the integrity of the core signal. The strategy will only allow one open position at a time.
    *   **Scaling Out:** As defined in the exit logic, the strategy will scale out of winning trades. The `strategy.exit` function will be used to close a percentage of the position (`qty_percent = 50`) at the first take-profit target.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the indicator logic into a `strategy` with the professional execution components defined above.

```pine
//@version=5
// 1. STRATEGY DECLARATION with realistic friction
strategy("Quantum Scalper - PRO", 
     overlay=true, 
     pyramiding=0, // No pyramiding, one trade at a time
     initial_capital=10000, 
     commission_type=strategy.commission.percent, 
     commission_value=0.04, // Realistic commission for futures/CFDs
     slippage=2, // 2 ticks of slippage on entry/exit
     calc_on_close=true) // Execute on bar close for reliability

// --- INPUTS FOR EXECUTION ---
riskPercent    = input.float(1.0, "Risk per Trade (%)", minval=0.1, maxval=5.0)
atrLen         = input.int(14, "ATR Length for Stop")
atrStopMult    = input.float(2.0, "ATR Stop Multiplier")
rrTarget       = input.float(1.5, "Initial R:R Target")
stagnationBars = input.int(75, "Max Bars in Trade")

// [ ... Paste the original indicator's calculation logic here for srHigh_fixed, bb_lower, qtf_lower, etc. ... ]
// Example:
// lookback = input.int(50, title="Lookback Period")
// srHigh = ta.highest(high, lookback)
// ... and so on for all indicators.

// --- 2. EXECUTION TRIGGERS ---
bool longTouch = low <= srLow_fixed and low <= bb_lower and low <= qtf_lower
bool longReject = close > srLow_fixed and close > bb_lower and close > qtf_lower
bool longCondition = longTouch and longReject and mtf_check_long and strategy.position_size <= 0

bool shortTouch = high >= srHigh_fixed and high >= bb_upper and high >= qtf_upper
bool shortReject = close < srHigh_fixed and close < bb_upper and close < qtf_upper
bool shortCondition = shortTouch and shortReject and mtf_check_short and strategy.position_size >= 0

// --- 3. RISK MANAGEMENT & EXIT CALCULATION ---
float atrValue = ta.atr(atrLen)
float stopLossDistance = atrValue * atrStopMult
float takeProfitDistance = stopLossDistance * rrTarget

// Position Sizing
float riskAmount = (strategy.equity * riskPercent) / 100
float positionSize = riskAmount / (stopLossDistance * syminfo.pointvalue)

// Time-based Exit
isEOD = (hour(time_close) == 16 and minute(time_close) >= 45) // Example for US Equities

// --- 4. STRATEGY ORDERS ---
// LONG ENTRY
if (longCondition)
    // Calculate SL/TP levels for this specific trade
    longStopPrice = close - stopLossDistance
    longTakeProfitPrice = close + takeProfitDistance
    
    // Place Entry Order
    strategy.entry("Long", strategy.long, qty=positionSize)
    
    // Place Bracket Orders (TP1 and SL) immediately after entry
    strategy.exit("L-Exit", from_entry="Long", qty_percent=50, profit=takeProfitDistance, loss=stopLossDistance)
    // Place Trailing Stop for the remainder (activated after entry)
    strategy.exit("L-Trail", from_entry="Long", trail_price=qtf_basis, trail_offset=0)

// SHORT ENTRY
if (shortCondition)
    // Calculate SL/TP levels for this specific trade
    shortStopPrice = close + stopLossDistance
    shortTakeProfitPrice = close - takeProfitDistance
    
    // Place Entry Order
    strategy.entry("Short", strategy.short, qty=positionSize)
    
    // Place Bracket Orders (TP1 and SL)
    strategy.exit("S-Exit", from_entry="Short", qty_percent=50, profit=takeProfitDistance, loss=stopLossDistance)
    // Place Trailing Stop for the remainder
    strategy.exit("S-Trail", from_entry="Short", trail_price=qtf_basis, trail_offset=0)

// TIME-BASED & STAGNATION EXITS
if (strategy.position_size != 0)
    if (bar_index - strategy.opentrades.entry_bar_index(0) > stagnationBars)
        strategy.close_all(comment="Stagnation Exit")
    if (isEOD)
        strategy.close_all(comment="EOD Exit")

```
    