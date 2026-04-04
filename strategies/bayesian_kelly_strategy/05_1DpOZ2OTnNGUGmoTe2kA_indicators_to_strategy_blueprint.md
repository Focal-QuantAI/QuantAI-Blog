
# Indicators to Strategy Blueprint

Here is the architectural breakdown for transforming the "Bayesian Kelly Strategy" into a production-grade automated execution framework.

The provided script is a portfolio allocation model, not a traditional trading strategy. It's always invested, merely adjusting its leverage based on a Bayesian-updated return expectation. This is common in quantitative finance but lacks the discrete entry, exit, and risk-control mechanisms essential for a robust, standalone trading system. Our goal is to build these mechanisms around its core signal, `f_bayes`.

### 1. Execution Triggers (Entry & Direction)

The original script is always long, varying its size. A production system requires clear, non-ambiguous states: **Long, Short, or Flat**. We will repurpose the `f_bayes` variable from a leverage factor into a directional conviction score.

*   **Core Signal Modification:** The original code clamps the signal at zero: `math.max(..., 0)`. This prevents any short signals. The first step is to remove this clamp to allow the signal to be both positive (bullish) and negative (bearish).
    `float f_bayes_raw = (p_prec * p_mu + s_prec * s_mu) * kelly_frac`

*   **Long Entry Condition:** A position is initiated only when the conviction score crosses a meaningful positive threshold. This filters out low-conviction noise around zero.
    `bool isLongEntry = f_bayes_raw > entry_threshold and strategy.position_size == 0`
    *(Where `entry_threshold` could be an input, e.g., `0.1`)*

*   **Short Entry Condition:** A short position is initiated when the conviction score crosses a meaningful negative threshold.
    `bool isShortEntry = f_bayes_raw < -entry_threshold and strategy.position_size == 0`

*   **Execution Nuance:** The strategy uses `process_orders_on_close=true`. This is a robust choice for a production system. It means all calculations are performed on the close of the current bar, and orders are submitted to be executed on the open of the *next* bar. This prevents lookahead bias and is replicable in a live environment. We will maintain this structure.

*   **Signal Reversals:** The system must be able to flip from a long to a short position without necessarily going flat first. If the `isShortEntry` condition becomes true while `strategy.position_size > 0`, the `strategy.entry()` command will automatically generate a market order to close the existing long and open the new short position. This is efficient but can incur higher slippage and commission. Our logic will explicitly handle this by first closing the existing position before evaluating a new entry on the next bar, providing more control.

### 2. Multi-Tiered Exit Logic

The original script's only exit mechanism is a periodic rebalance. This exposes the system to unlimited downside risk. A professional framework requires a hierarchy of exit rules.

*   **Initial Stop Loss (Volatility-Based):** A non-negotiable component. The stop loss will be calculated dynamically at the time of entry based on the Average True Range (ATR), ensuring it adapts to the current market volatility.
    *   **Logic:**
        1.  Calculate ATR over a specific period (e.g., 14 bars) at the time of entry.
        2.  For a **Long** entry, `StopLossPrice = entry_price - (atr_value * atr_multiplier)`.
        3.  For a **Short** entry, `StopLossPrice = entry_price + (atr_value * atr_multiplier)`.
    *   The `atr_multiplier` (e.g., 2.5) becomes a key input parameter.

*   **Take Profit/Trailing (Dynamic Trailing Stop):** The Bayesian logic implies letting winners run as long as the statistical edge persists. A fixed take profit is counterintuitive. A dynamic trailing stop is superior as it protects profits while allowing for trend continuation.
    *   **Mechanism:** An ATR-based trailing stop (Chandelier Exit).
    *   **Logic:**
        1.  For a **Long** position, the stop is trailed at `highest_price_since_entry - (atr_value * trail_atr_multiplier)`.
        2.  For a **Short** position, the stop is trailed at `lowest_price_since_entry + (atr_value * trail_atr_multiplier)`.
    *   The `strategy.exit()` function will be updated on every bar where the trailing stop price improves (moves in the direction of the trade).

*   **Time-Based Exits:** Capital should not be held indefinitely in stagnant trades.
    *   **Stagnation Exit:** If a position has been open for `X` bars (e.g., 20) and has not achieved a minimum profit target (e.g., 1x the initial risk), exit the position. This frees up capital for new opportunities.
    *   **Signal Invalidation Exit:** The original rebalancing logic can be repurposed as an exit signal. If we are in a long position and `f_bayes_raw` drops below a "hold threshold" (e.g., 0), it indicates the statistical edge has disappeared. The position should be closed.
        `bool closeLong = strategy.position_size > 0 and f_bayes_raw < hold_threshold`
        `bool closeShort = strategy.position_size < 0 and f_bayes_raw > -hold_threshold`

### 3. Capital Allocation & Risk Management

The original script's Kelly-based sizing is its core idea, but it's untethered from a stop-loss, making it theoretical. We will integrate its "conviction score" into a professional, risk-first position sizing model.

*   **Risk-Based Sizing:** The foundation of any serious strategy. We will risk a fixed percentage of account equity on every single trade.
    *   **Formula:**
        1.  `RiskPerTrade = strategy.equity * risk_percent`
        2.  `StopLossDistance = abs(entry_price - StopLossPrice)`
        3.  `PositionSize = RiskPerTrade / StopLossDistance`
    *   This formula calculates the exact number of shares/contracts to buy or sell so that if the trade hits the initial stop loss, the account loses exactly the predefined percentage of equity.

*   **Pyramiding & Scaling:** The original script's rebalancing is a form of scaling. We can formalize this with clearer rules.
    *   **Pyramiding (Scaling In):** We will disable pyramiding by default (`pyramiding=0`) to enforce a clean, one-trade-at-a-time structure. For more advanced versions, rules could be: "Allow one additional entry if the initial position is profitable by at least 2x the initial risk, and a new `isLongEntry` signal occurs." The size of the second entry would be calculated based on its own risk parameters.
    *   **Scaling Out:** Instead of a single trailing stop, we can define multiple profit targets to scale out of the position. For example:
        1.  Close 33% of the position at `entry_price + (StopLossDistance * 1)`.
        2.  Close 33% of the position at `entry_price + (StopLossDistance * 2)`.
        3.  Let the final 33% run with the dynamic trailing stop.
    *   This approach locks in gains methodically while still participating in a potential home run.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the transition from the allocation model to a structured trading strategy with robust risk and exit management.

```pine
//@version=5
// TRANSITIONED TO PRODUCTION-GRADE FRAMEWORK
strategy(
     "Production Bayesian Strategy", 
     overlay=true, 
     process_orders_on_close=true, 
     pyramiding=0, // Explicitly disable pyramiding for clean signal evaluation
     commission_type=strategy.commission.percent,
     commission_value=0.075, // Realistic broker commission (e.g., 0.075%)
     slippage=2 // Realistic slippage in ticks
)

// --- Inputs ---
// Bayesian Calculation Inputs
lookback_p = input.int(252, "Prior Lookback Window", minval=10)
lookback_e = input.int(60, "Evidence Lookback Window", minval=10)
kelly_frac = input.float(0.5, "Kelly Fraction", minval=0, maxval=1)

// Trade Execution & Risk Inputs
entry_threshold = input.float(0.05, "Entry Conviction Threshold")
risk_percent = input.float(1.0, "Risk Per Trade %") / 100
atr_period = input.int(14, "ATR Period")
sl_atr_mult = input.float(2.5, "Stop Loss ATR Multiplier")
trail_atr_mult = input.float(3.0, "Trailing Stop ATR Multiplier")
stagnation_bars = input.int(20, "Max Bars in Trade (Stagnation)")

// --- Core Signal Calculation ---
float ret = math.log(close / close[1])

float p_mu = ta.sma(ret, lookback_p)
float p_var = ta.variance(ret, lookback_p)
float p_prec = p_var > 0 ? 1 / p_var : 0

float s_mu = ta.sma(ret, lookback_e)
float s_var = ta.variance(ret, lookback_e)
float s_prec = s_var > 0 ? 1 / s_var : 0

// MODIFIED: Removed the math.max(..., 0) clamp to allow for negative (short) signals
float f_bayes_raw = (p_prec * p_mu + s_prec * s_mu) * kelly_frac

// --- Volatility & Risk Calculation ---
float atr = ta.atr(atr_period)

// --- Entry Conditions ---
bool isLongEntry = f_bayes_raw > entry_threshold and strategy.position_size == 0
bool isShortEntry = f_bayes_raw < -entry_threshold and strategy.position_size == 0

// --- Exit Conditions ---
bool closeLongSignal = strategy.position_size > 0 and (f_bayes_raw < 0 or bar_index - strategy.opentrades.entry_bar_index(0) > stagnation_bars)
bool closeShortSignal = strategy.position_size < 0 and (f_bayes_raw > 0 or bar_index - strategy.opentrades.entry_bar_index(0) > stagnation_bars)

// --- Position Sizing ---
float stopLossDistance = atr * sl_atr_mult
float riskValue = strategy.equity * risk_percent
float positionSize = riskValue / stopLossDistance

// --- Execution Logic ---
if (isLongEntry)
    // 1. Calculate Stop Loss Price
    float stopLossPrice = close - stopLossDistance
    // 2. Enter Long
    strategy.entry("Long", strategy.long, qty=positionSize)
    // 3. Set Initial Stop Loss & Trailing Stop
    strategy.exit("Exit Long", from_entry="Long", stop=stopLossPrice, trail_price=close, trail_offset=atr * trail_atr_mult)

if (isShortEntry)
    // 1. Calculate Stop Loss Price
    float stopLossPrice = close + stopLossDistance
    // 2. Enter Short
    strategy.entry("Short", strategy.short, qty=positionSize)
    // 3. Set Initial Stop Loss & Trailing Stop
    strategy.exit("Exit Short", from_entry="Short", stop=stopLossPrice, trail_price=close, trail_offset=atr * trail_atr_mult)

// --- Exit Signal Execution ---
if (closeLongSignal)
    strategy.close("Long", comment="Signal Invalidated")

if (closeShortSignal)
    strategy.close("Short", comment="Signal Invalidated")

// --- Plotting for Visualization ---
plot(f_bayes_raw, "Bayesian Conviction", color=f_bayes_raw > 0 ? color.green : color.red)
hline(entry_threshold, "Long Threshold", color=color.new(color.green, 50), linestyle=hline.style_dashed)
hline(-entry_threshold, "Short Threshold", color=color.new(color.red, 50), linestyle=hline.style_dashed)
hline(0, "Zero Line", color=color.new(color.gray, 50))
```
    