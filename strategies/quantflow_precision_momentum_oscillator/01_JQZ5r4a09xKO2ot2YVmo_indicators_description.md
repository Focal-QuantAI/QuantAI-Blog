
# Indicators Description

### 1. Component Deconstruction

#### **A. Cumulative MIDAS VWAP & Standard Deviation**
This is the foundational engine of the oscillator, establishing a dynamic mean and volatility baseline.

*   **Specific Configuration:**
    *   **Price Source:** `hlc3` (High + Low + Close / 3). This is a hard-coded, typical price measure.
    *   **Anchor Point:** The calculation is cumulative and resets based on the `anchorMethod` input.
        *   `Auto`: Resets on the daily (`D`) timeframe for intraday charts (<= 15min), weekly (`W`) for higher intraday, monthly (`M`) for daily, and yearly (`12M`) for weekly/monthly charts.
        *   `Timeframe`: User-defined timeframe (`anchorTf`).
        *   `Date`: User-defined start date and time (`anchorDate`).
    *   **Reset Detection:** The `isReset` boolean variable becomes `true` for a single bar when the anchor condition is met, triggering a reset of the cumulative sums.

*   **Functional Modification:**
    *   This is a custom, non-native implementation of a Volume-Weighted Average Price (VWAP) and its corresponding Volume-Weighted Standard Deviation.
    *   **VWAP Logic:** It uses three `var` variables (`sumVol`, `sumVolPrice`, `sumVolSq`) that persist their values across bars. On each bar, they are cumulatively updated. The VWAP is calculated as `midasVwap = sumVolPrice / sumVol`.
    *   **Standard Deviation Logic:** The script calculates the volume-weighted variance using the formula `variance = (sumVolSq / sumVol) - math.pow(midasVwap, 2)`. The standard deviation (`stdDev`) is the square root of this variance. This provides a statistically robust measure of price dispersion around the volume-weighted mean.

#### **B. Z-Score Normalization**
This component translates the raw price deviation into a standardized, unbounded oscillator.

*   **Specific Configuration:**
    *   **Source:** The calculation uses the current `close` price, the calculated `midasVwap`, and the calculated `stdDev`.
    *   **Formula:** `rawZ = (close - midasVwap) / stdDev`.

*   **Functional Modification:**
    *   The Z-Score is not a standard Pine Script® indicator but a mathematical construct. Its function is to normalize the output, expressing the current price's distance from the VWAP in terms of standard deviations. A value of `1.0` means the price is one standard deviation above the VWAP. This makes the oscillator's values comparable across different assets and volatility regimes.

#### **C. Zero-Lag Double EMA (DEMA)**
This component smooths the normalized Z-Score to create the final, responsive oscillator wave.

*   **Specific Configuration:**
    *   **Source:** The `rawZ` score.
    *   **Length:** `momLength` input, defaulting to `3`. A very short length is chosen to prioritize responsiveness over smoothing, aiming for lag reduction.

*   **Functional Modification:**
    *   The script manually implements the DEMA formula: `momOsc = 2 * ta.ema(rawZ, momLength) - ta.ema(ta.ema(rawZ, momLength), momLength)`.
    *   The intended effect is to significantly reduce the lag associated with a traditional Simple or Exponential Moving Average. By applying a double-smoothing process and recombining the results, the DEMA stays closer to the source data (`rawZ`), which is critical for timely signal generation in a momentum context.

#### **D. Pivot-Based Divergence Detector**
This system identifies decoupling between price momentum and the oscillator's momentum.

*   **Specific Configuration:**
    *   **Source:** `ta.pivothigh` and `ta.pivotlow` are applied to both the oscillator (`momOsc`) and price (`high`/`low`).
    *   **Lookback Periods:** A left-strength of `4` and a right-strength of `2` are hard-coded. This means a pivot point must be higher/lower than the 4 preceding bars and the 2 succeeding bars to be confirmed.

*   **Functional Modification:**
    *   The logic uses `ta.valuewhen` to fetch the value of the *previous* confirmed pivot point on both the oscillator and price.
    *   A bearish divergence (`isBearDiv`) is confirmed when a new price pivot high is higher than the previous one, while the corresponding oscillator pivot high is *lower* than its previous one. The `high[2]` and `momOsc[2]` offsets are used to align the signal with the bar where the pivot is confirmed (due to the `right=2` lookback).

#### **E. Volatility Compression Ratio**
A custom metric to gauge the state of the oscillator's volatility.

*   **Specific Configuration:**
    *   **Short-Term Lookback:** `10` bars for the current oscillator range (`oscRange`).
    *   **Long-Term Lookback:** `50` bars for the simple moving average of the oscillator's range (`oscAvgRange`).

*   **Functional Modification:**
    *   The script calculates a ratio: `compRatio = oscRange / oscAvgRange`.
    *   This normalizes the current 10-bar volatility against its 50-period average. It is not measuring price volatility, but the volatility *of the momentum oscillator itself*. A ratio below `0.7` signifies "Coiling" (a contraction in momentum swings), while a ratio above `1.4` signifies "Expanding" (an expansion of momentum swings).

### 2. Logic Layering & Confluence

The script's engine filters market noise through a hierarchical, multi-stage process.

*   **1. Foundational Context (The Mean):** The MIDAS VWAP acts as the primary "Gatekeeper." All subsequent calculations are relative to this dynamic anchor. The script is fundamentally uninterested in price unless it deviates significantly from this mean.

*   **2. Normalization & State Definition (The Zone):** The Z-Score and subsequent DEMA smoothing create the `momOsc`. The `fibThreshold` (default 0.618) then slices this oscillator's range into three zones:
    *   **Bullish Territory:** `momOsc > 0`
    *   **Bearish Territory:** `momOsc < 0`
    *   **Extreme Zones (Reversal Zones):** `abs(momOsc) > fibThreshold`.
    This is a **Threshold Cross** system. An entry into an Extreme Zone is the first-level alert that a mean-reversion opportunity may be forming.

*   **3. Velocity Analysis (The Predictive Trigger):** The script looks for a change in the oscillator's second derivative (its acceleration/deceleration).
    *   **Interaction Dynamics:** This is a **Confluence** filter. A "Spark" signal (`bullSpark` or `bearSpark`) requires two conditions:
        1.  The oscillator must be in a state of moderate extension (`momOsc > 0.382` or `< -0.382`).
        2.  The oscillator's velocity (`momOsc - momOsc[1]`) must cross zero.
    *   **Hierarchical Filtering:** The state of the oscillator (Condition 1) acts as a gatekeeper for the velocity event (Condition 2). A velocity cross near the zero line is ignored; it only becomes a "Spark" when momentum is already stretched, signaling a potential exhaustion point before the oscillator itself has reversed.

*   **4. Divergence Analysis (The Confirmation Trigger):** This is a classic **Divergence** filter that acts as a powerful confirmation signal.
    *   **Interaction Dynamics:** It confirms trend exhaustion by identifying a decoupling between price highs/lows and oscillator highs/lows. A divergence signal is independent of the other triggers but provides the highest level of confluence when it occurs while the oscillator is in an Extreme Zone.

### 3. The Execution Engine

The script's "triggers" are defined by the boolean logic that controls the visual markers and alert conditions.

*   **Zero Line Cross (`zeroCross`):**
    *   **Boolean Logic:** `ta.cross(momOsc, 0) == true`.
    *   **Function:** Signals a full reversal of momentum polarity. This is a lagging, trend-confirming signal.

*   **Early Reversal Spark (`bullSpark`, `bearSpark`):**
    *   **Boolean Logic (Bullish):** `ta.crossover(velocity, 0) AND momOsc < -0.382`.
    *   **Boolean Logic (Bearish):** `ta.crossunder(velocity, 0) AND momOsc > 0.382`.
    *   **Mathematical Constants:** The `0.382` constant is a Fibonacci retracement level used as a hard-coded threshold. It defines a zone of "moderate" extension, allowing the signal to trigger earlier than a cross from the primary `0.618` `fibThreshold` zone. This is designed to front-run the reversal.

*   **Momentum Extreme (State-based Alert):**
    *   **Boolean Logic:** `math.abs(momOsc) > fibThreshold`.
    *   **Mathematical Constants:** The `fibThreshold` (default `0.618`) defines the boundary for what the script considers a statistically significant deviation from the mean, priming the user for a potential reversal.

*   **Momentum Divergence (`isBearDiv`, `isBullDiv`):**
    *   **Boolean Logic (Bearish):** `(new oscillator pivot high exists)` AND `(new osc pivot high < previous osc pivot high)` AND `(price high at new pivot > price high at previous pivot)`.
    *   **Function:** This is a pattern-recognition trigger. It requires a specific sequence of highs and lows to be met, confirming that the upward thrust in price is not being supported by underlying momentum.
    