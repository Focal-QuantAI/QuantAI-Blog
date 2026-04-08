
# Indicators Description

### 1. Component Deconstruction

The script's core is a bespoke, multi-instance Volume Profile engine. It does not use standard TradingView indicators but builds its entire analytical framework from raw price and volume data.

#### **A. Custom Volume Profile Engine**

This is the central component, instantiated up to five times for different timeframes.

*   **Specific Configuration:**
    *   **Timeframe (`htf1` - `htf5`):** User-definable higher timeframes (e.g., "30", "60", "240", "1D"). Each instance of the engine analyzes a discrete period corresponding to its specified timeframe.
    *   **Vertical Resolution (`rows`):** A user-defined integer (default: 20) that dictates the number of price buckets the profile is divided into. The total vertical range of the HTF period (`htfH` - `htfL`) is segmented into this number of levels.
    *   **Volume Data Source (`lowerTF1` - `lowerTF5`):** A user-defined lower timeframe (default: "1"). This is the source of the volume data used to construct the HTF profile.

*   **Functional Modification (Data Synthesis):**
    The script employs a non-standard, granular method for profile construction. Instead of using the chart's native volume data for the HTF bar, it synthesizes the profile by fetching finer-grained data from a lower timeframe.
    1.  **Data Request:** For each HTF period, the script uses `request.security_lower_tf` to pull an array of `high`, `low`, and signed `volume` values from every bar within the specified `lowerTF`.
    2.  **Volume Allocation:** The engine iterates through each of these LTF data points. For a single LTF bar, it identifies the vertical price buckets (`rows`) that the bar's range (`ltfH` to `ltfL`) has crossed.
    3.  **Volume Distribution:** The volume of that single LTF bar is then divided equally among all the price buckets it touched. The mathematical logic is `div = data.V / (math.abs(upLev - dnLev) + 1)`, where `data.V` is the LTF bar's volume and `upLev`/`dnLev` are the start and end bucket indices. This prevents all of a bar's volume from being assigned to a single level, providing a more distributed and accurate representation of trading activity across the bar's range.

#### **B. Delta Calculation Engine**

This is an oscillating study integrated directly into the Volume Profile. It operates in two modes depending on the `lowerTF` input.

*   **Specific Configuration:**
    *   **Price Source:** `close`, `bid`, `ask`.
    *   **Lookback Period:** `1` (compares `close` to `close[1]`).

*   **Functional Modification (Trade Direction Approximation):**
    The script calculates "Delta" by signing the volume from the `lowerTF`. This is achieved via the `direction()` function.
    1.  **Tick-Rule Approximation (for `lowerTF = "1T"`):** If the data source is tick data, it approximates trade aggression.
        *   `close == bid`: The last trade occurred at the bid price, implying a seller-initiated trade (negative delta, `-1`).
        *   `close == ask`: The last trade occurred at the ask price, implying a buyer-initiated trade (positive delta, `+1`).
        *   Otherwise, it falls back to the standard uptick/downtick rule.
    2.  **Uptick/Downtick Rule (for all other `lowerTF`):** For standard timeframes, it uses `math.sign(close - close[1])`.
        *   If `close > close[1]`, the volume is considered buying volume (`+1`).
        *   If `close < close[1]`, the volume is considered selling volume (`-1`).
    The final signed volume (`ltfV`) is calculated as `volume * direction(lowerTF)` within the `request.security_lower_tf` call. This signed value is then accumulated into the `delta`, `buyVol`, and `sellVol` arrays of the profile.

#### **C. Value Area (VA) & Point of Control (POC) Calculation**

This is a statistical study performed on the generated `totalVol` array of each profile.

*   **Specific Configuration:**
    *   **VA Percentage:** Hardcoded at `0.7` (70%).
    *   **Data Source:** The `totalVol` array, which is the sum of absolute buy and sell volume for each price level.

*   **Mathematical Logic:**
    1.  **POC Identification:** The Point of Control is identified by finding the index of the maximum value in the `totalVol` array (`htfProfile.totalVol.max()`). The corresponding price level is the POC.
    2.  **Value Area Growth Algorithm:** The VA is calculated via an iterative expansion algorithm.
        *   It begins with the volume of the POC row.
        *   It then enters a loop, adding the volume from the row directly above and the row directly below the current range.
        *   This expansion continues symmetrically outwards from the POC until the cumulative volume within the expanding range (`sum`) meets or exceeds 70% of the total volume for the entire profile (`htfProfile.totalVol.sum() * 0.7`).
        *   The highest price level in this final range is the Value Area High (VAH), and the lowest is the Value Area Low (VAL).

### 2. Logic Layering & Confluence

The script's filtering mechanism is not sequential but **spatial and comparative**. It generates multiple independent data structures (the five profiles) and presents them simultaneously, relying on the user to identify confluence.

*   **Interaction Dynamics:**
    *   **Confluence of Levels:** The primary "signal" is the visual alignment of key levels from different timeframe profiles. The engine does not programmatically check for this; it provides the visual data for the analyst to do so. A high-probability zone is identified when, for example, the `POC` of the `htf1` ("30m") profile aligns with the `VAL` of the `htf3` ("240m") profile. This layering of significance across timeframes is the core noise-reduction technique.

*   **Hierarchical Filtering:**
    The script enables a two-tiered filtering process, moving from macro structure to micro confirmation.
    1.  **Gatekeeper (Structural Context):** The higher timeframe profiles (`htf3`, `htf4`, `htf5`) act as the primary gatekeepers. They define the major structural support and resistance zones (macro POCs, VAHs, VALs). A trade setup is only considered valid if price is interacting with one of these significant HTF levels. Activity occurring in the "middle of nowhere" is filtered out as noise.
    2.  **Confirmation (Aggression Analysis):** The `deltaVP` model acts as the confirmation filter. Once price reaches a key structural level identified by the traditional profiles, the user can switch the `Model` input to "Delta Profile".
        *   **Function:** This reveals the net buyer vs. seller aggression at each price level.
        *   **Signal:** At a key support level, the appearance of strong positive delta (large green bars) confirms that buyers are absorbing selling pressure, validating a long entry. Conversely, strong negative delta (large red bars) at a resistance level confirms seller dominance. This filters out trades at levels that are not showing the expected institutional response.

### 3. The Execution Engine

This script is a discretionary analysis tool; it has no automated execution engine. The "trigger" is a cognitive event for the trader, based on the confluence of data provided by the script.

*   **Boolean Logic (Cognitive Trigger):**
    The script facilitates a discretionary decision based on the following implied conditions:
    *   `isLocationValid` = `true` when `price` is at or near a key level (POC, VAH, VAL) from a primary HTF profile (e.g., `htf3`).
    *   `isConfluencePresent` = `true` when that same price level is also a key level on a secondary profile (e.g., `htf1`).
    *   `isConfirmationPresent` = `true` when the `deltaVP` model shows a net delta imbalance at that level that supports the intended trade direction (e.g., `delta > 0` for a long).

    A high-conviction "trigger" for the analyst occurs when:
    `isLocationValid AND isConfluencePresent AND isConfirmationPresent`

*   **Mathematical Constants:**
    *   **`0.7` (Value Area):** This hardcoded multiplier directly defines the boundaries of "accepted value." A higher value (e.g., 0.8) would result in a wider VAH/VAL range, classifying more of the profile as "value." A lower value would tighten the range, making the criteria for value more stringent.
    *   **`15` (Normalization Multiplier):** Found in the `normBuy`, `normSell`, and `normDelta` calculations. This is a purely **visual scaling factor**. It controls the horizontal length of the profile's bars on the chart. It has **zero impact** on the underlying volume/delta calculations or the determination of POC/VAH/VAL. Its purpose is to ensure the profile is visually legible regardless of the absolute volume traded.
    *   **`bar_index` Offsets (e.g., `+49`, `+59`):** These are hardcoded **positional constants** used for drawing. They dictate the horizontal placement of each profile on the chart, ensuring they appear side-by-side without overlapping. They have no influence on the mathematical engine or its analytical output.
    