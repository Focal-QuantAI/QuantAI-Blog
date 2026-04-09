
# Indicators Description

### 1. Component Deconstruction

The script's primary analytical component is not a standard library indicator (like RSI or MACD) but a custom-built structural pattern recognition engine. Its core building block is the `ta.pivothigh` and `ta.pivotlow` function.

*   **Pivot Points (`ta.pivothigh` / `ta.pivotlow`)**
    *   **Specific Configuration:**
        *   **Source:** `high` for `ta.pivothigh` and `low` for `ta.pivotlow`.
        *   **Left Strength (Lookback):** `INPUT_PIVOT_LEFT` (default: 5 bars). This defines the number of bars to the left of a potential pivot that must have lower highs (for a pivot high) or higher lows (for a pivot low).
        *   **Right Strength (Offset):** `INPUT_PIVOT_RIGHT` (default: 5 bars). This defines the number of bars to the right that must have lower highs or higher lows. This introduces a lag of `INPUT_PIVOT_RIGHT` bars, as a pivot can only be confirmed after this period has passed.
    *   **Functional Modification:** The script does not merely plot these pivots. It transforms them into a structured dataset for advanced analysis.
        1.  **Data Capture:** Upon detection, the price and bar index (`bar_index - INPUT_PIVOT_RIGHT`) of each pivot are captured into a custom `Coordinate` object.
        2.  **Stateful Buffering:** These `Coordinate` objects are stored in two persistent arrays, `pivot_highs` and `pivot_lows`. These arrays act as rolling buffers, maintaining a history of the last 20 significant market structure points. This transforms the stateless output of `ta.pivothigh`/`low` into a stateful dataset that forms the basis for trendline construction.

### 2. Logic Layering & Confluence

The script employs a hierarchical filtering system to identify high-probability wedge patterns and distinguish them from random market noise. The logic progresses from raw data collection to geometric validation and finally to breakout confirmation.

*   **Interaction Dynamics: Geometric Confluence**
    The engine's core function is to detect a specific geometric confluence between two trendlines derived from the pivot point buffers.
    1.  **Trendline Construction:** A potential pattern is considered only when both pivot arrays contain at least `MIN_TOUCHES_PER_LINE` (default: 3) points. The script then constructs two `Trendline` objects:
        *   **Upper Trendline:** Connects the `N`-th most recent pivot high to the most recent pivot high (where `N` is `MIN_TOUCHES_PER_LINE`).
        *   **Lower Trendline:** Connects the `N`-th most recent pivot low to the most recent pivot low.
    2.  **Slope-Based Classification:** The script calculates the slope of each trendline and looks for a specific mathematical relationship to classify the pattern:
        *   **Rising Wedge (Bearish Implication):** Identified if `tl_up.slope > 0` AND `tl_lo.slope > 0` AND `tl_lo.slope > tl_up.slope`. This mathematical condition confirms that both lines are rising, but the lower line (support) is rising faster than the upper line (resistance), forcing a convergence.
        *   **Falling Wedge (Bullish Implication):** Identified if `tl_up.slope < 0` AND `tl_lo.slope < 0` AND `tl_up.slope < tl_lo.slope`. This confirms both lines are falling, but the upper line (resistance) is falling faster than the lower line (support), again forcing a convergence.

*   **Hierarchical Filtering (Gatekeeper Logic)**
    The script applies a series of stringent checks to validate a potential pattern before it becomes "active."
    1.  **Gatekeeper 1: Data Sufficiency:** The entire detection logic is bypassed unless `array.size(pivot_highs)` and `array.size(pivot_lows)` are both greater than or equal to `MIN_TOUCHES_PER_LINE`. This ensures the pattern is based on a statistically relevant number of structural points.
    2.  **Gatekeeper 2: Geometric Convergence:** The slope conditions described above must be met. Any other slope combination (e.g., a channel, a broadening wedge) is ignored.
    3.  **Gatekeeper 3: Future Apex Projection:** The script calculates the theoretical intersection point (the "apex") of the two trendlines. A pattern is only considered valid if this apex occurs at a future bar index (`apex_idx > bar_index`). This filters out patterns that have already completed their consolidation phase.
    4.  **Gatekeeper 4: Pattern Integrity Check:** The `check_violation()` method acts as a critical noise filter. It iterates through all historical bars within the pattern's boundaries (from its starting pivot to the current bar) and invalidates the entire pattern if any bar's `close` has prematurely breached either the upper or lower trendline. This ensures the identified consolidation has remained intact.

### 3. The Execution Engine

The execution engine transitions the script from pattern observation to trade signal generation. It is governed by a precise set of boolean conditions and configurable risk parameters.

*   **Boolean Logic: The Trigger Condition**
    A trade signal is generated only when a confluence of three conditions is met on the same bar for an "active" wedge pattern:
    1.  **Directional Breakout:** The `close` of the current bar must cross the pattern boundary in the *expected* direction.
        *   **Long Trigger:** `close` > `upper_trendline_price` for a Falling Wedge.
        *   **Short Trigger:** `close` < `lower_trendline_price` for a Rising Wedge.
    2.  **Momentum Conviction:** The body of the breakout candle must represent a significant percentage of the candle's total range.
        *   **Formula:** `bodyPct = (math.abs(close - open) / (high - low)) * 100.0`
        *   **Condition:** `bodyPct >= MIN_BODY_PCT` (default: 50%). This filter is designed to reject weak breakouts or "doji-like" candles that signify indecision rather than conviction.
    3.  **Invalidation Avoidance:** The breakout must not be an "invalidation" (e.g., a close *above* a rising wedge or *below* a falling wedge).

    The final signal (`longSignal` or `shortSignal`) is `true` only when `correctBreakout AND strongBody` evaluates to `true`. A breakout in the correct direction but with a weak body (`correctBreakout AND NOT strongBody`) marks the pattern as `is_failed`, preventing future signals from it.

*   **Mathematical Constants & Risk Profile**
    Once a valid signal is triggered, the script calculates Entry, Stop Loss (SL), and Take Profit (TP) levels. The risk-to-reward profile is directly influenced by these user-defined settings.
    *   **Entry Price:** `close` of the breakout candle.
    *   **Stop Loss (`slPrice`):** The calculation is determined by the `SL_MODE` input:
        *   `"Breakout Candle"`: Uses the `low` (for longs) or `high` (for shorts) of the signal candle. This creates a tight, dynamic stop based on recent volatility.
        *   `"Previous Swing"`: Uses the price of the most recently confirmed pivot point (`recentSwingLow` or `recentSwingHigh`). This provides a wider, structure-based stop.
        *   `"Fixed Points"`: `entryPrice - (FIXED_SL_POINTS * syminfo.mintick)`. This uses a static risk distance defined by the user, where `FIXED_SL_POINTS` acts as a multiplier for the instrument's minimum price change (`syminfo.mintick`).
    *   **Take Profit (`tpPrice`):** The calculation is determined by the `TP_MODE` input:
        *   `"Risk Reward"`: The TP is a direct function of the risk taken.
            *   **Formula:** `tpPrice = entryPrice + (riskDistance * RR_RATIO)` for a long, where `riskDistance = entryPrice - slPrice`.
            *   The `RR_RATIO` (default: 1.0) is the key mathematical constant that defines the trade's reward potential relative to its risk.
        *   `"Fixed Points"`: `entryPrice + (FIXED_TP_POINTS * syminfo.mintick)`. This sets a static profit target, independent of the calculated risk.
    