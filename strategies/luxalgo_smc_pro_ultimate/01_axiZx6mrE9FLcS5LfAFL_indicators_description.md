
# Indicators Description

### 1. Component Deconstruction

This analysis deconstructs the script's components based on their default input configurations.

#### **A. Core Indicators & Oscillators**

*   **Average True Range (ATR)**
    *   **Specific Configuration:** `ta.atr(atrLen)` with `atrLen` = 9. The price source is the default for `atr`, which is a smoothed average of True Range (`math.max(high - low, math.abs(high - close[1]), math.abs(low - close[1]))`).
    *   **Functional Modification:** Standard implementation. Used exclusively for risk management to calculate stop-loss placement.

*   **Simple Moving Average (SMA)**
    *   **Specific Configuration:** `ta.sma(volume, 20)`. A 20-period SMA is calculated on `volume`.
    *   **Functional Modification:** Standard implementation. It serves as a baseline or "average volume" threshold for the Signal Ranking engine.

*   **Relative Strength Index (RSI)**
    *   **Specific Configuration:** `ta.rsi(close, 14)`. A standard 14-period RSI calculated on the `close` price.
    *   **Functional Modification:** Standard implementation. It is not used for overbought/oversold conditions but exclusively for detecting price-momentum divergence as an optional confirmation filter.

*   **Exponential Moving Average (EMA)**
    *   **Specific Configuration:** `ta.ema(close, 200)`. A standard 200-period EMA calculated on the `close` price.
    *   **Functional Modification:** Standard implementation. It is not part of the primary entry logic. It is used post-trade within the "Confluence Meter" to score the trend alignment of an open position.

*   **Bollinger Bands (BB)**
    *   **Specific Configuration:** `ta.bb(close, 20, 2.0)`. A standard 20-period Bollinger Band with a 2.0 standard deviation multiplier, calculated on the `close` price.
    *   **Functional Modification:** The script only extracts the middle band (`bbMid`), which is a 20-period SMA. The upper and lower bands are discarded. This component is effectively used as a 20-period SMA to act as a short-term momentum filter.

#### **B. Custom Mathematical & Logical Constructs**

*   **Premium & Discount Zones**
    *   **Specific Configuration:** `pdLookback` = 100.
    *   **Mathematical Logic:**
        1.  `rangeHigh = ta.highest(high, 100)`
        2.  `rangeLow = ta.lowest(low, 100)`
        3.  `equilibrium = (rangeHigh + rangeLow) / 2`
        The engine defines a trading range based on the highest high and lowest low over the last 100 bars. The midpoint of this range is the "Equilibrium". Any price below this midpoint is in a "Discount" zone, and any price above is in a "Premium" zone. This is a direct implementation of a core SMC concept.

*   **Fair Value Gaps (FVG)**
    *   **Specific Configuration:** This is a hard-coded 3-bar pattern recognition algorithm.
    *   **Mathematical Logic:**
        *   **Bullish FVG (`bFVG`):** `low > high[2] and close[1] > high[2]`. This identifies a gap where the current bar's low is higher than the high of two bars prior, and the previous bar's close confirms this separation.
        *   **Bearish FVG (`sFVG`):** `high < low[2] and close[1] < low[2]`. This identifies a gap where the current bar's high is lower than the low of two bars prior, with the previous close confirming the gap.
        This is a simplified but effective FVG detection method.

*   **Market Structure (MSS/BOS/CHoCH)**
    *   **Specific Configuration:** `internalLookback` = 9, `swingLookback` = 50.
    *   **Mathematical Logic:**
        1.  **Pivot Detection:** The script identifies pivot points (swing highs/lows) using a standard method: `high[lookback] == ta.highest(high, lookback * 2 + 1)`. A bar is a pivot if its high/low is the highest/lowest within a window of `lookback * 2 + 1` bars centered on it.
        2.  **State Persistence:** The price levels of the most recent "Internal" and "Swing" pivots (`lastISH`, `lastISL`, `lastSSH`, `lastSSL`) are stored using `var` variables, persisting their values across bars until a new pivot is formed.
        3.  **Structure Break:** A Market Structure Shift (`mssL` or `mssS`) is defined not by the formation of a new pivot, but by the `close` price crossing over a previous swing high (`lastISH`/`lastSSH`) or crossing under a previous swing low (`lastISL`/`lastSSL`). This transforms the pivot levels into dynamic support/resistance lines that trigger the primary signal upon being broken.

*   **Volume Momentum & Signal Ranking**
    *   **Specific Configuration:** `volMult` = 1.2, `volCandles` = 3.
    *   **Mathematical Logic:**
        1.  **Strength Check:** `isStrong` is true if `volume > volAvg * 1.2`. The current bar's volume must be at least 20% greater than the 20-period average volume.
        2.  **Momentum Check:** The `volIncreasing` flag is derived from a `for` loop. For this flag to remain `true`, the condition `volume[i] < volume[i+1]` must be `false` for `i` from 0 to 2. This means `volume[0] >= volume[1]` AND `volume[1] >= volume[2]`. The logic checks for a **monotonically non-increasing** volume profile over the last 3 bars. This is counter-intuitive to the variable name and likely a logical error; the intent was probably to check for `volume[i] > volume[i+1]`. As written, it rewards decreasing volume.

### 2. Logic Layering & Confluence

The script's engine is built on a hierarchical filtering system where a primary event must occur before subsequent confirmation filters are checked.

*   **Hierarchical Filtering:**
    1.  **Gatekeeper:** The **Market Structure Shift (MSS)** is the non-negotiable, primary catalyst. The boolean variables `mssL` (for long) and `mssS` (for short) must be `true` for the logic chain to proceed. Without a structural break of a prior pivot, no signal can be generated.
    2.  **Value Filter:** The **Premium & Discount Zone** acts as the second layer. If enabled (`requirePDZone = true`), a bullish MSS is only considered valid if the `close` is in a `Discount` zone. A bearish MSS is only valid if the `close` is in a `Premium` zone. This enforces the discipline of "buy low, sell high" relative to the 100-bar range.
    3.  **Precision & Confirmation Filters:** The remaining conditions are layered as optional "AND" clauses. They serve to increase the signal's specificity:
        *   **FVG Presence:** Requires the MSS and pullback to occur in the context of a price inefficiency.
        *   **Divergence:** Adds a classic momentum-based confirmation.
        *   **BB Filter:** Ensures the entry aligns with short-term momentum (price above the 20 SMA for longs).

*   **Interaction Dynamics:**
    *   **Threshold Cross:** The core of the system is the `ta.crossover` / `ta.crossunder` of the `close` price with a stored pivot level. This is a classic threshold-crossing trigger.
    *   **State-Based Filtering:** Following the trigger, the system checks the *state* of the market (e.g., `inDiscount`, `bFVG`). It is not looking for multiple indicators to move in unison (Convergence), but for a specific sequence of events: **Break -> Pullback to Value Zone -> Confirmation**.

### 3. The Execution Engine

#### **A. Trigger Conditions**

The final trade signal is a boolean composite of the layered logic.

*   **Boolean Logic for a Long Entry (`bTrigger`):**
    A long signal is generated (`bTrigger = true`) on a bar IF ALL of the following are met:
    1.  A bullish Market Structure Shift has occurred (`mssL`).
    2.  **AND** the Premium/Discount filter is disabled **OR** the `close` price is in the Discount zone.
    3.  **AND** the FVG filter is disabled **OR** a bullish FVG was present on the current or preceding bar.
    4.  **AND** the Divergence filter is disabled **OR** a bullish RSI divergence is present.
    5.  **AND** the Bollinger Band filter is disabled **OR** the `close` is above the 20-period middle band.

*   **Boolean Logic for a Short Entry (`sTrigger`):**
    A short signal is generated (`sTrigger = true`) on a bar IF ALL of the following are met:
    1.  A bearish Market Structure Shift has occurred (`mssS`).
    2.  **AND** the Premium/Discount filter is disabled **OR** the `close` price is in the Premium zone.
    3.  **AND** the FVG filter is disabled **OR** a bearish FVG was present on the current or preceding bar.
    4.  **AND** the Divergence filter is disabled **OR** a bearish RSI divergence is present.
    5.  **AND** the Bollinger Band filter is disabled **OR** the `close` is below the 20-period middle band.

#### **B. Exit Conditions & Mathematical Constants**

*   **Initial Stop Loss:**
    *   **Mathematical Constant:** `atrMult = 3.0`.
    *   **Logic:** The initial stop loss is placed a distance of `3.0 * ta.atr(9)` away from the entry bar's price (below the low for longs, above the high for shorts). This wide multiplier indicates a strategy designed to withstand significant volatility and aims for larger price swings, prioritizing a lower frequency of being stopped out over a tight risk definition.

*   **Take Profit 1 (TP1):**
    *   **Mathematical Constant:** `tp1RR = 1.5`.
    *   **Logic:** The first take profit target is calculated to be `1.5` times the initial risk. `TP1_Distance = Initial_Risk_Distance * 1.5`. Upon hitting TP1, 50% of the position is closed, and the stop loss is moved to the entry price (breakeven). This systematically de-risks the trade after a favorable move.

*   **Trailing Stop Loss:**
    *   **Logic:** After TP1 is hit, the stop loss begins to trail the price. For a long position, the stop is continuously updated to `low - (atr * atrMult)` if this new value is higher than the current stop loss (which is at breakeven). This creates a one-way trailing stop that only moves in the direction of the trade to lock in profits.

*   **Signal Decay:**
    *   **Mathematical Constant:** `decayRate = 2`.
    *   **Logic:** Used in the visual "Confluence Meter", this constant reduces the trade's score by 2 points for every bar it remains open. This mathematically quantifies the concept of "time risk," penalizing trades that fail to achieve their objective promptly.
    