
# Indicators Description

### 1. Component Deconstruction

#### **Moving Average (MA) Weave**
The core of the script is a ribbon composed of up to 12 moving averages ("filaments").

*   **Specific Configuration:**
    *   **MA Type:** User-selectable between Exponential (EMA), Simple (SMA), and Smoothed (SMMA/RMA). Default is `EMA`.
    *   **Price Source:** `close` for all MAs.
    *   **Base Period (`basePer`):** Lookback period of the fastest MA. Default: `5`.
    *   **Weave Depth (`ribbonN`):** Total number of MAs in the ribbon. Default: `8`.
    *   **Filament Spacing (`spacing`):** The arithmetic step added to the period of each subsequent MA. Default: `5`.
    *   **Static Periods (Example):** With default settings and Volatility Morphing disabled, the MA periods would be: 5, 10, 15, 20, 25, 30, 35, 40.

*   **Functional Modification (Volatility Morphing):**
    The script employs a non-standard, dynamic period calculation for all MAs when `adaptive` is enabled. This is the "Volatility Morphing" engine.
    *   **Mathematical Logic:**
        1.  A **Volatility Regime Ratio (`volReg`)** is calculated: `volReg = ta.atr(atrFast) / ta.atr(atrSlowP)`. By default, this is `ATR(10) / ATR(50)`. This ratio is > 1.0 in high-volatility environments and < 1.0 in low-volatility environments.
        2.  An **Adaptive Multiplier (`adaptMult`)** is derived from the inverse of this ratio: `adaptMult = 1.5 / volReg`.
        3.  This multiplier is then clamped between a floor (`adaptMin`, default 0.5) and a ceiling (`adaptMax`, default 2.0).
        4.  The final period for each MA is calculated as `math.round((basePeriod + n * spacing) * adaptMult)`.
    *   **Intended Effect:**
        *   **During Low Volatility:** `volReg` is low (< 1.0), causing `adaptMult` to be high (> 1.5). This *lengthens* the MA lookback periods, increasing their smoothing effect. The goal is to reduce sensitivity and filter out whipsaws during market consolidation.
        *   **During High Volatility:** `volReg` is high (> 1.0), causing `adaptMult` to be low (< 1.5). This *shortens* the MA lookback periods, reducing their lag. The goal is to make the weave more responsive to capture shifts in fast-moving, trending markets.

#### **Average True Range (ATR)**

*   **Impulse ATR:**
    *   **Specific Configuration:** `ta.atr(10)`.
    *   **Purpose:** Measures short-term volatility to feed the `volReg` ratio.
*   **Anchor ATR:**
    *   **Specific Configuration:** `ta.atr(50)`.
    *   **Purpose:** Measures long-term, baseline volatility to serve as the denominator in the `volReg` ratio.
*   **Reference ATR (`atrRef`):**
    *   **Specific Configuration:** `ta.atr(14)`.
    *   **Purpose:** Used as a standard normalization factor for the ribbon's spread (`spreadN`) and to dynamically position signal labels on the chart, ensuring they remain visible relative to price action.

#### **Percent Rank (`ta.percentrank`)**

*   **Compression Chamber (`spreadRank`):**
    *   **Specific Configuration:** `ta.percentrank(spread, sqLen)`, with `spread` being the raw price difference between the fastest and slowest MAs, and `sqLen` defaulting to `100`.
    *   **Purpose:** Quantifies the current ribbon width relative to its history over the last 100 bars. A value below the `sqPctile` threshold (default: 10) signifies an unusually tight compression, activating the `isSqueeze` state.
*   **Trend Strength (`spreadPct`):**
    *   **Specific Configuration:** `ta.percentrank(spreadN, 100)`, where `spreadN` is the `atrRef`-normalized spread.
    *   **Purpose:** Provides a 0-100 score of the trend's intensity relative to volatility. This is primarily used for the dashboard display and the "Drift Fade" signal.

### 2. Logic Layering & Confluence

The script's engine filters noise by layering its components hierarchically, demanding confluence for high-conviction signals.

*   **Interaction Dynamics:**
    *   **Threshold Crosses:** The system's primary state changes are governed by threshold crosses.
        *   `isSqueeze = spreadRank <= sqPctile`: This is the gatekeeper for the "Momentum Surge" signal. The engine is only looking for breakouts when this condition is met.
        *   `driftWarn = spreadPct <= 15`: This triggers a warning when trend strength decays below a critical percentile.
    *   **State Change Detection (Edge Detection):** The script focuses on the *transition* between states, not the states themselves. This improves signal precision by firing only on the bar where a condition is first met.
        *   **Squeeze Release:** `not isSqueeze and isSqueeze[1]` identifies the exact bar that a compression period ends.
        *   **Twist Confirmation:** `confirmedTwistBull and not confirmedTwistBull[1]` identifies the exact bar that a trend reversal has been sustained for the required number of periods.
        *   **Fan Activation:** `alignPct >= 100 and nz(alignPct[1]) < 100` identifies the exact bar that the MAs achieve perfect alignment.

*   **Hierarchical Filtering:**
    1.  **Level 1 (Trend Gatekeeper):** The primary filter is the weave's orientation: `bullTrend = fastest > slowest`. Most directional signals (e.g., `surgeUp`, `fanBull`) are fundamentally gated by this boolean. A bullish signal cannot occur in a bearish weave structure.
    2.  **Level 2 (Structural Condition):** The "Compression Chamber" (`isSqueeze`) acts as a powerful structural filter. The highest-priority breakout signal, "Momentum Surge," can *only* be considered if the market is exiting this compressed state.
    3.  **Level 3 (Momentum Confirmation):** The `rising`/`falling` status of the fastest MA and the `spreadRoc` (rate of change of the spread) act as secondary momentum confirmations. For instance, a "Momentum Surge" requires not only a squeeze release but also an expanding spread (`spreadRoc > 0`).
    4.  **Level 4 (Conviction Filter):** The `alignPct` (Alignment Score) serves as the final conviction filter. A score of 100% indicates maximum trend coherence and triggers the "Filament Fan" signal, confirming a well-established, orderly trend.

### 3. The Execution Engine

The script generates six distinct types of signals, each with a unique boolean logic trigger and a cooldown mechanism to prevent redundancy. All signals require `barstate.isconfirmed` to fire only on bar close.

*   **1. Weave Cross (`IGNITE`/`QUENCH`)**
    *   **Boolean Logic:** `ta.crossover(fastest, slowest)` or `ta.crossunder(fastest, slowest)`.
    *   **Analysis:** The most basic trend reversal signal, triggered by the intersection of the ribbon's fastest and slowest MAs.
    *   **Mathematical Constants:** `CD_CROSS = 8`. A cooldown of 8 bars is enforced.

*   **2. Momentum Surge (`SURGE`)**
    *   **Boolean Logic:** `(not isSqueeze and isSqueeze[1]) and spreadRoc > 0 and bullTrend`.
    *   **Analysis:** The script's primary breakout signal. It requires the confluence of three events: (1) the end of a compression period, (2) immediate expansion of the ribbon's width, and (3) alignment with the overall trend direction.
    *   **Mathematical Constants:** `CD_SURGE = 12`. A 12-bar cooldown. The trigger is defined by `sqPctile` (default 10) and `sqLen` (default 100).

*   **3. Twist Lock (`TWIST LOCK`)**
    *   **Boolean Logic:** `(twistBullCnt == twistConf) and not (twistBullCnt[1] == twistConf)`.
    *   **Analysis:** A more robust reversal signal than the simple cross. It waits for the new trend direction (post-cross) to be sustained for `twistConf` bars before firing. It is edge-detected to fire only once upon confirmation.
    *   **Mathematical Constants:** `twistConf = 2`. The new trend must hold for 2 bars. `CD_TWIST = 10`.

*   **4. Filament Fan (`FAN`)**
    *   **Boolean Logic:** `alignPct >= 100 and nz(alignPct[1]) < 100`.
    *   **Analysis:** A trend *confirmation* signal. It triggers on the first bar that all MAs achieve perfect sequential order, indicating maximum trend conviction.
    *   **Mathematical Constants:** `CD_FAN = 10`.

*   **5. Snap Recoil (`SNAP`)**
    *   **Boolean Logic:** `ta.crossover(close, ribbonMid) and not bullTrend and spreadN > 0.3`.
    *   **Analysis:** A mean-reversion or counter-trend signal. It detects when price crosses back over the ribbon's midpoint *against* the prevailing trend direction, but only when the trend has sufficient width (`spreadN > 0.3`) to avoid noise in ranges.
    *   **Mathematical Constants:** `0.3` is the hard-coded minimum normalized spread required for the signal to be considered valid. `CD_SNAP = 8`.

*   **6. Drift Fade (`DRIFT`)**
    *   **Boolean Logic:** `spreadPct <= 15 and nz(spreadPct[1]) > 15`.
    *   **Analysis:** A trend exhaustion warning. It triggers when the normalized trend strength falls below the 15th percentile for the first time, signaling that momentum is decaying into a "dormant" state.
    *   **Mathematical Constants:** `15` is the percentile threshold. `CD_DRIFT = 15`, the longest cooldown, reflecting a potential shift in market regime.
    