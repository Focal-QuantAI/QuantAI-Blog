
# Indicators Description

### 1. Component Deconstruction

This section deconstructs each mathematical component of the script's engine, detailing its configuration and any non-standard modifications.

#### **A. Heikin Ashi (HA) Foundation**
*   **Core Function:** `functionHeikinAshi`
*   **Specific Configuration:**
    *   **Source:** Standard `open`, `high`, `low`, `close` of the chart's primary series.
    *   **Logic:** Implements the classical Heikin Ashi formulas:
        *   `HAClose = (open + high + low + close) / 4`
        *   `HAOpen = (previous HAOpen + previous HAClose) / 2`
        *   `HAHigh = max(high, HAOpen, HAClose)`
        *   `HALow = min(low, HAOpen, HAClose)`
*   **Functional Modification:** The script does not use the HA values for plotting candles directly on the main chart (though it has an option to color real candles). Instead, it uses the calculated HA values as the raw input for a standalone momentum oscillator. This repurposes HA from a visual smoothing tool into a quantitative data source.

#### **B. HA-Derived Momentum Source (`haSignedBase`)**
*   **Specific Configuration:**
    *   **Source:** The calculated `HAHigh`, `HALow`, and `HAClose` values from the HA Foundation.
    *   **Mathematical Formula:**
        1.  `haRangeNormPct = ((HAHigh - HALow) / HAClose) * 100`
        2.  `haSignedBase = isHAUp ? haRangeNormPct : -haRangeNormPct`
*   **Functional Modification:** This is a custom-engineered metric. It is not a standard indicator.
    *   **Intended Effect:** It converts the Heikin Ashi trend into a single, quantifiable data series. The formula calculates the HA candle's range as a percentage of its closing price, effectively creating a volatility-normalized measure of the candle's size. This value is then given a positive or negative sign based on the HA trend direction (`HAClose > HAOpen`). The result is a raw momentum value where larger numbers (positive or negative) represent stronger directional pressure within the smoothed HA trend.

#### **C. Z-Score Normalization & Clamping (`f_zscoreClamp`)**
*   **Specific Configuration:**
    *   **Source:** The custom `haSignedBase` series.
    *   **Lookback Period (`normPeriod`):** Dynamically selected based on the chart's timeframe using Fibonacci numbers (e.g., 233 for <3min charts, 144 for <15min, 55 for <2hr) or a manual user input (`in_manualLookback`).
    *   **Mathematical Formula:** `_z = _src / ta.stdev(_src, _len)`
    *   **Clamping:** The output is hard-coded to be clamped between -15.0 and +15.0.
*   **Functional Modification:** This is a simplified, non-centered Z-score. A true Z-score is `(value - mean) / stdev`. By omitting the mean, the script normalizes the momentum value solely by its volatility (standard deviation).
    *   **Intended Effect:** This normalization standardizes the oscillator's output, making its magnitude more comparable across different assets and timeframes. It scales the raw `haSignedBase` value based on its own recent volatility. The clamp at `±15.0` is a ceiling to prevent extreme, outlier volatility events from distorting the oscillator's visual scale.

#### **D. Core Engine 1: "HA Range Base"**
*   **Specific Configuration:**
    *   **Source:** The normalized and clamped series (`haSignedPrepared`).
    *   **Type:** A single moving average, user-selectable between `EMA` and `SMA`.
    *   **Length (`in_rangeLen`):** User-defined, default is 55.
*   **Functional Modification:** This is a standard moving average calculation, but it is applied to the custom, normalized HA momentum source. It acts as a direct, single-pass smoothing filter on the momentum signal.

#### **E. Core Engine 2: "HA Blend"**
*   **Specific Configuration:**
    *   **Source:** The normalized and clamped series (`haSignedPrepared`).
    *   **Type:** A complex, custom-weighted average of multiple EMAs.
    *   **Lengths:** A fixed ladder of EMAs with Fibonacci lengths: 5, 8, 13, 21, 34, 55, 89, 144, 233.
*   **Functional Modification:** This is a highly non-standard smoothing algorithm.
    *   **Mathematical Logic:**
        1.  It calculates nine separate EMAs on the source data.
        2.  Based on a user preset ("Fast", "Balanced", "Slow"), it selects four pairs of adjacent EMAs from the ladder (e.g., "Balanced" uses 13/21, 21/34, 34/55, 55/89).
        3.  It calculates the simple average of each of these four pairs.
        4.  The final output (`blendEngineCore`) is the simple average of those four averaged pairs.
    *   **Intended Effect:** This creates a heavily smoothed output with significantly reduced noise compared to a single MA. By averaging averages of different lengths, it aims to reduce the lag inherent in a single long-period moving average while still filtering out high-frequency noise, thereby improving the signal-to-noise ratio.

#### **F. Final Smoothing & Output (`osc`)**
*   **Specific Configuration:**
    *   **Source:** The output of either the "HA Range Base" or "HA Blend" engine.
    *   **Type:** An optional, additional moving average (`EMA` or `SMA`).
    *   **Length (`in_finalSmoothLen`):** User-defined, default is 3.
*   **Functional Modification:** This is a final, optional low-pass filter applied after the main engine calculation. Its purpose is to provide one last layer of smoothing for visual clarity, at the cost of a small amount of additional lag. The final `osc` variable represents the fully processed oscillator value.

#### **G. Auxiliary Studies (Applied to `osc`)**
All subsequent studies are applied *to the final oscillator value (`osc`)*, not to price. They measure the momentum of the momentum.

*   **Adaptive Crossovers:**
    *   **Configuration:** Two moving averages (`fastLine`, `slowLine`) calculated on `osc`. Lengths are either manually set or automatically adjusted based on timeframe (e.g., 5/13 on <3min charts, 13/48 on <2hr charts).
    *   **Purpose:** To identify acceleration and deceleration in the oscillator itself. A bullish cross (`fastLine` > `slowLine`) indicates that the oscillator's momentum is increasing.

*   **SMA 20/50 Crossovers:**
    *   **Configuration:** A second, independent pair of MAs on `osc` with fixed default lengths of 20 and 50.
    *   **Purpose:** Provides a classic, non-adaptive momentum-of-momentum crossover signal for reference.

*   **Rolling Oscillator VWA (Volume-Weighted Average):**
    *   **Configuration:** A custom VWA calculated on `osc`. The weight is `volume / ta.sma(volume, oscVwaLen)`.
    *   **Modification:** This is not a standard VWMA. It weights the oscillator's value by the ratio of current volume to its moving average.
    *   **Effect:** It gives significantly more weight to oscillator values that occur on bars with volume spikes relative to the recent norm. This is a conviction filter.

*   **Trend Pivot Overlay:**
    *   **Configuration:** Uses `ta.pivothigh` and `ta.pivotlow` on `osc` with default lookbacks of `left=21`, `right=5`.
    *   **Modification:** It filters pivots, only accepting highs that form above zero and lows that form below zero. The overlay line (`overlayLine`) then tracks the most recent significant high or low pivot, creating a structural support/resistance level for the oscillator itself.

*   **Adaptive Guide Lines:**
    *   **Configuration:** The upper and lower guides are not fixed values.
        *   `upperGuide = ta.highest(osc, normPeriod) * 0.5`
        *   `lowerGuide = ta.lowest(osc, normPeriod) * 0.5`
    *   **Effect:** These create dynamic "overbought/oversold" zones that represent 50% of the oscillator's recent peak range. They expand and contract with the oscillator's volatility, providing a relative measure of extension rather than a fixed one.

### 2. Logic Layering & Confluence

The script's engine is a sequential, multi-stage filtering pipeline designed to refine a raw price signal into a high-clarity momentum reading.

*   **Hierarchical Filtering:** The logic flows in a strict hierarchy, where each stage processes the output of the previous one:
    1.  **Stage 1 (Price Smoothing):** Raw price is first filtered through **Heikin Ashi** calculations. This removes the primary layer of market noise and establishes a smoothed directional bias.
    2.  **Stage 2 (Quantification & Normalization):** The visual HA trend is converted into a quantitative, signed momentum value (`haSignedBase`) and then normalized by its own standard deviation (`f_zscoreClamp`). This transforms the signal into a standardized format.
    3.  **Stage 3 (Core Engine Smoothing):** The normalized signal is passed through one of two powerful smoothing engines ("Range Base" or "Blend"). This is the most significant noise reduction step, designed to isolate the core underlying momentum trend from short-term fluctuations.
    4.  **Stage 4 (Regime & Confirmation Analysis):** The final, clean oscillator signal (`osc`) is then analyzed for confluence using auxiliary studies. This is where the script looks for confirmation before generating a strong visual cue.

*   **Interaction Dynamics:** The script primarily relies on **Threshold Crosses** and **Convergences** for its signals.
    *   **Primary Threshold Cross (Regime Filter):** The `osc` crossing the **`zeroLine`** is the fundamental trigger. A cross above zero indicates that the underlying HA-derived momentum has shifted from bearish to bullish. This acts as the primary "Gatekeeper" signal.
    *   **Secondary Threshold Cross (Extension Filter):** The `osc` crossing the adaptive `upperGuide` or `lowerGuide` signals that momentum is becoming extended relative to its recent range. This is not a reversal signal but a warning of potential exhaustion.
    *   **Confluence (Momentum Confirmation):** The script seeks a convergence of signals. A high-conviction bullish setup occurs when:
        1.  The oscillator is above the `zeroLine` (bullish regime).
        2.  The `fastLine` crosses above the `slowLine` (momentum is accelerating).
    *   **Strategic Gatekeeper (HTF Plot):** The higher-timeframe oscillator plot (`htfOscFixed`) is designed to be a visual, strategic filter. A user would interpret a bullish signal on the current timeframe (e.g., 15-min) as having higher probability if the HTF plot (e.g., 1-hour) is also in a bullish state (above its own zero line).

### 3. The Execution Engine

As an `indicator`, the script's "Execution Engine" translates its mathematical state into visual triggers (colors and plot states) rather than trade orders.

*   **Boolean Logic for Visual Triggers:** The primary visual output is the color of the oscillator, determined by the `f_visualThresholdColor` function. A specific color is triggered by a combination of two boolean conditions: the oscillator's position relative to a threshold and its direction of movement.

    *   **Strong Bullish Signal (`in_colAbove50Rise`):**
        *   `osc >= (ta.highest(osc, normPeriod) * 0.5)` AND `osc > osc[1]`
        *   *Condition:* The oscillator has entered its upper dynamic zone AND is still rising.

    *   **Standard Bullish Signal (`in_colAboveRise`):**
        *   `osc >= 0` AND `osc > osc[1]`
        *   *Condition:* The oscillator is in positive territory AND is rising.

    *   **Bullish Deceleration (`in_colAboveFall`):**
        *   `osc >= 0` AND `osc < osc[1]`
        *   *Condition:* The oscillator is in positive territory BUT is now falling (losing momentum).

    *   **Strong Bearish Signal (`in_colBelow50Fall`):**
        *   `osc <= (ta.lowest(osc, normPeriod) * 0.5)` AND `osc < osc[1]`
        *   *Condition:* The oscillator has entered its lower dynamic zone AND is still falling.

    *   **Standard Bearish Signal (`in_colBelowFall`):**
        *   `osc < 0` AND `osc < osc[1]`
        *   *Condition:* The oscillator is in negative territory AND is falling.

    *   **Bearish Deceleration (`in_colBelowRise`):**
        *   `osc < 0` AND `osc > osc[1]`
        *   *Condition:* The oscillator is in negative territory BUT is now rising (bearish momentum is weakening).

*   **Significance of Mathematical Constants:**
    *   **`in_clampRange = 15.0`:** This hard-coded value acts as a circuit breaker. It caps the normalized input at 15 standard deviations, preventing extreme volatility events from making the oscillator pane unreadable. It ensures visual consistency.
    *   **Fibonacci Numbers (Lookbacks & EMAs):** The use of numbers from the Fibonacci sequence (5, 8, 13, 21, 34, 55...) for lookback periods and EMA lengths is a deliberate design choice rooted in classic technical analysis theory. The intent is to align the script's calculation periods with what are believed to be natural market rhythm cycles.
    *   **Pivot Lookbacks (`left=21`, `right=5`):** This configuration creates a specific type of pivot. The long left lookback (21 bars) ensures that a pivot high/low is a truly significant turning point relative to its recent history. The short right lookback (5 bars) allows for reasonably prompt confirmation of that pivot, balancing structural significance with acceptable lag.
    