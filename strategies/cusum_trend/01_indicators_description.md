
# Indicators Description

### 1. Component Deconstruction

#### Hull Moving Average (HMA)
*   **Specific Configuration:**
    *   **Price Source:** `close`
    *   **Length (`base_len`):** Dynamically configured based on the "Sensitivity" input.
        *   `Fast (Day Trade)`: 14 periods
        *   `Balanced (Swing)`: 21 periods
        *   `Slow (Trend)`: 50 periods
*   **Functional Modification:** The HMA is not used for direct crossover signals. Instead, it serves as a fast, low-lag dynamic baseline. Its primary function is to provide a reference point for calculating the `residual`, which is the core input for the CUSUM engine.

#### Standard Deviation (StDev)
*   **Specific Configuration:**
    *   **Price Source:** `residual` (defined as `close - hma_base`). This is a non-standard source.
    *   **Length (`base_len`):** The same dynamic length used by the HMA (14, 21, or 50).
*   **Functional Modification:** This is a custom volatility measurement. Instead of measuring the volatility of price itself, it measures the volatility of **price's deviation from the HMA baseline**. This creates an adaptive volatility value that is specific to the current trend conditions. It is used to normalize the CUSUM parameters (`k_drift` and `h_thresh`), making them responsive to changes in market character. A failsafe (`na(res_std) or res_std == 0 ? 0.001 : res_std`) prevents division-by-zero errors in flat markets.

#### CUSUM Pressure Accumulators (`bullPressure`, `bearPressure`)
*   **Specific Configuration:** These are custom-coded recursive variables, not standard Pine Script indicators.
    *   **Bull Pressure Formula:** `bullPressure := math.max(0, nz(bullPressure[1]) + residual - k_drift)`
    *   **Bear Pressure Formula:** `bearPressure := math.max(0, nz(bearPressure[1]) - residual - k_drift)`
*   **Functional Modification:** This is the mathematical core of the script, adapting the CUSUM control chart methodology.
    1.  **`nz(pressure[1])`**: It begins with the accumulated pressure from the previous bar.
    2.  **`+ residual` / `- residual`**: It adds the current bar's deviation from the HMA. For `bullPressure`, a positive `residual` (price > HMA) increases the value. For `bearPressure`, a negative `residual` (price < HMA) is inverted by the minus sign, also increasing its value.
    3.  **`- k_drift`**: This is a constant "decay" or "cost" factor applied on every bar. It ensures that pressure only accumulates if the `residual` is consistently larger than this noise-filtering threshold. If the `residual` is smaller than `k_drift`, the total pressure will decrease.
    4.  **`math.max(0, ...)`**: This is a critical "ratchet" mechanism. It prevents the pressure values from ever becoming negative. If the directional momentum falters and the calculation results in a negative number, the pressure is simply reset to zero. This ensures that past, opposing movements do not detract from a new accumulation of pressure.

### 2. Logic Layering & Confluence

The script employs a hierarchical filtering system to distinguish statistically significant trend shifts from market noise.

*   **Interaction Dynamics:** The primary dynamic is a **Threshold Cross** system. The engine does not look for convergences or divergences in the traditional sense but rather for the accumulation of evidence (`bullPressure` or `bearPressure`) to surpass a statistically defined threshold (`h_thresh`).

*   **Hierarchical Filtering:**
    1.  **Level 1 Filter (The Baseline):** The **HMA** acts as the foundational layer, establishing the market's short-term mean. All subsequent calculations are dependent on this baseline.
    2.  **Level 2 Filter (The Noise Gate):** The `k_drift` variable (`res_std * k_mult`) functions as the first gatekeeper. On each bar, the `residual` (price deviation) must be greater than this volatility-adjusted `k_drift` for any pressure to accumulate. This effectively filters out minor price oscillations around the HMA, which are classified as noise. The `k_mult` parameter directly controls the strictness of this gate.
    3.  **Level 3 Filter (The Confirmation Gate):** The `h_thresh` variable (`res_std * h_mult`) is the final gatekeeper. A signal is only generated when the *cumulative* pressure (`bullPressure` or `bearPressure`), built up over multiple bars, successfully breaches this much higher threshold. This ensures that a signal is the result of *persistent, directional momentum*, not a single anomalous price spike, thereby maximizing the signal-to-noise ratio.

### 3. The Execution Engine

#### Trigger Conditions
The script's entry signals are governed by a state machine (`regime`) which is updated by boolean trigger variables.

*   **Boolean Logic (Primary Triggers):**
    *   `trig_bull = (bullPressure > h_thresh) AND barstate.isconfirmed`
    *   `trig_bear = (bearPressure > h_thresh) AND barstate.isconfirmed`
    *   The inclusion of `barstate.isconfirmed` is a critical anti-repainting mechanism, ensuring that a trigger is only valid on the close of a bar.

*   **State Change Logic (Regime Definition):**
    *   **Entry into Bull Regime:** `regime` is set to `1` if `trig_bull` is true.
    *   **Entry into Bear Regime:** `regime` is set to `-1` if `trig_bear` is true.
    *   Upon a trigger event (`trig_bull` or `trig_bear`), both `bullPressure` and `bearPressure` are immediately reset to `0.0`. This re-arms the CUSUM engine, forcing it to wait for a new accumulation of evidence for the next signal.

*   **Signal Generation (`bull_start`, `bear_start`):** The actual alert and label events are triggered by detecting the *first bar* of a new regime.
    *   `bull_start = regime == 1 and regime[1] != 1`
    *   `bear_start = regime == -1 and regime[1] != -1`
    *   This is an edge-detection technique that isolates the exact moment the state change occurs.

#### Exit Conditions
The exit logic is simpler than the entry, defined by a reversion to a neutral "Ranging" state.

*   **Boolean Logic (Exit from Regime):**
    *   **Exit Bull Regime:** `regime` is reset to `0` if `regime == 1` and `close < dn_band` on a confirmed bar.
    *   **Exit Bear Regime:** `regime` is reset to `0` if `regime == -1` and `close > up_band` on a confirmed bar.
    *   The `up_band` and `dn_band` are defined as `hma_base +/- h_thresh`, effectively making the exit trigger a cross of the opposing confirmation threshold band. This band also functions as the plotted trailing stop.

#### Mathematical Constants
The script's behavior is heavily influenced by two user-abstracted multipliers.

*   **`k_mult` (Drift Multiplier):**
    *   **Values:** 0.4 (Fast), 0.5 (Balanced), 0.6 (Slow)
    *   **Significance:** This constant controls the **sensitivity of accumulation**. It determines the magnitude of the `k_drift` noise filter relative to volatility. A lower `k_mult` creates a smaller hurdle for the `residual`, allowing pressure to build more easily and increasing signal frequency at the risk of capturing more noise.

*   **`h_mult` (Threshold Multiplier):**
    *   **Values:** 2.0 (Fast), 3.0 (Balanced), 4.0 (Slow)
    *   **Significance:** This constant controls the **signal confirmation threshold**. It dictates how many standard deviations of accumulated pressure are required to validate a trend. A higher `h_mult` demands a greater body of evidence (more persistent momentum) before firing a signal, resulting in fewer but potentially higher-conviction trades. It directly impacts the script's lag and confirmation level.
