
# Indicators to Strategy Blueprint

The provided script is a sophisticated visualization tool for multi-timeframe volume profiles. It excels at displaying key market-generated levels like the Point of Control (POC), Value Area High (VAH), and Value Area Low (VAL). However, it contains no inherent trading logic. To transform this into a production-ready execution framework, we must first define a trading thesis based on the information it provides.

A common and robust approach for trading with volume profiles is **mean reversion**. This thesis assumes that price will tend to revert to the area of highest value (the POC) after testing the extremes of the value area (VAH/VAL). We will build our automated strategy around this principle.

### 1. Execution Triggers (Entry & Direction)

The core of the strategy is to identify when the market rejects a move outside the established value area of a significant higher timeframe (HTF). We will use one of the user-defined HTFs (e.g., `htf1` - the 30-minute profile) as our primary signal generator.

*   **Long Entry Condition:**
    *   **Thesis:** The market has tested below the Value Area Low (VAL) and is being rejected, signaling a probable return into the value area.
    *   **Boolean Logic:** `(low < VAL_HTF) and (close > VAL_HTF)`
    *   This condition triggers when a candle's low pierces the VAL, but the candle *closes* back above it. This "rejection" pattern is our entry signal.

*   **Short Entry Condition:**
    *   **Thesis:** The market has tested above the Value Area High (VAH) and is being rejected, signaling a probable return into the value area.
    *   **Boolean Logic:** `(high > VAH_HTF) and (close < VAH_HTF)`
    *   This is the mirror image of the long entry. The candle's high must exceed the VAH, but the close must be back below it.

*   **Execution Nuances:**
    *   **Execution Timing:** All entries must be executed at the **"Close" of the signal bar**. This is critical for this type of rejection strategy. Executing on real-time price action would lead to premature entries as price simply *touches* the level, without confirming rejection. In Pine Script, this is achieved with `process_orders_on_close = true`.
    *   **Signal Reversals:** The system must be capable of flipping its position. If the strategy is in a long position and a valid short entry condition triggers, the framework should automatically close the long and initiate a new short. Using a consistent ID for `strategy.entry` (e.g., "VP_Reversion") handles this reversal seamlessly.

### 2. Multi-Tiered Exit Logic

A professional exit strategy is not a single point but a series of rules designed to manage the trade from inception to completion.

*   **Initial Stop Loss (Volatility-Based):**
    *   An arbitrary percentage or point-based stop is inadequate as it ignores market volatility. The stop loss will be calculated using the Average True Range (ATR).
    *   **Long Trade Stop:** `Entry Price - (ATR(14) * 2)`
    *   **Short Trade Stop:** `Entry Price + (ATR(14) * 2)`
    *   This places the stop outside the typical noise of the market, giving the trade room to work. The multiplier (2x in this case) should be a configurable input.

*   **Take Profit / Trailing Mechanism (Multi-Stage):**
    *   The logical primary target for a mean reversion trade is the **Point of Control (POC)** of the same HTF profile. However, exiting fully at the POC can leave profits on the table if the momentum continues.
    *   **Stage 1 (Scale Out):** Exit 50% of the position when the price reaches the POC.
    *   **Stage 2 (Risk Removal):** Upon the partial exit at the POC, move the stop loss on the remaining 50% of the position to the entry price (breakeven).
    *   **Stage 3 (Trailing Stop):** For the remaining position, activate a trailing stop to capture further gains. A simple and effective method is to trail the stop by a multiple of the ATR (e.g., `high - 3 * ATR` for a long) or trail it below the low of the previous `N` bars (a "chandelier" exit).

*   **Time-Based Exits:**
    *   **End of Session:** For intraday strategies, it's critical to avoid unmanaged overnight risk. A rule will be implemented to flatten any open position `X` minutes before the session close (e.g., 16:45 EST for US Equities).
    *   **Stagnation Exit:** If a trade has been open for a significant number of bars (e.g., 75 bars) without hitting either the stop loss or the initial take profit, it indicates a lack of momentum. The position will be closed to free up capital for better opportunities.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term survival. We will implement a fixed-fractional risk model.

*   **Risk-Based Sizing:**
    *   The strategy will risk a fixed percentage of the total account equity on every single trade (e.g., 1%).
    *   The distance between the entry price and the initial stop loss defines the per-share/per-contract risk.
    *   **Formula:** `Position Size = (Account Equity * Risk Percentage) / |Entry Price - Stop Loss Price|`
    *   **Example:** With a $100,000 account and a 1% risk rule ($1,000 risk per trade), if the distance from entry to stop is $2.50, the position size would be `$1000 / $2.50 = 400 shares`. This ensures that a losing trade costs a predictable amount, regardless of the trade's specific parameters.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, the strategy will scale out of positions at the POC. This is a core feature.
    *   **Pyramiding (Scaling In):** For this mean-reversion strategy, pyramiding (adding to a winning position) is generally counter-intuitive and will be disabled (`pyramiding = 0`). The thesis is based on a move from an extreme to the mean, not a sustained trend. Adding to the position as it approaches its target (the POC) increases risk for diminishing reward.

### 4. Implementation Snippet (Pine Logic)

The following snippet demonstrates how to wrap the core concepts from the Volume Profile indicator into a professional `strategy` framework. This code is a conceptual blueprint, abstracting the complex profile calculations into placeholder functions (`getHTF_VAH`, `getHTF_VAL`, `getHTF_POC`) for clarity.

```pine
//@version=5
// STRATEGY DEFINITION
strategy("VP Mean Reversion Architect", 
     overlay=true, 
     process_orders_on_close=true, 
     pyramiding=0, 
     initial_capital=100000,
     commission_type=strategy.commission.percent, 
     commission_value=0.04, // Realistic commission for futures/stocks
     slippage=2) // Realistic slippage in ticks

// --- INPUTS ---
// Risk Management
riskPercent = input.float(1.0, "Risk per Trade %", minval=0.1, maxval=5.0) / 100
atrPeriod = input.int(14, "ATR Period")
atrMultiplier = input.float(2.0, "ATR Stop Multiplier")

// Time-Based Exits
useEodExit = input.bool(true, "Use End-of-Day Exit?")
eodTime = input.session("1645-1700", "End of Day Session")
isEod = time(timeframe.period, eodTime)

// --- DATA & INDICATORS ---
// Placeholder functions to represent fetching the complex VP data.
// In a real implementation, this would involve porting the logic from the original script
// to run historically and return these values.
var VAH_HTF = 0.0
var VAL_HTF = 0.0
var POC_HTF = 0.0

// This is a simplified representation of getting the HTF values.
// A full implementation would require careful state management.
isNewHtfBar = timeframe.change("30")
if isNewHtfBar
    // ... complex calculation logic from the original script would run here ...
    // For demonstration, we'll use simple pivots as placeholders.
    VAH_HTF := ta.pivothigh(high, 10, 1)[1]
    VAL_HTF := ta.pivotlow(low, 10, 1)[1]
    POC_HTF := (VAH_HTF + VAL_HTF) / 2

// Plot for visualization
plot(VAH_HTF, "VAH", color.red, style=plot.style_circles)
plot(VAL_HTF, "VAL", color.green, style=plot.style_circles)
plot(POC_HTF, "POC", color.blue, style=plot.style_cross)

// --- STRATEGY LOGIC ---
// Entry Conditions
longEntryCondition = low < VAL_HTF and close > VAL_HTF and VAL_HTF > 0
shortEntryCondition = high > VAH_HTF and close < VAH_HTF and VAH_HTF > 0

// Risk & Sizing Calculation
atrValue = ta.atr(atrPeriod)
stopDistance = atrValue * atrMultiplier

longStopPrice = close - stopDistance
shortStopPrice = close + stopDistance

positionSize = (strategy.equity * riskPercent) / stopDistance

// --- EXECUTION ENGINE ---
// Entry Logic
if (longEntryCondition)
    strategy.entry("VP_Reversion", strategy.long, qty=positionSize)
    // Set the partial take profit and stop loss for the long trade
    strategy.exit("Exit Long", from_entry="VP_Reversion", qty_percent=50, limit=POC_HTF)
    strategy.exit("Stop Long", from_entry="VP_Reversion", stop=longStopPrice)

if (shortEntryCondition)
    strategy.entry("VP_Reversion", strategy.short, qty=positionSize)
    // Set the partial take profit and stop loss for the short trade
    strategy.exit("Exit Short", from_entry="VP_Reversion", qty_percent=50, limit=POC_HTF)
    strategy.exit("Stop Short", from_entry="VP_Reversion", stop=shortStopPrice)

// Breakeven Logic: If position is profitable after hitting TP1, move stop to breakeven
if (strategy.position_size > 0 and strategy.opentrades.profit(0) > 0)
    strategy.exit("BE Long", from_entry="VP_Reversion", stop=strategy.opentrades.entry_price(0))

if (strategy.position_size < 0 and strategy.opentrades.profit(0) > 0)
    strategy.exit("BE Short", from_entry="VP_Reversion", stop=strategy.opentrades.entry_price(0))

// Time-Based Exit
if (useEodExit and isEod and strategy.position_size != 0)
    strategy.close_all(comment="EOD Exit")

```
    