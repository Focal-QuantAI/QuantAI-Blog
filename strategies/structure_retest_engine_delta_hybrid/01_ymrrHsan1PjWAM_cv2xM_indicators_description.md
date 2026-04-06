
# Indicators Description

### 1. Component Deconstruction

This section dissects the individual mathematical and logical components used to build the trading model.

#### **A. Price Action & Structure**

*   **Pivot High/Low (`ta.pivothigh`, `ta.pivotlow`)**
    *   **Specific Configuration:**
        *   **Price Source:** `high` for `pivothigh`, `low` for `pivotlow`.
        *   **Lookback Period:** Symmetrical lookback defined by `input.int(pivotLen, default=5)`. A value of 5 requires 5 bars to the left and 5 bars to the right of a candle for it to be confirmed as a pivot.
    *   **Functional Modification:** None. This is a standard implementation used to objectively identify significant swing points in the market structure.

#### **B. Volatility & Zone Definition**

*   **Average True Range (ATR) (`ta.atr`)**
    *   **Specific Configuration:**
        *   **Length:** Hard-coded at `14` periods. This is a common industry standard for volatility measurement.
    *   **Functional Modification:** The ATR value is not plotted directly. It is used as a multiplier to create a dynamic retest zone.
        *   **Mathematical Logic:** `buffer = ta.atr(14) * sensitivity`. The `sensitivity` input (default `0.5`) acts as a scalar for the ATR value.
        *   **Intended Effect:** This creates a volatility-adaptive "Retest Box" around the broken structure level (`activeLevel`). In high-volatility environments, the ATR increases, widening the retest zone to account for larger price swings. In low-volatility, the zone contracts, requiring a more precise retest. This improves the robustness of the retest detection mechanism compared to a static pip/point value.

#### **C. Momentum & Volume Analysis**

*   **"Delta Hybrid" (Custom Function `f_getDelta`)**
    *   **Specific Configuration:** This is a custom, volume-weighted calculation.
    *   **Functional Modification:**
        *   **Mathematical Logic:** `((close - open) / (high - low)) * volume`.
            *   The `(close - open) / (high - low)` term normalizes the candle's body relative to its total range, producing a value between -1.0 and +1.0. This serves as a "Candle Efficiency Ratio." A value of +1.0 means the candle closed at its high with no wicks; a value of -1.0 means it closed at its low.
            *   This ratio is then multiplied by the candle's `volume`.
        *   **Intended Effect:** This calculation serves as a proxy for net order flow or buying/selling pressure within a single candle. It is **not** true delta (which requires bid/ask volume data) but an estimation. A large positive value indicates a strong bullish candle on high volume, suggesting high conviction from buyers. This is used to gauge the strength of both the initial breakout and the sentiment during the retest phase.

*   **Simple Moving Average (SMA) (`ta.sma`)**
    *   **Specific Configuration:** Two distinct SMAs are utilized.
        1.  **Delta MA:** `ta.sma(math.abs(candleDelta), deltaMaLen)`, with `deltaMaLen` defaulting to `20`. The source is the *absolute* value of the custom Delta Hybrid calculation.
        2.  **Volume MA:** `ta.sma(volume, volMaLen)`, with `volMaLen` defaulting to `20`. The source is the standard `volume`.
    *   **Functional Modification:** Both SMAs are not used for trend-following but as dynamic thresholds for filtering signals.
        *   The **Delta MA** is used to determine if a breakout is a "high conviction" event. The condition `math.abs(breakoutDelta) > deltaMa * deltaThresh` checks if the breakout candle's delta was an outlier compared to its recent average.
        *   The **Volume MA** is used to validate the entry confirmation candle. The condition `volume > volMa * volThresh` ensures the retest confirmation is supported by above-average market participation, increasing the signal-to-noise ratio.

### 2. Logic Layering & Confluence

The script's engine filters market noise through a sequential, hierarchical process. A signal is only generated if price action successfully passes through each logical gate.

*   **Gate 1: Structure Identification & State Tracking**
    *   The script maintains a `trendState` variable (`1` for bullish, `-1` for bearish).
    *   It continuously monitors for a `crossover` of the last pivot high (`lastPH`) or a `crossunder` of the last pivot low (`lastPL`).

*   **Gate 2: The "Change of Character" (CHoCH) - The Primary Filter**
    *   **Hierarchical Filtering:** This is the master condition that "arms" the system. A CHoCH is defined as a structural break that opposes the current `trendState`.
        *   **Bullish CHoCH:** `bullishBreak` occurs while `trendState == -1`.
        *   **Bearish CHoCH:** `bearishBreak` occurs while `trendState == 1`.
    *   **Interaction Dynamics:** Upon a CHoCH, the system enters a "monitoring" state. It captures the broken price level (`activeLevel`), the new trend direction (`activeDir`), and the Delta Hybrid value of the breakout candle (`breakoutDelta`). All subsequent logic is dependent on this event having occurred.

*   **Gate 3: The Pullback & Retest Sequence**
    *   **Noise Filtering:** The script explicitly waits for a valid pullback by requiring the `hasLeftLevel` flag to be `true`. This flag is only set when price has fully exited the ATR-defined retest zone after the CHoCH. This prevents false signals from choppy price action that fails to move away from the broken level.
    *   **Threshold Cross:** The system then waits for price to re-enter the retest zone (`priceAtLevel = true`), defined as `low <= activeLevel + buffer and high >= activeLevel - buffer`.

*   **Gate 4: Confluence & Confirmation**
    *   This final layer seeks **Confluence** between price action, confirmation patterns, and optional volume/delta filters before generating an entry signal.
    *   The core logic requires price to have passed through Gates 1-3. It then checks for a specific confirmation type (`Touch`, `Close Outside Level`, or `Engulfing`) in conjunction with optional filters (`requireVol`, `requireCandle`).
    *   The "Delta Hybrid" analysis adds another layer of confluence. The `f_deltaTag` function assesses the quality of the setup by comparing the `breakoutStrong` status (conviction of the initial break) with the `deltaSupportsTrend` status (cumulative delta during the retest phase).

### 3. The Execution Engine

The trigger mechanism is a precise combination of boolean state flags and price-based conditions.

#### **A. Pre-Trigger State Conditions**

Before any entry pattern is evaluated, the following conditions must all be met:
1.  `not levelConsumed`: The active level has not already produced a signal (unless `allowReEntry` is enabled).
2.  `bar_index > lastCHoCHBar`: The current bar is not the same bar as the CHoCH event.
3.  `hasLeftLevel`: The price has demonstrably pulled back from the level and then returned.
4.  `priceAtLevel`: The current candle's high/low is intersecting the ATR-defined retest zone.

#### **B. Boolean Logic for Entry Triggers**

If the pre-trigger state is valid, the script evaluates one of three user-selected confirmation modes.

*   **Mode 1: "Touch"**
    *   **Boolean Logic:** `(activeDir == 1 AND respectsLevel AND bullCandleOK AND volumeOK) OR (activeDir == -1 AND respectsLevel AND bearCandleOK AND volumeOK)`
        *   `respectsLevel`: The candle must close on the "correct" side of the raw pivot line (`close > activeLevel` for bulls).
        *   `bull/bearCandleOK`: Optional filter requiring a bullish/bearish candle body.
        *   `volumeOK`: Optional filter requiring volume to be greater than `volMa * volThresh`.

*   **Mode 2: "Close Outside Level" (COL)**
    *   **Boolean Logic:** This is a stricter version of "Touch". The `respectsLevel` condition is replaced with a requirement for the close to be completely clear of the entire ATR zone.
        *   **Bullish:** `close > activeLevel + buffer`
        *   **Bearish:** `close < activeLevel - buffer`

*   **Mode 3: "Engulfing"**
    *   **Boolean Logic:** This replaces the price-level conditions with a classic candlestick pattern recognition.
        *   **Bullish:** `close > open AND close >= high[1] AND open <= close[1] AND volumeOK`
        *   **Bearish:** `close < open AND close <= low[1] AND open >= close[1] AND volumeOK`

#### **C. Mathematical Constants & Risk Profile Influence**

*   **`sensitivity = 0.5`:** This ATR multiplier directly impacts the entry criteria. A lower value creates a tighter retest zone, leading to fewer but potentially more precise entries. A higher value widens the zone, increasing the chance of an entry but potentially at a less optimal price. It directly influences the trade's initial risk.
*   **`deltaThresh = 1.5`:** A statistical constant defining an "outlier." By requiring the breakout delta to be 1.5x its moving average, the script filters for breakouts driven by abnormally high momentum, which theoretically have a higher probability of continuation.
*   **`volThresh = 1.30`:** A volume confirmation multiplier. Requiring the confirmation candle's volume to be 30% above its moving average filters out rejections that occur on low market participation, which are more likely to be noise. This constant directly manages the signal quality by demanding conviction at the point of entry.
    