
    # Indicators Description

    ### 1. Component Deconstruction

This section deconstructs the individual mathematical and logical components that form the foundation of the script's analysis engine.

#### **Core Delta Calculation (`f_delta`)**
*   **Function:** A foundational, reusable function to calculate basic volume delta for a given timeframe (`_tf`).
*   **Configuration:**
    *   **Price Source:** Open (`o`) and Close (`c`) of the candle.
    *   **Volume Source:** `volume` (`v`).
    *   **Lookahead:** `barmerge.lookahead_on`. This is a critical configuration choice, intentionally causing the function to use data from the still-forming higher timeframe candle. This provides real-time data at the cost of repainting.
*   **Mathematical Logic:**
    1.  If `close > open`, delta is `+volume`.
    2.  If `close < open`, delta is `-volume`.
    3.  If `close == open` (a doji), it uses the previous candle's close (`c[1]`) as a tie-breaker:
        *   If `c > c[1]` (price ticked up), delta is `+volume`.
        *   If `c < c[1]` (price ticked down), delta is `-volume`.
        *   Otherwise, delta is `0.0`.

#### **Intrabar Cumulative Volume Delta (CVD)**
*   **Function:** A non-standard, custom calculation that aggregates buying and selling pressure from all available lower-timeframe (`tf_low`) candles within the current primary chart candle. This provides a more granular view of order flow than the standard `f_delta`.
*   **Configuration:**
    *   **Data Source:** Arrays of OHLCV data from `tf_low` via `request.security_lower_tf`.
*   **Functional Modification:** Instead of simple up/down volume, it employs a price-weighted volume split for each sub-candle.
    *   **Buy Volume (`_sBuy`):** `_iv * (_ic - _il) / _sp` where `_iv` is sub-candle volume, `_ic` is close, `_il` is low, and `_sp` is the high-low spread. This attributes a portion of the volume to buyers based on how high the candle closed within its range.
    *   **Sell Volume (`_sSell`):** `_iv * (_ih - _ic) / _sp`. This attributes the remaining volume to sellers.
    *   **Total CVD (`ibCVD_total`):** The sum of all `_sBuy` minus the sum of all `_sSell` across all sub-candles.
    *   **Normalized CVD (`ibCVD_norm`):** `ibCVD_total / ibVol_total`. This scales the net CVD into a range of -1.0 to +1.0, representing the net pressure as a percentage of total intrabar volume.

#### **Delta Exhaustion Score**
*   **Function:** Detects potential trend exhaustion by analyzing the pattern of buying and selling pressure over a series of candles. It operates in two modes.
*   **Configuration:**
    *   **Lookback (Intrabar):** All available sub-candles (`_subDeltaArr`).
    *   **Lookback (Rolling):** Last 6 closed primary candles (`_rollDelta`).
    *   **Threshold:** `exhaustFlips` (default: 3) - the minimum number of times delta must change sign (e.g., from positive to negative).
*   **Functional Modification:** This is a hybrid system.
    *   **Mode Selection:** It uses the intrabar array (`_useIntrabar = _sdSize >= 3`) if at least 3 sub-candles are available; otherwise, it falls back to the rolling array of closed primary candles.
    *   **Flip Counting (`f_countFlips`):** Iterates through the selected array, incrementing a counter (`_fc`) each time the sign of the delta value differs from the previous non-zero delta's sign.
    *   **Magnitude Decline (`_md`):** A secondary condition that checks if the absolute delta of the most recent period is less than 70% of the prior period's delta, signaling fading momentum.
    *   **Trigger:** `deltaExhausted` is `true` if the flip count meets `exhaustFlips` OR if the flip count is at least 2 and magnitude is declining.

#### **Absorption Detection**
*   **Function:** Identifies candles where high volume fails to produce a significant price move, suggesting large passive orders are "absorbing" aggressive market participants.
*   **Configuration:**
    *   **Volume SMA Length:** `absorbSmaLen` (default: 20 bars).
    *   **Volume Multiplier Threshold:** `absorbThreshMult` (default: 1.3). Intrabar volume must be at least 1.3x its 20-period SMA.
    *   **Delta/Volume Ratio:** `absorbDeltaPct` (default: 0.25). The absolute normalized delta (`|ibCVD_total| / ibVol_total`) must be *less than* 25%.
*   **Mathematical Logic:** The `absorbed` boolean is `true` only when:
    1.  `ibVol_total / ta.sma(ibVol_total, 20) >= 1.3` (High Volume Condition)
    2.  `math.abs(ibCVD_total) / ibVol_total < 0.25` (Flat Delta Condition)
    *   **Accumulation vs. Distribution:** The script further classifies this by comparing the `tf_high` close to its open. Absorption on a down-close (`_c5 <= openH`) is labeled `absorbAccum`; on an up-close (`_c5 >= openH`) it's `absorbDist`.

#### **Intrabar Momentum Slope**
*   **Function:** Measures the acceleration or deceleration of delta within the current candle.
*   **Configuration:**
    *   **Lookback:** `slopeLen` (default: 3) sub-candles.
    *   **Minimum Change:** `slopeMinChg` (default: 0.0).
*   **Mathematical Logic:**
    *   `ibMomSlope` is calculated as the delta of the newest sub-candle minus the delta of the sub-candle `slopeLen` periods ago. This is a simplified linear slope calculation.
    *   `ibMomAccel` (Acceleration) is `true` if the slope is positive (and greater than `slopeMinChg`) and the overall `ibCVD_total` is also positive.
    *   `ibMomDecel` (Deceleration) is `true` if the slope is negative while `ibCVD_total` is positive (momentum fading) or if the slope is positive while `ibCVD_total` is negative (selling pressure is easing).

#### **Point of Control (POC) & VWAP**
*   **Intrabar POC:**
    *   **Logic:** Iterates through all `tf_low` sub-candles and identifies the one with the highest volume. The POC price is the midpoint `(high + low) / 2` of that specific sub-candle.
    *   **Rolling POC:** A 5-period simple moving average of the intrabar POC price from the last 5 closed primary candles. This provides a smoothed, lagging value area.
*   **Session VWAP:**
    *   **Configuration:** Anchor can be `UTC midnight` or `Rolling 24h`.
    *   **UTC Midnight Logic:** A standard cumulative VWAP calculation (`sum(tp*v)/sum(v)`) that resets its sums (`_vwapCumTV`, `_vwapCumVol`) when the UTC day changes. `tp` is `(h+l+c)/3`.
    *   **Rolling 24h Logic:** A non-standard implementation that uses `ta.sma` to create a volume-weighted moving average. It calculates `ta.sma(tp * volume, _barsIn24h)` and divides it by `ta.sma(volume, _barsIn24h)`.

#### **Other Standard & Modified Indicators**
*   **DV Flow (Delta Volume):** A standard calculation that splits a candle's volume based on where it closes within its range. `dvRatio` represents the proportion of buying pressure.
*   **Hidden Signal:** A specific application of DV Flow. It's a **divergence** trigger.
    *   `hiddenBuy`: `dvRatio > 0.55` (over 55% buy pressure) on a red candle.
    *   `hiddenSell`: `dvRatio < 0.45` (less than 45% buy pressure, i.e., >55% sell pressure) on a green candle.
*   **OBV Divergence:** A custom 3-bar divergence check. It compares the raw change in OBV (`_obv5 - _obv5[3]`) directly against the raw change in price (`_c5 - _c5[3]`). This is a non-standard, direct slope comparison, avoiding issues with cumulative OBV values.
*   **Trend Filter:**
    *   **Configuration:** Uses a `tf_trend` (default: 60m) timeframe.
    *   **Logic:** Calculates the delta on the trend timeframe (`deltaTrend`) and compares it to its own long-period SMA (`ta.sma(deltaTrend, _trend_len)`). `trend_bull` is `true` if delta is above its SMA.

---

### 2. Logic Layering & Confluence

The script's engine moves from raw data to a final signal through a sophisticated, multi-stage filtering and weighting process.

#### **1. The Primary Score: Weighted Delta**
The initial signal strength is a weighted sum of delta from two timeframes.
*   **Base Weights:** User-defined `wHigh` (65%) and `wLow` (35%).
*   **Dynamic Adjustment:** These weights are not static. They are multiplied by a volatility factor (`_fH`, `_fL`) which is the ratio of the current delta to its long-term SMA, clamped between 0.5 and 1.5. This amplifies the weight during high-delta moves and reduces it during quiet periods.
*   **Confluence Boosts:** Two additional components are added to this score:
    1.  **Hidden Signal Boost (`wtDV_bull`/`wtDV_bear`):** If a `hiddenBuy` or `hiddenSell` signal is present, a fixed bonus of 15% of the total base weight is added to the corresponding bull or bear score.
    2.  **Intrabar CVD Boost (`wtCVD_bull`/`wtCVD_bear`):** If the normalized intrabar CVD (`ibCVD_norm`) is strong ( > 0.2 or < -0.2), a proportional bonus of up to 10% of the total base weight is added.

#### **2. Hierarchical Filtering: The Veto System**
After the initial score (`bull_weight`, `bear_weight`) is calculated, it is passed through a series of "Gatekeeper" conditions that can invalidate a signal.
*   **Gatekeeper 1: Threshold (`thresh`)**
    *   The calculated weight must first exceed a minimum threshold, defined as `sigThresh` % (default 40%) of the total possible weight. A signal with a low weighted score is filtered out immediately.
*   **Gatekeeper 2: DV Veto (`dvVetoBuy`/`dvVetoSell`)**
    *   **Interaction:** This checks for a direct **Divergence**.
    *   **Logic:** A `BUY` signal is vetoed if there is a `hiddenSell` signal active and the bull weight is not overwhelmingly strong (i.e., less than 1.2x the threshold). This prevents entering a long trade when the underlying volume flow of the primary candle is bearish.
*   **Gatekeeper 3: Absorption Veto (`absVetoBuy`/`absVetoSell`)**
    *   **Interaction:** This is another **Divergence** filter.
    *   **Logic:** A `BUY` signal is vetoed if `absorbDist` (Distribution) is detected. This acts as a powerful gatekeeper, preventing trades into a move that is being actively faded by large passive sellers.

#### **3. The Secondary Engines: Reversal & Prediction**
These systems run in parallel to the primary execution engine, providing context and alternative signals.

*   **Reversal Engine:**
    *   **Initial Trigger:** Looks for a **Convergence** of two conditions: `conflict` (MTF deltas opposing) and `near_open` (price close to the candle's open).
    *   **Confluence Check:** It then counts how many of five additional confirming conditions are met (`_c1` to `_c5_rev`), which include DV Flow, Hidden Signals, OBV Divergence, Volume Spikes, and Intrabar CVD confirmation. A signal is only considered "ideal" (`revIdeal`) if it meets at least 3 of these 5 conditions.
*   **Composite Confidence Engine:**
    *   **Interaction:** This is a pure **Confluence** model. It synthesizes nine different metrics into a single score (0-100%).
    *   **Logic:** It takes the primary signal's weight (`_wScore`), intrabar CVD (`_cvdScore`), trend alignment (`_trendS`), and other factors, applies a specific weight to each (e.g., Delta weight is 25%, CVD is 20%), and sums them. This provides a holistic measure of signal quality, layering momentum, trend, and exhaustion factors.
*   **Next Candle Prediction Engine:**
    *   **Interaction:** A weighted voting system where seven factors contribute positively (for UP) or negatively (for DOWN).
    *   **Hierarchical Weighting:** Factors are not equal. `Hidden Signal` has the highest impact (weight 25), while `OBV Divergence` is lowest (weight 7). The final score is then multiplied by the `candlePhase`, giving more authority to predictions made late in the candle's life.

---

### 3. The Execution Engine

This section defines the precise boolean logic for the script's primary trade triggers.

#### **Primary Momentum Signal (`final_buy` / `final_sell`)**

*   **Boolean Logic for `final_buy`:**
    1.  `bull_weight >= ( (wHigh + wLow) * sigThresh / 100 )`
        *   The total calculated bullish score must meet the user-defined threshold.
    2.  AND `deltaH > 0`
        *   The main timeframe delta must be positive (a non-negotiable condition).
    3.  AND `NOT dvVetoBuy`
        *   The signal is not vetoed by an opposing Hidden Signal.
    4.  AND `NOT absVetoBuy`
        *   The signal is not vetoed by the detection of Distribution (absorption).

*   **Boolean Logic for `final_sell`:**
    1.  `bear_weight >= ( (wHigh + wLow) * sigThresh / 100 )`
        *   The total calculated bearish score must meet the user-defined threshold.
    2.  AND `deltaH < 0`
        *   The main timeframe delta must be negative.
    3.  AND `NOT dvVetoSell`
        *   The signal is not vetoed by an opposing Hidden Signal.
    4.  AND `NOT absVetoSell`
        *   The signal is not vetoed by the detection of Accumulation (absorption).

#### **Confirmed Reversal Signal (`revEntry_up_confirmed` / `revEntry_down_confirmed`)**

*   **Boolean Logic for `revEntry_up_confirmed`:**
    1.  `reversal_up` is true, which requires:
        *   `(deltaH < 0 AND deltaL > 0)` (Conflicting MTF delta)
        *   AND `math.abs(close - openH) / openH * 100 < revThresh` (Price is close to the open)
    2.  AND `revIdeal` is true, which requires:
        *   The `near_open` condition from above.
        *   AND `_condCount >= 3` (At least 3 of the 5 confluence conditions are met).
    3.  AND `_c2` is true.
        *   This is the `hiddenBuy` signal. It is a mandatory component for a *confirmed* reversal entry, acting as the final gatekeeper.

#### **Mathematical Constants & Their Influence**

*   **`sigThresh` (default: 40):** Controls the sensitivity of the primary momentum signal. A lower value allows weaker delta scores to generate a signal, increasing frequency but potentially reducing the signal-to-noise ratio. A higher value demands stronger delta confluence, resulting in fewer but higher-conviction signals.
*   **`revThresh` (default: 0.1):** Defines the "near open" zone for reversals as a percentage of the open price. A smaller value (e.g., 0.05) requires price to have moved very little, targeting tight consolidations or failed breakouts. A larger value (e.g., 0.2) allows for reversals after a more significant initial move.
*   **`absorbThreshMult` (default: 1.3):** Governs the sensitivity of the Absorption veto. A lower value (e.g., 1.1) will trigger the absorption logic on smaller volume increases, making the veto more frequent. A higher value (e.g., 2.0) requires a massive volume spike, reserving the veto for rare, high-impact events.
*   **`exhaustFlips` (default: 3):** Sets the threshold for detecting delta exhaustion. A lower number (e.g., 2) will flag exhaustion more readily, potentially acting as an early warning system. A higher number (e.g., 4) requires more definitive back-and-forth action, confirming a true "battle" before signaling exhaustion.
*   **`timePressureMin` (default: 80):** Determines when "Next Candle" and other time-sensitive signals become "live". At 80%, a signal on a 5-minute chart becomes active in the last minute. This constant directly controls the trade-off between getting an early signal and having more complete data from the current candle.
    