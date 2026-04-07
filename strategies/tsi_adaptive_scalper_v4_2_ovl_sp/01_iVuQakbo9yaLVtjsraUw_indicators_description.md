
# Indicators Description

### 1. Component Deconstruction

This analysis deconstructs the core mathematical and logical components of the `TSI Adaptive Scalper v4.2 OVL-SP` script.

#### **A. True Strength Index (TSI)**
The central oscillator of the strategy.

*   **Specific Configuration:**
    *   **Price Source:** `close`.
    *   **Long Period (`fLong`):** Adaptive, determined by a matrix of Timeframe Class and Instrument Type. Ranges from 13 (M1/Metal) to 40 (D1/Forex).
    *   **Short Period (`fShort`):** Adaptive, determined by the same matrix. Ranges from 7 (M1/Metal) to 20 (D1/Forex).
    *   **Signal Period (`fSignal`):** Adaptive, determined by the matrix. An EMA of the final TSI value. Ranges from 5 to 18.
    *   **Overbought/Oversold Levels (`lvlOB`/`lvlOS`):** Base levels are adaptive via the matrix (e.g., 25.0 for M1/Metal, 42.0 for D1/Index). These base levels are then dynamically adjusted.

*   **Functional Modification:** The script employs two significant, non-standard modifications to the TSI calculation:
    1.  **Adaptive Post-Smoothing (`tsiAdapt`):** The raw TSI value (`tsiRaw`) is subjected to a custom smoothing filter before being used as the final `tsi` value. The formula is `tsiAdapt := activeAlpha * tsiRaw + (1.0 - activeAlpha) * nz(tsiAdapt[1], tsiRaw)`.
        *   **Logic:** The smoothing factor, `activeAlpha`, is dynamically calculated based on the ratio of a fast ATR (10-period) to a slow ATR (40-period).
        *   **Intended Effect:** During high volatility (high ATR ratio), `activeAlpha` increases, making the TSI more responsive and reducing lag. During low volatility, `activeAlpha` decreases, increasing the smoothing effect to filter out market noise. This creates a volatility-normalized momentum reading.
    2.  **Dynamic Volatility Thresholds (`dynOB`/`dynOS`):** The OB/OS levels are not static. They are calculated by multiplying the base level (`baseOB`) by a volatility ratio (`clampedVol`).
        *   **Logic:** `clampedVol` is the ratio of a 14-period ATR to a 50-period ATR, clamped between a floor (0.6) and a ceiling (1.6).
        *   **Intended Effect:** In high-volatility regimes, the OB/OS bands widen, requiring a more significant price move to trigger an extreme reading. This reduces false signals during volatile expansion phases. In low-volatility regimes, the bands contract.

#### **B. Exponential Moving Average (EMA)**
Used as a primary trend filter.

*   **Specific Configuration:**
    *   **Price Source:** `close`.
    *   **Length (`emaLenFinal`):** Adaptive, based on Timeframe Class. Ranges from 21 (M1) to 60 (D1).
*   **Functional Modification:** None. It is a standard EMA used to establish a directional bias (`close > emaVal` for bullish).

#### **C. Average Directional Index (ADX)**
Used as a trend strength filter.

*   **Specific Configuration:**
    *   **DI Length:** 14 periods (`adxLen`).
    *   **ADX Smoothing Length:** 14 periods (`adxLen`).
    *   **Threshold (`adxThFinal`):** Adaptive, based on Timeframe Class. Ranges from 20.0 to 23.0.
*   **Functional Modification:**
    *   **Double Smoothing:** An optional EMA (`adxSmoothLen` = 5) is applied to the final ADX value. This is intended to reduce choppiness in the ADX line, providing a more stable reading of trend strength at the cost of minor additional lag.

#### **D. Bollinger Bands (BB) / BB Width**
Used as a consolidation and whipsaw filter.

*   **Specific Configuration:**
    *   **Price Source:** `close`.
    *   **Length (`bbPeriod`):** 20.
    *   **Standard Deviations (`bbStdDev`):** 2.0.
*   **Functional Modification:** The script does not use the bands for signals. It calculates the **Normalized Bollinger Band Width**: `(bbUpper - bbLower) / bbBasis * 100.0`.
    *   **Logic:** This value is compared to a threshold (`consolThr`) to determine if the market is in a state of consolidation. The threshold itself is adaptive, defaulting to the 25th percentile of the BB Width over the last 100 bars.
    *   **Intended Effect:** This module acts as a regime filter. When `isConsolidation` is true, it penalizes the signal score and increases the signal cooldown period, systematically reducing exposure in low-probability, ranging markets.

#### **E. Average True Range (ATR)**
A multi-purpose volatility engine.

*   **Specific Configuration:** Used in three separate contexts:
    1.  **TSI Reactivity:** 10-period (fast) and 40-period (slow).
    2.  **Dynamic Thresholds:** 14-period (fast) and 50-period (slow).
    3.  **TP/SL Calculation:** 14-period (`tpslATRPeriod`).
*   **Functional Modification:** ATR is never used as a direct signal. It serves exclusively as a quantitative measure of volatility to dynamically modify the behavior of other components (TSI, OB/OS levels, and risk parameters).

#### **F. Relative Volume**
A volume-based confirmation filter.

*   **Specific Configuration:**
    *   **Source:** `volume`.
    *   **Lookback Period:** 20-period SMA (`volRelPeriod`).
    *   **Thresholds:** Strong volume is `> 1.3x` the SMA; weak volume is `< 0.6x` the SMA.
*   **Functional Modification:** This is not a binary filter but a scoring modifier. It adds a progressive bonus (`volScoreAdj`) for signals occurring on high volume and a progressive penalty for signals on low volume, directly influencing the final confluence score.

---

### 2. Logic Layering & Confluence

The script's intelligence resides in its hierarchical filtering and confluence-based scoring system, designed to maximize the signal-to-noise ratio.

*   **Interaction Dynamics:** The engine primarily seeks **Confluence** and **Mean Reversion within a Trend**.
    *   **Core Signal:** A `crossover` of the TSI and its signal line.
    *   **Primary Setup (Mean Reversion):** The highest-value signal is a crossover occurring within an extreme zone (e.g., bullish crossover in the oversold zone). This represents the end of a pullback and the potential resumption of the primary trend.
    *   **Secondary Setup (Momentum):** A signal is also generated when the TSI *exits* an extreme zone (e.g., crosses back above the oversold line). This is a momentum-based trigger.
    *   **Divergences:** The script actively scans for classic bullish and bearish divergences between price pivots and TSI pivots, which contribute significantly to the confluence score.

*   **Hierarchical Filtering:** Signals must pass through a series of logical gates before being considered valid. The order of operations is critical:
    1.  **Session Gate:** Signals can be entirely blocked or have their potential score penalized based on the active trading session.
    2.  **Cooldown Gate:** A dynamic cooldown period, adjusted for both volatility (ATR) and consolidation (BB Width), prevents signal clustering.
    3.  **Slope Gate:** Crossovers with insufficient momentum (a shallow angle, measured by `crossSlope`) are immediately filtered out.
    4.  **Regime Gate (Anti-Whipsaw):** If the BB Width module detects consolidation (`isConsolidation`), the signal's score is multiplied by a penalty factor (`consolMultiplier`), and the cooldown is extended. This acts as a powerful filter against chop.
    5.  **Trend Gate (EMA + ADX):** For signals not occurring in extreme zones, they are gated by the primary trend filters. A bullish signal is only considered if `close > EMA` and `ADX > Threshold`. This forces the script to trade pullbacks, not reversals against a strong trend.
    6.  **Multi-Timeframe Gate:** An optional but important filter where the higher-timeframe TSI must either be aligned with the signal direction or, at a minimum, not be in the opposite extreme zone.

*   **Confluence Scoring:** Instead of a binary `true/false` outcome, the script quantifies signal quality via a weighted scoring system (`rawScoreBull`/`rawScoreBear`). Points are awarded for:
    *   **Signal Type:** Base score for a crossover or momentum signal.
    *   **Location:** Bonus for signals in OB/OS zones.
    *   **Confirmation:** Bonuses for ADX confirmation, MTF alignment, structural alignment (BOS/CHoCH), divergence, and strong crossover slope.
    *   **Volume:** A bonus is added for high relative volume; a penalty is subtracted for low volume.
    *   **Final Adjustment:** The raw score is then multiplied by factors for session, composite bias, and consolidation state to produce the final `adjScore`. A signal is only generated if this score surpasses a user-defined threshold.

---

### 3. The Execution Engine

The final trigger and risk management logic are defined by a combination of boolean conditions and mathematical constants.

*   **Boolean Logic (Trigger):**
    A bullish signal (`bullSignal`) is `true` if the cooldown period is over (`bullCDOK`) AND one of the following is `true`:
    1.  **Crossover Signal (`bullSignalCross`):**
        *   A crossover occurs (`ta.crossover(tsi, sig)`).
        *   It passes the slope filter (`slopeOK`).
        *   It passes the session filter (`sessionAllowSignal`).
        *   **AND** it meets one of these path-dependent conditions:
            *   **Path A (Extreme Zone):** The crossover occurs within the oversold zone (`inOSZone`). This is the highest-probability path and bypasses the main trend filter.
            *   **Path B (Near Extreme Zone):** The crossover occurs *near* the oversold zone (`nearOSZone`) AND the ADX confirms a strong trend (`adxOK`).
            *   **Path C (Standard Trend):** The crossover occurs outside extreme zones BUT aligns with all primary filters: trend EMA (`trendBullOK`), ADX (`adxOK`), and MTF (`bullMTFOK`).
    2.  **Momentum Signal (`bullSignalMom`):**
        *   The TSI crosses back *out* of the oversold zone (`momBullSignal`).
        *   It passes the session filter (`sessionAllowSignal`).

*   **Mathematical Constants & Risk Profile:**
    *   **`slATRMult` (Default: 1.2):** This constant sets the base Stop Loss distance as a multiple of the 14-period ATR. A value of 1.2 implies a relatively tight, volatility-adjusted stop suitable for scalping.
    *   **`tp1RR` (Default: 1.5) / `tp2RR` (Default: 2.5):** These constants define the Risk-to-Reward profile. A `tp1RR` of 1.5 sets the first take-profit target at a distance equal to 1.5 times the initial risk (the SL distance). This establishes a positive reward asymmetry from the outset.
    *   **Score-Adjusted Stop Loss (`calcSLMultiplier`):** This is a critical risk management function. It modifies the `slATRMult` based on the signal's final confluence score.
        *   **Logic:** `1.35 - score * 0.05`.
        *   **Effect:** A high-score signal (e.g., 9/10) results in a smaller multiplier (`0.9`), tightening the stop loss because confidence is high. A low-score signal (e.g., 4/10) results in a larger multiplier (`1.15`), widening the stop to allow for more noise. This dynamically links the assumed risk of a trade to its quantified quality.
    *   **`consolMultiplier` (Default: 0.75):** When consolidation is detected, the final score is reduced by 25%. This constant directly controls the script's sensitivity reduction during ranging markets.
    *   **`alertScoreMin` (Default: 4.0):** This hard-coded number acts as the final gatekeeper for execution. No matter how many filters a signal passes, if its final confluence score is below this threshold, it is considered low-probability and will not generate an alert or display TP/SL zones.
    