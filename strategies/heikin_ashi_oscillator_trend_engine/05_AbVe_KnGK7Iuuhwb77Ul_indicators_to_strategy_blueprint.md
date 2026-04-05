
# Indicators to Strategy Blueprint

Based on the provided "Heikin Ashi Oscillator Trend Engine," we will architect a professional-grade, automated execution framework. The original script is a sophisticated `indicator` that provides a wealth of visual information; our task is to distill this into a set of precise, non-discretionary rules for live trading.

The core signal we will build upon is the **Adaptive MA Crossover** (`fastLine` and `slowLine`), as it is designed to be the primary momentum-shift detector within the oscillator's engine.

### 1. Execution Triggers (Entry & Direction)

The transition from a visual indicator to an execution system requires absolute clarity on what constitutes a trade signal. We will use the adaptive crossover on the oscillator as our primary trigger, refined with a momentum filter to avoid entering at potential exhaustion points.

*   **Long Entry Condition:** A bullish crossover occurs when the `fastLine` crosses above the `slowLine`, AND the core oscillator (`osc`) is not in an over-extended state.
    *   `longCondition = ta.crossover(fastLine, slowLine) and (osc < upperGuide)`

*   **Short Entry Condition:** A bearish crossover occurs when the `fastLine` crosses below the `slowLine`, AND the core oscillator (`osc`) is not in an over-extended state.
    *   `shortCondition = ta.crossunder(fastLine, slowLine) and (osc > lowerGuide)`

*   **Execution Nuances:**
    *   **Execute on Bar Close:** All entry and exit decisions will be made on the close of the bar where the condition becomes true (`process_orders_on_close=true`). This prevents the strategy from firing multiple orders based on intra-bar price fluctuations (whipsaws) and ensures that the logic is consistent with how backtests are calculated, providing more reliable performance metrics.
    *   **Signal Reversals:** The system is designed for "close-and-reverse" logic. If the strategy is in a long position and a `shortCondition` becomes true, the existing long position will be closed, and a new short position will be opened simultaneously. This ensures the strategy is always positioned according to the most recent valid signal from the oscillator engine.

### 2. Multi-Tiered Exit Logic

A robust exit strategy is more critical than a perfect entry. We will move away from simplistic exits and implement a multi-layered system to manage risk and protect profits dynamically.

*   **Initial Stop Loss (Volatility-Based):** The stop loss will be calculated dynamically for each trade based on market volatility, not a fixed percentage. We will use the Average True Range (ATR).
    *   **Calculation:**
        *   For a **Long** trade: `Stop Loss Price = Entry Price - (ATR * ATR Multiplier)`
        *   For a **Short** trade: `Stop Loss Price = Entry Price + (ATR * ATR Multiplier)`
    *   The `ATR Multiplier` (e.g., 2.0) will be a user-configurable parameter, allowing the trader to adjust risk tolerance based on the asset's character.

*   **Take Profit / Trailing Mechanism:** We will employ a two-stage profit-taking and trailing stop mechanism to lock in gains while allowing for larger trends to run.
    *   **Stage 1: Initial Profit Target (TP1):** A primary profit target will be set at a multiple of the initial risk (R). For example, at 1.5R.
        *   `Take Profit Price = Entry Price + 1.5 * (Entry Price - Stop Loss Price)` for a long.
        *   Upon hitting TP1, the strategy will scale out, closing a portion of the position (e.g., 50%).
    *   **Stage 2: Breakeven & Trailing Stop:** Once TP1 is hit, the stop loss on the remaining portion of the position is immediately moved to the entry price (breakeven). Simultaneously, a trailing stop is activated.
        *   **Trailing Logic:** The stop will trail the price using the oscillator's `slowLine`. If in a long position, the trade will be exited if the `osc` value crosses back below the `slowLine`. This dynamically adjusts the exit point based on the underlying momentum structure, rather than a fixed price distance.

*   **Time-Based Exits:**
    *   **End of Day (EOD) Exit:** For intraday timeframes (e.g., below 1H), a mandatory "square-off" rule will be implemented. All open positions will be closed 15 minutes before the end of the trading session to avoid overnight risk and funding charges.
    *   **Stagnation Exit:** If a trade has been open for a specified number of bars (e.g., 50 bars) and has not yet reached TP1, it will be closed. This frees up capital from non-performing trades.

### 3. Capital Allocation & Risk Management

Professional trading is defined by its risk management protocol. We will implement a fixed-fractional position sizing model to ensure consistent risk exposure on every trade.

*   **Risk-Based Sizing:** The strategy will risk a fixed percentage of the total account equity on each trade.
    *   **Inputs:**
        *   `account_equity`: The total value of the trading account.
        *   `risk_per_trade_pct`: The percentage of equity to risk (e.g., 1.0 for 1%).
        *   `stop_loss_distance_currency`: The distance in price from entry to the initial stop loss (`ATR * ATR Multiplier`).
    *   **Position Size Formula:**
        *   `risk_amount_currency = account_equity * (risk_per_trade_pct / 100)`
        *   `position_size = risk_amount_currency / stop_loss_distance_currency`
    *   This calculation is performed *before* each entry, ensuring that a trade with a wider stop (higher volatility) results in a smaller position size, and vice-versa.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, the strategy will automatically sell 50% of the position at TP1. This is a core feature.
    *   **Pyramiding (Scaling In):** Adding to winning positions will be disabled by default to maintain a simple, robust risk profile. Advanced implementation could allow for one additional entry (a "pyramid") under strict conditions:
        1.  The initial trade must be profitable by at least 1R (i.e., the stop is at breakeven).
        2.  The oscillator must pull back to a key level (e.g., the zero-line) and then resume its trend direction.
        3.  The total risk of the initial trade and the pyramid combined must not exceed a master risk limit (e.g., 1.5% of equity).

### 4. Implementation Snippet (Pine Logic)

The following code block demonstrates how to transform the `indicator` into a `strategy` by incorporating the architectural components defined above. This snippet should be integrated into the main body of the original script after all the oscillator calculations are complete.

```pine
// ==============================================================================
// 🟢 REGION: STRATEGY EXECUTION FRAMEWORK
// ==============================================================================

// ──────────────────────────────────────────────
// 🟦 Strategy Declaration & Inputs
// ──────────────────────────────────────────────
strategy("HA Oscillator Execution Engine", 
     overlay=true, // Plot trades on the main chart
     pyramiding=0, // No pyramiding by default
     initial_capital=10000, 
     default_qty_type=strategy.fixed, 
     commission_type=strategy.commission.percent, 
     commission_value=0.075, // Realistic commission (e.g., 0.075%)
     slippage=2, // Realistic slippage in ticks
     process_orders_on_close=true) // Execute on bar close for reliability

// Risk Management Inputs
risk_pct = input.float(1.0, "Risk per Trade %", group="Risk Management", minval=0.1, maxval=5.0)
atr_len = input.int(14, "ATR Length for Stop Loss", group="Risk Management")
atr_mult = input.float(2.0, "ATR Multiplier for Stop Loss", group="Risk Management")
tp1_rr = input.float(1.5, "Take Profit 1 R:R", group="Risk Management")
stagnation_bars = input.int(50, "Max Bars in Trade (Stagnation Exit)", group="Risk Management")

// ──────────────────────────────────────────────
// 🟦 Risk & Position Sizing Calculation
// ──────────────────────────────────────────────
atr_val = ta.atr(atr_len)
stop_loss_distance = atr_val * atr_mult

// Calculate position size based on fixed-fractional risk
risk_amount = (strategy.equity * risk_pct) / 100
position_size = risk_amount / stop_loss_distance

// ──────────────────────────────────────────────
// 🟦 Entry & Exit Conditions
// ──────────────────────────────────────────────
// Entry Conditions (from Section 1)
long_condition = ta.crossover(fastLine, slowLine) and osc < upperGuide
short_condition = ta.crossunder(fastLine, slowLine) and osc > lowerGuide

// Exit Conditions
exit_long_trail = ta.crossunder(osc, slowLine)
exit_short_trail = ta.crossover(osc, slowLine)
stagnation_exit = barssince(strategy.opentrades > 0) > stagnation_bars

// ──────────────────────────────────────────────
// 🟦 Strategy Execution Logic
// ──────────────────────────────────────────────
// Only trade if not already in a position (for a clean, non-reversing system)
// To enable close-and-reverse, remove the strategy.opentrades == 0 check.
if (strategy.opentrades == 0)
    // Long Entry
    if (long_condition)
        // Define TP1 price based on R:R
        tp1_price_long = strategy.position_avg_price + (stop_loss_distance * tp1_rr)
        strategy.entry("Long", strategy.long, qty=position_size)
        // Use strategy.exit to place both SL and TP orders for 50% of the position
        strategy.exit("Long TP1/SL", from_entry="Long", qty_percent=50, limit=tp1_price_long, stop=strategy.position_avg_price - stop_loss_distance)
        // Place a stop loss for the remaining 50%
        strategy.exit("Long SL2", from_entry="Long", qty_percent=100, stop=strategy.position_avg_price - stop_loss_distance)

    // Short Entry
    if (short_condition)
        // Define TP1 price based on R:R
        tp1_price_short = strategy.position_avg_price - (stop_loss_distance * tp1_rr)
        strategy.entry("Short", strategy.short, qty=position_size)
        // Use strategy.exit to place both SL and TP orders for 50% of the position
        strategy.exit("Short TP1/SL", from_entry="Short", qty_percent=50, limit=tp1_price_short, stop=strategy.position_avg_price + stop_loss_distance)
        // Place a stop loss for the remaining 50%
        strategy.exit("Short SL2", from_entry="Short", qty_percent=100, stop=strategy.position_avg_price + stop_loss_distance)

// Trailing and Stagnation Exit Logic
if (strategy.position_size > 0) // If in a long position
    if (exit_long_trail or stagnation_exit)
        strategy.close("Long", comment = exit_long_trail ? "Trail Exit" : "Stagnation")

if (strategy.position_size < 0) // If in a short position
    if (exit_short_trail or stagnation_exit)
        strategy.close("Short", comment = exit_short_trail ? "Trail Exit" : "Stagnation")

// EOD Exit Logic (Example for assets with a session)
is_eod_exit_time = (hour(time, "America/New_York") == 15 and minute(time, "America/New_York") >= 45)
if (is_eod_exit_time)
    strategy.close_all(comment="EOD Exit")

// ==============================================================================
// 🔴 END REGION: STRATEGY EXECUTION FRAMEWORK
// ==============================================================================
```
    