
# Indicators Description

### 1. Component Deconstruction

The script's architecture is centered around a single, highly configurable, custom-built component: a **Multi-Timeframe Volume & Delta Profile Engine**. It does not use standard, pre-built indicators like RSI or EMA. Instead, it constructs its analytical tools from raw price and volume data.

#### **A. The Volume Profile Engine (Core Component)**

This is a bespoke implementation of a Volume Profile, built from the ground up.

*   **Specific Configuration:**
    *   **Timeframes (HTF):** The script can simultaneously compute up to five independent profiles, each tied to a user-defined higher timeframe (e.g., `30m`, `4H`, `1D`).
    *   **Data Source:** The engine's primary innovation is its data source. Instead of using the OHLCV of the higher timeframe bar itself, it uses `request.security_lower_tf` to pull in granular data from a user-specified lower timeframe (`LTF Vol`, e.g., `1m`). This allows it to build a high-fidelity, intra-bar profile for each HTF period.
    *   **Price Bins (Rows):** The vertical resolution of the profile is controlled by the `rows` input (default: 20). The total price range of the HTF period (`htfH` - `htfL`) is divided into this number of discrete price levels.
    *   **Profile Models:** The engine has two distinct operational modes selected via the `model` input:
        1.  `Traditional Volume Profile`
        2.  `Delta Profile`

*   **Functional Modification & Mathematical Logic:**
    *   **Volume Distribution Algorithm:** For each bar from the lower timeframe data, the script identifies the price bins (`rows`) it spans. The volume of that LTF bar is then divided equally among all the bins it touches. The formula is `div = data.V / (math.abs(upLev - dnLev) + 1)`, where `data.V` is the LTF bar's volume, and `upLev`/`dnLev` are the indices of the highest and lowest price bins the bar touched. This prevents a single high-volume, wide-ranging bar from unfairly weighting a single price bin.
    *   **Delta Calculation (`direction()` function):** This is the mathematical core of the `Delta Profile` model.
        *   It determines whether volume is "buying" or "selling" volume by calculating a `direction` multiplier (+1 or -1).
        *   For standard timeframes, it uses the simple proxy: `math.sign(close - close[1])`. A positive value indicates buying pressure; a negative value indicates selling pressure.
        *   For tick-based timeframes (`1T`), it attempts a more precise calculation using `close == bid` (selling) and `close == ask` (buying).
        *   The volume from each LTF bar is multiplied by this `direction` value. The `setVals()` method then accumulates this signed volume into a `delta` array for each price bin.

#### **B. Derived Studies**

These are not separate indicators but are calculated from the primary Volume Profile data.

*   **Point of Control (POC):**
    *   **Configuration:** This is not configurable. It is algorithmically determined.
    *   **Logic:** The script finds the price bin with the highest `totalVol` within the profile. The price level of this bin is designated as the POC.

*   **Value Area (VA):**
    *   **Configuration:** The percentage is hard-coded.
    *   **Logic:** The script calculates a Value Area based on a fixed **70%** of the total volume traded during the HTF period (`target = htfProfile.totalVol.sum() * 0.7`).
    *   **Algorithm:** It employs a standard "grow from POC" algorithm. It starts with the volume of the POC row, then iteratively adds the volume from the rows immediately above and below, expanding outwards until the cumulative sum of volume exceeds the 70% target. The highest price level in this range is the Value Area High (VAH), and the lowest is the Value Area Low (VAL).

### 2. Logic Layering & Confluence

The script's design philosophy is based on visual, structural filtering rather than sequential boolean logic.

*   **Interaction Dynamics:**
    *   **Structural Convergence:** The primary "signal" is the visual alignment (convergence) of key levels from different timeframe profiles. The engine's purpose is to make these convergences obvious. For example, a trader would look for a scenario where the **Daily VAH** (`htf4`) aligns horizontally with the **4-Hour POC** (`htf3`). This alignment of value-inflection points across multiple timeframes significantly increases the level's expected structural importance, filtering out noise from levels that are only significant on a single timeframe.

*   **Hierarchical Filtering:**
    *   **Timeframe Hierarchy:** The script establishes an explicit hierarchy of market structure. A level derived from a `1W` profile is inherently a stronger "Gatekeeper" than a level from a `30m` profile. The script does not programmatically enforce this; it presents the information visually, allowing the analyst to prioritize signals from higher-order structures. A trade setup is considered high-probability only when price action at a lower-order (e.g., `30m`) level is validated by a corresponding higher-order (e.g., `1D`) structural level.
    *   **Delta as a Confirmation Filter:** The `Delta Profile` model acts as a secondary, sophisticated filter. After identifying a key price level using the `Traditional Volume Profile` (the "what" and "where"), the user can switch to the `Delta Profile` to analyze the order flow aggression (the "who" and "how").
        *   **Example:** If price approaches a support level identified as a high-volume node, seeing a large, positive delta value at that level confirms that aggressive buyers are actively absorbing selling pressure, validating the strength of the support. Conversely, a negative delta would invalidate the setup.

### 3. The Execution Engine

This script is a discretionary analysis tool, not an automated strategy. It has no "Execution Engine" in the traditional sense of generating `strategy.entry()` calls. Its "triggers" are visual patterns presented to the analyst.

*   **Trigger Conditions (Discretionary):**
    *   The "trigger" is the confluence of multiple data points, interpreted by the user. A high-probability long setup might be defined by the following visual conditions:
        1.  Price is testing a level that is a **convergence** of a Daily POC and a 4-Hour VAL.
        2.  The `Delta Profile` for that level shows a significant **positive delta**, indicating buyer absorption.
        3.  The real-time price action shows signs of rejection (e.g., long lower wicks on the candlestick chart).

*   **Mathematical Constants & Their Significance:**
    *   `0.7`: The hard-coded 70% multiplier for the Value Area calculation. This is a standard in Auction Market Theory, approximating one standard deviation of volume distribution and representing the area of "fair value" where the majority of business was conducted.
    *   `15`: Used in the normalization formula (`* 15`). This is a purely **visual scaling factor**. It controls the maximum horizontal length of the profile bars to ensure they are visually coherent on the chart. It has **no impact** on the mathematical calculation of POC, VA, or Delta, only on their graphical representation.
    *   `bar_index + 49`, `+ 109`, etc.: These are hard-coded **horizontal plot offsets**. Their sole purpose is to position the five potential profiles side-by-side across the chart without overlapping. They are critical for the script's visual layout but have no analytical significance.
    *   `rows` (default `20`): This input directly controls the **granularity** of the analysis. A lower number creates a coarse, low-resolution profile, while a higher number provides a fine, high-resolution view of volume distribution. This parameter directly impacts the signal-to-noise ratio; too high a value can introduce noise, while too low a value can obscure important details.
    