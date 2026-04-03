
# Indicators to Strategy Blueprint

Based on the provided "CUSUM Trend" indicator, here is the architectural blueprint for its transformation into a production-grade automated execution framework.

### 1. Execution Triggers (Entry & Direction)

The original script correctly identifies the precise moment a new trend begins using the `bull_start` and `bear_start` variables. These will serve as our primary entry triggers.

*   **Long Entry Condition:** A long position is initiated when the market regime flips from non-bullish (ranging or bearish) to bullish.
    ```pinescript
    longCondition = regime == 1 and regime[1] != 1
    ```
*   **Short Entry Condition:** A short position is initiated when the market regime flips from non-bearish (ranging or bullish) to bearish.
    ```pinescript
    shortCondition = regime == -1 and regime[1] != -1
    ```

*   **Execution Nuance (Close vs. Real-time):** The original indicator is wisely built with `barstate.isconfirmed`, ensuring its signals are non-repainting and final only upon the close of a bar. To maintain this integrity and avoid false signals from intra-bar volatility, the strategy **must** execute on the bar's close. This is achieved in Pine Script by setting `process_orders_on_close=true` within the `strategy()` declaration. This means an entry signal on the 10:00 AM bar will result in an order being placed at the 10:00 AM closing price, to be filled at the open of the next bar (10:01 AM, assuming a 1-minute chart).

*   **Signal Reversals:** The logic is inherently a "stop-and-reverse" (SAR) system. If the strategy is in a long position and the `shortCondition` becomes true, the framework must automatically close the existing long position and initiate a new short position on the same bar. The `strategy.entry()` function handles this natively; calling `strategy.entry("Short", strategy.short, ...)` while a long position is open will automatically reverse the position.

### 2. Multi-Tiered Exit Logic

A professional strategy cannot rely on a single exit method. We will construct a layered defense to protect capital and profits.

*   **Initial Stop Loss (Volatility-Based):** The most critical addition. The original script's trailing stop can be very wide on the entry bar, exposing the trade to significant initial risk. We will implement an ATR-based stop loss set at the time of entry.
    *   **Calculation:** Upon a `longCondition`, the initial stop loss will be placed at `entry_price - (ATR * multiplier)`. For a `shortCondition`, it will be `entry_price + (ATR * multiplier)`. A typical multiplier is between 1.5 and 2.5.
    *   **Logic:** This creates a "maximum acceptable loss" for the trade idea, calculated based on recent market volatility, not an arbitrary percentage.

*   **Take Profit / Trailing Mechanism:** We will combine the indicator's native trailing stop with our initial stop for a dynamic exit system.
    *   **Phase 1 (Initial Risk):** The ATR-based stop loss remains the primary floor for the position.
    *   **Phase 2 (CUSUM Trail):** The indicator provides a natural trailing stop: the lower band (`dn_band`) for longs and the upper band (`up_band`) for shorts. Once the price moves favorably and the CUSUM trail moves *above* the initial ATR stop loss (for a long), the CUSUM trail becomes the new, active stop loss. This allows the trade to breathe while aggressively protecting accrued profits.
    *   **Exit Condition:**
        *   For Longs: Exit if `close < max(initial_long_stop, dn_band)`.
        *   For Shorts: Exit if `close > min(initial_short_stop, up_band)`.

*   **Time-Based Exits:** Capital is a finite resource; it should not be held hostage by stagnant trades.
    *   **End of Session:** For intraday strategies (e.g., on timeframes H1 or lower), a non-negotiable rule to close all open positions 5-10 minutes before the market session closes. This avoids overnight/weekend gap risk.
    *   **Stagnation Exit:** If a position has been open for `X` bars (e.g., 50 bars) and has not made a new high (for longs) or new low (for shorts) in the last `Y` bars (e.g., 20 bars), the position is closed. This frees up capital from "dead" trades that are neither winning nor losing decisively.

### 3. Capital Allocation & Risk Management

Signals are useless without a mathematical framework for position sizing. We will implement a fixed-fractional risk model.

*   **Risk-Based Sizing:** The core of professional risk management. The strategy will risk a fixed percentage of account equity on every single trade.
    *   **Inputs:**
        *   `account_equity`: The total value of the trading account (provided by `strategy.equity`).
        *   `risk_per_trade_pct`: A user-defined input (e.g., 1.0 for 1% risk).
    *   **Calculation Logic (for a Long trade):**
        1.  `risk_amount_per_trade = account_equity * (risk_per_trade_pct / 100)`
        2.  `entry_price = close` (since we execute on bar close)
        3.  `stop_loss_price = entry_price - (ta.atr(14) * 2.0)`
        4.  `risk_per_unit = entry_price - stop_loss_price`
        5.  `position_size = risk_amount_per_trade / risk_per_unit`
    *   This calculation is performed *before* every `strategy.entry` call to ensure each trade is sized according to its specific risk profile and the current account balance.

*   **Pyramiding & Scaling:** Adding to winning positions must be rule-based to avoid reckless over-exposure.
    *   **Pyramiding Condition:** A new position can be added only if:
        1.  The primary position is in profit.
        2.  A valid, secondary entry signal occurs (e.g., a pullback to the `hma_base` within an established trend).
        3.  The total number of open entries does not exceed a predefined limit (e.g., 3).
    *   **Risk Adjustment:** When pyramiding, the stop loss for *all* open entries must be moved to, at a minimum, the break-even price of the consolidated position. This ensures that adding to a winner does not turn the entire trade into a loser.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the `indicator` into a `strategy` incorporating the architectural principles above.

```pinescript
// CUSUM Trend converted to a Production-Ready Execution Engine

//@version=5
strategy("CUSUM Execution Engine", 
     overlay=true, 
     process_orders_on_close=true, // CRITICAL: Execute on bar close to match non-repainting logic
     pyramiding=0, // Set to 0 for stop-and-reverse. Set > 0 for pyramiding rules.
     commission_type=strategy.commission.percent,
     commission_value=0.075, // Example: 0.075% commission per trade
     slippage=2) // Example: 2 ticks of slippage per order

// ==========================================
// 1. USER INPUTS & RISK MANAGEMENT
// ==========================================
string sensitivity = input.string("Balanced (Swing)", "Sensitivity", options=["Fast (Day Trade)", "Balanced (Swing)", "Slow (Trend)"])
float risk_percent = input.float(1.0, "Risk Per Trade %", minval=0.1, maxval=10.0)
float atr_mult = input.float(2.0, "Initial Stop ATR Multiplier", minval=0.5)
int atr_len = input.int(14, "ATR Length")

// ==========================================
// 2. CUSUM ENGINE (Copied from original script)
// ==========================================
var int base_len = 21, var float k_mult = 0.5, var float h_mult = 3.0
if sensitivity == "Fast (Day Trade)"
    base_len := 14, k_mult := 0.4, h_mult := 2.0
else if sensitivity == "Balanced (Swing)"
    base_len := 21, k_mult := 0.5, h_mult := 3.0
else if sensitivity == "Slow (Trend)"
    base_len := 50, k_mult := 0.6, h_mult := 4.0

float hma_base = ta.hma(close, base_len)
float residual = close - hma_base
float res_std = ta.stdev(residual, base_len)
res_std := na(res_std) or res_std == 0 ? 0.001 : res_std
float k_drift = res_std * k_mult
float h_thresh = res_std * h_mult
var float bullPressure = 0.0, var float bearPressure = 0.0 
bullPressure := math.max(0, nz(bullPressure[1]) + residual - k_drift)
bearPressure := math.max(0, nz(bearPressure[1]) - residual - k_drift)
bool trig_bull = bullPressure > h_thresh, bool trig_bear = bearPressure > h_thresh
if trig_bull or trig_bear
    bullPressure := 0.0, bearPressure := 0.0

var int regime = 0
float up_band = hma_base + h_thresh, float dn_band = hma_base - h_thresh
if trig_bull
    regime := 1
else if trig_bear
    regime := -1
else if regime == 1 and close < dn_band
    regime := 0
else if regime == -1 and close > up_band
    regime := 0

// ==========================================
// 3. EXECUTION TRIGGERS & RISK CALCULATION
// ==========================================
bool longCondition = regime == 1 and regime[1] != 1
bool shortCondition = regime == -1 and regime[1] != -1

// Calculate position size based on ATR stop
float atr_val = ta.atr(atr_len)
float stop_distance = atr_val * atr_mult
float risk_per_unit = stop_distance * syminfo.pointvalue
float risk_capital = (risk_percent / 100) * strategy.equity
float position_size = risk_capital / risk_per_unit

// Define Stop Loss and Take Profit levels AT THE TIME OF ENTRY
var float long_stop_price = na
var float short_stop_price = na

if (longCondition)
    long_stop_price := close - stop_distance
    short_stop_price := na // Reset other direction's stop

if (shortCondition)
    short_stop_price := close + stop_distance
    long_stop_price := na // Reset other direction's stop

// Persist stop levels through the trade
long_stop_price := strategy.position_size > 0 ? nz(long_stop_price[1], long_stop_price) : na
short_stop_price := strategy.position_size < 0 ? nz(short_stop_price[1], short_stop_price) : na

// ==========================================
// 4. STRATEGY ORDERS
// ==========================================
// --- ENTRY LOGIC ---
if (longCondition)
    strategy.entry("Long", strategy.long, qty=position_size, comment="CUSUM Long Entry")

if (shortCondition)
    strategy.entry("Short", strategy.short, qty=position_size, comment="CUSUM Short Entry")

// --- MULTI-TIERED EXIT LOGIC ---
// A. Initial ATR Stop Loss (managed by strategy.exit)
if (strategy.position_size > 0)
    strategy.exit("Long Exit", from_entry="Long", stop=long_stop_price)

if (strategy.position_size < 0)
    strategy.exit("Short Exit", from_entry="Short", stop=short_stop_price)

// B. CUSUM Trailing Stop (managed by strategy.close as a conditional exit)
bool trail_long_exit = strategy.position_size > 0 and close < dn_band and dn_band > long_stop_price
bool trail_short_exit = strategy.position_size < 0 and close > up_band and up_band < short_stop_price

if (trail_long_exit)
    strategy.close("Long", comment="CUSUM Trail Stop")

if (trail_short_exit)
    strategy.close("Short", comment="CUSUM Trail Stop")

// C. Time-Based Exit (Example: End of Day for stocks)
isEOD = year(time_close) == year(timenow) and month(time_close) == month(timenow) and dayofmonth(time_close) == dayofmonth(timenow) and hour(time_close) >= 15 and minute(time_close) >= 50
if (isEOD)
    strategy.close_all(comment="End of Day Exit")

// ==========================================
// 5. VISUALIZATION (for debugging)
// ==========================================
plot(strategy.position_size > 0 ? long_stop_price : na, "Initial Long SL", color.red, style=plot.style_linebr)
plot(strategy.position_size < 0 ? short_stop_price : na, "Initial Short SL", color.red, style=plot.style_linebr)
plot(regime == 1 ? dn_band : regime == -1 ? up_band : na, "CUSUM Trail", color.aqua, style=plot.style_circles, linewidth=2)
```
