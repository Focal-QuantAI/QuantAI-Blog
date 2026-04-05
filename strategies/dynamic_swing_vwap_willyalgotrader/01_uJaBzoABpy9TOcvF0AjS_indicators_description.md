
# Indicators Description

### 1. Component Deconstruction

This script's engine is constructed from several core components, some standard and some functionally modified for this specific strategy.

#### Swing Pivot Detector
*   **Specific Configuration:**
    *   **Indicator:** `ta.highestbars()` and `ta.lowestbars()`
    *   **Lookback Period:** `prdInput` (default: `55` bars)
    *   **Price Source:** `high` for swing highs, `low` for swing lows.
*   **Functional Modification:**
    *   The script does not merely detect a pivot; it **classifies** it. After a new pivot is confirmed, its price (`ph` or `pl`) is compared to the price of the *previous opposing pivot* (`prev`).
    *   This comparison results in a structural label:
        *   **HH (Higher High):** New swing high is above the previous swing high.
        *   **LH (Lower High):** New swing high is below the previous swing high.
        *   **LL (Lower Low):** New swing low is below the previous swing low.
        *   **HL (Higher Low):** New swing low is above the previous swing low.
        *   **EH/EL (Equal High/Low):** Pivots at the same price level.
        *   **IH/IL (Initial High/Low):** The first pivots on the chart.
    *   This classification layer transforms a simple pivot signal into a market structure analysis tool, which is then used for filtering.

#### Volume Processor
*   **Specific Configuration:**
    *   **Function:** Custom `getVol()` function.
    *   **Inputs:** Raw `volume`, a 50-period `ta.median(volume)`, and a 20-period `ta.sma(volume)`.
*   **Functional Modification:**
    *   This is a "hacked" volume source designed to normalize volume data and improve the signal-to-noise ratio of the subsequent VWAP calculation.
    *   **Volume Capping:** If `volCapInput` is `true`, any volume bar exceeding 3x the 20-period SMA of volume is capped at that 3x level (`volSma20 * 3.0`). This prevents anomalous, high-volume events (e.g., news spikes, exchange glitches) from disproportionately skewing the VWAP.
    *   **Volume Floor:** For bars with zero volume, the script substitutes the 50-period median volume. This ensures the mathematical stability of the VWAP calculation, preventing division-by-zero errors and maintaining a logical flow on assets with intermittent volume.

#### Adaptive Price Tracking (APT) Engine
*   **Specific Configuration:**
    *   **Base Period:** `baseAptInput` (default: `21.0`), representing a baseline smoothing half-life.
    *   **Volatility Measure:** 50-period `ta.atr()` and a 50-period `ta.rma()` of the ATR.
    *   **Sensitivity Control:** `volBiasInput` (default: `10.0`), an exponent that controls the intensity of the adaptation.
*   **Functional Modification:**
    *   This is a custom volatility-adaptive smoothing mechanism. It dynamically adjusts the VWAP's responsiveness.
    *   **Mathematical Logic:**
        1.  A `ratio` is calculated: `atr / ta.rma(atr, 50)`. This ratio is `> 1.0` when current volatility is higher than its recent average and `< 1.0` when it is lower.
        2.  If adaptation is enabled (`useAdaptInput` is `true`), the effective APT is calculated as: `aptRaw = baseAptInput / math.pow(ratio, volBiasInput)`.
        3.  **Effect:** When volatility expands (`ratio` > 1), the denominator increases, causing the `aptRaw` value to *decrease*. A lower APT value translates to a larger EWMA alpha (faster smoothing), making the VWAP track price more aggressively. Conversely, in low volatility, the APT increases, resulting in a smoother, slower VWAP. The `volBiasInput` acts as a sensitivity multiplier for this effect.

#### Exponentially Weighted Moving Average (EWMA) VWAP
*   **Specific Configuration:**
    *   **Price Source:** `srcInput` (default: `hlc3`).
    *   **Volume Source:** The custom `getVol()` function.
    *   **Smoothing Factor (Alpha):** Not fixed. It is derived from the APT engine's output using the half-life formula: `alpha = 1.0 - math.exp(-math.log(2.0) / APT)`.
*   **Functional Modification:**
    *   This is not a continuous, standard VWAP. It is a **re-anchoring, segmented EWMA VWAP**.
    *   **Reset Mechanism:** The core of the script. Upon the detection of a valid swing pivot, the running EWMA sums for price-volume (`p`) and volume (`vol`) are completely discarded.
    *   **Re-Seeding:** The calculation is re-seeded with the price and volume values from the exact bar where the new pivot occurred.
    *   **Backfilling:** To ensure perfect continuity from the anchor point, the script performs a historical backfill. It iterates from the pivot bar forward to the current bar, recalculating the entire EWMA segment. This computationally intensive process guarantees the VWAP line originates precisely from the anchor pivot.
    *   **Smooth Re-Anchor:** An optional modification (`smoothAnchorInput`) that reduces initial jitter. For the first 5 bars of a new segment, the calculated `alpha` is ramped up from 50% to 100% of its full value. This eases the VWAP into its new trajectory.

#### ATR-Based Deviation Bands
*   **Specific Configuration:**
    *   **Central Line:** The Dynamic Swing VWAP itself.
    *   **Width Calculation:** `bandMultInput` (default: `0.618`) multiplied by a smoothed ATR value.
*   **Functional Modification:**
    *   The band width is not based on the raw `ta.atr()`. It uses an **EWMA of the ATR** (`ewmaAtr`).
    *   This `ewmaAtr` is calculated in lockstep with the VWAP itself, using the same adaptive `alpha` and subject to the same re-anchoring and backfilling logic. This ensures that the band width's responsiveness is perfectly synchronized with the VWAP's responsiveness.

### 2. Logic Layering & Confluence

The script's intelligence lies in its hierarchical filtering system, where each component acts as a gatekeeper for the next.

*   **Interaction Dynamics:** The primary dynamic is a **Signal Reset**. The system is not looking for indicators to agree (convergence) but for a primary structural signal (a pivot) to completely reset and re-contextualize a secondary tracking indicator (the EWMA VWAP).

*   **Hierarchical Filtering:**
    1.  **Level 1 Gatekeeper: Swing Pivot Detection.** The entire engine is dormant until `ta.highestbars` or `ta.lowestbars` returns `0`. This is the master trigger that initiates any potential change in state.
    2.  **Level 2 Gatekeeper: Structural Classification & Filtering.** Once a pivot is detected, it is not automatically acted upon. It is first classified (HH, LL, etc.). The `onlyLLHHInput` boolean then acts as a secondary filter.
        *   If `true`, only pivots that represent a continuation of the macro trend (`LL` in a downtrend, `HH` in an uptrend) are allowed to pass and trigger a VWAP reset.
        *   Contrarian pivots (`HL` in a downtrend, `LH` in an uptrend) are labeled but are filtered out, causing the existing VWAP segment to continue uninterrupted. This filter is designed to increase the probability of trading with the dominant structural trend.
    3.  **Core Engine Execution:** Only if a pivot signal passes both Level 1 and Level 2 gates does it trigger the **re-anchoring** of the EWMA VWAP engine. This layering ensures that the computationally expensive backfilling process is only initiated on high-conviction structural signals.

### 3. The Execution Engine

The script's primary output is the visual VWAP line, but its execution logic is defined by the alert conditions.

*   **Trigger Conditions (Boolean Logic):**
    The alert trigger is a boolean variable `wasAnchor`, which resolves to `true` only when a specific sequence of events occurs on a confirmed bar close.
    1.  `dirFlipped = dir != dir[1] and barstate.isconfirmed`: A new swing high or low has been officially confirmed on the close of the current bar. `dir` changes from `1` to `-1` or vice-versa.
    2.  `doAnchor`: This boolean evaluates whether the confirmed swing is of a type that should trigger a reset. Its logic is: `isFirstSwing or (onlyLLHHInput ? isLLHH : true)`.
        *   This means a reset will *always* happen on the first pivots (`isFirstSwing`).
        *   After that, if `onlyLLHHInput` is `false`, *any* pivot will cause a reset (`true`).
        *   If `onlyLLHHInput` is `true`, a reset will only occur if the pivot is a Higher High or Lower Low (`isLLHH`).
    3.  `wasAnchor = dirFlipped and doAnchor`: The final alert condition. An alert is generated only when a new swing is confirmed (`dirFlipped`) AND that swing passes the structural filter (`doAnchor`).

*   **Exit Conditions:**
    *   The script, as an `indicator`, does not contain explicit exit logic (e.g., take profit, stop loss). Its purpose is to define the trend and its dynamic support/resistance. An exit strategy would need to be built on top of this, for example, by triggering an exit when price crosses back over the VWAP or when a new, opposing `wasAnchor` signal is generated.

*   **Mathematical Constants:**
    *   **`3.0`:** The multiplier in the `volCap` calculation. Hard-coded to define an "extreme" volume spike as anything over 300% of the 20-bar average. It governs the script's resilience to volume outliers.
    *   **`math.log(2.0)` (approx. 0.693):** A fundamental constant in the formula to convert a "half-life" period into an equivalent EWMA smoothing factor (`alpha`). Its presence is a direct consequence of the mathematical definition of half-life.
    *   **`0.618`:** The default `bandMultInput`. This is the Golden Ratio, a number from Fibonacci sequence theory. Its use implies a design choice to set default deviation bands at a level with perceived technical significance, rather than a purely statistical one (like a standard deviation of 1.0 or 2.0).
    *   **`5`:** The number of bars used for the `smoothAnchorInput` ramp. This hard-coded value defines the duration of the transition period for a new VWAP segment, balancing smoothness against responsiveness.
    