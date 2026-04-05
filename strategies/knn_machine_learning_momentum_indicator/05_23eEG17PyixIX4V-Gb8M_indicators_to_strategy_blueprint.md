
# Indicators to Strategy Blueprint

Here is the architectural blueprint for transforming the KNN Machine Learning Momentum Indicator into a production-grade automated trading strategy.

### 1. Execution Triggers (Entry & Direction)

The provided indicator generates signals based on a KNN model's probability output. To trade these signals effectively, we must define precise, non-repainting entry conditions that account for market noise and trend context.

*   **Precise Entry Conditions:**
    *   **Long Entry:** A long position is initiated when all the following conditions are met:
        1.  The model's bullish probability crosses above the user-defined `prob_threshold`. (`ta.crossover(prob_up, prob_threshold)`)
        2.  The price is currently above the long-term trend filter (`close > ema`). This ensures we only take "Major Long" signals, filtering out higher-risk counter-trend trades.
        3.  There is no existing long position open (`strategy.position_size <= 0`). This prevents pyramiding by default and allows for reversals.
    *   **Short Entry:** A short position is initiated when all the following conditions are met:
        1.  The model's bearish probability crosses above the user-defined `prob_threshold`. (`ta.crossover(prob_down, prob_threshold)`)
        2.  The price is currently below the long-term trend filter (`close <= ema`). This validates the signal with the prevailing "Major Short" trend.
        3.  There is no existing short position open (`strategy.position_size >= 0`).

*   **Execution Nuances:**
    *   **Execute on Bar Close:** All calculations, including the complex KNN loop, must be finalized before an order is placed. Firing orders intra-bar based on a fluctuating probability model would lead to significant whipsaw and signal repainting. The strategy will be configured with `process_orders_on_close=true`. This ensures that a signal is only considered valid once the candle has closed, providing a stable state for the model's features.
    *   **Signal Reversals:** The system is designed for immediate reversals. If the strategy is in a long position and a valid `Short Entry` condition is met on a bar's close, the framework will automatically close the existing long position and initiate a new short one. This is achieved by allowing a new `strategy.entry` call to reverse the current market position, which is a more efficient execution model than separate `strategy.close` and `strategy.entry` commands.

### 2. Multi-Tiered Exit Logic

A profitable entry is meaningless without a disciplined exit strategy. We will replace the non-existent exit logic with a three-pronged approach: an initial volatility-based stop, a dynamic trailing mechanism, and a time-based kill switch.

*   **Initial Stop Loss (Volatility-Based):**
    *   An arbitrary percentage or point-based stop is brittle. We will calculate the stop loss using the Average True Range (ATR) to adapt to the market's current volatility.
    *   **Calculation:** Upon entry, an ATR with a period of `14` is calculated (`atr_val = ta.atr(14)`).
    *   **Placement:**
        *   For a **Long** trade, the initial Stop Loss is placed at `entry_price - (atr_val * atr_multiplier)`, where `atr_multiplier` is a user-defined input (e.g., 2.0).
        *   For a **Short** trade, the initial Stop Loss is placed at `entry_price + (atr_val * atr_multiplier)`.

*   **Take Profit / Trailing Stop:**
    *   We will implement a two-stage profit-taking and trailing stop mechanism to lock in gains while allowing winners to run.
    *   **Stage 1: Initial Target & Breakeven:**
        *   A primary Take Profit (TP1) target is set at a risk/reward ratio, for example, 1.5:1. `TP1_Long = entry_price + (entry_price - stop_loss_price) * 1.5`.
        *   When TP1 is hit, 50% of the position is closed.
        *   Simultaneously, the stop loss for the remaining 50% of the position is moved to the original entry price (breakeven).
    *   **Stage 2: Dynamic Trailing:**
        *   The remaining position is managed by a "Chandelier Exit." The trailing stop will continuously update on each bar's close.
        *   **For a Long:** The stop is trailed at `highest(high, trade_duration) - (atr_val * atr_multiplier)`.
        *   **For a Short:** The stop is trailed at `lowest(low, trade_duration) + (atr_val * atr_multiplier)`.
        *   This ensures the stop is always ratcheted up behind a winning trade but never moves down, effectively protecting profits while giving the trend room to extend.

*   **Time-Based Exits:**
    *   Momentum trades are time-sensitive. If the expected move doesn't materialize, the trade's premise is likely invalid.
    *   **Stagnation Exit:** If a trade has been open for `X` bars (e.g., 25 bars) and has not reached the breakeven point (TP1), the position is closed automatically. This cuts losses on trades that are "drifting" sideways and tying up capital.
    *   **End-of-Session Exit:** For intraday strategies, an explicit rule will be added to close all open positions `N` minutes before the market close (e.g., 15 minutes) to avoid overnight risk and gap moves.

### 3. Capital Allocation & Risk Management

Professional trading is synonymous with professional risk management. We will size every position to risk a fixed percentage of the total account equity, ensuring no single trade can cause catastrophic losses.

*   **Risk-Based Sizing:**
    *   The core of the risk engine. The strategy will risk a user-defined percentage of equity on every single trade (e.g., 1.0%).
    *   **Logic:**
        1.  Define `risk_per_trade_pct` (e.g., 1.0).
        2.  Calculate the dollar amount to risk: `risk_amount = strategy.equity * (risk_per_trade_pct / 100)`.
        3.  Calculate the distance to the initial stop loss in points: `stop_loss_distance = abs(entry_price - stop_loss_price)`.
        4.  Calculate the position size: `position_size = risk_amount / (stop_loss_distance * syminfo.pointvalue)`.
    *   This calculation is performed *before* each entry, ensuring that a trade in a volatile market (wide stop) will have a smaller position size than a trade in a quiet market (tight stop), keeping the dollar risk constant.

*   **Pyramiding & Scaling:**
    *   **Pyramiding (Default: Off):** The base strategy will not add to winning positions (`pyramiding = 0`). This is a conservative and robust starting point. Advanced configurations could allow pyramiding under strict conditions: the initial position must be in profit by at least `2 * ATR`, and a new, high-conviction signal must appear.
    *   **Scaling Out:** The multi-tiered exit logic inherently supports scaling out. The first exit at TP1 reduces the position size by a predefined fraction (e.g., 50%), immediately banking profit and reducing risk on the remainder of the trade.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the architectural transition from an `indicator` to a `strategy`, incorporating the principles of professional execution.

```pine
//@version=5
// --- STRATEGY DECLARATION ---
// Note: initial_capital, slippage, and commission are critical for realistic backtests.
strategy("Production KNN Execution Engine", 
     overlay=true, 
     process_orders_on_close=true,
     initial_capital=100000,
     default_qty_type=strategy.fixed, // We will calculate qty manually
     commission_type=strategy.commission.percent,
     commission_value=0.04, // Realistic broker commission
     slippage=2) // Slippage in ticks

// --- STRATEGY-SPECIFIC INPUTS ---
group_risk = "🛡️ Risk & Trade Management"
risk_per_trade_pct = input.float(1.0, "Risk Per Trade (%)", group=group_risk, minval=0.1, maxval=5.0)
atr_period         = input.int(14, "ATR Period", group=group_risk)
atr_multiplier     = input.float(2.0, "ATR Stop Multiplier", group=group_risk)
use_trend_filter   = input.bool(true, "Enable Trend Filter", group=group_risk)
use_time_exit      = input.bool(true, "Enable Time-Based Exit", group=group_risk)
stagnation_bars    = input.int(25, "Max Bars Before Stagnation Exit", group=group_risk)

// ... [Paste the entire original script from line 5 to the 'SIGNAL GENERATION' section] ...
// ... [This includes constants, inputs, feature engineering, PCA, and KNN core engine] ...

// ==========================================
// --- SIGNAL GENERATION (FROM ORIGINAL SCRIPT) ---
// ==========================================
bool raw_long_signal = ta.crossover(prob_up, prob_threshold)
bool raw_short_signal = ta.crossover(prob_down, prob_threshold)

// EMA Trend Filter
ema = ta.ema(close, ema_len)
ema_bull = close > ema
ema_bear = close <= ema

// --- FINAL EXECUTION TRIGGERS ---
bool long_entry_condition = raw_long_signal and (ema_bull or not use_trend_filter)
bool short_entry_condition = raw_short_signal and (ema_bear or not use_trend_filter)

// ==========================================
// --- RISK & POSITION SIZING ENGINE ---
// ==========================================
float atr_val = ta.atr(atr_period)

// Calculate Stop Loss Price *before* entry to determine size
float long_stop_price = close - (atr_val * atr_multiplier)
float short_stop_price = close + (atr_val * atr_multiplier)

// Risk-based position sizing calculation
float risk_per_unit = syminfo.pointvalue * atr_val * atr_multiplier
float risk_amount = (risk_per_trade_pct / 100) * strategy.equity
float position_size = risk_amount / risk_per_unit
position_size := math.max(position_size, 0.00001) // Ensure size is not zero

// ==========================================
// --- TRADE EXECUTION & MANAGEMENT ---
// ==========================================
// --- ENTRY LOGIC ---
if (long_entry_condition and strategy.position_size <= 0)
    // Close any existing short and go long
    strategy.entry("KNN Long", strategy.long, qty=position_size, comment="Entry Long")
    // We use strategy.exit to place the stop loss order immediately
    strategy.exit("SL/TP Long", from_entry="KNN Long", loss=long_stop_price)

if (short_entry_condition and strategy.position_size >= 0)
    // Close any existing long and go short
    strategy.entry("KNN Short", strategy.short, qty=position_size, comment="Entry Short")
    // We use strategy.exit to place the stop loss order immediately
    strategy.exit("SL/TP Short", from_entry="KNN Short", loss=short_stop_price)

// --- TIME-BASED EXIT LOGIC ---
bool in_long_trade = strategy.position_size > 0
bool in_short_trade = strategy.position_size < 0
int bars_since_entry = bar_index - strategy.opentrades.entry_bar_index(strategy.opentrades - 1)

if (use_time_exit and (in_long_trade or in_short_trade) and bars_since_entry >= stagnation_bars)
    strategy.close_all(comment="Stagnation Exit")

// Note: The multi-stage TP and dynamic trailing logic is more complex and often
// requires a state machine to manage partial exits. The above provides the robust
// foundation (entry + initial stop). A full implementation would expand the 
// strategy.exit calls or use separate strategy.close calls based on profit levels.
```
    