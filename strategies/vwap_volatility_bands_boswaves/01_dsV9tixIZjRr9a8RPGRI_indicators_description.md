
# Indicators Description

### 1. Component Deconstruction

This section deconstructs the individual mathematical components used to build the indicator's logic engine.

#### **A. Cumulative VWAP (Volume-Weighted Average Price)**
*   **Specific Configuration:**
    *   **Price Source:** `hlc3` (the average of the high, low, and close of each bar).
    *   **Anchor Period:** User-selectable between `Session`, `Week`, or `Month`. The calculation resets at the beginning of each new anchor period.
    *   **Reset Logic:** The `is_new_anchor` variable uses built-in functions to detect the first bar of a new period.
        *   `Session`: Resets on the first bar of the regular trading session or on a new day for intraday timeframes.
        *   `Week`: Resets when `weekofyear` changes.
        *   `Month`: Resets when `month` changes.
*   **Functional Modification:** This is not a rolling VWAP. It is a **cumulative** calculation that aggregates `hlc3 * volume` and total `volume` from the anchor point. The formula is `raw_vwap = cum_tpvol / cum_vol`. This grounds the indicator to a specific starting point, representing the average price paid per share/contract since the session, week, or month began.

#### **B. T3 Moving Average (Tim Tillson's T3)**
*   **Specific Configuration:**
    *   **Source:** The `raw_vwap` calculated above. This is a critical distinction; the T3 is smoothing the VWAP, not the price.
    *   **Length (`len`):** User-defined integer, default `28`. This is the lookback period for the nested EMAs within the T3 formula.
    *   **Factor (`factor`):** User-defined float, default `0.7`. This is the `a` coefficient in the T3 calculation, controlling the degree of smoothing and lag.
*   **Functional Modification:** The script employs a custom function `f_t3` that implements the full T3 formula, which is a form of triple-exponential smoothing designed for superior lag reduction compared to a simple EMA or DEMA.
    *   **Mathematical Logic:** The T3 is a weighted sum of six nested Exponential Moving Averages (EMAs) of the source (`raw_vwap`).
        *   `e1 = ema(src, len)`
        *   `e2 = ema(e1, len)`
        *   ...
        *   `e6 = ema(e5, len)`
    *   The final `t3` value is calculated as `c1*e6 + c2*e5 + c3*e4 + c4*e3`, where the coefficients `c1` through `c4` are derived from the `factor` (`a`). This complex weighting scheme results in an exceptionally smooth yet responsive moving average, aiming to filter out market noise from the core VWAP value.

#### **C. ATR (Average True Range)**
*   **Specific Configuration:**
    *   **Length (`atr_len`):** User-defined integer, default `14`.
    *   **Source:** Standard implementation using the bar's `high`, `low`, and `close`.
*   **Functional Modification:** None. This is a standard `ta.atr()` calculation used as a raw measure of market volatility. Its purpose is to provide a dynamic scaling factor for the volatility bands.

#### **D. ATR-Based Volatility Bands**
*   **Specific Configuration:**
    *   **Basis:** The `t3` (T3-smoothed VWAP) line acts as the central axis.
    *   **Width:** The `atr` value is multiplied by a series of hard-coded constants to define the band levels.
*   **Functional Modification:** These are not statistically derived bands like Bollinger Bands (which use standard deviation). They are volatility-derived.
    *   **Mathematical Logic:** The bands are calculated by adding or subtracting a multiple of the current `atr` value from the `t3` basis line.
    *   **Multipliers:** The script uses four fixed, asymmetrical multipliers: `0.5`, `1.0`, `1.5`, and `2.2`. These values define discrete volatility zones around the T3-VWAP mean.

### 2. Logic Layering & Confluence

The script's intelligence lies in how it layers these components to create a hierarchical filtering system.

*   **Hierarchical Filtering: The `score` Variable as a "Gatekeeper"**
    1.  **Primary Filter (Regime Definition):** The entire logical flow is governed by the `score` variable. This variable acts as a state machine defining the market regime.
    2.  **Calculation:** `score := t3 > t3[1] ? 1 : t3 < t3[1] ? -1 : nz(score[1])`
    3.  **Function:**
        *   If the T3-VWAP is rising (`t3 > t3[1]`), the regime is Bullish (`score = 1`).
        *   If the T3-VWAP is falling (`t3 < t3[1]`), the regime is Bearish (`score = -1`).
        *   If the T3-VWAP is flat, the `score` persists its previous state (`nz(score[1])`).
    4.  **Gatekeeping Action:** All subsequent visual and signal logic is conditional on the `score`. For example, the lower (bullish) bands are only plotted when `score == 1`, and the upper (bearish) bands are only plotted when `score == -1`. This immediately filters out half of the potential signals, forcing the user to only consider trades in the direction of the T3-VWAP's momentum.

*   **Interaction Dynamics: Threshold Crosses**
    The engine does not use complex pattern recognition or indicator divergence. Its logic is based entirely on a sequence of **Threshold Crosses**.
    1.  **Trend Inception Cross:** The first and most important cross is the T3 line crossing its own value from the previous bar (`t3 > t3[1]`). This flips the `score` and establishes the dominant trend direction. This is the primary entry trigger for a new trend.
    2.  **Exhaustion/TP Cross:** The secondary crosses are the `close` price crossing the outermost volatility bands (`band_u4` or `band_l4`). This is used to signal potential trade exits (Take Profit). This dynamic represents price "diverging" from the mean by a statistically significant, volatility-adjusted amount.

### 3. The Execution Engine

This section details the precise boolean logic for trade triggers and exits.

*   **Boolean Logic: Triggers & Exits**
    *   **Long Entry Signal (`long`):**
        *   `ta.crossover(t3, t3[1])`
        *   **Translation:** Returns `true` for the single bar where the T3-VWAP value becomes greater than its value on the prior bar. This marks the initiation of a bullish regime.
    *   **Short Entry Signal (`short`):**
        *   `ta.crossunder(t3, t3[1])`
        *   **Translation:** Returns `true` for the single bar where the T3-VWAP value becomes less than its value on the prior bar. This marks the initiation of a bearish regime.
    *   **"Bear TP" Signal (`tpbear`):**
        *   `show_tp and ta.crossover(close, band_u4) and score == 1`
        *   **Translation:** This condition is `true` ONLY IF:
            1.  The `score` indicates a **Bullish** trend is active (`score == 1`).
            2.  The `close` price crosses *above* the outermost upper band (`band_u4`).
            3.  This is an **exhaustion signal for a long position**. It indicates that in a strong uptrend, price has accelerated so far above the mean that a pullback or reversal is likely.
    *   **"Bull TP" Signal (`tpbull`):**
        *   `show_tp and ta.crossunder(close, band_l4) and score == -1`
        *   **Translation:** This condition is `true` ONLY IF:
            1.  The `score` indicates a **Bearish** trend is active (`score == -1`).
            2.  The `close` price crosses *below* the outermost lower band (`band_l4`).
            3.  This is an **exhaustion signal for a short position**. It indicates a potential capitulation washout in a downtrend, suggesting a prime moment to take profit.

*   **Mathematical Constants & Risk Profile**
    *   **T3 Factor (`0.7`):** This value is a well-established default for the T3, offering a robust balance between smoothing and responsiveness. A higher value would increase lag but provide a more stable trend definition, filtering out more whipsaws. A lower value would make the T3 react faster, leading to earlier signals but more potential noise.
    *   **ATR Multipliers (`0.5, 1.0, 1.5, 2.2`):** These hard-coded values are central to the script's risk mechanics. The final multiplier, `2.2`, sets the threshold for an exhaustion signal.
        *   **Significance:** A `2.2x` ATR move from the T3-VWAP is the trigger for a take-profit signal. This value directly controls the strategy's target. A smaller value (e.g., `1.8`) would generate TP signals more frequently and with smaller price moves. A larger value (e.g., `3.0`) would require a much stronger, potentially parabolic move to trigger a TP, aiming for larger gains but increasing the risk of price reversing before the target is hit. The `2.2` value is a pre-defined constant that dictates the required magnitude of a trend's "blow-off" top or bottom to be considered an exit point.
    