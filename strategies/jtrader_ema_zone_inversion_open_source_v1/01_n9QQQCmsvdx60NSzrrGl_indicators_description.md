
# Indicators Description

### 1. Component Deconstruction

#### **Exponential Moving Averages (EMAs)**
*   **Specific Configuration:**
    *   **EMA 1:** 33-period EMA, sourced from `close`.
    *   **EMA 2:** 50-period EMA, sourced from `close`.
    *   **EMA 3:** 200-period EMA, sourced from `close`.
*   **Functional Modification:**
    *   The script does not modify the core EMA calculation. Instead, it uses their relational positioning to define market regimes.
    *   `bullStack = ema1 > ema2 and ema2 > ema3`: A boolean flag that is `true` only when the 33 EMA is above the 50 EMA, and the 50 EMA is above the 200 EMA. This defines a strong bullish trend structure.
    *   `bearStack = ema1 < ema2 and ema2 < ema3`: A boolean flag that is `true` only when the 33 EMA is below the 50 EMA, and the 50 EMA is below the 200 EMA. This defines a strong bearish trend structure.
    *   `emaHigh` / `emaLow`: These variables calculate the upper and lower bounds of the entire EMA cloud at any given bar, used for filling the visual zone on the chart.

#### **Fair Value Gap (FVG) - Custom Implementation**
The script employs a custom, state-managed FVG detection engine rather than a simple, stateless pattern search.

*   **Specific Configuration:**
    *   **Lookback Period:** `fvgLookback` (default: 30 bars). This defines the maximum age of an FVG for it to be considered valid for a trade trigger.
*   **Functional Modification (Bearish FVG for Long Setups):**
    *   **`bearFVGloose`:** `low[2] - high[0] > 0 and open[1] >= close[1]`. This identifies the fundamental 3-bar FVG structure (the low of the bar 2 periods ago is higher than the high of the current bar) but adds a condition that the middle bar must be bearish or a doji.
    *   **`bearFVGstrict`:** `low[2] - high[0] > 0 and open[2] >= close[2] and open[1] >= close[1] and open[0] >= close[0]`. This is a significantly stricter version. It requires that all three bars involved in creating the FVG pattern are bearish (or dojis). This modification aims to increase the signal-to-noise ratio by filtering for FVGs created by aggressive, one-sided selling pressure.
    *   **State Machine:** The script uses a series of `var` variables (`bL3cActive`, `bL3cTop`, `bL3cBot`, etc.) to track the formation of a bearish price leg. Detection is "activated" by a `pivotHigh` and deactivated if that pivot is broken to the upside. During this active state, it continuously looks for and updates the boundaries of the widest `bearFVG` found, only storing it if a `bearFVGstrict` pattern is part of the sequence. This ensures the FVG is part of a broader pullback structure.

*   **Functional Modification (Bullish FVG for Short Setups):**
    *   The logic is an exact mirror of the bearish FVG detection for short setups, activated by a `pivotLow` and requiring bullish bars (`open <= close`) to form the gap.

#### **Pivot Points - Custom Implementation**
*   **Specific Configuration:**
    *   The script uses a fixed 3-bar lookback for pivot identification.
*   **Functional Modification:**
    *   **`pivotHigh`:** `high[1] > high[0] and high[1] > high[2]`. This is a standard 3-bar fractal high, identifying a bar with a higher high than its immediate neighbors.
    *   **`pivotLow`:** `low[1] < low[0] and low[1] < low[2]`. This is a standard 3-bar fractal low.
    *   **Pivot Counting:** The functions `f_countPivotLows` and `f_countPivotHighs` iterate backwards from the FVG's starting bar to count the number of 3-bar pivot structures within the pullback. This data is used by the `pivotFilter` input to qualify setups based on the complexity of the preceding price action (e.g., allowing only single, clean pullbacks).

### 2. Logic Layering & Confluence

The script's engine is built on a strict, hierarchical filtering process where each condition acts as a gatekeeper for the next. A signal is only generated if the market narrative unfolds in a precise sequence.

*   **Interaction Dynamics:** The primary dynamic is **Hierarchical Filtering** combined with a final **Threshold Cross** trigger. It does not use traditional convergence or divergence.

*   **Hierarchical Filtering - Long Entry Example:**
    1.  **State Activation (Structural Context):** The process begins when a `pivotHigh` is formed. This activates the `bL3cActive` state, telling the script to start monitoring for a counter-trend (bearish) FVG. This establishes the beginning of the pullback.
    2.  **Pattern Identification (Imbalance Creation):** While `bL3cActive` is true, the script searches for a `bearFVG`. It specifically requires at least one `bearFVGstrict` to occur, ensuring the FVG was created with significant momentum. The FVG's coordinates are stored in memory (`fvgTopL`, `fvgBotL`).
    3.  **The Trigger (Invalidation Event):** The script now waits for a **Threshold Cross**. The `ifvgFireL` condition becomes true only on the bar that `close`s above the FVG's upper boundary (`fvgTopL`). This is the "inversion" signal.
    4.  **Macro Trend Filter (Regime Confirmation):** At the moment of the trigger, the script checks if `bullStack` is `true`. If the EMAs are not in a bullish alignment, the signal is immediately discarded.
    5.  **Retrospective Analysis (Liquidity Raid Confirmation):** This is the most sophisticated layer. After the trigger, the script looks *back in time* from the FVG's starting point to the trigger bar.
        *   It uses `f_pivotLowInfo` to find the absolute lowest pivot low within this period.
        *   It then checks the condition `wickTouchedZone = pvLow <= ema33AtPivot and pvClose >= ema50AtPivot`. This mathematically confirms the "liquidity grab" narrative: the wick of the pivot low must have penetrated the 33 EMA, but the body must have closed above the 50 EMA, signaling a rejection from the dynamic support zone.
        *   Simultaneously, `f_allClosesAboveEma200` confirms that at no point during the pullback did the price close below the 200 EMA, ensuring macro trend integrity was maintained.
    6.  **Final Filter (Structural Complexity):** The `pivotOK` check validates the signal against the user's preference for single or multiple pivots within the pullback.

### 3. The Execution Engine

#### **Trigger Conditions (Long Signal)**

The final `ifvgValidL` signal returns `true` if and only if the following boolean logic is satisfied:

`ifvgFireL AND showLong AND (all conditions below)`

*   **`ifvgFireL` is defined as:**
    *   `not na(fvgTopL)`: A valid, strict, bearish FVG has been previously identified and its data is stored.
    *   `close > fvgTopL AND close[1] <= fvgTopL`: The current bar's close has crossed and closed *above* the top of the stored FVG, and the previous bar was still below or inside it. This isolates the exact moment of the breakout.
    *   `bullStack`: The 33, 50, and 200 EMAs are in a bullishly stacked formation *at the time of the close*.
    *   `(bar_index - fvgBarL) <= fvgLookback`: The FVG is not older than the user-defined `fvgLookback` period (default 30 bars).

*   **Post-Trigger Validation Checks:**
    *   `allAboveEma200`: A loop confirms every `close` from the start of the FVG formation to the trigger bar remained above the 200 EMA.
    *   `bullStackPivot`: The 50 EMA was above the 200 EMA at the historical bar where the key pivot low was formed.
    *   `wickTouchedZone`: The low of the key pivot (`pvLow`) was less than or equal to the 33 EMA at that bar, AND the close of that same bar (`pvClose`) was greater than or equal to the 50 EMA.
    *   `pivotOK`: The number of pivots counted during the pullback matches the user's filter setting.

#### **Exit Conditions & Mathematical Constants**

The script employs a fixed Risk-to-Reward framework based on structural price points, not volatility measures like ATR.

*   **Stop Loss (SL):**
    *   **Logic:** `slL := pivLowPx`
    *   **Significance:** The Stop Loss is placed at the absolute low of the pivot point that was identified as the "liquidity grab." This is a structurally defined stop, assuming that a break below this level invalidates the entire setup.

*   **Take Profit (TP):**
    *   **Logic:** `tpL := close + (close - pivLowPx)`
    *   **Significance:** This defines a fixed **1:1 Risk-to-Reward ratio**. The mathematical constant is the implicit `1` multiplier. The distance from the entry price (`close`) to the Stop Loss (`pivLowPx`) is calculated, and that same distance is projected upwards from the entry price to determine the Take Profit level. This makes the strategy's risk profile predictable and consistent for every trade.

*   **Trade State Management:**
    *   The `inTradeL` and `inTradeS` boolean variables act as simple state locks. Once a valid signal fires and a trade is considered "active," no new signals in the same direction can be generated until the TP or SL is hit (`high >= tpL or low <= slL`). This prevents signal spamming during a trending move.
    