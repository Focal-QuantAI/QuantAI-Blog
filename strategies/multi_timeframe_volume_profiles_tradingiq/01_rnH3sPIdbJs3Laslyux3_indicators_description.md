
# Indicators Description

### 1. Component Deconstruction

The script's architecture is centered around a single, powerful, and custom-built component: a high-resolution Volume Profile engine. This engine has two distinct operational modes.

#### **A. Core Engine: High-Resolution Volume Profile**

This is not a standard TradingView built-in function. It is a bespoke engine constructed from the ground up to achieve maximum data granularity.

*   **Specific Configuration:**
    *   **Price Source:** The engine does not use the chart's native `high` and `low`. Instead, it uses `request.security_lower_tf` to pull `high`, `low`, and `volume` data from a user-defined lower timeframe (LTF), with a default of "1" (1 minute). This is the cornerstone of its high-resolution nature.
    *   **Vertical Resolution (Rows):** The price range of each profile is discretized into a user-defined number of bins (`rows`, default = 20). The height of each bin is `(Session High - Session Low) / rows`.
    *   **Timeframe Anchors (HTF):** The script can generate up to five independent profiles, each anchored to a user-defined higher timeframe (HTF) (e.g., `30`, `60`, `240`, `1D`, `1W`).
    *   **Value Area (VA):** The Value Area is calculated using a hard-coded mathematical constant of **70%** (`target = htfProfile.totalVol.sum() * 0.7`).

*   **Functional Modification (Mathematical Logic):**
    1.  **Data Aggregation:** For a given HTF period (e.g., a single day for the "1D" profile), the script requests and stores an array of all constituent LTF bars (e.g., all 1-minute bars) within that session.
    2.  **Volume Distribution Algorithm:** The volume of a single LTF bar is not assigned to a single price level. Instead, it is distributed proportionally across all the profile rows (`bins`) that the LTF bar's range (`high` to `low`) intersects. The formula is: `div = data.V / (math.abs(upLev - dnLev) + 1)`, where `data.V` is the LTF bar's volume, and `upLev`/`dnLev` are the indices of the highest and lowest profile rows it touched. This prevents volume distortion from single volatile bars and provides a more accurate distribution.
    3.  **Value Area Calculation:** The VA is computed by starting at the Point of Control (POC) row and iteratively expanding outwards (both up and down), adding the volume of the next adjacent row with the higher volume until the cumulative sum reaches 70% of the session's total volume. The highest and lowest rows in this set define the Value Area High (VAH) and Value Area Low (VAL).

#### **B. Operational Mode: Delta Profile**

This is a functional modification of the core engine, altering how volume is interpreted and visualized.

*   **Specific Configuration:**
    *   Activated via the `model` input (`modelType.deltaVP`).
    *   It uses the same `rows`, `HTF`, and `LTF` configuration as the traditional profile.

*   **Functional Modification (Mathematical Logic):**
    1.  **Volume Signing:** The core modification occurs during the initial data request. The volume from the LTF is signed based on price direction. The `direction(lowerTF)` function returns `+1` or `-1`.
        *   For tick data (`"1T"`), it uses `close == ask` for `+1` (buy) and `close == bid` for `-1` (sell).
        *   For all other timeframes, it uses `math.sign(close - close[1])`. A positive value indicates buying pressure; a negative value indicates selling pressure.
    2.  **Delta Calculation:** The signed volume (`volume * direction()`) is then aggregated into a `delta` array for each price row. A positive value in a row's delta indicates net buying pressure at that level, while a negative value indicates net selling pressure.
    3.  **Visualization Logic:** Unlike the traditional profile which shows buy and sell volume side-by-side, the Delta Profile visualizes only the *net imbalance*. The `addPoints` method draws a bar representing the absolute delta value, colored based on its sign (up/buy or down/sell). This immediately highlights levels of aggressive buyer absorption or seller initiative.

### 2. Logic Layering & Confluence

The script's filtering mechanism is not based on combining disparate indicators but on the hierarchical layering of a single, robust concept across time.

*   **Interaction Dynamics:** The primary dynamic is **Price-Level Intersection**. The engine's purpose is to pre-calculate and render structurally significant price zones (POC, VAH, VAL). The system generates an "insight" when the current market price interacts with one of these static, historical levels. It does not use moving averages or oscillators for confirmation.

*   **Hierarchical Filtering:**
    1.  **Primary Filter (Macro Structure):** The highest timeframes configured (e.g., Weekly, Daily) act as the primary "Gatekeeper." The POC, VAH, and VAL from these profiles represent major areas of liquidity and value consensus. A trader using this script would first identify where the current price is in relation to these macro levels.
    2.  **Secondary Filter (Intraday Structure):** The lower HTFs (e.g., 4H, 1H) provide context for the intraday auction. They reveal how the market is rotating *within* or *reacting to* the larger macro structure.
    3.  **Confluence as a Filter:** The script's core value proposition is visualizing confluence. A high-conviction zone is identified where levels from different timeframes overlap. For example, a Weekly VAL that aligns with a Daily POC creates a powerful support zone. The script filters noise by focusing the user's attention on these clustered levels, implying a multi-timeframe consensus on value that is harder to breach.
    4.  **Data Resolution as a Filter:** By using LTF data (e.g., 1-minute) to construct the HTF profiles, the script improves the signal-to-noise ratio. It builds a precise map based on where transactions *actually occurred*, filtering out the ambiguity of using the chart's native (and often coarse) bar data for profile construction.

### 3. The Execution Engine

This script is a decision-support tool, not an automated strategy. Its "Execution Engine" is designed to provide visual triggers for a discretionary trader.

*   **Boolean Logic (Calculation Trigger):**
    *   The entire calculation and drawing process is encapsulated within a `if barstate.islast` block. This is a critical performance optimization, ensuring the computationally expensive process of iterating through hundreds or thousands of LTF bars and redrawing objects only occurs on the most recent, real-time bar.
    *   The `timeframe.change(HTF)` boolean acts as a reset trigger, clearing the historical LTF data and restarting the profile calculation at the beginning of each new HTF session.
    *   `htfUse1` through `htfUse5` are simple boolean toggles that determine whether a given profile's calculation and drawing logic is executed at all.

*   **Trigger Conditions (Visual, Not Automated):**
    *   The script does not generate `alertcondition` or strategy entry/exit signals.
    *   The "Trigger" is a visual event defined by the user: **`current_price` intersects `{POC | VAH | VAL}`**.
    *   A higher-probability trigger is a **Confluence Trigger**: **`current_price` intersects a cluster of levels** (e.g., `Daily_VAL` ≈ `4H_POC`).
    *   When using the **Delta Profile** model, the trigger can be further refined by observing the delta at the level of interest. A test of a key support level (e.g., a Daily VAL) accompanied by a large positive delta at that level suggests strong buyer absorption, providing a more nuanced entry trigger.

*   **Mathematical Constants:**
    *   **`0.7` (70%):** This constant defines the Value Area. It is a widely accepted standard in Auction Market Theory, representing the price range where 70% of a session's volume was traded. It has a direct impact on the perceived "fair value" zone and does not influence risk-to-reward directly, but rather the location of potential trade entries and exits.
    *   **`15`:** This hard-coded number in the `normBuy`/`normSell`/`normDelta` calculation is a **visual scaling factor**. It normalizes the volume/delta data to a maximum horizontal width of approximately 15 `bar_index` units. It has **zero impact** on the mathematical integrity of the profile or any risk/reward calculation; it purely governs the aesthetic width of the drawn profile on the chart.
    