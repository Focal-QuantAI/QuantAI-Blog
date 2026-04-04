
# Indicators Description

### 1. Component Deconstruction

#### **A. Hurst Exponent (Custom Implementation)**
This is the strategic core of the script, implemented via a custom variance scaling method rather than a built-in function.

*   **Specific Configuration:**
    *   **Lookback Period (`active_h_period`):** Default is 60 bars. This defines the window for variance calculations.
    *   **Scaling Lag (`active_h_lag`):** Default is 10 bars. This is the `q` in the variance ratio, representing the longer-term return horizon.
    *   **Price Source:** `close` price is used exclusively for all calculations.

*   **Functional Modification:**
    *   The script calculates the Hurst exponent `H` by comparing the variance of 1-period returns to the variance of `q`-period returns over a `p`-bar lookback window.
    *   **Mathematical Logic:**
        1.  `var1 = ta.variance(close - close[1], active_h_period)`: Calculates the variance of single-period price changes. This represents short-term, high-frequency price fluctuation.
        2.  `varq = ta.variance(close - close[active_h_lag], active_h_period)`: Calculates the variance of `q`-period price changes. This represents longer-term price fluctuation.
        3.  `H_raw = math.log(varq / var1) / (2.0 * math.log(active_h_lag))`: This is the variance-scaling formula. `H` is derived from the logarithmic relationship between the ratio of long-term to short-term variance and the scaling lag `q`.
        4.  `H = math.max(0.0, math.min(H_raw, 1.0))`: The raw `H` value is clamped between 0.0 and 1.0 to maintain theoretical bounds and prevent calculation errors from extreme market data.
        5.  `safeH = nz(H, 0.5)`: A "safe" version of `H` is created. If `H` is `na` (during the initial bars of the chart), it defaults to 0.5, representing a neutral, random-walk assumption.

#### **B. Kalman Filter (Custom, 1-D)**
This acts as the script's primary signal line, replacing a traditional moving average to reduce lag while smoothing price.

*   **Specific Configuration:**
    *   **Base Tracking Gain (`active_kf_gain`):** Default is 0.2. This is the baseline smoothing factor before adaptation.
    *   **Price Source:** The filter is applied directly to the `close` price.

*   **Functional Modification:**
    *   The script employs a non-standard, adaptive Kalman Filter where the tracking gain is dynamically modulated by the Hurst exponent.
    *   **Mathematical Logic:**
        1.  `adaptive_gain = active_kf_gain * (0.5 + safeH)`: The tracking gain is not static. It is scaled by a factor derived from the Hurst exponent.
            *   **Trending Market (H -> 1.0):** `adaptive_gain` approaches `active_kf_gain * 1.5`. The filter becomes more sensitive and tracks price more aggressively, reducing lag.
            *   **Mean-Reverting Market (H -> 0.0):** `adaptive_gain` approaches `active_kf_gain * 0.5`. The filter becomes less sensitive and provides more smoothing, ignoring noise.
            *   **Random Walk Market (H = 0.5):** `adaptive_gain` equals `active_kf_gain * 1.0`. The filter operates at its base configuration.
        2.  `kf := kf[1] + adaptive_gain * (close - kf[1])`: This is the recursive update formula for a simple one-dimensional Kalman filter (or more accurately, an Exponential Moving Average with a dynamic alpha). The new filter value is the previous value plus a fraction (the `adaptive_gain`) of the error between the current price and the previous filter value.

#### **C. Average True Range (ATR)**
This component provides the raw volatility measurement used to calculate the band width.

*   **Specific Configuration:**
    *   **Length (`active_atr_len`):** Default is 10 periods.
    *   **Source:** Standard `high`, `low`, and `close` prices via the `ta.atr()` function.

*   **Functional Modification:**
    *   The ATR value itself is standard. Its modification occurs in how it is multiplied and applied in the Supertrend framework.

#### **D. Supertrend Framework (Custom, Adaptive)**
This structure provides the dynamic upper and lower bands that serve as the trigger levels.

*   **Specific Configuration:**
    *   **Signal Line:** The Kalman Filtered price (`kf`) is used as the central pivot, not the `close` price.
    *   **Base Multiplier (`active_atr_base`):** Default is 1.5. This sets the minimum possible band width in ATR units.
    *   **Hurst Scale Factor (`active_atr_hscale`):** Default is 3.0. This controls the magnitude of band widening in low-Hurst regimes.

*   **Functional Modification:**
    *   This is a heavily modified Supertrend where the ATR multiplier is not a fixed constant but a dynamic variable controlled by the Hurst exponent.
    *   **Mathematical Logic:**
        1.  `h_mult = active_atr_base + active_atr_hscale * (1.0 - safeH)`: This formula calculates the dynamic ATR multiplier.
            *   **Strong Trend (H -> 1.0):** The term `(1.0 - safeH)` approaches 0. The multiplier `h_mult` collapses to its minimum value, `active_atr_base`. This results in tighter bands for more responsive entries and exits.
            *   **Strong Mean-Reversion (H -> 0.0):** The term `(1.0 - safeH)` approaches 1. The multiplier `h_mult` expands to its maximum value, `active_atr_base + active_atr_hscale`. This results in significantly wider bands to filter out whipsaws in choppy markets.
        2.  `band = ta.atr(active_atr_len) * h_mult`: The final band offset is the product of the standard ATR and this Hurst-adaptive multiplier.
        3.  The core Supertrend logic (`upBand`, `dnBand`, `trend`) is then applied using the Kalman Filter (`kf`) as the price source and this dynamic `band` offset.

### 2. Logic Layering & Confluence

The script's engine is built on a strict hierarchical filtering model, where each layer processes information from the layer above it, progressively refining the signal and reducing noise.

*   **Hierarchical Filtering:**
    1.  **Primary Layer (Regime Diagnosis):** The **Hurst Exponent** acts as the master "Gatekeeper." It does not generate signals but provides a single, quantitative measure of the market's character (trending, random, or mean-reverting). All subsequent calculations are subordinate to its output.
    2.  **Secondary Layer (Signal Processing):** The **Kalman Filter** takes the raw `close` price and smooths it. Its behavior is directly governed by the Hurst Exponent from the primary layer. It becomes a "regime-aware" signal line, lagging less in trends and smoothing more in chop.
    3.  **Tertiary Layer (Trigger Definition):** The **Adaptive Supertrend Bands** form the final filter. Their distance from the Kalman Filter signal line is also governed by the Hurst Exponent. They use the processed output from the secondary layer (`kf`) as their central axis.

*   **Interaction Dynamics:**
    *   The system is fundamentally based on **Confluence**. A trade signal is only generated when the market regime (Hurst), smoothed momentum (Kalman), and volatility-based price action (Supertrend cross) align.
    *   The core interaction is a **Threshold Cross**. The trigger is the Kalman Filter (`kf`) crossing one of the adaptive Supertrend bands. However, the thresholds themselves are dynamic, constantly being adjusted by the Hurst exponent, making it a "smart" threshold system.

### 3. The Execution Engine

The final signal is the result of a clear boolean state machine based on the interaction between the Kalman Filter and the adaptive bands.

*   **Boolean Logic:**
    *   **Trend State (`trend`):** A state variable (`trend`) holds the current market direction (1 for Bullish, -1 for Bearish).
    *   **Bullish Trigger (`turnedBullish`):** This becomes `true` on the single bar where the trend flips from bearish to bullish. The condition for this flip is `kf > prevDn` (the Kalman Filter crosses above the previous period's lower band).
    *   **Bearish Trigger (`turnedBearish`):** This becomes `true` on the single bar where the trend flips from bullish to bearish. The condition for this flip is `kf < prevUp` (the Kalman Filter crosses below the previous period's upper band).
    *   **Exit Logic:** The system operates on a "flip-and-reverse" basis. The trigger for a new bullish trend is simultaneously the exit signal for a bearish trend, and vice versa.

*   **Mathematical Constants:**
    *   **`active_atr_base` (Default: 1.5):** This is a critical risk-management constant. It establishes a **minimum risk floor**. It ensures that even in a perfectly trending market (H=1.0), the stop-loss level (the band) is at least 1.5 ATRs away from the Kalman Filter line, preventing excessively tight stops that are prone to noise.
    *   **`active_atr_hscale` (Default: 3.0):** This constant dictates the **defensive posture** of the strategy. It controls how aggressively the bands widen in choppy markets. A higher value means the system will require a much larger volatility-adjusted price move to trigger a signal when the Hurst exponent is low, significantly increasing its immunity to whipsaws.
    *   **`0.5` in `(0.5 + safeH)`:** This is a centering constant for the Kalman gain. It ensures that in a perfectly random market (H=0.5), the adaptive gain multiplier is exactly 1.0, causing the filter to operate at its user-defined base gain.
    *   **`1.0` in `(1.0 - safeH)`:** This is an inversion constant for the band multiplier. It translates a high Hurst value (trending) into a low multiplier and a low Hurst value (choppy) into a high multiplier, achieving the desired adaptive behavior.
    