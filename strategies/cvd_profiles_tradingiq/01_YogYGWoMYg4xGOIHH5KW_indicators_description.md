
# Indicators Description

### 1. Component Deconstruction

#### **A. Cumulative Volume Delta (CVD) Engine**

*   **Specific Configuration:**
    *   **Price Source:** The delta calculation is derived from a lower timeframe (`TF`) specified by the user (default: "1").
    *   **Directional Logic (`direction()` function):** The script approximates tick-by-tick directionality.
        *   For `TF = "1T"` (1 Tick), it attempts to use `bid` and `ask` prices. A trade at the `ask` is considered a buy (`+1`), and a trade at the `bid` is a sell (`-1`). If price is between `bid` and `ask`, it falls back to the standard method.
        *   For all other timeframes, it uses the `math.sign(close - close[1])` method, also known as the "Up/Down Tick Rule". A positive sign indicates buying volume for that bar, and a negative sign indicates selling volume.
    *   **Aggregation:** The delta for each lower-timeframe bar (`ltfV`) is summed up (`ltfV.sum()`) and then cumulatively added to a running total (`cvd += nz(ltfV.sum())`) on each bar of the chart's primary timeframe.

*   **Functional Modification:**
    *   The script's primary innovation is not the CVD itself, but its projection onto a vertical price histogram, creating a **CVD Profile**.
    *   Instead of a single cumulative line, the script distributes the calculated delta (`div = data / (math.abs(end - start) + 1)`) across all the price "ticks" or rows that a lower-timeframe bar traversed. This transforms the one-dimensional CVD data into a two-dimensional profile, answering not just "who is in control?" but "*where* are they exerting control?".

#### **B. Custom Profile Engine**

*   **Specific Configuration:**
    *   **Row Height (`tickAmount`):** The vertical resolution of the profile is determined by `tickAmount`.
        *   If `ticks` input is `0` (default), the row height is adaptive, calculated as `ta.atr(14) * syminfo.mintick`. This links the profile's granularity to market volatility.
        *   If `ticks` is set to a non-zero value, it uses that fixed number of ticks for the row height.
    *   **Session Period:** The profile accumulates data over a period defined by the `timeframe` input (default: "1D" via `timeframe.change("1D")`). The calculation resets at the start of each new session. A "Fixed Start" option allows for a continuous profile from a user-defined timestamp.

*   **Functional Modification:**
    *   This is a bespoke profiling engine built from the ground up using Pine Script's drawing objects (`box.new`). It does not use any built-in profile functions.
    *   Its core function is to act as a canvas for different data models selected by the user (`model` input):
        1.  **CVD:** Profiles the raw Cumulative Volume Delta at each price level.
        2.  **Imbalance Ratio:** Profiles the ratio of buying volume to selling volume (`nz(TP.buyVol.get(x) / TP.sellVol.get(x))`) at each price level.
        3.  **Activity:** Profiles the percentage of total session volume that occurred at each price level.
        4.  **USD Volume:** Profiles the notional value traded at each price level (`absAmount * ltfC.get(i)`).

#### **C. Value Area (VA) & Point of Control (POC)**

*   **Specific Configuration:**
    *   **POC Source:** The Point of Control is the price level with the highest `totalVol` (total volume traded), not the highest delta.
    *   **VA Percentage (`vaCumu`):** The Value Area is configured to contain `70%` (default) of the session's total volume.
    *   **Calculation Logic:** The VA is calculated by starting at the POC and symmetrically expanding outwards, level by level, until the cumulative volume within the expanding range meets or exceeds the `vaCumu` percentage.

*   **Functional Modification:**
    *   While the calculation method is standard for volume profiles, its application here is unique. The VA and POC are calculated based on **total volume activity** but are used to contextualize the **delta-based data models** (CVD, Imbalance Ratio). This creates a framework where "fair value" is defined by volume, but the sentiment within that value area is judged by order flow.

#### **D. Off-Chart Oscillators (CVD Delta & CVD Acceleration)**

*   **Specific Configuration:**
    *   **CVD Delta:** A first-order derivative of the cumulative CVD line (`cvd - cvd[1]`). It measures the rate of change or momentum of the order flow.
    *   **CVD Acceleration:** A second-order derivative of the cumulative CVD line (`cvdDelta - cvdDelta[1]`). It measures the rate of change of the momentum, highlighting inflection points.
    *   **Source:** These are calculated from the main `cvd` variable on the chart's timeframe.

*   **Functional Modification:**
    *   The coloring of these oscillators is dynamic. The color intensity is graded against a rolling array of the 50 most recent highest absolute values (`maxAmt`). This normalizes the visualization, ensuring that significant spikes are highlighted relative to recent activity, rather than an arbitrary fixed scale.

### 2. Logic Layering & Confluence

The script is a visual analysis tool, not an automated strategy. Its logic is layered to provide the trader with a contextual framework for interpreting order flow.

*   **Interaction Dynamics:** The engine is designed to spot **Divergences** and analyze activity at **Thresholds**.
    *   **Price vs. Order Flow Divergence:** The primary use case is identifying a decoupling between price action and the CVD Profile. For example, if price makes a new low but the corresponding CVD at that low is less negative than at the previous low (or even positive), it signals seller exhaustion and absorption by passive buyers.
    *   **Imbalance Ratio Threshold:** The "Imbalance Ratio" model is built around the threshold of `1.0`. Ratios significantly greater than 1 indicate strong initiative buying, while ratios approaching 0 indicate overwhelming selling. The color gradient is explicitly designed to highlight these imbalances.

*   **Hierarchical Filtering:** The script employs a multi-stage filtering process to improve the signal-to-noise ratio.
    1.  **Primary Gatekeeper (The Value Area):** The VA (VAH, VAL) and POC act as the highest-level filter. They define the session's "fair value" range. Activity inside the VA is considered balancing/rotational, while activity outside is initiative/directional.
    2.  **Secondary Filter (Location):** The script's logic implicitly prioritizes signals at these key structural levels. A large CVD imbalance occurring at the VAH or VAL is considered a far more significant event (potential breakout or rejection) than the same imbalance occurring in the middle of the range.
    3.  **Tertiary Filter (The Data Model):** The chosen `model` (CVD, Imbalance, etc.) provides the final layer of confirmation. A trader first identifies the context (price at VAL), then observes the specific order flow characteristic (e.g., a strong buying imbalance) to form a complete trade thesis. The engine does not generate a signal unless this hierarchy of conditions is visually met.

### 3. The Execution Engine

As an `indicator`, the script has no automated execution engine. Its "triggers" are visual confluences designed for discretionary trader interpretation.

*   **Boolean Logic (Visual Interpretation):**
    *   **High-Probability Long Trigger (Example):**
        *   `price` is testing or has swept below the `Value Area Low (VAL)`.
        *   **AND** The CVD Profile at that price level shows a positive value OR the "Imbalance Ratio" is `> 1`.
        *   **AND** The `offChart` CVD Delta oscillator prints a strong positive bar.
    *   **High-Probability Short Trigger (Example):**
        *   `price` is testing or has swept above the `Value Area High (VAH)`.
        *   **AND** The CVD Profile at that price level shows a negative value OR the "Imbalance Ratio" is `< 1`.
        *   **AND** The `offChart` CVD Delta oscillator prints a strong negative bar.

*   **Mathematical Constants & Their Significance:**
    *   **`vaCumu = 70` (%):** This is a market profile standard derived from statistical distribution, representing approximately one standard deviation of activity. It defines the boundary between "value" and "excess." Modifying this directly impacts the sensitivity of the primary filter; a lower value creates a tighter VA, leading to more frequent "breakout" signals, while a higher value creates a wider VA, filtering for only the most significant directional moves.
    *   **`Imbalance Ratio Threshold = 1.0`:** This is the mathematical equilibrium point between buying and selling volume at a price level. The script's coloring logic for the Imbalance model pivots on this constant, visually separating buyer-dominated levels from seller-dominated ones.
    *   **`atr(14)`:** The lookback period of `14` for the ATR is a standard industry choice. Its use here ensures that the profile's resolution (`tickAmount`) is dynamically tied to a medium-term volatility baseline, preventing the profile from becoming too coarse in quiet markets or too fine in volatile ones.
    