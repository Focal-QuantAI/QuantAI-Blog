
# Indicators Description

### 1. Component Deconstruction

#### **Average True Range (ATR)**
*   **Specific Configuration:**
    *   **Price Source:** Implicit `hlc3` (High, Low, Close average) as per the standard `ta.atr` function implementation.
    *   **Lookback Period:** `atrLengthInput` = `200` bars (default). This long period is chosen to establish a stable, long-term volatility baseline, reducing the influence of short-term volatility spikes on the stop's placement.
*   **Functional Modification:**
    *   The ATR value itself is not modified. It is a standard, unmodified `ta.atr()` calculation. Its application, however, is central to every dynamic calculation in the script, serving as the fundamental unit of risk and distance.

#### **Custom Sigmoid Function (`sigmoid(t)`)**
This is a non-standard, engineered component designed to create a specific, non-linear transition curve.
*   **Specific Configuration:**
    *   **Input `t`:** A normalized progress value from `0.0` to `1.0`, representing the fraction of the transition's duration (`sigLengthInput`) that has passed.
*   **Functional Modification (Mathematical Logic):**
    1.  **Range Mapping:** The input `t` (range `[0, 1]`) is first mapped to a new variable `x` in the range `[-6, 6]` via the linear transformation `x = -6.0 + 12.0 * t`. This range is deliberately chosen because it covers the steepest part of the standard logistic curve, providing the most significant change.
    2.  **Standard Logistic Function:** The script calculates `sig = 1.0 / (1.0 + math.exp(-x))`. This is the mathematical definition of a sigmoid (logistic) function. As `x` moves from -6 to +6, `sig` moves from a value very close to 0 to a value very close to 1.
    3.  **Output Normalization:** The raw `sig` value does not perfectly span the `[0, 1]` range. To correct this, the script pre-calculates the theoretical minimum (`sMin` at x=-6) and maximum (`sMax` at x=+6) and normalizes the output using the formula: `(sig - sMin) / (sMax - sMin)`.
    *   **Intended Effect:** This complex modification ensures the function returns a value that starts at exactly `0.0`, accelerates smoothly through `0.5` at the midpoint of the transition, and decelerates to end at exactly `1.0`. This creates an "S-shaped" adjustment curve, which is slow at the beginning and end but rapid in the middle. This is designed to move the trailing stop aggressively during the core of a momentum burst while being gentle at the start and finish of the adjustment phase.

---

### 2. Logic Layering & Confluence

The script's engine is a state machine that operates in one of two primary modes: **Trailing** or **Adjusting**. The layering of logic determines which state is active and how the trailing stop is calculated within that state.

*   **Interaction Dynamics:** The core dynamic is not a confluence of different indicators but rather a monitoring of the **distance between price and the trailing stop itself**. This distance is measured in units of ATR. The system uses a **Threshold Cross** on this distance metric to transition between states.

*   **Hierarchical Filtering:** The logic is strictly hierarchical, preventing conflicting signals.
    1.  **Level 1: Trend Direction Filter (The Primary State)**
        *   The `direction` variable (`1` for Bull, `-1` for Bear) is the highest-level filter. All subsequent logic is conditional on this state. A price close across the `trailingStop` line is the only event that can change this primary state.
    2.  **Level 2: Adjustment Trigger (The Gatekeeper)**
        *   The script only considers an adjustment if it is *not* already in an adjustment phase (`not isAdjusting`).
        *   This gatekeeper checks if the distance between the `close` and the `trailingStop` has exceeded a volatility-defined threshold: `currentDist > kDist`, where `kDist` is `atrMultInput * atr`.
        *   Therefore, the **Sigmoid Transition engine is gated by the primary trend filter**. It can only activate when a trend is established *and* price has accelerated significantly away from the trailing stop.
    3.  **Level 3: Adjustment Execution (The Active State)**
        *   Once triggered, the `isAdjusting` flag becomes `true`, locking the script into the adjustment logic.
        *   In this state, the trailing stop calculation is taken over by the sigmoid function. The original trailing logic is bypassed.
        *   This state persists until one of two termination conditions is met: the adjustment duration (`sigLengthInput`) expires, or the stop gets too close to the price (`minDistMultInput`).

---

### 3. The Execution Engine

#### **A. Trend Flip Conditions (Primary Trigger)**

This is the mechanism for establishing or reversing the primary trend direction.

*   **Boolean Logic:**
    *   **Bull to Bear Flip:** `direction == 1 AND close < trailingStop`
    *   **Bear to Bull Flip:** `direction == -1 AND close > trailingStop`
*   **Execution on Flip:**
    1.  The `direction` variable is inverted.
    2.  The `isAdjusting` state is immediately terminated and reset to `false`.
    3.  The `trailingStop` is re-initialized to a new position based on the opposite side of the price action.
        *   For a new Bull trend: `trailingStop` is set to `low - atrMultInput * atr`.
        *   For a new Bear trend: `trailingStop` is set to `high + atrMultInput * atr`.

#### **B. Sigmoid Transition Conditions (Secondary Trigger & Execution)**

This is the profit-locking and risk-reduction mechanism.

*   **Trigger - Boolean Logic:**
    *   `not isAdjusting AND currentDist > kDist`
    *   This translates to: The system is in its normal trailing mode AND the distance from the closing price to the current trailing stop is greater than the value of `atrMultInput` multiplied by the current `atr`.
*   **Execution on Trigger:**
    1.  `isAdjusting` is set to `true`.
    2.  `sigCounter` is reset to `0`.
    3.  The current `trailingStop` value is stored in `startLevel` to serve as the anchor for the adjustment.
    4.  The total potential adjustment distance, `targetOffset`, is calculated as `sigAmpMultInput * atr`.
*   **Execution during Adjustment:**
    *   On each subsequent bar where `isAdjusting` is `true`:
        1.  A progress ratio `t` is calculated: `sigCounter / sigLengthInput`.
        2.  This ratio is fed into the `sigmoid(t)` function to get a `sigFactor` between 0 and 1.
        3.  A proposed new stop level, `candidate`, is calculated. For a bull trend, this is `startLevel + (targetOffset * sigFactor)`. The stop moves from its starting point towards the price by a fraction of the total intended offset.
        4.  The `candidate` value only replaces the `trailingStop` if it moves the stop closer to the price (i.e., `math.max(trailingStop, candidate)` for longs), ensuring the "trailing" property is never violated.
*   **Termination Conditions:**
    *   **Time-Out:** `sigCounter >= sigLengthInput`. The adjustment phase has completed its pre-defined number of bars.
    *   **Proximity-Out:** `newDist < minDist`. The `candidate` stop level is closer to the current price than the minimum allowed distance, defined by `minDistMultInput * atr`. This acts as a crucial safety brake to prevent whipsaws.
*   **Mathematical Constants & Risk Profile Influence:**
    *   **`atrMultInput` (Default: 3.0):** Defines the baseline risk. A higher value creates a wider stop, reducing sensitivity to noise but increasing potential loss on a reversal. It also sets a higher bar for the Sigmoid Transition to trigger.
    *   **`sigAmpMultInput` (Default: 3.0):** Controls the *aggressiveness* of the profit-locking adjustment. It dictates how much of the gap between price and the stop will be closed during the transition. A higher value leads to a more aggressive "catch-up," locking in gains faster but also bringing the stop closer to the price.
    *   **`minDistMultInput` (Default: 0.5):** A critical risk management parameter. It defines the "personal space" around the price that the trailing stop cannot enter during an adjustment. This prevents the aggressive sigmoid move from placing the stop so close that a minor pullback would trigger an exit, thus balancing the aggressiveness of `sigAmpMultInput`.
    