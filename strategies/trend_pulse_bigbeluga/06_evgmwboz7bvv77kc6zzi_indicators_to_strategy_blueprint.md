
# Indicators to Strategy Blueprint

Based on the "Trend Pulse [BigBeluga]" indicator, here is the architectural blueprint for a production-ready, automated execution framework. The original script excels at identifying trend direction but lacks the critical components for managing risk, capital, and trade execution in a live environment. This framework addresses those gaps.

### 1. Execution Triggers (Entry & Direction)

The core logic of the indicator is sound but must be translated into explicit order commands. The system is inherently a "reversal" system, meaning it is always either in a long or short state.

*   **Long Entry Condition:** A long position is initiated when the `direction` flips from bearish (`false`) to bullish (`true`). This occurs on the bar *after* the price closes above the adaptive trend band.
    *   **Boolean Logic:** `direction[1] == false and direction == true`
    *   This is equivalent to `ta.crossover(close, sma)` from the original script.

*   **Short Entry Condition:** A short position is initiated when the `direction` flips from bullish (`true`) to bearish (`false`). This occurs on the bar *after* the price closes below the last confirmed pivot low.
    *   **Boolean Logic:** `direction[1] == true and direction == false`
    *   This is equivalent to `ta.crossunder(close, pivot)` from the original script.

*   **Execution Nuance: "Close of the Bar" vs. "Real-time"**
    The original script correctly uses `barstate.isconfirmed`, meaning signals are only valid *after* a bar has fully closed. This is the **required execution model** for this strategy.
    *   **Why:** Executing on bar close prevents "repaint" and whipsaws from intra-bar price fluctuations crossing the `sma` or `pivot` levels, only to reverse before the bar closes. It ensures the signal is stable and confirmed.
    *   **Implementation:** Orders should be sent at the open of the next bar (or as close to it as possible) after a signal is confirmed on the close of the previous bar.

*   **Signal Reversals:**
    The strategy should not wait for a stop loss to be hit before entering a new trade in the opposite direction. If a valid entry signal occurs, it must first close any existing position and then immediately open the new one.
    *   **Logic:**
        *   If `longCondition` is true AND `strategy.position_size < 0` (currently short), execute `strategy.close()` for the short position, then execute `strategy.entry()` for the new long position.
        *   If `shortCondition` is true AND `strategy.position_size > 0` (currently long), execute `strategy.close()` for the long position, then execute `strategy.entry()` for the new short position.

### 2. Multi-Tiered Exit Logic

A professional strategy never relies solely on reversal signals for exits. It requires a defensive plan to manage risk and protect profits.

*   **Initial Stop Loss (Volatility-Based):**
    The stop loss must be dynamic and adapt to market volatility. We will use the Average True Range (ATR) for this calculation.
    *   **Long Position Stop:** `Entry Price - (ATR(14) * 2.5)`. The multiplier (e.g., 2.5) should be an input parameter for optimization. The stop is placed a volatility-adjusted distance below the entry. A more conservative placement would be below the low of the signal bar: `low[1] - (ATR(14) * 1.5)`.
    *   **Short Position Stop:** `Entry Price + (ATR(14) * 2.5)`. Similarly, this is placed above the entry. A more conservative placement would be above the high of the signal bar: `high[1] + (ATR(14) * 1.5)`.

*   **Take Profit / Trailing Mechanism:**
    We will implement a two-stage exit to lock in gains while allowing for larger trends to run.
    1.  **Scale-Out at Fixed Target:** Exit a portion of the position (e.g., 50%) when the trade reaches a predefined risk-reward multiple.
        *   **Logic:** Calculate the initial risk (`Initial Risk = abs(Entry Price - Stop Loss Price)`). The first take-profit target is `Entry Price + (Initial Risk * 2.0)` for longs, and `Entry Price - (Initial Risk * 2.0)` for shorts. A 2:1 RR is a common starting point.
    2.  **Dynamic Trailing Stop (Chandelier Exit):** After the first target is hit, the stop loss for the remaining position is moved to breakeven. From there, it begins to trail using a volatility-based mechanism.
        *   **Long Trail Logic:** The stop is trailed at `highest(high, 20) - (ATR(14) * 3)`. It follows the highest high of the last 20 bars, giving the trade room to pull back without being stopped out prematurely.
        *   **Short Trail Logic:** The stop is trailed at `lowest(low, 20) + (ATR(14) * 3)`.

*   **Time-Based Exits:**
    Capital should not be tied up in stagnant trades.
    *   **End of Day (EOD):** For intraday timeframes (e.g., 1H or lower), all open positions should be squared off 15 minutes before the session close to avoid overnight risk.
    *   **Stagnation Exit:** If a trade has been open for a specified number of bars (e.g., 75 bars) and has not reached the first take-profit target, exit the position. This "time stop" prevents capital from being locked in a non-performing trade.

### 3. Capital Allocation & Risk Management

Position sizing is arguably the most critical component of a profitable strategy. We will use a fixed-fractional risk model.

*   **Risk-Based Sizing:**
    The core principle is to risk a fixed percentage of account equity on every single trade. This ensures that a losing streak does not catastrophically damage the account and normalizes position size across varying levels of volatility.
    *   **Inputs:**
        *   `accountEquity`: The total current value of the trading account (`strategy.equity`).
        *   `riskPercent`: The percentage of equity to risk per trade (e.g., 1.0 for 1%).
    *   **Calculation:**
        1.  `riskAmount = accountEquity * (riskPercent / 100)`
        2.  `tradeRiskPerShare = abs(entryPrice - stopLossPrice)`
        3.  **`positionSize = riskAmount / tradeRiskPerShare`**
    *   This calculation is performed *immediately before* an entry order is placed, ensuring the size is always appropriate for the current market conditions and account size.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, the strategy will automatically scale out 50% of the position at the first take-profit target. This is a mandatory risk-reduction feature.
    *   **Pyramiding (Scaling In):** Adding to a winning position is an advanced technique. We can establish a simple, robust rule:
        *   **Rule:** A second unit (pyramid entry) can be added *only if* the initial position has reached at least a 1:1 risk-reward ratio, and a new, higher-probability pullback entry occurs (e.g., price pulls back to and bounces off the 20-period EMA while the main `direction` is still bullish).
        *   **Risk Management:** Upon adding a second unit, the stop loss for the *entire* position (both units) must be moved to the breakeven point of the initial entry. This ensures the trade cannot become a loser. The strategy's `pyramiding` parameter must be set to a value greater than 0.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates how to wrap the original indicator's logic within a professional `strategy` framework, incorporating the components detailed above.

```pine
//@version=5
// 1. STRATEGY DECLARATION
strategy("Trend Pulse Execution Engine", 
     overlay=true, 
     initial_capital=100000, 
     pyramiding=0, // Pyramiding disabled for this example for simplicity
     commission_type=strategy.commission.percent, 
     commission_value=0.075, // Realistic commission
     slippage=2, // Realistic slippage in ticks
     default_qty_type=strategy.percent_of_equity, // Base sizing on equity
     default_qty_value=1 // Placeholder, will be overridden by dynamic sizing
)

// 2. INPUTS FOR RISK & TRADE MANAGEMENT
riskPercent = input.float(1.0, "Risk per Trade %", minval=0.1, step=0.1)
atrPeriod = input.int(14, "ATR Period")
stopAtrMult = input.float(2.5, "Stop Loss ATR Multiplier", minval=0.1)
tpRr = input.float(2.0, "Take Profit R:R Multiple", minval=0.1)

// --- Paste Original Indicator's Input & Calculation Logic Here ---
// (pivLen, smaMax, smaMult, etc. and the calculations for `direction`, `pivot`, `sma`)
// NOTE: For brevity, the original indicator code is omitted but assumed to be present.
// Let's assume `direction`, `sma`, and `pivot` are calculated as in the original script.
// --- End of Pasted Logic ---

// 3. DEFINE EXECUTION TRIGGERS
longCondition = direction and not direction[1]
shortCondition = not direction and direction[1]

// 4. CALCULATE VOLATILITY & RISK PARAMETERS
atrValue = ta.atr(atrPeriod)
entryPrice = strategy.opentrades.entry_price(strategy.opentrades - 1) // Get entry price for open trade

// Calculate Stop Loss levels at the time of signal
stopLossLong = close - (atrValue * stopAtrMult)
stopLossShort = close + (atrValue * stopAtrMult)

// Calculate Position Size based on risk
riskAmount = strategy.equity * (riskPercent / 100)
tradeRiskPerUnit_Long = close - stopLossLong
tradeRiskPerUnit_Short = stopLossShort - close

positionSizeLong = riskAmount / tradeRiskPerUnit_Long
positionSizeShort = riskAmount / tradeRiskPerUnit_Short

// Calculate Take Profit levels
takeProfitLong = close + (tradeRiskPerUnit_Long * tpRr)
takeProfitShort = close - (tradeRiskPerUnit_Short * tpRr)

// 5. EXECUTION LOGIC
// --- ENTRY LOGIC ---
if (longCondition)
    // If we are currently short, close the position first (reversal)
    strategy.close("Short", comment="Reversal to Long")
    // Enter new long position with calculated size and exits
    strategy.entry("Long", strategy.long, qty=positionSizeLong)
    strategy.exit("Exit Long", from_entry="Long", stop=stopLossLong, limit=takeProfitLong, qty_percent=50) // Exit 50% at SL or TP1
    strategy.exit("Trail Long", from_entry="Long", stop=stopLossLong) // Protective stop for the remaining 50%

if (shortCondition)
    // If we are currently long, close the position first (reversal)
    strategy.close("Long", comment="Reversal to Short")
    // Enter new short position with calculated size and exits
    strategy.entry("Short", strategy.short, qty=positionSizeShort)
    strategy.exit("Exit Short", from_entry="Short", stop=stopLossShort, limit=takeProfitShort, qty_percent=50) // Exit 50% at SL or TP1
    strategy.exit("Trail Short", from_entry="Short", stop=stopLossShort) // Protective stop for the remaining 50%

// --- TIME-BASED EXIT LOGIC (Example for EOD) ---
isLastBarOfDay = (dayofweek != dayofweek[1])
if isLastBarOfDay
    strategy.close_all(comment="EOD Close")

// Note: A more sophisticated trailing stop (Chandelier Exit) would require manual management
// of the stop level in a 'var float' and using strategy.exit with a dynamically updated 'stop' value.
// The dual strategy.exit calls provide a simpler, yet effective, scale-out and trail mechanism.
```
    