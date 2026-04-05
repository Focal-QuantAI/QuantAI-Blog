
# Indicators Description

### 1. Component Deconstruction

This section deconstructs each technical component of the `Price Action Toolkit Lite [UAlgo]` script, detailing its configuration and any functional modifications.

#### **A. Market Structure Engine (Custom ZigZag)**

*   **Core Function:** This module identifies significant swing highs and lows to define market structure. It does not use a built-in ZigZag function but constructs one from basic price action logic.
*   **Specific Configuration:**
    *   **Lookback Period (`zigzagLen`):** `9`
    *   **Price Source:** `high` and `low`.
    *   **Detection Logic:**
        *   A new swing high is detected if the high of the bar `zigzagLen` periods ago (`high[zigzagLen]`) is the highest high over the entire `zigzagLen` lookback period (`ta.highest(high, zigzagLen)`).
        *   A new swing low is detected if the low of the bar `zigzagLen` periods ago (`low[zigzagLen]`) is the lowest low over the entire `zigzagLen` lookback period (`ta.lowest(low, zigzagLen)`).
*   **Functional Modification:**
    *   The script maintains a state variable `trend` (`1` for up, `-1` for down). The trend flips only when a new opposing swing point is confirmed. For example, if `trend` is `1` (uptrend), it will only flip to `-1` upon the confirmation of a new swing low (`to_down`).
    *   Confirmed swing points (price and time) are stored in four arrays: `highValIndex`, `highVal`, `lowValIndex`, and `lowVal`. These arrays form the foundational database for all subsequent structural analysis (BoS/CHoCH).

#### **B. Liquidity Levels (Pivot-Based)**

*   **Core Function:** Identifies potential liquidity pools resting above swing highs and below swing lows.
*   **Specific Configuration:**
    *   **Indicator:** `ta.pivothigh` and `ta.pivotlow`.
    *   **Lookback/Lookforward Period (`liquidity_len`):** `30`. This means a pivot high is confirmed only if the high of a candle is higher than the highs of the `30` preceding and `30` succeeding candles.
    *   **Price Source:** `high` for pivot highs, `low` for pivot lows.
*   **Functional Modification:**
    *   This is a standard implementation of pivots. The script draws a horizontal line from the confirmed pivot point forward in time. The "modification" lies in its lifecycle management: the line is extended until price sweeps it, at which point its style is changed to `dashed`, an 'x' label is plotted, and it is removed from the active tracking array (`bullishLiquidity` or `bearishLiquidity`).

#### **C. Order Blocks (OB) - Custom Implementation**

*   **Core Function:** Pinpoints the last opposing candle before a structural break, representing a potential zone of institutional interest.
*   **Specific Configuration:**
    *   **Trigger:** An Order Block is only identified *after* a Break of Structure (BoS) or Change of Character (CHoCH) event.
    *   **Price Source:** `high` and `low`.
    *   **Volatility Component:** `ta.atr(14)`.
*   **Functional Modification:**
    *   This is a "hacked" or custom-built indicator. Its mathematical logic is as follows:
        1.  **Identify Impulse Leg:** When a BoS/CHoCH occurs, the script defines the "impulse leg" as the series of bars between the prior swing point and the current bar that broke the structure.
        2.  **Locate Extreme Candle:** It iterates backwards through this impulse leg.
            *   For a **bullish OB** (following an upward break), it finds the lowest `low` within the leg.
            *   For a **bearish OB** (following a downward break), it finds the highest `high` within the leg.
        3.  **Define Zone Boundaries:** The OB is drawn as a box.
            *   **Bullish OB:** The bottom is the identified lowest `low` (`min`). The top is `min + ta.atr(14)`.
            *   **Bearish OB:** The top is the identified highest `high` (`max`). The bottom is `max - ta.atr(14)`.
    *   **Intended Effect:** The use of ATR makes the Order Block's size dynamic and volatility-adjusted. In high-volatility environments, the zone is thicker, accounting for wider price swings. This differs from standard OB definitions that use the fixed high/low of a single candle.

#### **D. Fair Value Gaps (FVG) / Imbalances**

*   **Core Function:** Detects three-bar price inefficiencies.
*   **Specific Configuration:**
    *   **Price Source:** `high` and `low`.
    *   **Detection Logic (Standard 3-Bar Pattern):**
        *   **Bullish FVG:** The current bar's `low` is greater than the `high` of two bars prior (`low > high[2]`). The gap exists between `high[2]` (bottom) and `low` (top).
        *   **Bearish FVG:** The current bar's `high` is less than the `low` of two bars prior (`high < low[2]`). The gap exists between `high` (bottom) and `low[2]` (top).
*   **Functional Modification:** No mathematical modification is applied to the core detection logic. The script's contribution is in the lifecycle management: an FVG box is drawn and extended forward in time until price mitigates it (a `low` enters a bullish FVG or a `high` enters a bearish FVG), at which point the box is deleted.

#### **E. Trend Lines (Pivot-Based)**

*   **Core Function:** Draws and extends trend lines based on recent swing points.
*   **Specific Configuration:**
    *   **Indicator:** `ta.pivothigh` and `ta.pivotlow`.
    *   **Lookback/Lookforward Period (`trendLineLength`):** `20`.
    *   **Anchor Points:** The script uses `ta.valuewhen` to get the bar index and price of the last two confirmed pivot lows (for an uptrend line) and the last two confirmed pivot highs (for a downtrend line).
*   **Functional Modification:**
    *   The script includes a custom function, `extendTrendline`, which calculates the slope between the two anchor pivots (`(y2 - y1) / (x2 - x1)`). It then uses this slope to project the line's end point to the current bar, ensuring the trend line is always extended to the right edge of the chart.
    *   A filter is applied: an uptrend line is only drawn if its calculated slope is positive (`slopeBullish > 0`), and a downtrend line is only drawn if its slope is negative (`slopeBearish < 0`).

### 2. Logic Layering & Confluence

The script's engine filters market noise through a distinct hierarchical process.

*   **Layer 1: Foundational Structure (ZigZag)**
    *   The custom ZigZag logic is the absolute foundation. It is the first filter, converting raw price data into a structured map of swing highs and lows. Nothing else related to market structure (BoS, CHoCH, OBs) can occur until this layer confirms a swing point.

*   **Layer 2: The "Gatekeeper" Event (BoS / CHoCH)**
    *   This layer acts as the primary gatekeeper for trade-relevant signals. It constantly monitors the current `close` against the most recent swing high/low stored in the arrays from Layer 1.
    *   **Interaction Dynamic:** This is a **Threshold Cross**. A signal is generated only when `close` crosses above the last `highVal` or below the last `lowVal`.
    *   **Hierarchical Filtering:**
        *   The script maintains a `lastState` variable ('up' or 'down') to track the direction of the most recent break.
        *   If a new break occurs in the *same direction* as the `lastState`, it is labeled **"BoS" (Break of Structure)**, confirming trend continuation.
        *   If a new break occurs in the *opposite direction*, it is labeled **"CHoCH" (Change of Character)**, signaling a potential trend reversal. This contextual labeling is a critical secondary filter.

*   **Layer 3: Consequential Signals (Order Blocks)**
    *   The identification of an Order Block is entirely dependent on Layer 2. An OB is a *consequence* of a BoS/CHoCH event.
    *   **Hierarchical Filtering:** The OB detection logic is only activated *after* a BoS/CHoCH is confirmed. It then analyzes the price action that *led* to the break to find its origin point. This ensures OBs are only marked in zones that have proven their ability to move the market.

*   **Layer 4: Independent Confluence Layers (FVG & Liquidity)**
    *   FVGs and Liquidity levels operate independently of the structural hierarchy. They are detected based on their own unique patterns (3-bar pattern for FVG, pivot confirmation for Liquidity).
    *   **Interaction Dynamic:** The script does not programmatically check for confluence between these layers. Instead, it presents all layers on the chart simultaneously. The confluence is intended for discretionary analysis by the user. For example, a high-probability setup would be a Bullish Order Block (Layer 3) that forms inside or adjacent to a Bullish Fair Value Gap (Layer 4), after price has swept a Bearish Liquidity level (Layer 4).

### 3. The Execution Engine

This script is a visualization tool; its "Execution Engine" pertains to the logic that triggers the creation, modification, and deletion of visual elements.

#### **A. Trigger Conditions (Boolean Logic)**

*   **BoS/CHoCH Label & Line:**
    *   `close > array.get(highVal, array.size(highVal) - 1)` OR `close < array.get(lowVal, array.size(lowVal) - 1)`
    *   This must be the first time this break has occurred, controlled by the `drawUp` and `drawDown` boolean flags.

*   **Order Block Box:**
    *   The BoS/CHoCH condition must be `true`.
    *   The `orderblockBool` input must be `true`.

*   **Fair Value Gap Box:**
    *   `isBullishFvg = low > high[2]` OR `isBearishFvg = high < low[2]`
    *   The `fvgBool` input must be `true`.

*   **Liquidity Line:**
    *   `not na(ta.pivothigh(high, liquidity_len, liquidity_len))` OR `not na(ta.pivotlow(low, liquidity_len, liquidity_len))`
    *   The `liquidityBool` input must be `true`.

#### **B. "Exit" / Invalidation Conditions**

*   **Order Block Mitigation:**
    *   **Bullish OB:** `close < testOrderblock.value` (where `value` is the bottom of the OB). The block is considered invalidated and is deleted.
    *   **Bearish OB:** `close > testOrderblock.value` (where `value` is the top of the OB). The block is invalidated and deleted.

*   **Fair Value Gap Mitigation:**
    *   **Bullish FVG:** `low < f.top` (price wicks into the gap). The FVG is considered filled/mitigated and is deleted.
    *   **Bearish FVG:** `high > f.bottom` (price wicks into the gap). The FVG is considered filled/mitigated and is deleted.

*   **Liquidity Sweep:**
    *   **Bearish Liquidity (Highs):** `high > testLiquidity.value`. The line style is changed, and the object is removed from the active array.
    *   **Bullish Liquidity (Lows):** `low < testLiquidity.value`. The line style is changed, and the object is removed from the active array.

#### **C. Mathematical Constants & Their Influence**

*   **`zigzagLen = 9`:** A relatively small value, making the structure detection sensitive to shorter-term swings. This increases the frequency of BoS/CHoCH signals.
*   **`liquidity_len = 30`:** A large value, ensuring that only structurally significant and well-established swing points are marked as liquidity. This improves the signal-to-noise ratio for these specific levels.
*   **`1.0 * ta.atr(14)`:** This is the implicit multiplier used for Order Block thickness. The ATR value is added (bullish) or subtracted (bearish) from the extreme price point of the impulse leg. This directly impacts the size of the zone of interest and, by extension, the potential entry area and stop-loss placement for a discretionary trader. A larger ATR widens the zone, offering a larger area for entry but requiring a wider stop.
    