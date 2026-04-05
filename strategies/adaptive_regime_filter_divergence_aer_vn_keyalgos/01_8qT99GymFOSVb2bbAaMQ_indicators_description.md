
# Indicators Description

### 1. Component Deconstruction

#### Kaufman's Efficiency Ratio (ER)
*   **Specific Configuration:**
    *   **Lookback Period (`length`):** 10 bars.
    *   **Price Source:** `close`.
*   **Functional Modification:** This is a standard implementation of the Kaufman ER. The mathematical engine is as follows:
    1.  **Displacement (`displacement`):** Calculates the absolute net price change over the lookback period.
        *   `math.abs(close - close[length])`
    2.  **Path Distance (`pathDistance`):** Calculates the cumulative, bar-to-bar price change over the same lookback period. This represents the total "noise" or volatility.
        *   `math.sum(math.abs(close - close[1]), length)`
    3.  **Efficiency Ratio (`er`):** The ratio of net displacement to total path distance.
        *   `er = displacement / pathDistance`
    *   **Intended Effect:** The ER quantifies trend efficiency. A value near 1.0 indicates a highly efficient trend (low noise, high signal), while a value near 0 indicates a choppy, non-directional market (high noise, low signal).

#### Average True Range (ATR)
*   **Specific Configuration:**
    *   **Lookback Period (`atrLength`):** 14 bars.
*   **Functional Modification:** Standard implementation (`ta.atr`). It serves as the raw input for measuring the current volatility magnitude.

#### Simple Moving Average (SMA) of ATR
*   **Specific Configuration:**
    *   **Lookback Period (`atrMeanLen`):** 50 bars.
    *   **Source:** The 14-period `currentAtr` value.
*   **Functional Modification:** This is a custom composite metric. It calculates a long-term moving average of the short-term ATR.
    *   **Intended Effect:** It creates a historical baseline for "normal" volatility. By comparing the current ATR to this 50-period mean, the script can determine if the market is in a high-volatility or low-volatility regime *relative to its recent history*.

#### Adaptive Volatility Threshold
*   **Specific Configuration:**
    *   **Base Threshold (`baseEr`):** 0.25 (default).
    *   **Maximum Cap (`maxErCap`):** 0.65 (default).
*   **Functional Modification:** This is a custom-engineered component that dynamically adjusts the threshold for defining a trend.
    1.  **Volatility Ratio (`atrRatio`):** Normalizes current volatility against its long-term mean.
        *   `atrRatio = currentAtr / meanAtr`
        *   A value > 1.0 signifies above-average volatility; a value < 1.0 signifies below-average volatility.
    2.  **Raw Threshold (`rawThreshold`):** Scales the base ER threshold by the volatility ratio.
        *   `rawThreshold = baseEr * atrRatio`
        *   In high-volatility periods (`atrRatio` > 1), the threshold increases, requiring greater trend efficiency to be classified as a trend. In low-volatility periods (`atrRatio` < 1), the threshold decreases, making it easier to qualify as a trend.
    3.  **Capped Threshold (`dynamicThreshold`):** The final threshold is capped at `maxErCap`.
        *   `dynamicThreshold = math.min(rawThreshold, maxErCap)`
        *   **Intended Effect:** This acts as a safety mechanism. It prevents the required efficiency level from becoming unrealistically high during extreme volatility spikes, ensuring the logic remains functional in all market conditions.

---

### 2. Logic Layering & Confluence

The script's engine uses a hierarchical filtering model to classify the market into one of four distinct regimes.

#### Interaction Dynamics
The primary interaction is a **Threshold Cross** between the Efficiency Ratio (`er`) and the `dynamicThreshold`. This comparison acts as the primary gatekeeper for the entire logic system.

#### Hierarchical Filtering
The classification follows a two-stage decision tree:

*   **Filter 1: Trend Efficiency (The Primary Gatekeeper)**
    *   The script first determines if the market's price action is efficient enough to be considered a trend.
    *   **Condition:** `isTrending = er > dynamicThreshold`
    *   If this condition is `true`, the logic proceeds to the "Trending" branch. If `false`, it proceeds to the "Non-Trending" branch.

*   **Filter 2: Secondary Classification**
    1.  **If `isTrending` is `true`:** The script then filters by direction.
        *   **Uptrend:** `isTrending AND (close > close[length])`
        *   **Downtrend:** `isTrending AND (close < close[length])`
        *   This layer adds a directional vector to the efficiency classification.

    2.  **If `isTrending` is `false` (i.e., `isNonTrending` is `true`):** The script then filters by the volatility regime.
        *   **Chop:** `isNonTrending AND (currentAtr > meanAtr)`
            *   *Interpretation:* Inefficient price action combined with above-average volatility.
        *   **Consolidation:** `isNonTrending AND (currentAtr <= meanAtr)`
            *   *Interpretation:* Inefficient price action combined with below-average volatility (a quiet, range-bound state).

This layered approach ensures that a "trend" is not just directional movement, but *efficient* directional movement, adjusted for the current volatility environment.

---

### 3. The Execution Engine

The execution engine is designed to detect four types of divergences between price and the Efficiency Ratio oscillator. It operates independently of the regime classification but provides the core trade triggers.

#### Pivot Point Detection
The engine first identifies swing highs and swing lows using a non-repainting confirmation method.

*   **Swing High Confirmation (`swingHighConfirm`):**
    *   **Boolean Logic:** `(close[1] == ta.highest(close, divLength)[1]) AND (close < close[1])`
    *   **Mechanics:** This returns `true` only on the bar *after* a pivot high is formed. It confirms that the previous bar's `close` was the highest `close` in the last `divLength` bars, and the current bar's `close` has moved lower, locking in the swing high.

*   **Swing Low Confirmation (`swingLowConfirm`):**
    *   **Boolean Logic:** `(close[1] == ta.lowest(close, divLength)[1]) AND (close > close[1])`
    *   **Mechanics:** Symmetrically, this confirms the previous bar's `close` was the lowest `close` in the last `divLength` bars, and the current bar has moved higher.

#### Divergence Trigger Logic
Upon the confirmation of a new swing point, the script compares its price and ER values to the *previously stored* swing point of the same type (high-to-high, low-to-low).

*   **Boolean Logic for Regular Bearish Divergence (`regBear`):**
    *   `currentHighPrice > lastHighPrice` (Price makes a Higher High)
    *   **AND**
    *   `currentHighER < lastHighER` (Efficiency Ratio makes a Lower High)
    *   **Signal:** Trend exhaustion. The new price high was achieved with less efficiency, indicating weakening momentum.

*   **Boolean Logic for Regular Bullish Divergence (`regBull`):**
    *   `currentLowPrice < lastLowPrice` (Price makes a Lower Low)
    *   **AND**
    *   `currentLowER > lastLowER` (Efficiency Ratio makes a Higher Low)
    *   **Signal:** Selling pressure is waning. The new price low was formed with less directional efficiency than the previous one.

*   **Boolean Logic for Hidden Bearish Divergence (`hidBear`):**
    *   `currentHighPrice < lastHighPrice` (Price makes a Lower High)
    *   **AND**
    *   `currentHighER > lastHighER` (Efficiency Ratio makes a Higher High)
    *   **Signal:** Potential trend continuation. A pullback in price occurs with a surge in underlying efficiency, suggesting the pullback is weak and the dominant downtrend may resume.

*   **Boolean Logic for Hidden Bullish Divergence (`hidBull`):**
    *   `currentLowPrice > lastLowPrice` (Price makes a Higher Low)
    *   **AND**
    *   `currentLowER < lastLowER` (Efficiency Ratio makes a Lower Low)
    *   **Signal:** Potential trend continuation. A pullback in an uptrend occurs with very inefficient (choppy) price action, suggesting the pullback lacks conviction and the primary uptrend may resume.

#### Mathematical Constants & Parameters
The script's behavior is governed by its input parameters, which act as adjustable constants.

*   **`divLength = 10`:** This defines the lookback period for identifying a swing high/low. A smaller value will detect more frequent, shorter-term divergences. A larger value will filter for only major, longer-term divergences, increasing the signal-to-noise ratio at the cost of fewer signals.
*   **`baseEr = 0.25`:** This sets the minimum efficiency required to be considered a trend in a market with average volatility. It directly controls the sensitivity of the regime filter.
*   **`maxErCap = 0.65`:** This is a risk-control parameter. It prevents the `dynamicThreshold` from rising to an impossible level during a "volatility supernova," ensuring the script does not cease to function by classifying everything as non-trending. It defines the absolute maximum efficiency the script will ever demand to confirm a trend.
    