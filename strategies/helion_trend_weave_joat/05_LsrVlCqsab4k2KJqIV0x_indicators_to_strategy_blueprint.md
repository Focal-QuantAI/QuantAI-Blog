
# Indicators to Strategy Blueprint

The "Helion Trend Weave" is an exceptionally well-structured indicator, providing a rich dataset on trend phase, strength, and structure. To weaponize this for automated execution, we must translate its descriptive signals into prescriptive trading commands, focusing on robust entry triggers, dynamic risk management, and realistic market friction.

The core philosophy will be to use the primary trend signals for entries while leveraging the indicator's rich contextual data (alignment, spread, compression) as filters to improve signal quality.

### 1. Execution Triggers (Entry & Direction)

The indicator provides multiple signal types. For a primary trend-following strategy, the "Weave Cross" (`bullCross`/`bearCross`) and "Momentum Surge" (`surgeUp`/`surgeDn`) are the most potent entry triggers. We will combine them with filters to create high-probability entry conditions.

*   **Long Entry Condition (`enterLong`):**
    A long position is initiated when a primary bullish signal occurs, but only if the market structure supports the move. We will require either a confirmed bullish crossover or a momentum surge, filtered by minimum trend alignment.
    ```
    // A. Primary Bullish Crossover
    isBullCrossSignal = bullCross and (alignPct > 50) // Cross with at least 50% filament alignment

    // B. Bullish Momentum Surge
    isBullSurgeSignal = surgeUp and (spreadPct > 30) // Squeeze release with developing or dominant force

    // C. Final Entry Trigger
    enterLong = (isBullCrossSignal or isBullSurgeSignal) and strategy.position_size == 0
    ```

*   **Short Entry Condition (`enterShort`):**
    The logic mirrors the long condition for bearish setups.
    ```
    // A. Primary Bearish Crossover
    isBearCrossSignal = bearCross and (alignPct > 50) // Cross with at least 50% filament alignment

    // B. Bearish Momentum Surge
    isBearSurgeSignal = surgeDn and (spreadPct > 30) // Squeeze release with developing or dominant force

    // C. Final Entry Trigger
    enterShort = (isBearCrossSignal or isBearSurgeSignal) and strategy.position_size == 0
    ```

*   **Execution Nuances:**
    *   **Execution Timing:** All entries will be executed **at the close of the bar** where the condition becomes true. The original script's use of `barstate.isconfirmed` already enforces this, which is critical for preventing repaint and ensuring backtest results are replicable. Executing on real-time price action against a multi-MA ribbon is prone to severe whipsaws and is not recommended for a foundational strategy.
    *   **Signal Reversals:** The system is designed as "flip-and-reverse." If the strategy is in a long position and a valid `enterShort` signal occurs, the long position will be closed, and a new short position will be opened on the same bar. This is managed by using the same `id` for both `strategy.entry` calls.

### 2. Multi-Tiered Exit Logic

A static stop or take profit is insufficient. The exit logic must adapt to the volatility and character of the trend as defined by the Helion Weave itself.

*   **Initial Stop Loss (Volatility-Based):**
    The stop loss will be placed based on a combination of the ribbon's structure and current market volatility (ATR). This provides a more logical and dynamic buffer than a fixed percentage.
    *   **Calculation:** For a long entry, the stop is placed a fraction of an ATR below the `slowest` MA. For a short, it's placed above. This uses the ribbon's "anchor" filament as a dynamic support/resistance zone.
    *   **Logic:**
        *   `longStopPrice = slowest - (atrVal * 0.75)`
        *   `shortStopPrice = slowest + (atrVal * 0.75)`
    This ensures the stop is outside the main body of the trend structure, giving the trade room to breathe.

*   **Take Profit / Trailing Stop:**
    We will implement a two-stage profit-taking mechanism to lock in gains while allowing for runners.
    1.  **Initial Target (TP1):** A fixed Risk-to-Reward target. For example, at 1.5R, we close 50% of the position. The distance from entry to the initial stop loss defines 1R.
    2.  **Trailing Stop (ATR-Based):** After TP1 is hit, the stop loss for the remaining position is moved to breakeven. From there, it begins to trail the price using an ATR multiple. The `slowest` MA itself can serve as an excellent trailing stop line.
    *   **Trailing Logic:** Once in a profitable long trade, the stop loss will be `max(current_stop, slowest - atrVal * 0.25)`. This ensures the stop only moves up, never down, and trails just under the ribbon's anchor.

*   **Time-Based & Conditional Exits:**
    *   **Trend Exhaustion Exit:** The indicator's `driftWarn` signal is a purpose-built exit trigger. It fires when trend strength decays into the "DORMANT" phase. We will use this to exit any remaining position.
        *   `exitOnDrift = driftWarn and strategy.position_size != 0`
    *   **End of Day (EOD) Exit:** For intraday timeframes, a hard time-based exit is non-negotiable. The strategy will be configured to square all open positions 15 minutes before the session close.
    *   **Stagnation Exit:** If a trade is open for more than $X$ bars (e.g., 50) and has not reached TP1, it signifies a dead market. The position will be closed to free up capital.

### 3. Capital Allocation & Risk Management

Position sizing is the most critical component for long-term viability. We will use a fixed-fractional risk model.

*   **Risk-Based Sizing:**
    The strategy will risk a fixed percentage of the total account equity on every single trade.
    *   **Inputs:**
        *   `riskPercent`: User-defined percentage of equity to risk (e.g., 1.0 for 1%).
    *   **Logic:**
        1.  Calculate the dollar amount to risk: `riskAmount = strategy.equity * (riskPercent / 100)`.
        2.  Calculate the risk per share/contract: `riskPerUnit = abs(entry_price - stop_loss_price)`.
        3.  Calculate the final position size: `positionSize = riskAmount / riskPerUnit`.
    This logic ensures that losses are capped at the desired percentage, regardless of the trade's volatility or stop distance.

*   **Pyramiding & Scaling:**
    *   **Pyramiding:** The `fanBull` and `fanBear` signals are ideal triggers for adding to a winning position. They confirm that the trend has achieved perfect structural alignment.
        *   **Rule:** If in a profitable long position (e.g., price is > entry + 1R) and a `fanBull` signal occurs, a second unit (e.g., 50% of the initial size) can be added. The stop loss for the *entire* combined position is then moved to the original entry price (breakeven).
    *   **Scaling Out:** As defined in the exit logic, the strategy will automatically scale out by closing a portion of the position at TP1. This reduces risk and improves the strategy's win rate and psychological profile.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the `indicator` into a `strategy`, incorporating the professional execution logic defined above. It should be appended to or integrated with the existing script code.

```pine
// ══════════════════════════════════════════════════════════════════════════════
//                  STRATEGY EXECUTION FRAMEWORK
// ══════════════════════════════════════════════════════════════════════════════

// ──────────────────── STRATEGY CONFIGURATION ────────────────────────────────
//@version=5
// Convert indicator to strategy and set realistic execution parameters
strategy("Helion Trend Weave [Strategy]", 
     overlay=true, 
     initial_capital=100000,
     pyramiding=1, // Allow one additional entry per direction
     commission_type=strategy.commission.percent,
     commission_value=0.04, // Realistic commission for futures/crypto
     slippage=2) // 2 ticks of slippage on market orders

// ──────────────────── STRATEGY INPUTS ───────────────────────────────────────
string GRP_RISK = "Risk & Position Sizing"
float riskPercent      = input.float(1.0, "Risk Per Trade (%)", minval=0.1, maxval=5.0, step=0.1, group=GRP_RISK)
float atrStopMult      = input.float(0.75, "ATR Stop Offset Multiplier", minval=0.25, maxval=3.0, group=GRP_RISK)
float takeProfitRR     = input.float(1.5, "Take Profit 1 (R:R)", minval=1.0, maxval=5.0, group=GRP_RISK)
bool  useDriftExit     = input.bool(true, "Use Drift Fade for Exit?", group=GRP_RISK)
bool  usePyramiding    = input.bool(true, "Pyramid on 'Fan' Signal?", group=GRP_RISK)

// ──────────────────── ENTRY & EXIT CONDITIONS ───────────────────────────────
// Entry Logic
isBullCrossSignal = bullCross and (alignPct > 50)
isBullSurgeSignal = surgeUp and (spreadPct > 30)
enterLong = (isBullCrossSignal or isBullSurgeSignal) and strategy.position_size <= 0

isBearCrossSignal = bearCross and (alignPct > 50)
isBearSurgeSignal = surgeDn and (spreadPct > 30)
enterShort = (isBearCrossSignal or isBearSurgeSignal) and strategy.position_size >= 0

// Exit Logic
exitOnDrift = useDriftExit and driftWarn and strategy.position_size != 0

// Pyramiding Logic
pyramidLong = usePyramiding and fanBull and strategy.position_size > 0 and close > (strategy.opentrades.entry_price(0) + strategy.opentrades.entry_price(0) - (slowest[1] - atrVal[1] * atrStopMult))
pyramidShort = usePyramiding and fanBear and strategy.position_size < 0 and close < (strategy.opentrades.entry_price(0) - ((slowest[1] + atrVal[1] * atrStopMult) - strategy.opentrades.entry_price(0)))

// ──────────────────── POSITION SIZING & EXECUTION ───────────────────────────
// Calculate Stop Loss levels for sizing calculation
longStopPrice  = slowest - (atrVal * atrStopMult)
shortStopPrice = slowest + (atrVal * atrStopMult)

// Dynamic Risk-Based Position Sizing
riskAmount     = strategy.equity * (riskPercent / 100)
riskPerUnitLng = close - longStopPrice
riskPerUnitSht = shortStopPrice - close
positionSizeLng = riskAmount / riskPerUnitLng
positionSizeSht = riskAmount / riskPerUnitSht

// Calculate Take Profit levels
longTakeProfitPrice = close + (riskPerUnitLng * takeProfitRR)
shortTakeProfitPrice = close - (riskPerUnitSht * takeProfitRR)

// --- EXECUTION ENGINE ---
if (enterLong)
    strategy.entry("HTW_Long", strategy.long, qty=positionSizeLng, comment="Enter Long")
    strategy.exit("TP1/SL_L", "HTW_Long", qty_percent=50, profit=longTakeProfitPrice, stop=longStopPrice, comment="TP1/SL Long")
    strategy.exit("SL2_L", "HTW_Long", stop=longStopPrice, comment="Final SL Long")

if (enterShort)
    strategy.entry("HTW_Short", strategy.short, qty=positionSizeSht, comment="Enter Short")
    strategy.exit("TP1/SL_S", "HTW_Short", qty_percent=50, profit=shortTakeProfitPrice, stop=shortStopPrice, comment="TP1/SL Short")
    strategy.exit("SL2_S", "HTW_Short", stop=shortStopPrice, comment="Final SL Short")

// Pyramid Entry
if (pyramidLong)
    strategy.entry("HTW_L_Pyr", strategy.long, qty=positionSizeLng * 0.5, comment="Pyramid Long")
    // Move stop for original trade to breakeven
    strategy.exit("SL2_L", "HTW_Long", stop=strategy.opentrades.entry_price(0), comment="Move SL to B/E")

if (pyramidShort)
    strategy.entry("HTW_S_Pyr", strategy.short, qty=positionSizeSht * 0.5, comment="Pyramid Short")
    strategy.exit("SL2_S", "HTW_Short", stop=strategy.opentrades.entry_price(0), comment="Move SL to B/E")

// Conditional Exit
if (exitOnDrift)
    strategy.close_all(comment="Exit on Drift Fade")

// EOD Exit (Example for daily session, adjust session time as needed)
isEod = time_close("1D") and strategy.position_size != 0
if (isEod)
    strategy.close_all(comment="EOD Exit")

```
    