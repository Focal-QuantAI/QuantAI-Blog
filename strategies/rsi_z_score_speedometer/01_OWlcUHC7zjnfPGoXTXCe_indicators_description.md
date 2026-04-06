
# Indicators Description

### 1. Component Deconstruction

This script's engine is built upon two primary, independent mathematical components which are then normalized and visualized.

#### A. Relative Strength Index (RSI) Speedometer

*   **Core Calculation:** The script utilizes the standard `ta.rsi()` function. This is a momentum oscillator that measures the speed and change of price movements by comparing the magnitude of recent gains to recent losses.
*   **Specific Configuration:**
    *   **Price Source (`rsi_src`):** `close` (by default). This is the standard input for RSI calculations.
    *   **Lookback Period (`rsi_len`):** `14` (by default). This is the classic period for RSI, balancing responsiveness with signal smoothness.
*   **Functional Modification:**
    *   There are no mathematical modifications to the core RSI calculation itself. The script uses the raw, canonical RSI value.
    *   The output, `rsi_metric`, is a floating-point number between 0 and 100. This value is used directly to control the angle of the speedometer needle, providing a linear, 1:1 mapping where `0 RSI = 0 degrees` (far left) and `100 RSI = 180 degrees` (far right).

#### B. Z-Score Speedometer

*   **Core Calculation:** The Z-Score is a statistical measurement that describes a value's relationship to the mean of a group of values. It is measured in terms of standard deviations from the mean. A Z-Score of 0 indicates the data point is identical to the mean.
    *   **Mean Calculation:** `z_mean = ta.sma(z_src, z_len)` - A Simple Moving Average (SMA) is used to establish the statistical mean.
    *   **Standard Deviation Calculation:** `z_std = ta.stdev(z_src, z_len)` - The standard deviation of the source price over the same lookback period.
    *   **Raw Z-Score:** `z_raw = (z_src - z_mean) / z_std` - The classic formula for a Z-Score. A guard condition (`z_std != 0.0`) prevents division-by-zero errors during periods of zero volatility.
*   **Specific Configuration:**
    *   **Price Source (`z_src`):** `close` (by default).
    *   **Lookback Period (`z_len`):** `20` (by default). This is a common period for statistical calculations like Bollinger Bands, upon which this logic is based.
    *   **Limit (`z_limit`):** `3.0` (by default). This is a critical mathematical constant defining the maximum number of standard deviations to be considered.
*   **Functional Modification:** The raw Z-Score is unbounded, which is unsuitable for a fixed-gauge speedometer. The script performs a two-step modification to "tame" the value:
    1.  **Clamping:** `z_clamped = math.max(-z_limit, math.min(z_limit, z_raw))`. The raw Z-Score is capped at the upper and lower bounds defined by `z_limit`. Any value greater than `3.0` is treated as `3.0`, and any value less than `-3.0` is treated as `-3.0`. This reduces the impact of extreme black-swan events on the visualization, focusing on more common statistical deviations.
    2.  **Normalization/Rescaling:** `z_metric = ((z_clamped + z_limit) / (2.0 * z_limit)) * 100.0`. This is the key mathematical transformation. It converts the clamped Z-Score range (e.g., `[-3.0, +3.0]`) into a bounded 0-100 oscillator, identical in scale to the RSI.
        *   **At `-z_limit`:** `(-3.0 + 3.0) / 6.0 * 100 = 0`
        *   **At `0` (the mean):** `(0 + 3.0) / 6.0 * 100 = 50`
        *   **At `+z_limit`:** `(3.0 + 3.0) / 6.0 * 100 = 100`
    This normalized `z_metric` is then used to drive the second speedometer needle.

### 2. Logic Layering & Confluence

The script does not employ programmatic logic layering or hierarchical filtering. Instead, it presents two independent data streams side-by-side for **visual confluence**, leaving the synthesis to the discretionary trader.

*   **Interaction Dynamics:** The engine is designed for the user to identify **Visual Convergence**. The primary "signal" occurs when both indicators simultaneously point to an extreme condition.
    *   **Example (Overbought):** The RSI needle is in the "Strong Sell" zone (RSI > 80) *at the same time* the Z-Score needle is in the "> +2σ" zone (normalized value > 83.3, corresponding to a raw Z-Score > 2.0).
    *   **Example (Oversold):** The RSI needle is in the "Strong Buy" zone (RSI < 20) *at the same time* the Z-Score needle is in the "< -2σ" zone (normalized value < 16.7, corresponding to a raw Z-Score < -2.0).
*   **Hierarchical Filtering:** There is **no gatekeeping** mechanism. The RSI calculation is completely independent of the Z-Score calculation. One indicator does not need to meet a condition before the other is calculated or displayed. They are presented as two equal, parallel streams of information. The filtering is performed by the user's cognitive process, not by the script's code. This design improves the signal-to-noise ratio by requiring two different types of extremity (momentum-based and statistical) to align.

### 3. The Execution Engine

As an `indicator`, this script has no automated execution engine. Its "triggers" are visual cues for discretionary action, defined by the graphical representation of the underlying math.

*   **Boolean Logic (Visual Equivalent):** The intended trigger is a visual "AND" condition.
    *   **Potential Short Signal:** `(rsi_metric >= 80) AND (z_metric >= 83.3)`
    *   **Potential Long Signal:** `(rsi_metric <= 20) AND (z_metric <= 16.7)`
    *   *Note: The numerical values correspond to the "Strong" zones on the speedometers. The Z-Score value of 83.3 is derived from the normalization formula for a raw Z-Score of +2.0: `((2.0 + 3.0) / 6.0) * 100`.*

*   **Mathematical Constants & Visual Thresholds:** The script's behavior is heavily influenced by how it translates numerical values into visual zones.
    *   The speedometer is divided into 5 equal sectors, each representing 20 points on the 0-100 scale.
    *   **RSI Zones:**
        *   `80-100`: Strong Sell (Red)
        *   `60-80`: Sell (Orange)
        *   `40-60`: Neutral (Gray)
        *   `20-40`: Buy (Lime)
        *   `0-20`: Strong Buy (Green)
    *   **Z-Score Zones (and their raw Z-Score equivalents with `z_limit=3.0`):**
        *   `80-100` ("> +2σ"): Raw Z-Score from `+1.8` to `+3.0`
        *   `60-80` ("+1σ"): Raw Z-Score from `+0.6` to `+1.8`
        *   `40-60` ("0"): Raw Z-Score from `-0.6` to `+0.6`
        *   `20-40` ("-1σ"): Raw Z-Score from `-1.8` to `-0.6`
        *   `0-20` ("< -2σ"): Raw Z-Score from `-3.0` to `-1.8`
    *   The labels (e.g., "> +2σ") are placed at specific points for readability (`90%` mark for "> +2σ") and are approximations of the zone they represent. The true trigger is the needle's position relative to the colored sectors, which are mathematically precise.

The script's engine is therefore a data processing and visualization pipeline: it ingests price, runs it through two parallel mathematical transformations (one momentum-based, one statistical), normalizes both outputs to a common 0-100 scale, and renders them as intuitive speedometer gauges for qualitative analysis.
    