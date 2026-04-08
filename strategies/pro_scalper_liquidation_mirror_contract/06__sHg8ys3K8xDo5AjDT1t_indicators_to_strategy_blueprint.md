
# Indicators to Strategy Blueprint

Here is the architectural breakdown for transforming the "Pro Scalper: Liquidation Mirror" indicator into a production-ready automated execution framework.

### 1. Execution Triggers (Entry & Direction)

The core logic of the indicator identifies potential exhaustion points, which we will use as our primary entry signals. The goal is to enter *after* the liquidation spike bar has closed, confirming the event.

*   **Long Entry Condition (`bottomFishing`):**
    *   `direction > 0` (Price is above the Supertrend, indicating a macro uptrend).
    *   `low <= ta.lowest(low[1], breakoutLen)` (Current bar's low makes a new low over the lookback period).
    *   `volume > (ta.sma(volume, 20) * volLimit)` (Volume spike confirms a potential capitulation/liquidation event).
    *   `ta.dmi(14, 14)[2] > adxMin` (ADX confirms the prior trend had sufficient strength to lead to a meaningful exhaustion).
    *   `(hist < 0 and hist > hist[1] * 1.02)` (MACD histogram shows decelerating bearish momentum).

*   **Short Entry Condition (`topEscaping`):**
    *   `direction < 0` (Price is below the Supertrend, indicating a macro downtrend).
    *   `high >= ta.highest(high[1], breakoutLen)` (Current bar's high makes a new high over the lookback period).
    *   `volume > (ta.sma(volume, 20) * volLimit)` (Volume spike confirms a potential euphoria/liquidation event).
    *   `ta.dmi(14, 14)[2] > adxMin` (ADX confirms the prior trend had sufficient strength).
    *   `(hist > 0 and hist < hist[1] * 1.02)` (MACD histogram shows decelerating bullish momentum).

#### Execution Nuances:

*   **Execution Timing:** The conditions rely on the `high`, `low`, and `volume` of the current bar. These values are only final at the **Close of the bar**. Attempting to execute intra-bar would lead to repainting and unreliable signals. Therefore, the strategy must be configured to `process_orders_on_close = true` or, more robustly, place orders that execute on the open of the *next* bar. This ensures the signal is fully confirmed before capital is committed.
*   **Signal Reversals:** This strategy is designed to catch reversals, not to stay in a trend indefinitely. If a `longCondition` appears while the strategy is already short, the system must first close the existing short position before entering the new long. This is handled by using unique IDs for long and short entries (`"Long"`, `"Short"`) and issuing a `strategy.close()` command for the opposing position immediately before the new `strategy.entry()` call.

### 2. Multi-Tiered Exit Logic

A static exit strategy is insufficient. We will implement a dynamic, multi-layered approach to manage the trade from inception to completion.

*   **Initial Stop Loss (Volatility-Based):**
    *   **Calculation:** The stop loss will be based on the Average True Range (ATR) to adapt to current market volatility. We will calculate a 14-period ATR (`ta.atr(14)`).
    *   **Placement:**
        *   For a **Long** trade, the initial stop loss will be placed at `entry_price - (ATR * atrMultiplier)`. A typical `atrMultiplier` is between 1.5 and 3.0.
        *   For a **Short** trade, the initial stop loss will be placed at `entry_price + (ATR * atrMultiplier)`.
    *   This ensures the trade has enough room to fluctuate without being stopped out by normal market noise.

*   **Take Profit / Trailing Mechanism:**
    We will employ a two-stage profit-taking mechanism to lock in gains while allowing for further profit potential.
    *   **Stage 1: Initial Profit Target (IPT):** A primary take-profit order is set at a multiple of the initial risk. For example, at a 1.5:1 Risk/Reward Ratio. If the initial risk (distance to stop) is `$R`, the IPT is placed at `entry_price + 1.5 * R` for longs and `entry_price - 1.5 * R` for shorts. This target can be used to scale out a portion of the position (e.g., 50%).
    *   **Stage 2: Dynamic Trailing Stop (Chandelier Exit):** After the IPT is hit (or from the trade's inception), a trailing stop is activated. A robust method is the Chandelier Exit, which trails the stop at a multiple of ATR from the highest high (for longs) or lowest low (for shorts) since the trade was entered.
        *   **Long Trail:** `Trail_Stop = ta.highest(high, bars_since_entry) - (ATR * trailAtrMultiplier)`
        *   **Short Trail:** `Trail_Stop = ta.lowest(low, bars_since_entry) + (ATR * trailAtrMultiplier)`
        The stop only ever moves in the direction of the trade.

*   **Time-Based Exits:**
    *   **End of Day (EOD) Exit:** For intraday trading, all open positions must be squared off before the market close to avoid overnight risk and funding charges. This is implemented by checking if the current bar's time is within a specified window before the session close (e.g., 15 minutes) and issuing a `strategy.close_all()` command.
    *   **Stagnation Exit:** If a trade is open for more than $X$ bars without hitting either its stop loss or take profit, it indicates a lack of momentum. We will implement a "Max Bars in Trade" exit to close such positions and free up capital for more promising opportunities.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term viability. We will move from fixed-lot sizing to a dynamic risk-based model.

*   **Risk-Based Sizing:**
    *   **Rule:** The strategy will risk a fixed percentage of total account equity on every single trade (e.g., 1% of equity).
    *   **Calculation:**
        1.  Define `riskPerTrade` (e.g., 0.01 for 1%).
        2.  Calculate `equity` using `strategy.equity`.
        3.  Calculate `dollarRisk` = `equity * riskPerTrade`.
        4.  Calculate `stopLossDistance` = `abs(entry_price - stop_loss_price)`.
        5.  Calculate `positionSize` = `dollarRisk / stopLossDistance`.
    *   This formula ensures that a losing trade will only reduce the account by the predefined percentage, regardless of the stop loss width in pips or points.

*   **Pyramiding & Scaling:**
    *   **Pyramiding (Scaling In):** For this specific "liquidation mirror" strategy, pyramiding is **not recommended**. The entry signal is an acute event (exhaustion spike) and is not expected to repeat in a way that provides a logical secondary entry point. Attempting to add to the position later would contradict the core premise of the strategy.
    *   **Scaling Out:** This is highly recommended. As defined in the exit logic, the strategy can be configured to sell a portion of the position at the Initial Profit Target (IPT).
        *   **Example:** Enter a trade with 1 contract. Set a `strategy.exit` call to close 0.5 contracts at the 1.5R target. The remaining 0.5 contracts are then managed by the trailing stop until they are stopped out. This requires managing two separate exit orders for the same entry.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the transition from the `indicator` to a professional `strategy`, incorporating the architectural components discussed above.

```pine
//@version=5
// STRATEGY DECLARATION: From Indicator to a full-fledged execution engine
strategy("Pro Scalper: Liquidation Mirror (Strategy)", 
     overlay=true, 
     pyramiding=0, // No pyramiding for this reversal strategy
     process_orders_on_close=true, // Ensures signals are confirmed before execution
     default_qty_type=strategy.calculated, // Enables risk-based position sizing
     commission_value=0.05, // Example commission per trade (in currency)
     slippage=2) // Example slippage in ticks

// ============================================================================
// 1. INPUTS (Original + New Risk/Exit Inputs)
// ============================================================================
// --- Original Inputs
grp_trend    = "Trend Filter"
emaLength    = input.int(50, "Macro EMA Length", group=grp_trend)
stFactor     = input.float(3.0, "Supertrend Factor", group=grp_trend)
stPeriod     = input.int(10, "Supertrend Period", group=grp_trend)
grp_rev      = "Reversal Logic"
breakoutLen  = input.int(20, "Lookback Length (High/Low)", group=grp_rev)
volLimit     = input.float(1.5, "Liquidation Vol Multiplier", step=0.1, group=grp_rev)
adxMin       = input.int(30, "Min ADX (Trend Strength)", group=grp_rev)

// --- New Risk & Exit Management Inputs
grp_risk     = "Risk & Exit Management"
riskPercent  = input.float(1.0, "Risk per Trade (%)", minval=0.1, maxval=10, group=grp_risk)
atrLen       = input.int(14, "ATR Length (for Stops)", group=grp_risk)
stopAtrMult  = input.float(2.0, "Stop Loss ATR Multiplier", group=grp_risk)
tpRr         = input.float(1.5, "Take Profit R:R", group=grp_risk, tooltip="Initial Take Profit based on Risk/Reward ratio.")
maxBarsInTrade = input.int(120, "Max Bars in Trade (Stagnation Exit)", group=grp_risk)

// ============================================================================
// 2. CORE CALCULATIONS (Unchanged + ATR)
// ============================================================================
macroEma = ta.ema(close, emaLength)
[_, direction] = ta.supertrend(stFactor, stPeriod)
[_, _, adx] = ta.dmi(14, 14)
[_, _, hist] = ta.macd(close, 12, 26, 9)
volSma = ta.sma(volume, 20)
isLiquidation = volume > (volSma * volLimit) 
momentumFade = (hist > 0 and hist < hist[1] * 1.02) or (hist < 0 and hist > hist[1] * 1.02)
adxHigh = adx > adxMin
recentHigh = ta.highest(high[1], breakoutLen)
recentLow  = ta.lowest(low[1], breakoutLen)
atr = ta.atr(atrLen)

// ============================================================================
// 3. ENTRY & EXIT LOGIC
// ============================================================================
// --- Entry Conditions
topEscaping = (direction < 0) and (high >= recentHigh) and isLiquidation and adxHigh and momentumFade
bottomFishing = (direction > 0) and (low <= recentLow) and isLiquidation and adxHigh and momentumFade

// --- Position Sizing Calculation
stopLossPoints = atr * stopAtrMult * syminfo.pointvalue
dollarRisk = (riskPercent / 100) * strategy.equity
positionSize = dollarRisk / stopLossPoints

// --- Execute Trades
if (strategy.position_size == 0) // Only take new trades if flat
    // LONG ENTRY
    if (bottomFishing)
        stopLossPrice = low - (atr * stopAtrMult)
        takeProfitPrice = close + (close - stopLossPrice) * tpRr
        strategy.entry("Long", strategy.long, qty=positionSize)
        strategy.exit("Long Exit", from_entry="Long", stop=stopLossPrice, limit=takeProfitPrice)

    // SHORT ENTRY
    if (topEscaping)
        stopLossPrice = high + (atr * stopAtrMult)
        takeProfitPrice = close - (stopLossPrice - close) * tpRr
        strategy.entry("Short", strategy.short, qty=positionSize)
        strategy.exit("Short Exit", from_entry="Short", stop=stopLossPrice, limit=takeProfitPrice)

// --- Time-Based Exit (Stagnation)
if (strategy.position_size != 0 and bar_index - strategy.opentrades.entry_bar_index(0) > maxBarsInTrade)
    strategy.close_all(comment="Stagnation Exit")

// --- EOD Exit (Example for US Equities session)
isEod = hour(time_close, "America/New_York") == 15 and minute(time_close, "America/New_York") >= 45
if isEod
    strategy.close_all(comment="EOD Exit")

// ============================================================================
// 4. VISUALS (Plotting stops and TPs for verification)
// ============================================================================
plot(strategy.position_size > 0 ? strategy.opentrades.entry_price(0) : na, "Entry Price", color.yellow, style=plot.style_linebr)
plot(strategy.position_size != 0 ? strategy.opentrades.stop_price(0) : na, "Stop Loss", color.red, style=plot.style_linebr)
plot(strategy.position_size != 0 ? strategy.opentrades.limit_price(0) : na, "Take Profit", color.green, style=plot.style_linebr)
```
    