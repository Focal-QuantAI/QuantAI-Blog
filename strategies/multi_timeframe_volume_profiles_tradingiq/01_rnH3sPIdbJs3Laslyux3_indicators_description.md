
# Indicators Description

### 1. Component Deconstruction

The script's architecture is centered on a custom-built, multi-timeframe volume profiling engine. It does not use standard library indicators like RSI or EMA; instead, it constructs its analytical tools from raw price and volume data.

#### **A. Core Engine: Volume Profile**

This is a bespoke implementation, not a built-in TradingView function. Its mechanics are as follows:

*   **Specific Configuration:**
    *   **Vertical Resolution:** Controlled by `rows` (default: 20). This integer dictates the number of horizontal price bins the total price range of a period is divided into. A higher number increases the granularity of the profile at the cost of computational intensity.
    *   **Price Range Source:** The engine dynamically calculates the high and low of the selected higher timeframe (`HTF`) period (e.g., the last 60 minutes). This range (`htfH` - `htfL`) is then divided by `rows` to determine the height of each price bin.
    *   **Volume Data Source:** The engine's key feature is its use of `request.security_lower_tf`. It fetches `high`, `low`, and `volume` data from a user-defined lower timeframe (`lowerTF`, default: "1"). This provides a high-fidelity data set for constructing the profile, as opposed to using the single volume value from the higher timeframe bar.

*   **Functional Modification:** The script offers two distinct mathematical models for processing volume, selected via the `model` input.

    1.  **Traditional Volume Profile (`regVP`):**
        *   **Mathematical Logic:** In this mode, the script calculates total buying and selling volume for each price row. The `direction()` function determines if a lower-timeframe bar's volume is "buying" or "selling" based on a simple up-tick/down-tick proxy: `math.sign(close - close[1])`. The absolute value of this signed volume is then added to the `buyVol` or `sellVol` array for each price row the bar touched. The `totalVol` is the sum of `buyVol` and `sellVol`.
        *   **Intended Effect:** This provides a classic volume-at-price histogram, separating volume into buying and selling pressure for visual analysis of supply and demand at specific levels.

    2.  **Delta Profile (`deltaVP`):**
        *   **Mathematical Logic:** This model focuses on the *net difference* between buying and selling pressure. The `direction()` function again produces a signed volume (+V for buying, -V for selling). The `setVals` method adds this signed value directly to the `delta` array for each relevant price row. The `totalVol` array still accumulates the absolute volume.
        *   **Intended Effect:** This model is designed to reveal the net aggressive activity at each price level. A large positive delta at a level indicates a strong dominance of buyers, while a large negative delta indicates seller dominance. A delta near zero on a high-volume node suggests absorption and two-sided trade. This serves as a conviction filter.

#### **B. Derived Study: Value Area (VA)**

*   **Specific Configuration:**
    *   **Threshold:** The Value Area is calculated using a hard-coded constant of **70%** of the total volume for the period (`target = htfProfile.totalVol.sum() * 0.7`).
*   **Functional Modification (Algorithmic Implementation):**
    1.  The Point of Control (POC)—the price row with the maximum `totalVol`—is identified.
    2.  An iterative algorithm begins at the POC, summing its volume.
    3.  It then expands outwards, alternately adding the volume from the price row above and the price row below the currently included range.
    4.  This process continues until the accumulated volume sum (`sum`) meets or exceeds the 70% `target`.
    5.  The highest price level in this range becomes the Value Area High (VAH) and the lowest becomes the Value Area Low (VAL). This is a standard, industry-accepted method for VA calculation.

#### **C. Derived Study: Point of Control (POC)**

*   **Specific Configuration:** The POC is defined as the single price row with the maximum value in the `totalVol` array.
*   **Functional Modification:** There is no modification to the standard definition. The script finds it using `htfProfile.totalVol.indexof(htfProfile.totalVol.max())` to locate the index of the highest volume and then retrieves the corresponding price level.

### 2. Logic Layering & Confluence

The script's filtering mechanism is not based on a sequence of indicator checks but on the simultaneous visualization of structural information across multiple timeframes.

*   **Interaction Dynamics:** The core principle is **Structural Confluence**. The script does not compute signals based on indicator interactions (e.g., `RSI > 50 AND VAH_Cross`). Instead, it renders up to five independent volume profiles, each representing a different timeframe. The analytical value is derived by the user visually identifying where key levels from different profiles align.
    *   **Example:** A VAH on the 30-minute profile (`htf1`) gains significant analytical weight if it coincides with the POC of the 4-hour profile (`htf3`). This alignment suggests a micro-level resistance is reinforced by a macro-level area of accepted value, creating a high-probability zone for price reaction.

*   **Hierarchical Filtering:** The script facilitates a visual hierarchy but does not enforce it programmatically.
    *   **Macro Filter:** Higher timeframe profiles (e.g., Daily, Weekly) act as the macro context or "Gatekeeper." Their POC, VAH, and VAL levels define the significant structural zones for the entire session or week.
    *   **Micro Triggers:** Lower timeframe profiles (e.g., 15-min, 30-min) show the intra-day auction's development. A trading setup is considered higher probability when price action at a micro-level (e.g., testing the 15-min VAL) occurs at a pre-defined macro-level support (e.g., the Daily VAL). The script provides the map for this analysis; the trader performs the filtering.

### 3. The Execution Engine

This script is a decision-support tool, not an automated strategy. It has no "Execution Engine" in the traditional sense of generating `strategy.entry` or `alertcondition` calls. Its "engine" is purely for data processing and visualization, designed to inform a discretionary trader's execution.

*   **Boolean Logic:** The primary logical condition governing the entire calculation and drawing process is `if barstate.islast`. This is a critical performance optimization. All intensive calculations—iterating through lower-timeframe data, building the profile arrays, calculating VA/POC, and managing drawing objects—are executed only once, on the last historical bar and on every real-time tick. This prevents the script from re-calculating the entire profile on every bar of the chart's history, which would be computationally prohibitive.

*   **Visual Triggers (Informing Execution):** The script's output is a set of visual levels that a trader uses as a basis for execution. The "triggers" are pattern-based:
    *   **Level Test:** Price approaching a VAH, VAL, or POC.
    *   **Level Rejection/Acceptance:** Candlestick patterns confirming a bounce from or a breakout through one of these levels.
    *   **Confluence:** The trigger is amplified when the level being tested is significant on multiple timeframes simultaneously.
    *   **Real-time Price Marker:** A small circle (`●`) is drawn at the price level corresponding to the current `close`, providing an immediate visual reference of where the current price is within the established volume structure.

*   **Mathematical Constants:**
    *   `0.7`: The 70% multiplier for the Value Area calculation. This is a market standard derived from statistical principles where ~68.3% of values fall within one standard deviation of the mean in a normal distribution. 70% is a widely accepted adaptation for market profiles.
    *   `15`: Used in the normalization formula (`* 15`). This is a **cosmetic scaling factor**. It controls the maximum horizontal length of the drawn volume histograms, ensuring they fit neatly on the chart without excessive width. It has **no impact** on the analytical calculation of POC/VA or the script's risk profile; it only affects the visual representation of volume magnitude.
    *   `bar_index + 49`, `+ 109`, `+ 169`, etc.: These are **horizontal plot offsets**. They are used to draw each of the five profiles in its own "lane" on the right side of the chart, preventing them from overlapping. They have no analytical meaning.
    