
# Indicators to Strategy Blueprint

Based on the provided "Sigmoid Transition Trailing Stop" indicator, here is the architectural blueprint for transforming it into a production-grade automated execution framework.

### 1. Execution Triggers (Entry & Direction)

The core of the provided script is a trend-following system where the `direction` variable dictates the market state. An entry signal is generated at the exact moment this state flips.

*   **Long Entry Condition:** A long position is initiated when the system flips from a bearish state to a bullish state.
    *   `boolean condition`: `direction == 1 and direction[1] == -1`

*   **Short Entry Condition:** A short position is initiated when the system flips from a bullish state to a bearish state.
    *   `boolean condition`: `direction == -1 and direction[1] == 1`

*   **Execution Nuance (Close vs. Real-time):**
    The script's logic `if close < trailingStop` or `if close > trailingStop` is fundamentally based on the **closing price of a completed bar**. Attempting to execute this intra-bar would lead to significant repainting and unreliable signals, as `close` would be the real-time price, causing the `direction` to flicker back and forth.

    **Execution Mandate:** The strategy must wait for a bar to close to confirm a signal. The entry order should then be placed at the **open of the next bar**. In Pine Script, this is achieved by setting `process_orders_on_close = true` within the `strategy()` declaration. This ensures backtest results are realistic and non-repainting.

*   **Signal Reversals:**
    This system is inherently a **reversal strategy**. A signal to exit a long is simultaneously a signal to enter a short, and vice-versa. The execution logic must handle this cleanly. Instead of simply flipping, the robust approach is to:
    1.  Explicitly close the existing position using `strategy.close()`.
    2.  Immediately open the new position in the opposite direction using `strategy.entry()`.
    This two-step process provides clearer logging and more precise control over risk for each distinct trade.

### 2. Multi-Tiered Exit Logic

The provided script is already a sophisticated exit mechanism. We will build upon it by formalizing its role and adding layers for robustness.

*   **Initial Stop Loss & Trailing Mechanism:**
    The `trailingStop` variable calculated by the script serves as the **primary and dynamic stop loss**.
    *   **Initial Placement:** Upon entry, the stop loss is the value of `trailingStop` on the entry bar. This is already volatility-adjusted, using the `atrMultInput`.
    *   **Dynamic Trailing:** The script's "Sigmoid Transition" is an advanced trailing stop. When the price moves strongly in the trade's favor, the stop loss automatically accelerates towards the price, locking in profits more aggressively. This behavior is the core exit logic and should be implemented using a `strategy.exit()` call that updates its `stop` parameter on every bar.

*   **Take Profit/Scaling Out (Enhancement):**
    While the trailing stop will capture the bulk of a trend, adding discrete take-profit targets can improve the strategy's consistency by realizing profits along the way.
    *   **Multi-Stage Targets:** Define take-profit levels based on multiples of the initial risk or ATR at the time of entry.
        *   **TP1:** `Entry Price + (N * ATR at entry)` for longs. Close 33% of the position.
        *   **TP2:** `Entry Price + (M * ATR at entry)` for longs. Close another 33% of the position.
        *   The remaining position is left to be stopped out by the main `trailingStop`.
    *   This creates a "scaling out" mechanism that secures gains while still allowing a portion of the trade to ride a powerful trend to its conclusion.

*   **Time-Based Exits:**
    A professional strategy cannot leave capital exposed indefinitely.
    *   **End of Session/Day:** For instruments with distinct trading sessions (e.g., equities, futures), all open positions must be squared before the session close to avoid overnight risk and funding charges. This is a non-negotiable rule for most day-trading systems.
        *   `Logic:` Check if `time` is approaching the session close and execute `strategy.close_all()`.
    *   **Trade Stagnation:** If a trade is open for more than `X` bars (e.g., 50 bars) and has not reached a profitable state (e.g., price is still below `entry price + 0.5 * ATR`), exit the position. This frees up capital from non-performing trades.

### 3. Capital Allocation & Risk Management

Moving from a "signal generator" to a "trading business" requires strict mathematical rules for position sizing. We will discard fixed-lot sizing in favor of a dynamic risk-based model.

*   **Risk-Based Sizing:**
    The cornerstone of professional risk management. The strategy will risk a fixed percentage of total account equity on every single trade.
    *   **Inputs:**
        *   `accountEquity = strategy.equity`
        *   `riskPerTradePercent = 1.5` (e.g., 1.5%)
    *   **Calculation:**
        1.  **Risk Amount ($):** `riskAmount = accountEquity * (riskPerTradePercent / 100)`
        2.  **Stop Distance (Per Share/Contract):** `stopDistance = abs(entryPrice - initialStopLoss)`
        3.  **Position Size:** `positionSize = riskAmount / stopDistance`
    *   This calculation must be performed *immediately before* each `strategy.entry()` call to ensure the size is tailored to the specific conditions of that trade.

*   **Pyramiding & Scaling:**
    The current logic is "all-in or all-out." A more advanced implementation could add to winning positions.
    *   **Pyramiding Rule:** Consider adding to a position *only* after a "Sigmoid Transition" has successfully completed, confirming a strong continuation of the trend.
    *   **Pyramiding Sizing:** The size of the additional entry should also be risk-managed. A common technique is to add a new position while moving the stop loss for the *entire* combined position to the break-even price of the most recent entry. This prevents a winning trade from turning into a loser after pyramiding.
    *   **Scaling Out Logic:** As defined in the "Take Profit" section, `strategy.exit()` can be used with a `qty_percent` parameter to close portions of the trade at different targets.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the indicator into a fully-featured `strategy` incorporating the principles above.

```pine
//@version=5
// 1. STRATEGY DECLARATION: Professional settings for execution reality
strategy("Sigmoid TS Execution Framework", 
     overlay=true, 
     process_orders_on_close=true, // CRITICAL: Execute on next bar's open after signal on close
     slippage=2, // Example: 2 ticks of slippage per order
     commission_type=strategy.commission.percent, 
     commission_value=0.075) // Example: 0.075% commission

// == INPUTS ==
// Original Inputs
atrLengthInput   = input.int(200, "ATR Length")
atrMultInput     = input.float(3.0, "ATR Multiplier")
sigLengthInput   = input.int(20, "Sigmoid Length (Bars)")
sigAmpMultInput  = input.float(3.0, "Sigmoid Amplitude (ATR Units)")
minDistMultInput = input.float(0.5, "Min Distance (ATR Units)")
// Risk Management Inputs
riskPercent      = input.float(1.0, "Risk Per Trade %", minval=0.1, maxval=100)
useEodExit       = input.bool(true, "Use End-of-Day Exit?")
eodHour          = input.int(15, "EOD Exit Hour (24h)")
eodMinute        = input.int(45, "EOD Exit Minute")

// == CORE LOGIC (Ported from indicator) ==
// [The entire logic block from the original script for calculating 'direction' and 'trailingStop' goes here]
// ... (clamp, sigmoid, atr, state tracking, flip logic, adjustment logic) ...
// For brevity, this part is omitted but should be copied verbatim.

// == EXECUTION TRIGGERS ==
longEntryCondition = direction == 1 and direction[1] == -1
shortEntryCondition = direction == -1 and direction[1] == 1

// == RISK MANAGEMENT CALCULATION ==
riskAmount = (riskPercent / 100) * strategy.equity
stopDistance = atrMultInput * ta.atr(atrLengthInput) // The initial distance to the stop
positionSize = riskAmount / (stopDistance * syminfo.pointvalue)

// == STRATEGY EXECUTION ENGINE ==
// 1. ENTRY LOGIC
if (longEntryCondition)
    // Close any existing short position before entering long (reversal)
    strategy.close("Short", comment="Short Reversal")
    // Enter the new long position with calculated risk
    strategy.entry("Long", strategy.long, qty=positionSize, comment="Enter Long")

if (shortEntryCondition)
    // Close any existing long position before entering short (reversal)
    strategy.close("Long", comment="Long Reversal")
    // Enter the new short position with calculated risk
    strategy.entry("Short", strategy.short, qty=positionSize, comment="Enter Short")

// 2. EXIT LOGIC
// The primary exit is the trailing stop. This must be active on every bar for open trades.
if strategy.position_size > 0
    strategy.exit("Exit Long", from_entry="Long", stop=trailingStop)
if strategy.position_size < 0
    strategy.exit("Exit Short", from_entry="Short", stop=trailingStop)

// 3. TIME-BASED EXIT
isEod = useEodExit and (hour == eodHour) and (minute >= eodMinute)
if isEod
    strategy.close_all(comment="EOD Exit")

// == VISUALS (For chart analysis) ==
// [The plotting logic from the original script can be kept here for visual verification]
// ... (plot, fill, etc.) ...
```
    