
# Indicators to Strategy Blueprint

Here is the architectural breakdown for transforming the "Algo Trend System TG" indicator into a production-ready automated execution framework.

---

The provided script is an excellent signal generation `indicator`. It successfully identifies potential trade setups with clear visual aids and alert functionality. However, to transition this into an automated `strategy`, we must replace its internal trade-tracking simulation with a robust execution engine that accounts for real-world market dynamics, risk management, and order placement.

### 1. Execution Triggers (Entry & Direction)

The core entry logic is sound, combining trend (Supertrend) with momentum/regime (EMA Cloud). We will formalize this for the strategy engine.

*   **Precise Boolean Conditions:**
    *   **Long Entry:** A trade is triggered only when a Supertrend "flip to buy" occurs *while* the price is already in a bullish regime (Fast EMA > Slow EMA) and within the allowed trading session.
        ```pine
        // Core Logic
        [supertrend, direction] = ta.supertrend(factor, atrPeriod)
        isUpTrend = ta.ema(close, cloudFast) > ta.ema(close, cloudSlow)
        superTrendFlipsUp = ta.crossunder(direction, 0)

        // Final Entry Condition
        longCondition = superTrendFlipsUp and isUpTrend and inAllowedTime
        ```
    *   **Short Entry:** A trade is triggered only when a Supertrend "flip to sell" occurs *while* the price is already in a bearish regime (Fast EMA < Slow EMA) and within the allowed trading session.
        ```pine
        // Core Logic
        isDownTrend = ta.ema(close, cloudFast) < ta.ema(close, cloudSlow)
        superTrendFlipsDown = ta.crossover(direction, 0)

        // Final Entry Condition
        shortCondition = superTrendFlipsDown and isDownTrend and inAllowedTime
        ```

*   **Execution Nuances (Close vs. Real-time):**
    *   The original script implicitly calculates on the close of each bar. For a production strategy, this is the most reliable approach. It ensures signals are stable, non-repainting, and consistent between backtesting and live execution.
    *   **Decision:** We will execute orders on the close of the signal bar. This is configured in the `strategy()` call with `process_orders_on_close=true`. Attempting to execute on real-time price action (`calc_on_every_tick=true`) with this logic would lead to numerous false signals as the Supertrend and EMAs fluctuate intra-bar.

*   **Signal Reversals:**
    *   The current logic prevents new entries in the same direction but doesn't explicitly handle a reversal (e.g., being in a long trade when a valid short signal appears).
    *   **Professional Approach:** A reversal signal should first **close the existing position** and then enter the new one. This ensures risk is properly managed and prevents the strategy from being simultaneously long and short. We will implement a `strategy.close()` command that triggers on a counter-signal before the new `strategy.entry()` is called.

### 2. Multi-Tiered Exit Logic

Fixed dollar-distance targets are brittle and fail to adapt to market volatility. We will replace them with a dynamic, multi-layered exit framework.

*   **Initial Stop Loss (Volatility-Based):**
    *   Instead of a fixed `$20`, the stop loss will be calculated using the Average True Range (ATR). This ensures the stop is wider during volatile periods and tighter in calm markets, giving the trade an appropriate amount of "breathing room."
    *   **Calculation:** `Stop Loss = Entry Price - (ATR * Multiplier)`. A multiplier of 2 or 3 is a common starting point.
    *   *Example:* For a long trade, `longStopPrice = close - (ta.atr(14) * 2.5)`.

*   **Take Profit / Trailing Mechanism:**
    *   **Multi-Stage Take Profit:** The concept of TP1, TP2, and TP3 is excellent for scaling out. We will define these based on multiples of the initial risk (Risk/Reward ratio) or ATR. For instance, TP1 could be at 1.5x the initial ATR stop distance.
    *   **Dynamic Trailing Stop:** Once a profit target (e.g., TP1) is hit, we should protect profits aggressively. The most logical choice here is to use the **Supertrend line itself as a dynamic trailing stop**. Once TP1 is breached, the strategy will move the stop loss to the current value of the Supertrend line, locking in gains while still allowing the trade to capture further trend moves.

*   **Time-Based Exits:**
    *   **End of Session/Day:** A critical component for many markets (especially futures and equities) is to avoid holding positions overnight or through session closes. We will add a rule to automatically close any open position a few minutes before the end of the defined trading session.
    *   **Stagnation Exit:** A trade that goes nowhere ties up capital. We will implement a "Max Bars in Trade" exit. If a position has been open for $X$ bars without hitting a SL or TP target, it will be closed. This frees up capital for higher-probability setups.

### 3. Capital Allocation & Risk Management

This is the most critical transformation from an indicator to a professional trading system. We will move from abstract signals to concrete position sizes based on account equity and risk.

*   **Risk-Based Sizing:**
    *   This is non-negotiable for a production system. The strategy will risk a fixed percentage of account equity on every single trade.
    *   **Logic:**
        1.  Define Risk per Trade (e.g., 1% of equity). `riskAmount = strategy.equity * 0.01`.
        2.  Calculate the dollar risk based on the volatility-based stop loss: `dollarRiskPerUnit = abs(entryPrice - stopLossPrice)`.
        3.  Determine Position Size: `positionSize = riskAmount / dollarRiskPerUnit`.
    *   This ensures that a losing trade always results in a predictable, manageable loss relative to the account size, regardless of asset price or volatility.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** Our multi-tier TP logic is a form of scaling out. We will use multiple `strategy.exit()` orders with different `qty_percent` values to sell portions of the position at predefined profit levels (e.g., sell 50% at TP1, sell the remaining 50% when the trailing stop is hit).
    *   **Pyramiding (Scaling In):** The current logic is designed for one entry per trend move. Therefore, we will disable pyramiding (`pyramiding = 0`) to maintain the integrity of the original signal logic. Adding to winning positions should only be considered after this baseline strategy has been proven robust and profitable.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion from an `indicator` to a `strategy`, incorporating the professional-grade components discussed above.

```pine
//@version=5
// STRATEGY CONVERSION: From signal indicator to execution engine
strategy("Pro Algo Trend System", 
     overlay=true, 
     pyramiding=0, // Only one entry per direction
     process_orders_on_close=true, // Execute on bar close for stability
     default_qty_type=strategy.calculated, // Use our custom risk-based sizing
     commission_value=0.04, // Example commission per trade (in percent)
     slippage=2) // Example slippage in ticks

// --- 1. PROFESSIONAL INPUTS ---
// Core Trend Inputs (from original script)
atrPeriod = input.int(10, "ATR Period for Trend")
factor = input.float(3.0, "Trend Sensitivity Factor")
cloudFast = input.int(50, "Cloud Fast EMA")
cloudSlow = a= input.int(150, "Cloud Slow EMA")

// Risk Management Inputs
riskPercent = input.float(1.0, "Risk per Trade (%)", minval=0.1, maxval=10)
slAtrMultiplier = input.float(2.5, "Stop Loss ATR Multiplier")
tp1Rr = input.float(1.5, "Take Profit 1 R:R")
useTrailingStop = input.bool(true, "Use Supertrend as Trailing Stop after TP1?")
session = input.session("0930-1555", "Trading Session")
timeZone = input.string("America/New_York", "Timezone")

// --- 2. CORE LOGIC & CONDITIONS ---
[supertrend, direction] = ta.supertrend(factor, atrPeriod)
atrValue = ta.atr(14) // Use a standard 14-period ATR for risk calculation

isUpTrend = ta.ema(close, cloudFast) > ta.ema(close, cloudSlow)
isDownTrend = ta.ema(close, cloudFast) < ta.ema(close, cloudSlow)
inAllowedTime = not na(time(timeframe.period, session, timeZone))

longCondition = ta.crossunder(direction, 0) and isUpTrend and inAllowedTime
shortCondition = ta.crossover(direction, 0) and isDownTrend and inAllowedTime

// --- 3. DYNAMIC RISK & POSITION SIZING ---
// Calculate stop loss price BEFORE entry to determine size
longStopPrice = close - (atrValue * slAtrMultiplier)
shortStopPrice = close + (atrValue * slAtrMultiplier)

// Calculate position size based on risk
riskAmount = strategy.equity * (riskPercent / 100)
longPositionSize = riskAmount / (close - longStopPrice)
shortPositionSize = riskAmount / (shortStopPrice - close)

// --- 4. EXECUTION ENGINE ---
// Entry Logic
if (longCondition)
    strategy.close("Short", comment="Reversal") // Close any existing short
    strategy.entry("Long", strategy.long, qty=longPositionSize)

if (shortCondition)
    strategy.close("Long", comment="Reversal") // Close any existing long
    strategy.entry("Short", strategy.short, qty=shortPositionSize)

// Exit Logic (Multi-Tiered)
if (strategy.position_size > 0) // We are in a long position
    // Define TP based on initial risk
    initialRisk = close - strategy.position_avg_price
    tp1Price = strategy.position_avg_price + (initialRisk * tp1Rr)
    
    // Trailing Stop Logic
    trailingStopPrice = supertrend
    useTrail = useTrailingStop and high > tp1Price // Activate trail only after TP1 is feasible
    
    // Exit 50% at TP1 or initial SL
    strategy.exit("Long TP1/SL", from_entry="Long", qty_percent=50, loss=strategy.position_avg_price - initialRisk, profit=tp1Price)
    
    // Exit remaining 50% on trailing stop or initial SL
    strategy.exit("Long Trail", from_entry="Long", qty_percent=100, loss=useTrail ? trailingStopPrice : strategy.position_avg_price - initialRisk)

if (strategy.position_size < 0) // We are in a short position
    // Define TP based on initial risk
    initialRisk = strategy.position_avg_price - close
    tp1Price = strategy.position_avg_price - (initialRisk * tp1Rr)

    // Trailing Stop Logic
    trailingStopPrice = supertrend
    useTrail = useTrailingStop and low < tp1Price // Activate trail only after TP1 is feasible

    // Exit 50% at TP1 or initial SL
    strategy.exit("Short TP1/SL", from_entry="Short", qty_percent=50, loss=strategy.position_avg_price + initialRisk, profit=tp1Price)

    // Exit remaining 50% on trailing stop or initial SL
    strategy.exit("Short Trail", from_entry="Short", qty_percent=100, loss=useTrail ? trailingStopPrice : strategy.position_avg_price + initialRisk)

// Time-based Exit: Close all positions 5 minutes before session end
isEod = (hour(time_close) == 15 and minute(time_close) >= 55)
if (isEod)
    strategy.close_all(comment="End of Session Close")

```
    