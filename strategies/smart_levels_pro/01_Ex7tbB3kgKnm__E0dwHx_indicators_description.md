
# Indicators Description

### 1. Component Deconstruction

This script is a pure price-action visualization tool. It does not utilize standard oscillating or momentum indicators (like RSI, MACD, etc.). Instead, its core components are custom-built state machines for tracking and plotting structural price levels.

#### **Component 1: Higher-Timeframe (HTF) Level Tracker**
This is the primary engine for identifying significant historical price points. It operates identically for Daily, Weekly, and Monthly periods.

*   **Specific Configuration:**
    *   **Price Source:** `high` and `low` of each bar.
    *   **Lookback Period:** The lookback is not a fixed number of bars but is dynamically defined by calendar periods:
        *   **Daily:** A full trading day, defined as the period between 22:00 GMT on one day and 21:59 GMT on the next. The changeover is triggered by the boolean `is_new_day`.
        *   **Weekly:** A full trading week, from market open to market close. The changeover is triggered by `is_new_week`.
        *   **Monthly:** A full calendar month. The changeover is triggered by `is_new_month`.
    *   **Parameters:** User-configurable aesthetics (color, width, style) and a line extension parameter (`extend_bars`) for visualization.

*   **Functional Modification (Custom State Machine Logic):**
    The script employs a persistent state-tracking mechanism using `var` declared variables (e.g., `pd_high`, `d_high`). This is a non-standard, custom implementation to achieve its goal.
    1.  **Initialization:** Variables like `d_high` and `d_low` are initialized to `na`.
    2.  **Intra-Period Tracking:** For every bar *within* a given period (e.g., inside the current day), the script compares the bar's `high` and `low` to the period's stored `d_high` and `d_low`. If a new extreme is made, the corresponding `var` variable is updated.
        *   `if high >= nz(d_high, high)`: This line updates the current day's high. The `nz()` function handles the initial `na` value on the very first bar of the period.
    3.  **Period Rollover Logic:** On the *first bar* of a new period (e.g., when `is_new_day` becomes `true`):
        *   The "Previous" period's level (`pd_high`) is assigned the final value of the "Current" period's tracker (`d_high`).
        *   The "Current" period's tracker (`d_high`) is reset to the `high` of this new bar.
    *   **Intended Effect:** This logic creates a **step-function** for the "Previous" levels (`pd_high`, `pw_high`, etc.). These values remain constant for the entire duration of the new period, providing a static, historical reference point against which current price action can be measured.

#### **Component 2: Intraday Session Range Tracker**
This component isolates the price extremes of specific, high-liquidity trading sessions.

*   **Specific Configuration:**
    *   **Price Source:** `high` and `low` of each bar.
    *   **Time Windows (GMT):** The sessions are defined by hard-coded GMT time ranges, ensuring consistency regardless of the user's chart timezone.
        *   **Sydney:** 21:00 - 05:59 GMT
        *   **Tokyo (Asia):** 23:00 - 07:59 GMT
        *   **London:** 07:00 - 15:59 GMT
        *   **New York:** 12:00 - 20:59 GMT
    *   **Note:** The script correctly models the natural overlaps between sessions (e.g., London/NY overlap from 12:00-15:59 GMT).

*   **Functional Modification (Time-Based State Machine):**
    The logic mirrors the HTF tracker but uses time-based boolean flags (`inSydney`, `inLdn`, etc.) as its trigger.
    1.  **Session Start Detection:** A "new bar" flag (e.g., `sydNewBar = inSydney and not inSydney[1]`) detects the exact moment the session begins (a rising edge detection).
    2.  **Reset on Start:** When a session starts, its high/low trackers (`syd_h`, `syd_l`) are reset to the current bar's `high` and `low`.
    3.  **Intra-Session Update:** While the time is within the session (`inSydney` is true), the trackers are continuously updated with `math.max()` and `math.min()` to capture the session's developing range.
    *   **Intended Effect:** To dynamically build a visual representation of the session's trading range as it forms, providing real-time context for intraday support and resistance.

#### **Component 3: Equilibrium Lines**
This is a simple derived calculation, not an independent tracker.

*   **Specific Configuration:**
    *   **Source:** The calculated `pd_high`/`pd_low`, `pw_high`/`pw_low`, and `pm_high`/`pm_low` values.
    *   **Mathematical Formula:** `Equilibrium = (Period_High + Period_Low) / 2`

*   **Functional Modification:** This is a standard calculation for a 50% level of a given range.
    *   **Intended Effect:** To identify the mean or "fair value" point of a prior significant range. This level often acts as a powerful magnet for price or a secondary inflection point.

### 2. Logic Layering & Confluence

The script's intelligence lies not in programmatic filtering but in its visual presentation, which enables the user to perform hierarchical analysis.

*   **Interaction Dynamics:** The system is built to identify two primary dynamics:
    *   **Threshold Crosses:** The fundamental event is price approaching, touching, or breaching one of the plotted lines (e.g., `close` nearing `pd_high`). This is the primary "catalyst" event.
    *   **Convergences:** The script does not calculate confluence programmatically. Instead, by plotting all levels simultaneously, it allows the user to visually identify price zones where multiple levels cluster (e.g., a Previous Weekly High that is also the high of the prior day's London session). These zones represent areas of heightened structural significance.

*   **Hierarchical Filtering:** The script presents a clear visual hierarchy of importance, which acts as a noise filter for the analyst.
    *   **Gatekeeper by Timeframe:** The core filtering principle is that higher timeframe levels hold more weight. A monthly low (`pml`) is a more significant structural level than a daily low (`pdl`). A trader can use this implicit hierarchy to filter out trades at minor levels when a major HTF level is nearby.
    *   **Intraday Context Filter:** The session ranges act as a secondary filter. For example, a test of the Previous Day's High (`pd_h`) that occurs *during* the London session open carries a different weight than one that occurs in the low-volume period between sessions. The session boxes frame the HTF level interaction, adding a layer of contextual data.

### 3. The Execution Engine

This script contains **no automated execution engine**. Its purpose is decision support. The "triggers" are visual events on the chart, interpreted by the trader.

*   **Boolean Logic (State Management, Not Trade Execution):** The script's boolean logic is used exclusively for managing the state of the level trackers and optimizing drawing operations.
    *   `is_new_day = gmtH_lvl == 22 and hour(time[1], "GMT+0") != 22`: This is a precise rising-edge trigger that fires only on the first bar where the GMT hour becomes 22. It is the core of the daily state machine.
    *   `inLdn = gmtT >= 700 and gmtT < 1600`: This defines a state (being inside the London session). It is not a trigger itself but enables the session high/low tracking logic.
    *   `barstate.islast`: This is a performance optimization. It ensures that the code for drawing and updating the primary `line` and `label` objects runs only once, on the last (real-time) bar of the chart. This dramatically reduces the script's computational load.

*   **Mathematical Constants:**
    *   **`22:00 GMT`:** This is the most critical constant. It defines the Forex market's daily boundary (the close of the New York session) and ensures the Previous Day High/Low (PDH/PDL) are calculated based on the correct institutional trading day.
    *   **Session Time Integers (e.g., `700`, `1600`):** These represent the start and end times of trading sessions in `HHMM` format. They are industry-standard and crucial for the accuracy of the session range component.
    *   **`pip = syminfo.mintick * 4`:** This is a minor cosmetic constant. It creates a small price offset for placing labels slightly above or below their corresponding lines to improve readability. It has no influence on the strategy's logic or risk profile.
    *   **`5` in `ta.sma(math.sign(delta), 5)`:** This constant is used only for the visual gradient fill inside session boxes. It smooths the color transition over 5 bars, preventing rapid, noisy color flickering on choppy candles. It has zero impact on the calculation of the levels themselves.
    