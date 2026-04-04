
# Indicators Description

### 1. Component Deconstruction

#### **Structure & Volatility Indicators**

*   **Pivot High / Pivot Low (`ta.pivothigh`, `ta.pivotlow`)**
    *   **Specific Configuration:**
        *   `source`: `high` for `pivothigh`, `low` for `pivotlow`.
        *   `leftbars`: `pivotLength` (default: 3).
        *   `rightbars`: `pivotLength` (default: 3).
    *   **Functional Modification:** The script does not use the raw output of these functions directly in the trigger logic. Instead, it uses them to update two state variables, `lastPivotHigh` and `lastPivotLow`, declared with the `var` keyword. This modification transforms the pivot functions from a simple plot into a stateful tracking mechanism. The variables `lastPivotHigh` and `lastPivotLow` perpetually hold the price value of the *most recently confirmed* pivot, creating a dynamic, horizontal support/resistance level that persists across subsequent bars until a new pivot is formed.

*   **Average True Range (`ta.atr`)**
    *   **Specific Configuration:**
        *   `length`: 14 (hard-coded). This is the standard lookback period.
    *   **Functional Modification:** The ATR value is not used as a direct signal. Its sole purpose is to serve as a dynamic unit of measurement for market volatility. It is multiplied by the `atrMult` input to create a volatility-normalized threshold (`currentAtr * atrMult`). This threshold defines the maximum allowable price range for a valid consolidation period.

*   **Highest High / Lowest Low (`ta.highest`, `ta.lowest`)**
    *   **Specific Configuration:**
        *   `source`: `high[1]` for `ta.highest`, `low[1]` for `ta.lowest`. The `[1]` offset is critical, as it ensures the calculation is performed on *closed* bars, excluding the current, developing bar.
        *   `length`: `consLength` (default: 10).
    *   **Functional Modification:** These functions are used to calculate the `zoneRange` (`pastHigh - pastLow`). This value represents the absolute price range over the specified lookback period. It is the core input for the consolidation filter.

### 2. Logic Layering & Confluence

The script's engine is built on a hierarchical filtering model where multiple conditions must be met sequentially or concurrently. A signal is only generated if it passes through all logical gates.

*   **Gatekeeper 1: The Consolidation Filter (`isConsolidating`)**
    *   **Interaction Dynamics:** This is the primary filter that determines if the market is in a state of low-volatility compression suitable for a breakout.
    *   **Hierarchical Filtering:** It acts as the first major gatekeeper. The logic is `zoneRange <= (currentAtr * atrMult)`.
        *   The script first calculates the absolute price range (`zoneRange`) over the `consLength` period.
        *   It then calculates the maximum allowed range by scaling the current 14-period ATR by the `atrMult` factor.
        *   A `true` state is achieved only if the observed range is less than or equal to this volatility-adjusted threshold. If this condition is false, no breakout signal can be generated, regardless of price action. This filter effectively reduces the signal-to-noise ratio by ignoring breakouts from high-volatility, choppy environments.

*   **Gatekeeper 2: The Cooldown Filter (`canEnter`)**
    *   **Interaction Dynamics:** This filter is purely temporal and serves to prevent signal clustering and over-trading.
    *   **Hierarchical Filtering:** It functions as a time-based lock. The logic is `bar_index - lastTradeBar >= cooldownBars`.
        *   The script maintains a state variable, `lastTradeBar`, which records the bar index of the last valid signal.
        *   For each new bar, it checks if the number of bars elapsed since the last signal is greater than or equal to the `cooldownBars` input.
        *   If this condition is false, the `canEnter` flag is `false`, and the execution engine is blocked, even if all other structural and volatility conditions are met.

*   **Gatekeeper 3: The Structural Confirmation Filter**
    *   **Interaction Dynamics:** This filter validates the integrity of the breakout move itself, looking for a classic market structure pattern.
    *   **Hierarchical Filtering:** This is a final confirmation layer applied concurrently with the breakout trigger.
        *   **For Longs:** The condition `low > lastPivotLow` must be true. This ensures that the breakout candle's low is higher than the most recent significant swing low. It filters out breakouts where price spikes above resistance but immediately violates the underlying support structure.
        *   **For Shorts:** The condition `high < lastPivotHigh` must be true. This confirms the breakout candle's high is lower than the most recent significant swing high, validating the bearish pressure.

### 3. The Execution Engine

The final trigger is a boolean AND operation across all filtered conditions.

*   **Boolean Logic: Long Trigger (`bullishBreakout`)**
    A `true` signal is returned if and only if all of the following are true on the same bar:
    1.  `isBullishCandle`: The candle's `close` is greater than its `open`.
    2.  `ta.crossover(close, lastPivotHigh)`: The closing price crosses *above* the stored value of the last confirmed pivot high. This is the primary catalyst.
    3.  `low > lastPivotLow`: The low of the breakout candle is higher than the last confirmed pivot low (Structural Confirmation).
    4.  `isConsolidating`: The Consolidation Filter is permissive.
    5.  `canEnter`: The Cooldown Filter is permissive.

*   **Boolean Logic: Short Trigger (`bearishBreakout`)**
    A `true` signal is returned if and only if all of the following are true on the same bar:
    1.  `isBearishCandle`: The candle's `close` is less than its `open`.
    2.  `ta.crossunder(close, lastPivotLow)`: The closing price crosses *below* the stored value of the last confirmed pivot low. This is the primary catalyst.
    3.  `high < lastPivotHigh`: The high of the breakout candle is lower than the last confirmed pivot high (Structural Confirmation).
    4.  `isConsolidating`: The Consolidation Filter is permissive.
    5.  `canEnter`: The Cooldown Filter is permissive.

*   **Mathematical Constants & Risk Profile**
    *   **Risk Calculation:** The risk is defined structurally, not by a fixed value or volatility measure.
        *   **Long Risk:** `risk = entryPrice - stopPrice`, where `entryPrice` is `close` and `stopPrice` is `lastPivotLow`.
        *   **Short Risk:** `risk = stopPrice - entryPrice`, where `entryPrice` is `close` and `stopPrice` is `lastPivotHigh`.
        This methodology ties the trade's risk directly to the market structure preceding the breakout. A wider consolidation range naturally results in a larger initial risk.
    *   **Target Calculation (`targetMult`):** The take-profit level is a direct function of the calculated risk.
        *   **Long Target:** `targetPrice = entryPrice + (risk * targetMult)`.
        *   **Short Target:** `targetPrice = entryPrice - (risk * targetMult)`.
        The `targetMult` (default: 2.0) is a hard-coded multiplier that establishes a fixed risk-to-reward ratio for every trade. A value of 2.0 enforces a 1:2 risk-reward profile. This constant is the primary determinant of the strategy's potential profitability per trade.
    