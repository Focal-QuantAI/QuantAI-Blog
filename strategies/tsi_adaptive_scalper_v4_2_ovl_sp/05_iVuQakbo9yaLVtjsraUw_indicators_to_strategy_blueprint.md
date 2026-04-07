
# Indicators to Strategy Blueprint

This transformation reframes the "TSI Adaptive Scalper v4.2" from a sophisticated signal generator into a fully-fledged, automated execution framework. The original script's strength lies in its multi-factor confluence score; our strategy will leverage this score as the primary decision-making input, moving beyond simple crossover events.

### 1. Execution Triggers (Entry & Direction)

The core of the entry logic will pivot from observing signals to acting upon a quantifiable edge. The script's `adjScoreBull` and `adjScoreBear` provide the perfect mechanism for this.

*   **Entry Condition:** A trade is triggered only when a signal's confluence score surpasses a user-defined threshold. This ensures we only commit capital to the highest-probability setups identified by the indicator's extensive filtering.
    *   **Long Entry:** `bullSignal and (adjScoreBull >= entryScoreThreshold)`
    *   **Short Entry:** `bearSignal and (adjScoreBear >= entryScoreThreshold)`
    *   `entryScoreThreshold` will be a new input parameter for the strategy, allowing the user to tune the system's aggression (e.g., a value of 6.0 would only take "Strong" or better signals).

*   **Execution Nuance:** All calculations in the source script are based on closed-bar data. Therefore, our strategy will execute **at the open of the next bar** following a valid signal. This is a critical real-world constraint (`process_orders_on_close=true` or simply executing on the next bar's open). Attempting to execute "on the close" of the signal bar is a common backtesting fallacy that is impossible to replicate in live trading. Our orders will be market orders sent at the next available price.

*   **Signal Reversals:** The system must handle a change of bias gracefully. If a position is currently open (e.g., Long) and a valid, high-scoring Short signal appears, the framework will execute the following sequence:
    1.  An order is sent to `strategy.close("LONG_ID")` to flatten the existing long position.
    2.  Simultaneously, a new order is sent via `strategy.entry("SHORT_ID", strategy.short)` to initiate the new short position.
    This "close-and-reverse" logic ensures the strategy is always aligned with the latest high-probability signal, preventing it from being locked into a trade when the market has clearly turned. The script's built-in `cdDynamic` (cooldown) naturally mitigates excessive flipping.

### 2. Multi-Tiered Exit Logic

A professional exit strategy is not a single point of failure. It's a dynamic system designed to protect capital and maximize profit.

*   **Initial Stop Loss (Volatility-Based):** We will discard arbitrary percentage-based stops. The initial stop loss will be calculated dynamically at the time of entry, using the script's own ATR-based logic.
    *   **Calculation:** `Stop Loss Price = Entry Price - (tpslATR * slATRMult * scoreAdjustment)` for a long trade.
    *   The `slATRMult` and the brilliant `useScoreAdjSL` (score-adjusted SL) features from the original script will be directly integrated. A higher score leads to a tighter, more confident stop, while a lower-score entry gets more breathing room. This is a hallmark of a professional system.

*   **Take Profit / Trailing Mechanism:** We will implement a two-stage take-profit plan combined with a trailing stop to lock in gains while allowing winners to run.
    1.  **TP1 (Scaling Out):** Upon reaching the `tp1RR` level (e.g., 1.5R), the strategy will exit 50% of the position.
    2.  **Move to Breakeven:** Immediately after the TP1 partial exit, the stop loss for the remaining 50% of the position is moved to the entry price. This makes the remainder of the trade "risk-free."
    3.  **Trailing Stop Activation:** After TP1 is hit and the stop is at breakeven, a trailing stop is activated for the remaining position. A logical choice would be a trailing stop based on `high - (N * ATR)` for a long trade, where `N` is a configurable multiplier (e.g., 2.0). This allows the trade to capture a larger trend if one develops, moving beyond the fixed `tp2RR` target.

*   **Time-Based Exits:** Capital is a resource; it cannot be tied up indefinitely in stagnant trades.
    *   **End of Day (EOD) Squaring:** For a scalping/intraday strategy, holding positions overnight introduces unnecessary gap risk. A mandatory `strategy.close_all()` function will be triggered at a specified time (e.g., 15 minutes before market close) to flatten all open positions.
    *   **Trade Stagnation Exit:** An input `maxBarsInTrade` will be introduced. If a trade has been open for more than this number of bars and has not hit either TP1 or its stop loss, it will be closed automatically. This frees up capital for new opportunities.

### 3. Capital Allocation & Risk Management

Chart signals are meaningless without a mathematical framework for managing risk.

*   **Risk-Based Sizing:** Every single trade will risk a fixed percentage of the total account equity. This is non-negotiable for long-term survival.
    *   **Inputs:** `strategy.equity` (current account value) and a user-defined `riskPerTradePercent` (e.g., 1.0 for 1% risk per trade).
    *   **Logic:**
        1.  On a valid entry signal, calculate the risk distance in currency: `riskDistance = abs(entryPrice - stopLossPrice)`.
        2.  Calculate the total dollar amount to risk: `riskAmount = strategy.equity * (riskPerTradePercent / 100)`.
        3.  Calculate the precise position size: `positionSize = riskAmount / riskDistance`.
    *   This calculation ensures that whether the stop is wide (low volatility) or tight (high volatility), the maximum potential loss on any given trade is capped at the desired percentage of the account.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** This is our default behavior, as defined in the multi-tiered exit logic (exiting 50% at TP1).
    *   **Pyramiding (Scaling In):** This is an advanced and high-risk technique. Rules must be strict. We will allow for one additional entry (pyramiding) under the following conditions only:
        1.  The initial trade is open and has already moved to a risk-free status (i.e., TP1 has been hit, and the stop is at breakeven).
        2.  A *new* signal in the *same direction* occurs with a score greater than or equal to the `entryScoreThreshold`.
        3.  The new position size will be calculated based on the same risk percentage, but applied to the new entry/stop distance. This prevents adding to a losing position and only reinforces demonstrated momentum.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the indicator's logic into a `strategy` engine, focusing on the new execution components.

```pine
// -------------------------------------------------------------------------- //
// |                 TSI Adaptive Scalper - STRATEGY ENGINE                 | //
// -------------------------------------------------------------------------- //
//@version=5
// 1. STRATEGY DECLARATION
strategy("TSI Scalper [Strategy Engine]", 
     overlay=true, 
     initial_capital=10000, 
     commission_type=strategy.commission.percent, 
     commission_value=0.075, 
     slippage=2, // Slippage in ticks
     process_orders_on_close=true) // Ensures execution on the next bar's open

// --- [PASTE THE ENTIRE ORIGINAL INDICATOR CODE HERE, FROM LINE 10 to ~1000] ---
// ... all the functions, inputs, and calculations from the original script ...

// -------------------------------------------------------------------------- //
// |                      2. STRATEGY-SPECIFIC INPUTS                         | //
// -------------------------------------------------------------------------- //
string g_strat = "⚙️ Strategy Engine"
entryScoreThreshold = input.float(6.0, "Min. Entry Score (0-10)", group=g_strat, minval=0, maxval=10, step=0.5)
riskPerTradePercent = input.float(1.0, "Risk per Trade (%)", group=g_strat, minval=0.1, maxval=5.0, step=0.1)
useTrailingStop   = input.bool(true, "Use Trailing Stop After TP1?", group=g_strat)
trailingAtrMult   = input.float(2.0, "Trailing Stop ATR Multiplier", group=g_strat)
eodHour           = input.int(20, "EOD Close Hour (UTC)", group=g_strat, minval=0, maxval=23)
eodMinute         = input.int(45, "EOD Close Minute (UTC)", group=g_strat, minval=0, maxval=59)

// -------------------------------------------------------------------------- //
// |                         3. EXECUTION LOGIC                               | //
// -------------------------------------------------------------------------- //

// --- Entry Conditions ---
bool longCondition = bullSignal and (adjScoreBull >= entryScoreThreshold) and strategy.opentrades == 0
bool shortCondition = bearSignal and (adjScoreBear >= entryScoreThreshold) and strategy.opentrades == 0

// --- Time-based Exit ---
bool isEod = (hour(time_close, "UTC") == eodHour and minute(time_close, "UTC") >= eodMinute)

if (isEod)
    strategy.close_all(comment="EOD Close")

// --- Position Sizing & Order Placement ---
if (longCondition)
    // 1. Calculate Stops & Targets
    sl_adj   = calcSLMultiplier(adjScoreBull)
    sl_dist  = tpslATR * slATRMult * sl_adj
    sl_price = close - sl_dist
    tp1_price = close + sl_dist * tp1RR
    
    // 2. Calculate Position Size based on Risk
    riskDistancePoints = close - sl_price
    riskAmountEquity = (strategy.equity * riskPerTradePercent) / 100
    positionSize = riskAmountEquity / (riskDistancePoints * syminfo.pointvalue)

    // 3. Execute Orders
    strategy.entry("Long", strategy.long, qty=positionSize, comment="L Entry @" + str.tostring(adjScoreBull, "#.#"))
    strategy.exit("TP1/SL", from_entry="Long", qty_percent=50, profit=tp1_price, stop=sl_price, comment="L TP1/SL")
    // The remaining 50% is managed by the trailing stop logic below

if (shortCondition)
    // 1. Calculate Stops & Targets
    sl_adj   = calcSLMultiplier(adjScoreBear)
    sl_dist  = tpslATR * slATRMult * sl_adj
    sl_price = close + sl_dist
    tp1_price = close - sl_dist * tp1RR

    // 2. Calculate Position Size based on Risk
    riskDistancePoints = sl_price - close
    riskAmountEquity = (strategy.equity * riskPerTradePercent) / 100
    positionSize = riskAmountEquity / (riskDistancePoints * syminfo.pointvalue)

    // 3. Execute Orders
    strategy.entry("Short", strategy.short, qty=positionSize, comment="S Entry @" + str.tostring(adjScoreBear, "#.#"))
    strategy.exit("TP1/SL", from_entry="Short", qty_percent=50, profit=tp1_price, stop=sl_price, comment="S TP1/SL")

// --- Trailing Stop & Breakeven Management ---
var float trailingStopPrice = na

if (strategy.position_size > 0) // We are in a long position
    // Check if TP1 was hit (by checking closed trades) and set stop to breakeven
    if (strategy.closedtrades.exit_comment(strategy.closedtrades-1) == "L TP1/SL")
        strategy.exit("BE/Trail", from_entry="Long", stop=strategy.opentrades.entry_price(0), comment="L B/E")
    
    // If trailing is enabled, manage the trailing stop
    if (useTrailingStop and strategy.opentrades.entry_price(0) < close) // Only trail if in profit
        newTrailingStop = close - (ta.atr(14) * trailingAtrMult)
        trailingStopPrice := na(trailingStopPrice) ? newTrailingStop : math.max(newTrailingStop, trailingStopPrice)
        strategy.exit("BE/Trail", from_entry="Long", stop=trailingStopPrice)

if (strategy.position_size < 0) // We are in a short position
    if (strategy.closedtrades.exit_comment(strategy.closedtrades-1) == "S TP1/SL")
        strategy.exit("BE/Trail", from_entry="Short", stop=strategy.opentrades.entry_price(0), comment="S B/E")

    if (useTrailingStop and strategy.opentrades.entry_price(0) > close)
        newTrailingStop = close + (ta.atr(14) * trailingAtrMult)
        trailingStopPrice := na(trailingStopPrice) ? newTrailingStop : math.min(newTrailingStop, trailingStopPrice)
        strategy.exit("BE/Trail", from_entry="Short", stop=trailingStopPrice)

// Reset trailing stop price on new bar if not in a trade
if (strategy.position_size == 0)
    trailingStopPrice := na
```
    