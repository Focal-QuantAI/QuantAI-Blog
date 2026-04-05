
# Indicators Description

### 1. Component Deconstruction

#### **Oscillators & Derived Metrics**

*   **Relative Strength Index (RSI)**
    *   **Specific Configuration:** Three separate RSI calculations are used as base features.
        *   `f_rsi_s`: `ta.rsi(close, 2)`
        *   `f_rsi_m`: `ta.rsi(close, 3)`
        *   `f_rsi_l`: `ta.rsi(close, 4)`
    *   **Functional Modification:** The raw RSI values are not used directly for signaling. They serve as inputs for two subsequent transformations: Z-Score Normalization and the creation of an RSI Signal Line Distance feature.

*   **Simple Moving Average (SMA)**
    *   **Specific Configuration:** SMAs are utilized in three distinct functional roles, never for direct plotting or crossover signals.
        1.  **Mean Reversion Base:** Three SMAs are calculated on `close[1]` (the previous bar's close) to serve as a baseline for price deviation features. This use of `close[1]` is a critical detail to prevent lookahead bias within the feature calculation itself.
            *   `ta.sma(close[1], 2)`
            *   `ta.sma(close[1], 3)`
            *   `ta.sma(close[1], 4)`
        2.  **RSI Signal Line:** An SMA is calculated on the historical values of each RSI (`f_rsi_s[1]`, `f_rsi_m[1]`, `f_rsi_l[1]`) to create a signal line.
            *   `ta.sma(f_rsi_s[1], 4)`
            *   `ta.sma(f_rsi_m[1], 4)`
            *   `ta.sma(f_rsi_l[1], 4)`
        3.  **Normalization Statistics:** Within the `normalize` function, `ta.sma(src[1], window_size)` is used to calculate the mean of a feature over the entire learning window.
    *   **Functional Modification:** The SMA is not a standalone indicator here but a mathematical tool used to derive more complex features (mean reversion, momentum acceleration) and for statistical normalization.

*   **Exponential Moving Average (EMA)**
    *   **Specific Configuration:** A single EMA is used as a post-signal classification filter.
        *   `ema = ta.ema(close, 20)`
    *   **Functional Modification:** The EMA is not part of the core prediction engine. It operates on the output of the KNN model to categorize generated signals as either "Major" (pro-trend) or "Counter" (counter-trend).

#### **Custom-Engineered Features**

*   **Price Deviation from MA (%):**
    *   **Mathematical Logic:** `(close - ta.sma(close[1], length)) / ta.sma(close[1], length) * 100`
    *   **Intended Effect:** This feature quantifies the current price's percentage deviation from its recent mean. It is a measure of mean-reversionary "tension." A large positive value indicates an overbought state relative to its short-term average, while a large negative value indicates an oversold state. Three versions are created using MA lengths of 2, 3, and 4.

*   **RSI Signal Line Distance:**
    *   **Mathematical Logic:** `RSI - ta.sma(RSI[1], signal_len)`
    *   **Intended Effect:** This feature functions as a momentum oscillator for the RSI itself, effectively measuring the RSI's rate-of-change or "acceleration." A positive value means the RSI is currently above its own moving average, indicating accelerating momentum. A negative value indicates decelerating momentum. Three versions are created for each of the base RSIs.

*   **Z-Score Normalization (`normalize` function):**
    *   **Mathematical Logic:** `(source - mean) / stdev`, where `mean` and `stdev` are calculated over the `window_size`. The calculation `ta.sma(src[1], len)` and `ta.stdev(src[1], len)` ensures that the statistics for normalization are not influenced by the current bar's value. A small constant (`0.00001`) is added to the standard deviation to prevent division-by-zero errors in flat markets.
    *   **Intended Effect:** This is a standard statistical procedure to transform all nine raw features onto a common scale (mean of 0, standard deviation of 1). This is critical for the KNN algorithm, as it prevents features with naturally larger numerical ranges (like RSI values from 0-100) from disproportionately influencing the distance calculation compared to features with smaller ranges (like MA deviations).

*   **PCA "Lite" (Heuristic Dimensionality Reduction):**
    *   **Mathematical Logic:** When `use_pca` is `true`, the nine normalized features are compressed into three "Principal Components" via simple summation.
        *   `pc1`: Sum of the three normalized RSI features (Represents aggregate momentum).
        *   `pc2`: Sum of the three normalized MA Deviation features (Represents aggregate mean-reversion pressure).
        *   `pc3`: Sum of the three normalized RSI Signal Line Distance features, scaled by 0.5 (Represents aggregate momentum acceleration).
    *   **Intended Effect:** This is a non-standard, heuristic approach to dimensionality reduction. Instead of calculating a mathematical covariance matrix and eigenvectors, it groups features by their conceptual "family." The goal is to reduce the dimensionality of the feature space from nine to three, aiming to improve the signal-to-noise ratio by focusing on the broader character of momentum, reversion, and acceleration rather than the noisy specifics of any single feature.

### 2. Logic Layering & Confluence

*   **Interaction Dynamics:** The script's engine is fundamentally based on **Confluence** through pattern recognition. It does not rely on simple threshold crosses of individual indicators.
    1.  **Multi-dimensional State Vector:** At each bar, the script constructs a state vector representing the market's "fingerprint." This vector consists of `[pc1, pc2, pc3]` if PCA is enabled, or a selection of the normalized features if disabled.
    2.  **Minkowski Distance:** The engine quantifies the similarity between the current state vector and historical state vectors (within the `window_size`) using the Minkowski distance formula: `(Σ|x_i - y_i|^p)^(1/p)`. This calculation provides a single numerical value for the "distance" between two multi-dimensional points in the feature space.
    3.  **Weighted Voting (Gaussian Kernel):** The script identifies the `k` nearest historical neighbors. Instead of a simple 1-vote-per-neighbor system, it employs a **Gaussian Kernel** for weighting: `weight = exp(-d² / 2σ²)`. This means neighbors that are extremely close in the feature space have an exponentially higher influence on the final probability score than neighbors that are merely "close." This is a sophisticated form of confluence where the quality of the match is paramount.

*   **Hierarchical Filtering:** The script uses a multi-stage filtering process to move from raw data to a final, classified signal.
    1.  **Gatekeeper Filter (KNN Probability):** The primary and most important filter is the KNN engine's output itself. A potential signal is only generated if the weighted probability (`prob_up` or `prob_down`) crosses the `prob_threshold`. This acts as a high-conviction gatekeeper, filtering out all market states that do not have a strong historical precedent for a directional move.
    2.  **State Machine Filter (`last_dir`):** A secondary filter is implemented to reduce signal noise and prevent "chatter." The `last_dir` variable ensures that a new long signal can only be generated if the previous signal was not also long (and vice-versa for shorts). This enforces an alternating signal sequence.
    3.  **Classification Filter (EMA Trend):** The final layer does not block signals but classifies them. After a signal passes the first two filters, its position relative to the 20-period EMA determines its label.
        *   **Major Signal:** A long signal above the EMA or a short signal below the EMA. This implies the momentum burst is aligned with the prevailing short-term trend.
        *   **Counter Signal:** A long signal below the EMA or a short signal above the EMA. This implies the signal is anticipating a potential trend reversal or a counter-trend pullback.

### 3. The Execution Engine

*   **Boolean Logic (Trigger Conditions):**
    *   **Long Signal (`long_signal`):** A long signal is triggered (`true`) for a single bar if the following two conditions are met simultaneously:
        1.  `ta.crossover(prob_up, prob_threshold)` is `true`. This means the calculated probability for a bullish outcome has just crossed above the user-defined confidence threshold on the current bar.
        2.  `last_dir <= 0`. This means the last valid signal generated was either bearish (`-1`) or neutral (`0`), preventing consecutive long signals.
    *   **Short Signal (`short_signal`):** A short signal is triggered (`true`) for a single bar if the following two conditions are met simultaneously:
        1.  `ta.crossover(prob_down, prob_threshold)` is `true`. This means the calculated probability for a bearish outcome has just crossed above the user-defined confidence threshold.
        2.  `last_dir >= 0`. This means the last valid signal generated was either bullish (`1`) or neutral (`0`).

*   **Mathematical Constants & Their Significance:**
    *   **`k_neighbors` (Default: 5):** Controls the number of historical data points used for voting. A smaller `k` makes the model highly sensitive to local patterns but more susceptible to noise (higher variance). A larger `k` provides a smoother, more stable prediction but may generalize too much and miss nuanced setups (higher bias).
    *   **`window_size` (Default: 400):** Defines the lookback period for the historical "training" data. This is the database of patterns the model learns from. A larger window provides more data but increases computational load and risks including data from irrelevant market regimes.
    *   **`prob_threshold` (Default: 0.9):** This is the single most critical risk management and signal quality parameter. A value of 0.9 requires that the weighted consensus of the `k` nearest neighbors is 90% confident in a directional outcome. This aggressively filters for high-conviction setups, directly trading signal frequency for theoretical precision.
    *   **`momentum_window` (Default: 4):** Defines the prediction target for the supervised learning labels. The model is explicitly trained to predict whether a new high (for a bull case) or new low (for a bear case) will be made within the next 4 bars. This constant directly sets the intended time horizon of the predictive signals.
    *   **`p_param` (Default: 2.0):** The exponent for the Minkowski distance metric. A value of `2.0` makes it the standard **Euclidean distance** (straight-line distance in multi-dimensional space). A value of `1.0` would make it the **Manhattan distance** (sum of absolute differences along each axis). This parameter subtly changes how "similarity" between two market states is geometrically defined.
    