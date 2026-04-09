
# Indicators to Strategy Blueprint

The provided "Institutional Flow Scalper [IFS] v4" is a sophisticated `indicator` that synthesizes multiple order flow and momentum concepts into a single confidence score. While it effectively simulates trade entries and exits using internal state variables, this approach is insufficient for rigorous backtesting or live execution.

To transition this from a visual tool to a professional automated strategy, we must replace the internal simulation with Pine Script's native `strategy()` engine and introduce robust risk management and execution logic.

### 1. Execution Triggers (Entry & Direction)

The core entry logic is already well-defined by a confluence of "pillars" and environmental filters. We will formalize these into precise boolean triggers for the strategy engine.

*   **Long Entry Condition:** A trade is initiated when all the following conditions are met on a confirmed bar (`barstate.isconfirmed`):
    *   `bullishBias > bearishBias`
    *   `bullPillarCount >= 2`
    *   `confidence >= minConfidence`
    *   `not isChoppy` (Chop filter pass)
    *   `inKillZone` (Session filter pass)
    *   `bar_index - lastSignalBar >= 3` (Signal cooldown)

*   **Short Entry Condition:** A trade is initiated when all the following conditions are met on a confirmed bar:
    *   `bearishBias > bullishBias`
    *   `bearPillarCount >= 2`
    *   `confidence >= minConfidence`
    *   `not isChoppy`
    *   `inKillZone`
    *   `bar_index - lastSignalBar >= 3`

#### Execution Nuances: Close vs. Real-time

The original script uses `barstate.isconfirmed`, meaning signals are only valid *after* a bar has fully closed. In a `strategy`, this translates to an order being sent at the **open of the next bar**.

*   **Execution at "Close":** This is the standard, non-repainting approach. It ensures that the conditions that generated the signal are locked in and will not change. However, for a scalping strategy, the delay of waiting for the next bar's open can result in significant slippage or a missed entry, as the price may move substantially between the signal bar's close and the next bar's open.
*   **Execution on "Real-time Price Action":** This would require setting `calc_on_every_tick=true` in the `strategy` declaration. While this allows for faster entries within the signal bar, it introduces the risk of "repainting" signals. A condition might be true for a moment, triggering an entry, only to become false by the time the bar closes. **For a production system, we will stick to execution at the open of the next bar for backtesting reliability and add realistic slippage to model the real-world cost of this delay.**

#### Signal Reversals

The original script explicitly prevents reversals by requiring `posState == 0` for a new entry. This is a safety measure but can be inefficient, forcing the strategy to wait for an exit before acting on a new, opposing signal. A professional framework should handle this gracefully.

*   **Logic:** If the strategy is currently `Long` and a `shortSignal` condition becomes true, the system should first close the existing long position and then immediately open a new short position on the same bar. This is achieved by using `strategy.close()` followed by `strategy.entry()`.

### 2. Multi-Tiered Exit Logic

A single Take Profit and Stop Loss is a fragile approach. A robust system layers multiple exit conditions to adapt to market behavior.

*   **Initial Stop Loss (Volatility-Based):** The script's use of an ATR-multiple for the stop loss is excellent. We will retain this. The stop loss will be placed at `Entry Price - (ATR * atrMultSL)` for longs and `Entry Price + (ATR * atrMultSL)` for shorts. This is a non-negotiable hard stop.

*   **Take Profit / Trailing Mechanism (Dynamic):** A static TP can leave profits on the table. We will implement a two-stage exit strategy.
    1.  **Initial Target (TP1):** A primary take-profit target will be set at a 1:1 Risk/Reward ratio (i.e., `atrMultTP = atrMultSL`). At this point, we will exit 50% of the position.
    2.  **Trailing Stop Activation:** Once TP1 is hit, the stop loss for the remaining 50% of the position is moved to breakeven. From this point, a trailing stop is activated, trailing the price by `1.5 * ATR` from the highest high (for longs) or lowest low (for shorts) achieved since the entry. This allows the winning portion of the trade to run while protecting profits.

*   **Time-Based & Conditional Exits:**
    1.  **Momentum Failure:** The script's `longMomExit` and `shortMomExit` conditions are valuable. We will use these as a conditional exit to close the entire position if the underlying momentum that triggered the trade evaporates before any profit target is hit.
    2.  **End of Day (EOD) Exit:** As this is a scalping strategy, holding positions overnight introduces unacceptable risk. A hard rule will be implemented to close all open positions at a specified time, for example, 15 minutes before the market close (e.g., 15:45 EST for US Equities).

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component of a trading system's long-term viability. We will move from an assumed fixed size to a dynamic, risk-based model.

*   **Risk-Based Sizing:** The strategy will risk a fixed percentage of account equity on every trade.
    *   **Rule:** Risk `1%` of `strategy.equity` per trade.
    *   **Calculation:**
        1.  Define `risk_percent = 0.01`.
        2.  Calculate `risk_per_trade_usd = strategy.equity * risk_percent`.
        3.  Calculate `stop_loss_distance_usd = abs(entry_price - stop_loss_price)`.
        4.  **`position_size = risk_per_trade_usd / stop_loss_distance_usd`**.
    *   This ensures that a losing trade, regardless of the ATR or volatility at the time of entry, will only cost the account a predefined percentage, creating consistent risk exposure.

*   **Pyramiding & Scaling:**
    *   **Scaling Out:** As defined in the exit logic, we will scale out of positions by exiting a partial quantity at TP1. This is a core part of the strategy.
    *   **Pyramiding (Adding to Winners):** For this scalping framework, pyramiding is **disadvised**. It adds significant complexity, increases exposure, and is difficult to manage on short timeframes. The strategy will be configured with `pyramiding = 0` to enforce a "one position per signal" rule until a full exit occurs.

### 4. Implementation Snippet (Pine Logic)

Here is the pseudocode and Pine Script logic demonstrating the transformation from an `indicator` to a professional `strategy`. This snippet focuses on the execution engine, assuming all the indicator's calculations (`longSignal`, `shortSignal`, `atrVal`, etc.) are present.

```pine
// @version=5
// 1. STRATEGY DECLARATION - Professional Grade
strategy(
    "Institutional Flow Scalper [STRATEGY]",
    shorttitle="IFS v4 [Strat]",
    overlay=true,
    pyramiding=0, // No adding to positions
    initial_capital=100000,
    default_qty_type=strategy.percent_of_equity,
    default_qty_value=100, // Placeholder, we will calculate size manually
    commission_type=strategy.commission.cash_per_order,
    commission_value=2.5, // Realistic commission for futures (e.g., $2.50 entry, $2.50 exit)
    slippage=2 // Realistic slippage of 2 ticks for a scalping system
)

// --- [Assume all of the original script's calculations for 'longSignal', 'shortSignal', 'atrVal', etc. are here] ---

// 2. CAPITAL ALLOCATION & RISK MANAGEMENT
risk_percent = input.float(1.0, "Risk per Trade (%)", minval=0.1, maxval=5.0) / 100
use_eod_exit = input.bool(true, "Use End of Day Exit?")
eod_exit_hour = input.int(15, "EOD Exit Hour (24h)", minval=0, maxval=23)
eod_exit_min = input.int(45, "EOD Exit Minute", minval=0, maxval=59)

// Calculate position size based on risk
stop_loss_points = atrVal * atrMultSL
risk_per_share = stop_loss_points * syminfo.pointvalue
risk_amount = strategy.equity * risk_percent
position_size = risk_amount / risk_per_share

// 3. EXECUTION TRIGGERS
// The original script's logic is used, but we remove the `posState` check to allow reversals
longEntry  = longSignal and signalCooldown
shortEntry = shortSignal and signalCooldown

// 4. MULTI-TIERED EXIT LOGIC
// Initial Stop Loss and Take Profit 1 (50% exit)
long_sl_price = close - stop_loss_points
long_tp1_price = close + stop_loss_points // 1:1 R/R for first target

short_sl_price = close + stop_loss_points
short_tp1_price = close - stop_loss_points

// Conditional Exits
longMomExit  = strategy.position_size > 0 and momentumBearish and pressureIndex < -5
shortMomExit = strategy.position_size < 0 and momentumBullish and pressureIndex > 5

// Time-Based Exit
is_eod_exit_time = (hour == eod_exit_hour and minute >= eod_exit_min)
time_exit_signal = use_eod_exit and is_eod_exit_time and strategy.position_size != 0

// 5. STRATEGY EXECUTION ENGINE
if (longEntry)
    // Close any existing short position before entering long (reversal)
    strategy.close("Short", comment="Short Reversal")
    // Enter Long with 2 separate exit orders for scaling out
    strategy.entry("Long", strategy.long, qty=position_size)
    strategy.exit("Long TP1", from_entry="Long", qty_percent=50, profit=stop_loss_points)
    strategy.exit("Long SL", from_entry="Long", loss=stop_loss_points)

if (shortEntry)
    // Close any existing long position before entering short (reversal)
    strategy.close("Long", comment="Long Reversal")
    // Enter Short with 2 separate exit orders for scaling out
    strategy.entry("Short", strategy.short, qty=position_size)
    strategy.exit("Short TP1", from_entry="Short", qty_percent=50, profit=stop_loss_points)
    strategy.exit("Short SL", from_entry="Short", loss=stop_loss_points)

// Trailing Stop Logic (activates after TP1 is hit and position is moved to B/E)
// Note: Pine's native trailing is simpler to implement here for demonstration
var bool isTrailingActive = false
if (strategy.position_size > 0 and high > strategy.opentrades.profit_target(0)) // Simplified check for TP1 hit
    isTrailingActive := true
if (strategy.position_size == 0)
    isTrailingActive := false

if (isTrailingActive)
    strategy.exit("Long Trail", from_entry="Long", trail_points=stop_loss_points)


// Execute Conditional & Time-Based Exits
if (longMomExit)
    strategy.close("Long", comment="Momentum Exit")

if (shortMomExit)
    strategy.close("Short", comment="Momentum Exit")

if (time_exit_signal)
    strategy.close_all(comment="EOD Exit")

```
    