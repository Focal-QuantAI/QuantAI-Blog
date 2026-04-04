
# Indicators Description

### 1. Component Deconstruction

#### **Average True Range (ATR)**
*   **Specific Configuration:**
    *   **Function:** `ta.atr()`
    *   **Length:** `atrLengthInput` = `200` (default). This is a long lookback period, designed to capture a stable, long-term volatility baseline rather than reacting to short-term spikes.
    *   **Smoothing:** The `ta.atr` function uses a Wilder's Smoothing Method (an RMA, which is equivalent to an EMA with `alpha = 1 / length`).
    *   **Price Source:** Implicitly uses `high`, `low`, and `close` to calculate the True Range on each bar.
*   **Functional Modification:**
    *   The ATR value itself is not modified. However, its application is dynamic. It serves as the fundamental unit for three distinct calculations: the initial stop distance, the adjustment trigger threshold, and the minimum proximity guardrail.

#### **Base Volatility Bands**
*   **Specific Configuration:**
    *   **Upper Band Formula:** `high + atrMultInput * atr`
    *   **Lower Band Formula:** `low - atrMultInput * atr`
    *   **ATR Multiplier:** `atrMultInput` = `3.0` (default).
    *   **Price Source:** The basis for the bands are the raw `high` and `low` of the current bar, not a smoothed moving average. This makes the bands highly reactive to the immediate price action for the purpose of trend flips.
*   **Functional Modification:**
    *   These are not standard Bollinger or Keltner Channels. They function solely as the initial placement level for the `trailingStop` immediately after a trend direction flips. They do not act as a continuous channel but as a one-time calculation upon a crossover event.

#### **Custom Sigmoid Function**
*   **Specific Configuration:**
    *   The function `sigmoid(float t)` takes a single argument `t`, which represents the normalized progress of the adjustment phase (a value from 0.0 to 1.0).
*   **Functional Modification:**
    *   This is a custom, re-scaled, and normalized sigmoid function. Its mathematical anatomy is as follows:
        1.  **Input Clamping:** `clamp(t, 0.0, 1.0)` ensures the progress variable `t` never exceeds its logical bounds.
        2.  **Input Scaling:** `x = -6.0 + 12.0 * ...` This is the critical step. It maps the normalized progress `t` from its `[0.0, 1.0]` domain to a `[-6.0, 6.0]` domain. This specific range is chosen because it represents the most dynamic part of the standard logistic curve `1 / (1 + e^-x)`. Outside this range, the curve is nearly flat (asymptotic to 0 or 1). This scaling ensures the *entire* S-curve transition is utilized over the `sigLengthInput` duration.
        3.  **Boundary Calculation:** `sMin` and `sMax` calculate the sigmoid function's raw output at the scaled boundaries of -6.0 and +6.0.
        4.  **Standard Sigmoid:** `sig = 1.0 / (1.0 + math.exp(-x))` calculates the raw sigmoid value for the scaled input `x`.
        5.  **Output Normalization:** `(sig - sMin) / (sMax - sMin)` re-scales the output. The raw sigmoid output for an input of `[-6, 6]` is approximately `[0.0025, 0.9975]`. This final step normalizes this output to a clean, predictable `[0.0, 1.0]` range. This normalized value, `sigFactor`, can then be used as a reliable percentage multiplier for the adjustment.

### 2. Logic Layering & Confluence

The script's engine is a **state machine** with two primary states: **"Baseline Trailing"** and **"Sigmoid Adjustment."**

#### **State 1: Baseline Trailing (The Gatekeeper)**
*   **Interaction Dynamics:** This is the default state. The primary logic is a simple **Threshold Cross**. The `trailingStop` value persists from the previous bar, only moving in the direction of the trend (e.g., `math.max(trailingStop, candidate)` for a bull trend).
*   **Hierarchical Filtering:** The `direction` variable acts as the highest-level filter. All subsequent logic is conditional on whether the trend is bullish (`1`) or bearish (`-1`). A trend flip, defined by `close` crossing the `trailingStop`, acts as a hard reset for the entire system, forcing it back into this baseline state and resetting all adjustment-related variables (`isAdjusting`, `sigCounter`).

#### **State 2: Sigmoid Adjustment (The Active Phase)**
*   **Interaction Dynamics:** The transition from State 1 to State 2 is the core of the script's intelligence. It is not based on a simple indicator cross but on a **volatility-based distance threshold**.
    *   **Trigger Condition:** The engine continuously calculates `currentDist`, the absolute distance between `close` and `trailingStop`.
    *   **Confluence:** The trigger fires only when `currentDist` exceeds `kDist` (`atrMultInput * atr`). This is a confluence event: the price has not only maintained its trend but has accelerated away from its trailing stop by a distance greater than the initial, volatility-defined buffer. This is interpreted as a high signal-to-noise ratio, confirming trend strength.
*   **Hierarchical Filtering:** The trigger condition `not isAdjusting and currentDist > kDist` acts as a gate. The `not isAdjusting` check prevents the trigger from firing repeatedly during an ongoing adjustment. Once triggered:
    1.  The `isAdjusting` flag is set to `true`, locking the system into the adjustment state.
    2.  The current `trailingStop` is stored in `startLevel`.
    3.  The total adjustment magnitude, `targetOffset`, is calculated as `sigAmpMultInput * atr`.
    4.  The `sigCounter` begins, feeding the `sigmoid()` function to calculate the `sigFactor`.
    5.  The `trailingStop` is now calculated based on the sigmoid transition, moving from `startLevel` towards price.

### 3. The Execution Engine

#### **Trend Flip Trigger**
*   **Boolean Logic (Bull to Bear):** `direction[1] == 1 and close < trailingStop[1]`
*   **Boolean Logic (Bear to Bull):** `direction[1] == -1 and close > trailingStop[1]`
*   **Effect:** When `true`, `direction` is inverted, `isAdjusting` is reset to `false`, `sigCounter` is reset to `0`, and the `trailingStop` is repositioned using the Base Volatility Bands (`upperBand` or `lowerBand`).

#### **Adjustment Phase Trigger**
*   **Boolean Logic:** `not isAdjusting and ((direction == 1 ? close - trailingStop : trailingStop - close) > atrMultInput * atr)`
*   **Effect:** When `true`, the system enters the "Sigmoid Adjustment" state. `isAdjusting` becomes `true`, and the `sigCounter` is initiated.

#### **Adjustment Phase Execution & Exit**
*   **Execution Logic:** During each bar where `isAdjusting` is `true`:
    1.  **Progress Calculation:** `t = float(sigCounter) / float(sigLengthInput)` normalizes the time elapsed.
    2.  **Adjustment Factor:** `sigFactor = sigmoid(t)` calculates the non-linear adjustment multiplier (from 0.0 to 1.0).
    3.  **Target Stop Level:** A `candidate` stop is calculated:
        *   **Bull:** `startLevel + (targetOffset * sigFactor)`
        *   **Bear:** `startLevel - (targetOffset * sigFactor)`
    4.  **Trailing Constraint:** The final `trailingStop` is updated using `math.max` (for bull) or `math.min` (for bear) to ensure it only ever moves closer to the price, never away from it.
*   **Exit Logic:** The adjustment phase terminates and `isAdjusting` is set to `false` if either of two conditions is met:
    1.  **Proximity Guard:** `(direction == 1 ? close - candidate : candidate - close) < minDist`. The proposed `candidate` stop is within the minimum allowed distance (`minDistMultInput * atr`) from the current `close`. This is a safety override to prevent the stop from getting too tight.
    2.  **Timeout:** `sigCounter >= sigLengthInput`. The adjustment has run for its full, pre-defined duration in bars.
*   **Mathematical Constants:**
    *   `atrMultInput` (3.0): Governs the **risk profile** and **trigger sensitivity**. A higher value creates a wider initial stop (more room for volatility) but requires a stronger price move to initiate the adjustment phase.
    *   `sigAmpMultInput` (3.0): Controls the **aggressiveness of profit protection**. It defines the total distance, in ATR units, that the stop will travel towards the price during the transition. A higher value results in a much tighter final stop.
    *   `minDistMultInput` (0.5): A **risk management constraint**. It defines a hard floor on how tight the stop can become, ensuring a minimum "breathing room" of `0.5 * ATR` is maintained, preventing premature stop-outs from minor price fluctuations.
    *   `sigLengthInput` (20): Dictates the **duration of the transition**. A shorter length results in a faster, more aggressive tightening of the stop. A longer length creates a smoother, more gradual adjustment.
    