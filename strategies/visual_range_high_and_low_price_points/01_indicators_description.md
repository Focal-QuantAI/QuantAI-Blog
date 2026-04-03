
# Indicators Description

### 1. Component Deconstruction

This script's engine is built upon a minimalist set of components, eschewing traditional oscillating or moving average studies in favor of direct price and time analysis.

*   **Component 1: Price Action Extrema**
    *   **Indicators Used:** `high` and `low` (built-in series variables).
    *   **Specific Configuration:**
        *   **Price Source:** The raw, unmodified `high` and `low` price of each historical bar.
        *   **Length/Period:** Not applicable in the traditional sense. The lookback period is not a fixed integer but is dynamically determined by the visible chart area.
        *   **Smoothing:** None. The analysis is performed on unfiltered price data to capture the absolute price extremes.
    *   **Functional Modification:** There is no mathematical modification to the `high` or `low` values themselves. The innovation lies entirely in the *selection criteria* for which bars' `high` and `low` values are considered for analysis.

*   **Component 2: Dynamic Lookback Filter**
    *   **Indicators Used:** `time`, `chart.left_visible_bar_time`, `chart.right_visible_bar_time` (built-in time and chart-state variables).
    *   **Specific Configuration:**
        *   **Left Boundary:** `chart.left_visible_bar_time` - The UNIX timestamp of the leftmost bar currently visible on the user's screen.
        *   **Right Boundary:** `chart.right_visible_bar_time` - The UNIX timestamp of the rightmost bar currently visible on the user's screen.
    *   **Functional Modification:** This is the core "hack" of the script. It replaces a static, integer-based lookback period (e.g., `200` bars) with a dynamic, time-based window. The script iterates backwards from the current bar and uses the timestamp of each bar (`time[i]`) as a filter. If a bar's timestamp falls outside the visible window, it is excluded from the extrema calculation. This has a profound effect on the signal-to-noise ratio, as it ensures the identified high and low are always relevant to the trader's immediate field of view, automatically discarding stale, irrelevant price history.

### 2. Logic Layering & Confluence

The script's logic is not layered in the traditional sense of combining multiple indicators (e.g., RSI > 50 AND Price > EMA). Instead, it employs a hierarchical filtering process to manage computational load and define the analytical dataset.

*   **Hierarchical Filtering:**
    *   **Level 1 Gatekeeper: The Execution Trigger (`barstate.islast`)**
        *   The entire analytical engine is encapsulated within an `if barstate.islast` block. This is the highest-level filter. It dictates that the script performs **zero calculations** on any historical bar. The entire process of looping, comparing, and drawing only executes once, on the single most recent bar of the chart. This is a critical optimization that ensures maximum performance and prevents the script from slowing down the chart during historical data loading.

    *   **Level 2 Gatekeeper: The Visibility Window (`for` loop with time check)**
        *   Once the `barstate.islast` gate is passed, the script initiates a `for` loop to iterate backwards through historical bars. This loop contains the second filter: `if time[i] < chart.left_visible_bar_time`.
        *   This condition acts as a "break" trigger for the loop. It defines the dataset for the analysis. The loop effectively asks, "Is this historical bar visible on the screen?" If the answer is no, the loop terminates, and no further historical bars are analyzed. This dynamically constrains the search for the high and low to only the bars the trader can currently see.

*   **Interaction Dynamics:**
    *   The script's primary dynamic is **Threshold Identification**. It does not look for convergences or divergences between different mathematical studies. Its sole purpose is to process the price data within the filtered visibility window to find two values:
        1.  `visMax`: The absolute highest `high` within the visible bars.
        2.  `visMin`: The absolute lowest `low` within the visible bars.
    *   These two values are then projected horizontally across the visible chart area. The "confluence" is intended to occur between the live market price and these objectively identified threshold levels, providing a basis for discretionary breakout or mean-reversion trade decisions.

### 3. The Execution Engine

The execution engine is responsible for the calculation of the extrema and the rendering of the visual elements. Its logic is deterministic and event-driven.

*   **Trigger & Calculation Logic:**
    *   **Primary Trigger:** The engine's one and only trigger is the Boolean condition `barstate.islast` returning `true`.
    *   **Internal Boolean Logic:** Inside the `for` loop, the following conditions are evaluated for each bar `i` within the visible range:
        1.  `na(visMax) or high[i] > visMax`: This evaluates to `true` if either a) the maximum value has not yet been initialized, or b) the `high` of the current bar being inspected is greater than the highest value found so far. If `true`, `visMax` is updated.
        2.  `na(visMin) or low[i] < visMin`: This evaluates to `true` if either a) the minimum value has not yet been initialized, or b) the `low` of the current bar being inspected is less than the lowest value found so far. If `true`, `visMin` is updated.
    *   **Drawing Logic:** After the loop completes, two final checks occur:
        1.  `if not na(visMax)`: If a valid maximum was found, the script proceeds to delete any previously drawn maximum line/label and draws new ones at the updated price level (`visMax`).
        2.  `if not na(visMin)`: If a valid minimum was found, the script deletes the old minimum line/label and draws new ones at the updated price level (`visMin`).

*   **Mathematical Constants & Constraints:**
    *   **`4999`:** This hard-coded integer in the `for i = 0 to 4999` loop represents the **absolute maximum lookback period**. It acts as a performance and safety constraint. If a user zooms out so far that more than 5,000 bars are visible on the screen, this script will only analyze the most recent 5,000. For the vast majority of use cases, the `chart.left_visible_bar_time` filter will terminate the loop long before this limit is reached. It primarily serves to prevent script timeouts on charts with extremely long histories.
    *   **Cosmetic Constants:** Numbers such as `width=1` and the `30` in `color.new(color.red, 30)` are purely for visual styling (line width and 30% opacity, respectively). They have no impact on the mathematical calculation or the price levels identified.
