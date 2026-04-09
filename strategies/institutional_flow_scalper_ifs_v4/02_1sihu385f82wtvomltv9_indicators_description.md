
# Indicators Description

### 1. Component Deconstruction

This section dissects each individual indicator and calculation engine within the script, detailing its configuration and any mathematical modifications.

#### **Volatility & Sensitivity Engine**

*   **Average True Range (ATR)**
    *   **Specific Configuration:** `ta.atr(atrLength)` with a default `atrLength` of 14.
    *   **Functional Modification:** The raw ATR value (`atrVal`) is used as the base unit for risk management (Stop Loss/Take Profit). It is also fed into an adaptive sensitivity modifier.

*   **Adaptive Sensitivity Multiplier (`adaptiveMult`)**
    *   **Specific Configuration:** Uses `ta.percentrank(atrVal, 100)` to rank the current ATR value against its last 100 values.
    *   **Functional Modification:** This is a custom volatility regime filter. It modifies the final `Confidence Score` based on market volatility:
        *   **High Volatility (`atrPctRank > 70`):** `adaptiveMult` becomes `0.7`. This *reduces* the confidence score, making signals harder to trigger. The logic is that high volatility produces more noise and false signals, requiring a higher conviction threshold.
        *   **Low Volatility (`atrPctRank < 30`):** `adaptiveMult` becomes `1.3`. This *inflates* the confidence score, making signals easier to trigger. The logic is to increase sensitivity in quiet markets where significant moves are less frequent.
        *   **Normal Volatility:** `adaptiveMult` is `1.0` (no change).
        *   The user can override this with fixed "Low" (0.7) or "High" (1.3) settings.

#### **Volume & Momentum Analysis**

*   **Synthetic Volume Delta (CVD)**
    *   **Specific Configuration:** This is a custom, non-standard calculation. It does not use `request.security()` to pull tick data but rather approximates delta from bar data.
    *   **Functional Modification:**
        1.  **Close Position Ratio:** `closePosition = (close - low) / (high - low)` calculates the closing price's position within the bar's range (0 for closing at the low, 1 for closing at the high).
        2.  **Volume Apportionment:** `buyVolume = volume * closePosition` and `sellVolume = volume * (1.0 - closePosition)`. This model assumes buying pressure is proportional to how high the bar closes, and vice-versa.
        3.  **Bar Delta:** `volumeDelta = buyVolume - sellVolume`.
        4.  **Session Accumulation:** `sessionCVD` accumulates `volumeDelta` throughout a single trading day (`isNewSession = ta.change(time("D")) != 0`), resetting at the start of each new day. This provides a session-specific view of net buying/selling pressure.
        5.  **Smoothing:** The raw `sessionCVD` is smoothed using `ta.ema` with a default length of 5 (`cvdSmoothing`) to reduce noise and identify the underlying trend in order flow.

*   **CVD Divergence**
    *   **Specific Configuration:** Uses `ta.pivotlow` and `ta.pivothigh` with a lookback of 10 (`swingLookback`) to identify price swing points.
    *   **Functional Modification:** It implements a classic divergence detection engine:
        *   **Bullish Divergence:** Triggers `true` if a new price pivot low is lower than the previous one, BUT the corresponding `cvdSmoothed` value at the new pivot is higher than at the previous one.
        *   **Bearish Divergence:** Triggers `true` if a new price pivot high is higher than the previous one, BUT the corresponding `cvdSmoothed` value is lower. This is a non-standard implementation as it uses `ta.pivotlow(-high)` to find high pivots.

*   **Pressure Index**
    *   **Specific Configuration:** `ta.ema(volumeDelta / math.max(volume, 1) * 100, cvdSmoothing)`.
    *   **Functional Modification:** This oscillator normalizes the `volumeDelta` by the total volume of the bar, creating a percentage-based measure of net pressure. It is then smoothed with an EMA (default length 5). It is designed to be a more sensitive, lag-reduced indicator of immediate buying or selling pressure compared to the cumulative CVD.

#### **Event-Based Pattern Detection**

*   **Liquidity Sweep**
    *   **Specific Configuration:** Uses `ta.pivothigh` and `ta.pivotlow` with a lookback of 10 (`swingLookback`) to establish recent swing points.
    *   **Functional Modification:** A sweep is a multi-condition boolean event. A `bearishSweep` requires:
        1.  Price pierces the most recent swing high (`high > recentSwingHigh`).
        2.  Price fails to hold the level, closing back below it (`close < recentSwingHigh`).
        3.  The candle is bearish (`close < open`).
        4.  The event occurs on high volume (`volume > ta.sma(volume, 20) * sweepThreshold`).
        The logic for `bullishSweep` is the inverse.

*   **Order Absorption**
    *   **Specific Configuration:** Compares current volume to a 20-period SMA of volume.
    *   **Functional Modification:** Identifies candles indicative of large passive orders absorbing aggressive market orders. It requires:
        1.  **High Volume:** `volume > avgVolume * absorbVolMult` (default: > 2.0x average).
        2.  **Indecision:** The candle's body is a small fraction of its total range (`(bodySize / barRange) < absorbBodyRatio`, default: < 30%). This signifies a battle between buyers and sellers with no clear winner, despite high volume.

#### **Market Structure & Regime Filters**

*   **Institutional VWAP (Volume-Weighted Average Price)**
    *   **Specific Configuration:** This is a session-based VWAP, manually calculated to reset daily. It also calculates the session's standard deviation of price from the VWAP.
    *   **Functional Modification:**
        *   **VWAP:** `sumPV / sumVol`.
        *   **Standard Deviation:** `math.sqrt(sumPV2 / sumVol - vwapVal * vwapVal)`. This is the correct mathematical formula for population standard deviation for a weighted dataset.
        *   **Bands:** The standard deviation is multiplied by user-defined values (`vwapBandMult1`, `vwapBandMult2`) to create deviation bands, which act as dynamic support/resistance zones.

*   **Point of Control (POC)**
    *   **Specific Configuration:** `ta.sma(typicalPrice * volume, pocLength) / ta.sma(volume, pocLength)`. Default `pocLength` is 20.
    *   **Functional Modification:** This is **not** a traditional volume profile POC. It is a **moving VWAP** with a fixed lookback period of 20 bars, making it a shorter-term measure of the volume-weighted price center.

*   **Choppiness Index**
    *   **Specific Configuration:** Standard implementation with a default `chopLength` of 14.
    *   **Functional Modification:** Calculates `100 * log10(ATR_Sum / HL_Range) / log10(Length)`. A value above the `chopThreshold` (default 65.0) defines the market as "choppy," deactivating all trade signals. This acts as a master "regime filter."

*   **Session Awareness**
    *   **Specific Configuration:** Uses the `time()` function to check if the current bar falls within user-defined session windows (NY, London).
    *   **Functional Modification:** Creates a boolean flag `inKillZone`. This acts as a gatekeeper, only allowing signals during these specified high-liquidity periods. It also applies a `sessionMult` of 1.2 to the `Confidence Score` during these times, increasing the likelihood of a signal.

### 2. Logic Layering & Confluence

The script's engine avoids reliance on any single indicator by layering multiple conditions in a hierarchical structure.

*   **Interaction Dynamics:** The core logic is built on **Confluence** and **Threshold Crosses**. It does not simply look for indicators to agree, but quantifies the degree of agreement through a scoring system.

*   **Hierarchical Filtering:** The signal generation follows a strict, multi-stage filtering process:

    1.  **Stage 1: The "Pillars" (Confluence Check):**
        *   The script defines 7 distinct bullish and 7 bearish "pillars" of evidence (e.g., CVD Divergence, Liquidity Sweep, VWAP Bounce).
        *   A trade setup is only considered if a minimum of **2 pillars** are present (`bullPillarCount >= 2`). This is the first and most crucial confluence requirement. A single event, like a sweep, is insufficient on its own.

    2.  **Stage 2: The "Bias" (Directional Check):**
        *   A separate, weighted scoring system (`bullishBias`, `bearishBias`) determines the dominant market pressure. Conditions deemed more significant (e.g., a Sweep or Divergence) are given 2 points, while others (e.g., EMA alignment) receive 1.
        *   The trigger requires the bias to align with the trade direction (e.g., `bullishBias > bearishBias` for a long). This prevents taking long trades in an overwhelmingly bearish environment, even if two bullish pillars are present.

    3.  **Stage 3: The "Confidence Score" (Strength Check):**
        *   This is a granular scoring system (0-100) where different components contribute points to a `rawConfidence` total (e.g., a CVD divergence adds 20 points, an absorption event adds 6).
        *   This score is then dynamically adjusted by the `adaptiveMult` (volatility) and `sessionMult` (time of day).
        *   The final `confidence` must exceed the `minConfidence` threshold (default 30). This acts as a quantitative filter for signal quality.

    4.  **Stage 4: The "Gatekeepers" (Regime & State Check):**
        *   Even if all the above conditions are met, a signal is only generated if it passes three final boolean checks:
            *   `passesChop`: The market must **not** be in a choppy state.
            *   `passesSession`: The trade must occur within a designated "Kill Zone."
            *   `posState == 0`: The system must be flat (no active trade). This prevents pyramiding.
            *   `signalCooldown`: A minimum of 3 bars must pass between signals.

### 3. The Execution Engine

#### **Trigger Logic**

The final boolean logic for a long entry (`longEntry`) is the culmination of the entire filtering hierarchy:

`longSignal = (bullishBias > bearishBias) and (bullPillarCount >= 2) and (confidence >= minConfidence) and (not isChoppy) and (inKillZone) and barstate.isconfirmed`

`longEntry = longSignal and (bar_index - lastSignalBar >= 3) and (posState == 0)`

*   **`bullishBias > bearishBias`**: The weighted directional evidence must be bullish.
*   **`bullPillarCount >= 2`**: At least two distinct bullish patterns must have occurred.
*   **`confidence >= minConfidence`**: The quantitatively measured signal strength must pass the minimum threshold.
*   **`not isChoppy` and `inKillZone`**: The market regime and session time must be permissive.
*   **`signalCooldown` and `posState == 0`**: State management rules must be satisfied.

#### **Exit Logic**

The script employs a hybrid exit strategy with both static and dynamic conditions.

*   **Static Exits (Risk Management):**
    *   **Stop Loss:** `close - atrVal * atrMultSL` (for longs). The stop is placed at a distance determined by a multiple of the ATR at the time of entry.
    *   **Take Profit:** `close + atrVal * atrMultTP` (for longs). The target is also based on the entry-time ATR.
    *   **Mathematical Constants:**
        *   `atrMultSL` (default 1.5): A multiplier of 1.5 means the initial risk is 1.5 times the 14-period ATR. A smaller value implies a tighter stop and higher risk of being stopped out by noise.
        *   `atrMultTP` (default 1.0): A multiplier of 1.0 sets the initial reward target at 1x the 14-period ATR. With default settings, this establishes an initial Risk-to-Reward ratio of **1.5 : 1.0**, meaning the potential loss is greater than the potential reward. This profile is typical of high-win-rate scalping strategies where the goal is to capture small, probable moves.

*   **Dynamic Exit (Momentum-Based):**
    *   **`longMomExit = posState == 1 and momentumBearish and pressureIndex < -5`**
    *   This condition allows for an early exit if momentum shifts against the position before the SL or TP is hit. It requires:
        1.  A bearish divergence between price ROC and CVD ROC (`momentumBearish`).
        2.  The immediate selling pressure (`pressureIndex`) to cross a hard-coded threshold of -5.
    *   This acts as a mechanism to protect profits or cut losses early if the reversal thesis of the trade entry fails to materialize.
    