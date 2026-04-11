
# Indicators to Strategy Blueprint

Based on the provided "1-Min Rapid Scalper" script, here is the architectural blueprint for its transformation into a production-grade automated execution system. The original script, while functional, relies on naive position sizing and a simplistic exit model. This overhaul introduces robust risk management and dynamic trade control suitable for the friction of live markets.

### 1. Execution Triggers (Entry & Direction)

The core entry logic is based on a specific three-bar Heikin Ashi (HA) momentum pattern, filtered by the ADX indicator.

*   **Long Entry Condition:** A `longTrigger` is `true` when:
    1.  The candle 3 bars ago was a bearish HA candle (`colorFlipBull`).
    2.  The candle 2 bars ago was a bullish HA candle with no lower wick (`bull1`).
    3.  The candle 1 bar ago was a bullish HA candle with no lower wick, and its body was larger than the previous candle's body (`bull2`).
    4.  The current (forming) candle is a bullish HA candle with no lower wick, and its body is larger than the previous candle's body (`bull3`).
    5.  The current ADX(7) value is greater than the `adxThresh` (default 20).

*   **Short Entry Condition:** A `shortTrigger` is `true` when:
    1.  The candle 3 bars ago was a bullish HA candle (`colorFlipBear`).
    2.  The candle 2 bars ago was a bearish HA candle with no upper wick (`bear1`).
    3.  The candle 1 bar ago was a bearish HA candle with no upper wick, and its body was larger than the previous candle's body (`bear2`).
    4.  The current (forming) candle is a bearish HA candle with no upper wick, and its body is larger than the previous candle's body (`bear3`).
    5.  The current ADX(7) value is greater than the `adxThresh` (default 20).

#### Execution Nuances & Reality Check

*   **Execution Timing:** The script uses `process_orders_on_close = false`. This is correct for a scalping strategy. It means the `longTrigger` or `shortTrigger` condition will be evaluated on every incoming tick. An order will be sent to the broker the instant the conditions are met intra-bar.
*   **Heikin Ashi vs. Real Price:** This is the most critical execution reality. The strategy logic is based on Heikin Ashi values, which are calculated averages (`(O+H+L+C)/4`, etc.). These are not tradable prices. The trigger will fire based on HA data, but the `strategy.entry` order will be filled at the **actual market bid/ask price** at that exact moment. This will inevitably cause a discrepancy (slippage) between the backtest's assumed entry and the live fill price. This must be accounted for with realistic slippage settings.
*   **Signal Reversals:** The current logic allows for a `longTrigger` to occur while the strategy is in a short position (and vice-versa). The default behavior of `strategy.entry` is to automatically close the existing position and open the new one in the opposite direction. For a production system, it's cleaner to explicitly close the existing trade first before entering the new one to ensure precise risk calculation on the new entry.

### 2. Multi-Tiered Exit Logic

The original script's default "exit after 1 bar" is too simplistic. The optional ATR exit is a good start, but we will professionalize it with dynamic layers.

*   **Initial Stop Loss (Volatility-Based):**
    *   **Calculation:** The stop loss will be placed at `Entry Price - (ATR * slMult)` for longs, and `Entry Price + (ATR * slMult)` for shorts. The `ATR` value used should be the one from the bar *preceding* the entry to avoid lookahead bias.
    *   **Catastrophic Stop:** The `thePlug` input acts as a failsafe. The final stop loss will be the *tighter* of the ATR stop and the fixed percentage stop (`math.max()` for long stops, `math.min()` for short stops). This prevents a single high-volatility candle from creating an unacceptably large risk exposure.

*   **Take Profit / Trailing Mechanism (Dynamic):**
    A static take-profit is suboptimal for a scalping strategy that needs to capture quick bursts of momentum. A multi-stage, dynamic exit is superior.
    1.  **Stage 1 - Initial Profit Target (Scaling Out):** Set an initial take-profit target at a 1:1 Risk/Reward ratio (i.e., `Entry Price + (ATR * slMult)` for longs). At this level, exit 50% of the position.
    2.  **Stage 2 - Breakeven Stop:** Once the Stage 1 target is hit, the stop loss for the remaining 50% of the position is moved to the entry price (breakeven). This makes the remainder of the trade risk-free.
    3.  **Stage 3 - Trailing Stop:** For the remaining position, activate a trailing stop to capture further momentum. A simple but effective method is to trail the stop below the low of the previous 2-3 candles for a long, or above the high of the previous 2-3 candles for a short. This allows the trade to continue as long as momentum persists but quickly exits when it reverses.

*   **Time-Based Exits:**
    *   **Stagnation Exit:** For a scalper, capital velocity is key. If a trade has been open for more than $X$ bars (e.g., 10-15 bars on a 1-min chart) and has not hit either the Stage 1 TP or the stop loss, it should be closed. The trade is "stagnant," and the capital is better deployed on a new opportunity.
    *   **End-of-Session Exit:** All open positions should be squared up 5-10 minutes before the market close to avoid overnight risk and gap volatility.

### 3. Capital Allocation & Risk Management

This is the most significant overhaul. The original `percent_of_equity` sizing is flawed because it does not account for trade-specific volatility. We will implement a professional risk-based sizing model.

*   **Risk-Based Sizing:**
    *   **The Rule:** The strategy will risk a fixed percentage of account equity (e.g., 0.5% - 1.0%) on every single trade.
    *   **The Math:**
        1.  Define `risk_per_trade_percent` (e.g., `1.0`).
        2.  Calculate `equity_at_risk = strategy.equity * (risk_per_trade_percent / 100)`.
        3.  Calculate `risk_per_share_in_dollars` = `abs(entry_price - stop_loss_price)`.
        4.  **`position_size = equity_at_risk / risk_per_share_in_dollars`**.
    *   This calculation ensures that whether the stop loss is wide (high volatility) or tight (low volatility), the maximum potential loss from the trade is always capped at the desired percentage of total equity.

*   **Pyramiding & Scaling:**
    *   **Pyramiding (Adding to Winners):** **Not Recommended** for this specific strategy. The entry signal is a very specific, short-term reversal/momentum-ignition pattern. It is not designed to identify a sustained trend. Attempting to pyramid into such a move adds significant complexity and risk. The focus should be on a clean entry and exit.
    *   **Scaling Out:** **Highly Recommended.** As detailed in the exit logic, scaling out is a core component of the professional framework. It secures partial profits, reduces psychological pressure, and allows a portion of the trade to run for potentially larger gains.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the transition to a professional `strategy()` with risk-based sizing and a bracketed exit order.

```pine
// ==============================================================================
// 0. STRATEGY DEFINITION - PRODUCTION GRADE
// ==============================================================================
// Note: default_qty_type is now strategy.fixed because we calculate it manually.
// Slippage and Commission are CRITICAL for scalping reality.
strategy('1-Min Rapid Scalper (Pro Framework)',
     overlay = true,
     process_orders_on_close = false,
     initial_capital = 10000,
     default_qty_type = strategy.fixed, // We now calculate size manually
     commission_type = strategy.commission.percent,
     commission_value = 0.04, // Realistic broker commission
     slippage = 2) // Realistic slippage in ticks for a 1-min chart

// ==============================================================================
// 3. CAPITAL & RISK INPUTS
// ==============================================================================
grp_risk = 'Risk & Capital Management'
riskPercent = input.float(1.0, title = 'Risk Per Trade (%)', minval = 0.1, maxval = 5.0, step = 0.1, group = grp_risk)
stagnationBars = input.int(15, title = 'Exit if Stagnant after X Bars', group = grp_exit)

// [Keep original Indicator and Logic sections here]
// ... longTrigger and shortTrigger logic remains the same ...

// ==============================================================================
// 4. PROFESSIONAL EXECUTION ENGINE
// ==============================================================================
// Calculate Stop Loss Price BEFORE entry to determine position size
atrVal = ta.atr(atrLen)
longStopPrice = close - atrVal * slMult
shortStopPrice = close + atrVal * slMult

// Calculate Risk-Based Position Size
riskPerShare = math.abs(close - longStopPrice)
equityToRisk = (riskPercent / 100) * strategy.equity
positionSize = riskPerShare > 0 ? math.floor(equityToRisk / riskPerShare) : 0

// --- ENTRY & BRACKET EXIT LOGIC ---
if (longTrigger and strategy.position_size == 0 and positionSize > 0)
    // Define TP1 (1:1 R/R) and Final Stop
    tp1_price = close + riskPerShare
    sl_price = longStopPrice

    // Entry Order
    strategy.entry('Long', strategy.long, qty = positionSize, comment = 'Long Entry')
    // Exit Bracket: Scale out 50% at TP1, place final stop for 100%
    strategy.exit('Exit Long TP1/SL', from_entry = 'Long', qty_percent = 50, limit = tp1_price)
    strategy.exit('Exit Long SL', from_entry = 'Long', stop = sl_price)

if (shortTrigger and strategy.position_size == 0 and positionSize > 0)
    // Define TP1 (1:1 R/R) and Final Stop
    riskPerShareShort = math.abs(close - shortStopPrice)
    tp1_price = close - riskPerShareShort
    sl_price = shortStopPrice

    // Entry Order
    strategy.entry('Short', strategy.short, qty = positionSize, comment = 'Short Entry')
    // Exit Bracket: Scale out 50% at TP1, place final stop for 100%
    strategy.exit('Exit Short TP1/SL', from_entry = 'Short', qty_percent = 50, limit = tp1_price)
    strategy.exit('Exit Short SL', from_entry = 'Short', stop = sl_price)

// --- DYNAMIC TRADE MANAGEMENT (POST-ENTRY) ---
// Breakeven + Trailing Stop Logic
// This logic runs on every bar after the initial TP is hit.
// (This is a simplified representation; a full implementation would use vars to track state)
isTradeMature = strategy.opentrades.size(strategy.opentrades.last) < positionSize and strategy.position_size > 0
if isTradeMature
    strategy.exit('BE Trail', from_entry = 'Long', stop = strategy.opentrades.entry_price(strategy.opentrades.last)) // Move to Breakeven

// Stagnation Exit
if strategy.position_size != 0 and bar_index - strategy.opentrades.entry_bar_index(strategy.opentrades.last) >= stagnationBars
    strategy.close_all(comment = 'Stagnation Exit')

```
    