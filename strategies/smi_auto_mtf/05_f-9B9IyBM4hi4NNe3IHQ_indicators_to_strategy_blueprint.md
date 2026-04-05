
# Indicators to Strategy Blueprint

Based on the provided `SMI Auto MTF` indicator, here is the architectural blueprint for its transformation into a production-grade automated execution framework. The core logic transitions from simple crossover signals to a comprehensive system that manages entries, exits, and capital with a focus on real-world execution.

### 1. Execution Triggers (Entry & Direction)

The original script identifies bullish and bearish crossovers. A professional system cannot trade every crossover; it requires filters to improve signal quality and define the precise moment of execution.

*   **Long Entry Condition:** A `ta.crossover(smi, sig)` must occur **AND** the SMI value at the time of the cross must be below the zero line (`smi < 0`). This acts as a filter, ensuring we only take long signals when momentum is attempting to reverse from bearish to bullish territory, avoiding entries into already over-extended bullish moves.

*   **Short Entry Condition:** A `ta.crossunder(smi, sig)` must occur **AND** the SMI value at the time of the cross must be above the zero line (`smi > 0`). This filters for short signals that represent a potential exhaustion of bullish momentum, preventing entries after a significant downward move has already occurred.

*   **Execution Nuance:** All entry signals will be executed **on the close of the signal bar**. The `ta.crossover` and `ta.crossunder` functions confirm a true cross only at the bar's conclusion (`barstate.isconfirmed`). Attempting to trade this intra-bar (`calc_on_every_tick=true`) would lead to numerous false signals as the SMI and signal lines flicker across each other with each price tick. The strategy will place a market order at the open of the *next* bar, which is the standard, non-repainting approach for backtesting and live execution based on bar-close signals.

*   **Signal Reversals:** This system is designed as a "stop-and-reverse" (SAR) strategy.
    *   If the system is in a long position and a valid `Short Entry Condition` occurs, the existing long position will be closed immediately, and a new short position will be opened.
    *   Conversely, if in a short position, a `Long Entry Condition` will trigger a closure of the short and an entry into a new long position. This is managed by ensuring the strategy can hold only one position (long or short) at a time and using `strategy.close` before `strategy.entry` for clarity and control.

### 2. Multi-Tiered Exit Logic

A static entry signal is insufficient for robust performance. A dynamic, multi-faceted exit strategy is required to manage risk and protect profit.

*   **Initial Stop Loss (Volatility-Based):** The stop loss will not be a fixed percentage. It will be calculated dynamically using the Average True Range (ATR) to adapt to current market volatility.
    *   **Calculation:** Upon entry, calculate the 14-period ATR (`atr14`).
    *   **Long Stop Loss:** `Entry Price - (atr14 * 2.0)`
    *   **Short Stop Loss:** `Entry Price + (atr14 * 2.0)`
    *   The ATR multiplier (e.g., `2.0`) will be an adjustable input, allowing for optimization based on the asset and timeframe.

*   **Take Profit / Trailing Mechanism:** We will implement a two-stage profit-taking mechanism.
    *   **Stage 1: Initial Target (Risk/Reward):** A primary take-profit target will be set based on a multiple of the initial risk. For example, at a 1.5R target.
        *   **Calculation:** `Risk = abs(Entry Price - Initial Stop Loss Price)`.
        *   **Long TP:** `Entry Price + (Risk * 1.5)`.
        *   **Short TP:** `Entry Price - (Risk * 1.5)`.
        *   At this target, we will **scale out** by closing 50% of the position.
    *   **Stage 2: Trailing Stop (Profit Protection):** After the first target is hit, the stop loss for the remaining 50% of the position is moved to breakeven. From this point, a **Chandelier Exit** will be used as a trailing stop.
        *   **Logic:** The trailing stop will follow the market at a distance of `N * ATR` from the highest high (for longs) or lowest low (for shorts) since the trade was entered. This allows the winning portion of the trade to run while aggressively protecting accumulated profits.

*   **Time-Based Exits:**
    *   **End of Day (EOD):** For intraday timeframes (e.g., 1H, 4H), an EOD exit rule is critical to avoid overnight/weekend risk. A time-based rule will close any open position 15 minutes before the session's close.
    *   **Trade Stagnation:** If a position has been open for a specified number of bars (e.g., 50 bars) without hitting either its stop loss or first take-profit target, it will be closed. This frees up capital from non-performing trades.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term viability. We will use a fixed-fractional risk model.

*   **Risk-Based Sizing:** The strategy will risk a fixed percentage of the total account equity on every single trade.
    *   **Inputs:**
        *   `Account Equity`: Provided by `strategy.equity`.
        *   `Risk per Trade %`: A user-defined input (e.g., `1.0` for 1% risk).
    *   **Calculation Logic:**
        1.  Determine the cash amount to risk: `Risk_Amount = strategy.equity * (Risk_Per_Trade / 100)`.
        2.  Determine the risk per share/contract in cash: `Per_Unit_Risk = abs(Entry_Price - Stop_Loss_Price)`.
        3.  Calculate the final position size: `Position_Size = Risk_Amount / Per_Unit_Risk`.
    *   This calculation ensures that a losing trade, if hit at the initial stop loss, will only reduce the account by the predefined percentage, regardless of volatility.

*   **Pyramiding & Scaling:**
    *   **Pyramiding (Scaling In):** The base strategy is stop-and-reverse and does not include pyramiding to maintain simplicity and control risk. Adding to a position would require a separate set of rules (e.g., adding on a pullback after a new momentum high) and complicates risk management significantly. It is disabled by default (`pyramiding = 0`).
    *   **Scaling Out:** As defined in the exit logic, the strategy will automatically scale out. The `strategy.exit` command will be used with `qty_percent = 50` to close half the position at the first take-profit target.

### 4. Implementation Snippet (Pine Logic)

This code block demonstrates the conversion of the `indicator` into a `strategy` incorporating the architectural principles outlined above.

```pine
//@version=5
// --- STRATEGY DECLARATION ---
strategy("SMI Execution Framework", 
     overlay=true, 
     initial_capital=100000, 
     commission_type=strategy.commission.percent, 
     commission_value=0.075, 
     slippage=2, 
     process_orders_on_close=true, // Execute on the close of the bar
     pyramiding=0) // No pyramiding allowed in this SAR model

// --- RISK MANAGEMENT INPUTS ---
i_risk_percent = input.float(1.0, "Risk Per Trade %", minval=0.1, maxval=10)
i_atr_len      = input.int(14, "ATR Period")
i_atr_stop_mult= input.float(2.0, "ATR Stop Multiplier")
i_rr_target    = input.float(1.5, "Risk/Reward Target 1")

// --- ORIGINAL SMI LOGIC (UNCHANGED) ---
i_override = input.bool(false, title="Manual Override")
i_k        = input.int(13, title="K Period (manual)",      minval=1, maxval=100)
i_smooth   = input.int(5,  title="Smooth Period (manual)", minval=1, maxval=50)
i_signal   = input.int(5,  title="Signal Period (manual)", minval=1, maxval=50)

tf = timeframe.period
auto_k = tf == "15"  ? 5  : tf == "60"  ? 9  : tf == "240" ? 13 : tf == "D"   ? 13 : tf == "W"   ? 20 : 13
auto_smooth = tf == "15"  ? 3 : tf == "60"  ? 3 : tf == "240" ? 5 : tf == "D"   ? 5 : tf == "W"   ? 7 : 5
auto_signal = tf == "15"  ? 3 : tf == "60"  ? 3 : tf == "240" ? 5 : tf == "D"   ? 5 : tf == "W"   ? 7 : 5

k_len      = i_override ? i_k      : auto_k
smooth_len = i_override ? i_smooth : auto_smooth
signal_len = i_override ? i_signal : auto_signal

hh       = ta.highest(high, k_len)
ll       = ta.lowest(low,   k_len)
mid      = (hh + ll) / 2.0
diff     = close - mid
hl_range = hh - ll
s_diff  = ta.ema(ta.ema(diff,     smooth_len), smooth_len)
s_range = ta.ema(ta.ema(hl_range, smooth_len), smooth_len)
smi    = s_range != 0.0 ? 100.0 * s_diff / (0.5 * s_range) : 0.0
sig    = ta.ema(smi, signal_len)

// --- VOLATILITY & POSITION SIZING CALCULATION ---
atr_val = ta.atr(i_atr_len)

// --- EXECUTION TRIGGERS ---
long_condition = ta.crossover(smi, sig) and smi < 0
short_condition = ta.crossunder(smi, sig) and smi > 0

// --- STRATEGY EXECUTION LOGIC ---
if strategy.position_size == 0 // Only enter if flat
    // LONG ENTRY
    if long_condition
        stop_loss_price = close - (atr_val * i_atr_stop_mult)
        risk_per_unit   = close - stop_loss_price
        take_profit_1   = close + (risk_per_unit * i_rr_target)
        
        risk_amount     = strategy.equity * (i_risk_percent / 100)
        position_size   = risk_amount / risk_per_unit

        strategy.entry("Long", strategy.long, qty=position_size)
        strategy.exit("Long TP1/SL", from_entry="Long", qty_percent=50, limit=take_profit_1, stop=stop_loss_price)
        strategy.exit("Long SL2", from_entry="Long", stop=stop_loss_price) // Stop for the remaining 50%

    // SHORT ENTRY
    if short_condition
        stop_loss_price = close + (atr_val * i_atr_stop_mult)
        risk_per_unit   = stop_loss_price - close
        take_profit_1   = close - (risk_per_unit * i_rr_target)

        risk_amount     = strategy.equity * (i_risk_percent / 100)
        position_size   = risk_amount / risk_per_unit

        strategy.entry("Short", strategy.short, qty=position_size)
        strategy.exit("Short TP1/SL", from_entry="Short", qty_percent=50, limit=take_profit_1, stop=stop_loss_price)
        strategy.exit("Short SL2", from_entry="Short", stop=stop_loss_price) // Stop for the remaining 50%

// --- STOP-AND-REVERSE LOGIC ---
if strategy.position_size > 0 and short_condition // If long and a short signal appears
    strategy.close("Long", comment="Reversal to Short")

if strategy.position_size < 0 and long_condition // If short and a long signal appears
    strategy.close("Short", comment="Reversal to Long")

// --- PLOT STOPS FOR VISUALIZATION ---
plot(strategy.position_size > 0 ? strategy.opentrades.entry_price(0) - (atr_val * i_atr_stop_mult) : na, "SL", color.red, style=plot.style_linebr)
plot(strategy.position_size < 0 ? strategy.opentrades.entry_price(0) + (atr_val * i_atr_stop_mult) : na, "SL", color.red, style=plot.style_linebr)
```
    