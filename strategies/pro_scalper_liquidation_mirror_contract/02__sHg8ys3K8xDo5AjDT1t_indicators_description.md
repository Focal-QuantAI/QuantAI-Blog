
# Indicators Description

### 1. Component Deconstruction

This section provides a granular breakdown of each technical indicator and custom calculation used in the script's engine.

#### **Trend Baseline Indicators**

*   **Exponential Moving Average (`macroEma`)**
    *   **Specific Configuration:**
        *   **Function:** `ta.ema()`
        *   **Source:** `close`
        *   **Length:** `emaLength` (Default: 50)
    *   **Functional Modification:** None. This is a standard EMA calculation. It serves as a visual reference for the macro trend direction but is **not** used in the boolean logic for signal generation. Its role is purely discretionary for the user.

*   **Supertrend (`direction`)**
    *   **Specific Configuration:**
        *   **Function:** `ta.supertrend()`
        *   **ATR Period:** `stPeriod` (Default: 10)
        *   **Factor/Multiplier:** `stFactor` (Default: 3.0)
    *   **Functional Modification:** Standard implementation. The script exclusively utilizes the second return value, `direction`, which is `1` for an uptrend and `-1` for a downtrend. This component acts as the primary directional "gatekeeper" for the entire strategy.

#### **Momentum & Volatility Studies**

*   **Average Directional Index (`adx`)**
    *   **Specific Configuration:**
        *   **Function:** `ta.dmi()`
        *   **DI Length:** 14 (Hard-coded)
        *   **ADX Smoothing Period:** 14 (Hard-coded)
    *   **Functional Modification:** Standard implementation. The script isolates the third return value, `adx`, which quantifies trend *strength* irrespective of its direction. It is used as a filter to ensure the strategy only operates in high-momentum regimes.

*   **Moving Average Convergence Divergence (`hist`)**
    *   **Specific Configuration:**
        *   **Function:** `ta.macd()`
        *   **Source:** `close`
        *   **Fast EMA Length:** 12 (Standard)
        *   **Slow EMA Length:** 26 (Standard)
        *   **Signal Line SMA Length:** 9 (Standard)
    *   **Functional Modification:** Standard implementation. The script exclusively uses the third return value, the MACD Histogram (`hist`), which represents the difference between the MACD line and the Signal line. This value is a proxy for the rate-of-change of momentum.

*   **Liquidation Proxy (`isLiquidation`)**
    *   **Specific Configuration:**
        *   A 20-period Simple Moving Average (`ta.sma`) is calculated on the `volume` series.
    *   **Functional Modification:** This is a custom boolean flag, not a standard indicator. It compares the current bar's `volume` to the 20-period `volSma`. The logic is: `isLiquidation = volume > (volSma * volLimit)`.
        *   **Intended Effect:** This creates a dynamic threshold for identifying anomalous volume. A bar is flagged as a "liquidation event" only if its volume is significantly higher (default: > 150%) than the recent average volume. It is designed to detect climactic volume spikes that often accompany stop-cascades or capitulation.

#### **Price Structure Analysis**

*   **Price Extremes (`recentHigh`, `recentLow`)**
    *   **Specific Configuration:**
        *   **Functions:** `ta.highest()` and `ta.lowest()`
        *   **Lookback Period:** `breakoutLen` (Default: 20)
        *   **Source:** `high[1]` and `low[1]`
    *   **Functional Modification:** The use of the historical operator `[1]` is critical. The script is not looking at the highest high of the last 20 bars *including the current one*. It is defining a price channel based on the high/low of the **previous 20 bars**. The trigger condition then checks if the *current* bar's high/low has breached this pre-defined channel.

### 2. Logic Layering & Confluence

The script's engine is built on a hierarchical filtering model, where each condition must be met in a specific sequence to validate a signal. This layering is designed to maximize the signal-to-noise ratio.

*   **Layer 1: The Directional Gatekeeper (Macro Trend Filter)**
    *   The **Supertrend `direction`** acts as the master switch. The entire "Bottom Fishing" (Long) logic is disabled if `direction` is not `> 0` (bullish). Conversely, the "Top Escaping" (Short) logic is inert unless `direction` is `< 0` (bearish). This ensures the strategy only takes entries in the direction of the dominant trend defined by the Supertrend.

*   **Layer 2: The Regime Filter (Trend Strength)**
    *   The **ADX (`adxHigh`)** acts as the second gatekeeper. Even if the Supertrend indicates a valid trend direction, the script will not proceed unless the ADX is above the `adxMin` threshold (default: 30).
    *   **Interaction Dynamics:** This filter mandates that the market must be in a strong, trending state. It prevents the algorithm from attempting to catch reversals in low-momentum, choppy, or range-bound conditions where the core philosophy is invalid.

*   **Layer 3: The Event Trigger (Price & Volume Confluence)**
    *   This layer looks for the specific "liquidation event." It requires the **Convergence** of two conditions:
        1.  **Price Breakout:** The current bar's high must exceed the `recentHigh` (for shorts) or its low must fall below the `recentLow` (for longs). This identifies the sharp, counter-trend price thrust.
        2.  **Volume Anomaly:** The `isLiquidation` flag must be `true`. The price breakout must be accompanied by the anomalous volume spike.
    *   **Interaction Dynamics:** A price breakout without a volume spike is ignored as noise. A volume spike without a price breakout is also ignored. The engine requires both to occur on the same bar to identify a potential capitulation event.

*   **Layer 4: The Confirmation Filter (Momentum Exhaustion)**
    *   The **MACD Histogram (`momentumFade`)** is the final confirmation. It programmatically searches for a **Divergence** between price and momentum.
    *   **Interaction Dynamics:** While price is making a new extreme (Layer 3), the `momentumFade` logic checks if the force behind that move is already waning.
        *   For a short (`Top Escaping`), price makes a new high, but the bullish MACD histogram is lower than its previous value, indicating decelerating upward momentum.
        *   For a long (`Bottom Fishing`), price makes a new low, but the bearish MACD histogram is less negative (closer to zero) than its previous value, indicating decelerating downward momentum. This confirms the counter-trend spike is losing steam, creating the vacuum for the primary trend to reassert itself.

### 3. The Execution Engine

The final signal is a boolean `true` value generated only when all layered conditions are met simultaneously on a single bar.

#### **Top Escaping (Short Signal)**

*   **Boolean Logic:** A `true` signal is generated if and only if:
    1.  `direction < 0` **AND**
    2.  `high >= recentHigh` **AND**
    3.  `isLiquidation` is `true` **AND**
    4.  `adxHigh` is `true` **AND**
    5.  `momentumFade` is `true`

#### **Bottom Fishing (Long Signal)**

*   **Boolean Logic:** A `true` signal is generated if and only if:
    1.  `direction > 0` **AND**
    2.  `low <= recentLow` **AND**
    3.  `isLiquidation` is `true` **AND**
    4.  `adxHigh` is `true` **AND**
    5.  `momentumFade` is `true`

#### **Mathematical Constants & Their Influence**

*   **`volLimit = 1.5`:** This multiplier directly controls the sensitivity of the event detector. A lower value (e.g., 1.2) will generate more signals by classifying smaller volume increases as "liquidations." A higher value (e.g., 2.0) will be far more selective, requiring a much more significant volume anomaly, thus reducing signal frequency but potentially increasing signal quality.
*   **`adxMin = 30`:** This constant defines the minimum required trend strength. It acts as the primary whipsaw filter. Setting it higher (e.g., 35) restricts the strategy to only the most powerful trends. Setting it lower (e.g., 25) allows it to operate in less decisive trends, increasing signal frequency at the risk of engaging in weaker setups.
*   **`breakoutLen = 20`:** This lookback period defines the "short-term" range. A shorter length (e.g., 10) will make the system more sensitive to recent price action, triggering on smaller counter-trend moves. A longer length (e.g., 30) requires a more significant price extension to break the established range, resulting in fewer but potentially more meaningful signals.
*   **`1.02` (in `momentumFade`):** This hard-coded multiplier introduces a 2% tolerance into the momentum exhaustion check. Instead of requiring the MACD histogram to be strictly less than its prior value, it allows for a negligible increase. This makes the condition slightly more robust and prevents valid signals from being missed due to micro-fluctuations in the histogram's value. It marginally reduces the strictness of the divergence requirement.
    