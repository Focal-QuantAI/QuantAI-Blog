
    # Indicators to Strategy Blueprint

    This `indicator` is a sophisticated piece of real-time analysis, leveraging lower timeframe data to build a high-confidence picture of market dynamics within a single candle. However, its reliance on `lookahead` and `security_lower_tf` makes it a classic "chart painter" that is notoriously difficult to automate. To transform this into a professional execution framework, we must shift focus from the beautiful, repainting dashboard to a set of discrete, non-repainting, and actionable rules.

The core philosophy is to **wait for confirmation**. The script's own `timePressure` and `_nxtActive` variables are the key. They acknowledge that the rich intra-bar data is only reliable in the final moments of a candle's life. Our strategy will formalize this concept.

---

### 1. Execution Triggers (Entry & Direction)

The provided script offers a multitude of signals. For a production strategy, we must distill these into a single, non-conflicting trigger. We will build a primary trigger based on the script's main signal (`final_buy`/`final_sell`) and filter it with its confidence score (`confPct`).

*   **Execution Timing:** All decisions will be made at the **close of the bar**. The script's intra-bar calculations (`lookahead=barmerge.lookahead_on`) are useful for real-time monitoring, but executing on them is a recipe for chasing phantom signals. We will use the state of the indicator on the bar's final tick (i.e., its confirmed, historical value) to make a decision for the *next* bar's open. This is simulated in a backtest by checking the condition on the previous bar (`condition[1]`).

*   **Long Entry Condition (`enterLong`):**
    A confluence of three factors, confirmed at bar close:
    1.  **Primary Signal:** The main `final_buy` signal must be true.
    2.  **Confidence Filter:** The `confPct` must exceed a minimum threshold (e.g., 55%). This filters out weak signals.
    3.  **Signal Stability:** The `timePressure` variable must be true. This ensures the signal is generated late in the candle's formation, making it less likely to have been a temporary flicker.

    ```plaintext
    // Boolean Logic for Long Entry
    minConfidence = input.int(55, "Minimum Confidence % for Entry")
    enterLong = final_buy[1] and confPct[1] >= minConfidence and timePressure[1]
    ```

*   **Short Entry Condition (`enterShort`):**
    The logic is symmetrical to the long entry.
    ```plaintext
    // Boolean Logic for Short Entry
    enterShort = final_sell[1] and confPct[1] >= minConfidence and timePressure[1]
    ```

*   **Signal Reversals:** The framework will treat a reversal as a distinct "close" and "entry" event. If we are in a long position and an `enterShort` signal is generated, the system will:
    1.  Generate a `strategy.close("Long")` order.
    2.  Generate a new `strategy.entry("Short")` order.
    This ensures risk for the new position is calculated correctly based on the new entry price and stop loss, rather than just flipping the position size.

### 2. Multi-Tiered Exit Logic

A static stop-loss or take-profit is insufficient. The exit logic must be as dynamic as the market conditions the indicator is designed to read.

*   **Initial Stop Loss (Volatility-Based):**
    The stop loss will be placed based on the Average True Range (ATR) of the execution timeframe. This adapts the risk distance to current market volatility. A secondary, more advanced option is to use the `rollingPOC` calculated by the script, placing the stop on the other side of this high-volume node.

    *   **Calculation:** `Stop Price (Long) = Entry Price - (ATR(14) * 2)`
    *   **Calculation:** `Stop Price (Short) = Entry Price + (ATR(14) * 2)`
    The ATR multiplier (here, `2`) should be an adjustable input.

*   **Take Profit / Trailing Mechanism:**
    We will implement a two-stage exit strategy to lock in gains while allowing for runners.
    1.  **Initial Profit Target (TP1):** A first take-profit target is set at a multiple of the initial risk (R). For example, at 1.5R (i.e., profit is 1.5 times the distance from entry to stop loss). At this point, 50% of the position is closed.
    2.  **Breakeven Stop:** Upon hitting TP1, the stop loss for the remaining 50% of the position is moved to the entry price. The trade is now risk-free.
    3.  **Trailing Stop:** The remaining position is managed with an ATR-based trailing stop. The stop will trail the price by a multiple of ATR (e.g., `ATR(14) * 2.5`), only moving in the direction of the trade. This allows the winning trade to continue as long as momentum persists.

*   **Time-Based Exits:**
    1.  **End of Session:** A non-negotiable rule for day trading strategies. All open positions will be squared off 15 minutes before the session close (e.g., at 23:45 if trading a 24h market like crypto) to avoid overnight risk and funding fees.
    2.  **Stagnation Exit:** If a trade has been open for $X$ bars (e.g., 25 bars on a 5-minute chart) and has not hit TP1, it will be closed. This prevents capital from being tied up in trades that have lost momentum.

### 3. Capital Allocation & Risk Management

Position sizing is not an afterthought; it is the core of risk management. We will use a fixed-fractional model.

*   **Risk-Based Sizing:**
    The strategy will risk a fixed percentage of account equity on every single trade. For example, 1% of the total account value. The position size is derived from this risk parameter and the stop-loss distance.

    *   **Logic:**
        1.  Define `riskPercent` (e.g., 1.0 for 1%).
        2.  On an entry signal, calculate the `stopLossDistance` in price points (e.g., `entryPrice - stopLossPrice`).
        3.  Calculate `riskAmount` = `strategy.equity * (riskPercent / 100)`.
        4.  Calculate `positionSize` = `riskAmount / stopLossDistance`.

*   **Pyramiding & Scaling:**
    The script's `strengthLabel` ("Medium", "Strong", "Very Strong") and `confPct` are perfect for this.
    *   **Scaling In (Pyramiding):** An initial position can be taken on a "Strong" signal (`confPct > 55`). If, while in the trade, a subsequent bar generates another signal that is upgraded to "Very Strong" (`confPct > 75`) and price has moved in our favor, a second, smaller unit (e.g., 0.5% risk) can be added. The overall stop for the combined position would be moved up to maintain the new blended cost basis.
    *   **Scaling Out:** This is handled by our multi-tiered exit logic (closing 50% at TP1).

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the indicator's logic into a professional `strategy` framework. It integrates the triggers, exits, and risk management defined above.

```pine
// @version=5
// NOTE: This is a strategy wrapper. The original indicator code must be pasted above this block.
strategy("BTC Delta MTF v3 [Execution Engine]", 
     overlay=true, 
     process_orders_on_close=true, // Decide on close, execute on next bar's open
     slippage=2, // Realistic slippage in ticks for limit/market orders
     commission_type=strategy.commission.percent, 
     commission_value=0.075) // Realistic exchange commission

// === STRATEGY INPUTS ===
grpStrat = "Strategy Settings"
riskPercent     = input.float(1.0, "Risk per Trade (%)", group=grpStrat, step=0.25)
minConfidence   = input.int(55, "Minimum Confidence % for Entry", group=grpStrat, minval=40, maxval=90, step=5)
atrPeriod       = input.int(14, "ATR Period", group=grpStrat)
atrSlMultiplier = input.float(2.0, "ATR Stop Loss Multiplier", group=grpStrat, step=0.1)
atrTsMultiplier = input.float(2.5, "ATR Trailing Stop Multiplier", group=grpStrat, step=0.1)
tp1RR           = input.float(1.5, "Take Profit 1 R:R", group=grpStrat, step=0.1)
stagnationBars  = input.int(25, "Exit if Stagnant after X Bars", group=grpStrat)

// === RISK & EXIT CALCULATION ===
atrValue = ta.atr(atrPeriod)

// Entry conditions based on previous bar's confirmed state to avoid repainting issues
bool enterLong  = final_buy[1] and confPct[1] >= minConfidence and timePressure[1] and strategy.position_size == 0
bool enterShort = final_sell[1] and confPct[1] >= minConfidence and timePressure[1] and strategy.position_size == 0

// Time-based exit
isEod = hour(time, "UTC") == 23 and minute(time, "UTC") >= 45
if (isEod)
    strategy.close_all(comment="End of Day Exit")

// === EXECUTION LOGIC ===
if (enterLong)
    // 1. Calculate Stop Loss and Position Size
    entry_price = open // We enter on the open of the new bar
    stop_loss_price = entry_price - (atrValue[1] * atrSlMultiplier)
    risk_per_unit = entry_price - stop_loss_price
    position_size = (strategy.equity * (riskPercent / 100)) / risk_per_unit

    // 2. Calculate Take Profit
    take_profit_1_price = entry_price + (risk_per_unit * tp1RR)

    // 3. Execute Orders
    strategy.entry("Long", strategy.long, qty=position_size, comment="Entry Long")
    // Place an exit order that closes 50% at TP1
    strategy.exit("TP1/SL Long", from_entry="Long", qty_percent=50, limit=take_profit_1_price, stop=stop_loss_price)
    // Place a separate exit order for the remaining 50% with only the stop loss
    strategy.exit("SL2 Long", from_entry="Long", qty_percent=100, stop=stop_loss_price)

if (enterShort)
    // 1. Calculate Stop Loss and Position Size
    entry_price = open
    stop_loss_price = entry_price + (atrValue[1] * atrSlMultiplier)
    risk_per_unit = stop_loss_price - entry_price
    position_size = (strategy.equity * (riskPercent / 100)) / risk_per_unit

    // 2. Calculate Take Profit
    take_profit_1_price = entry_price - (risk_per_unit * tp1RR)

    // 3. Execute Orders
    strategy.entry("Short", strategy.short, qty=position_size, comment="Entry Short")
    strategy.exit("TP1/SL Short", from_entry="Short", qty_percent=50, limit=take_profit_1_price, stop=stop_loss_price)
    strategy.exit("SL2 Short", from_entry="Short", qty_percent=100, stop=stop_loss_price)

// === DYNAMIC TRADE MANAGEMENT ===
// Breakeven and Trailing Stop Logic
var bool isTrailingActive = false
if (strategy.position_size > 0 and strategy.position_avg_price < take_profit_1_price[1] and close >= take_profit_1_price[1])
    strategy.exit("BE/Trail Long", from_entry="Long", stop=strategy.position_avg_price, comment="Move to BE")
    isTrailingActive := true
if (strategy.position_size < 0 and strategy.position_avg_price > take_profit_1_price[1] and close <= take_profit_1_price[1])
    strategy.exit("BE/Trail Short", from_entry="Short", stop=strategy.position_avg_price, comment="Move to BE")
    isTrailingActive := true

// Activate ATR Trailing Stop after TP1 is hit
if (isTrailingActive)
    if (strategy.position_size > 0)
        trail_stop_price = close - (atrValue * atrTsMultiplier)
        strategy.exit("ATR Trail Long", from_entry="Long", stop=trail_stop_price)
    if (strategy.position_size < 0)
        trail_stop_price = close + (atrValue * atrTsMultiplier)
        strategy.exit("ATR Trail Short", from_entry="Short", stop=trail_stop_price)

// Stagnation Exit
if (strategy.position_size != 0 and bar_index - strategy.opentrades.entry_bar_index(0) >= stagnationBars)
    strategy.close_all(comment="Stagnation Exit")

// Reset trailing flag when position is closed
if (strategy.position_size == 0)
    isTrailingActive := false
```
    