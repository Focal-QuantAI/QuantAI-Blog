
# Indicators Description

### 1. Component Deconstruction

The script's engine is built from first-principle statistical measures rather than traditional technical indicators.

*   **Logarithmic Returns (`ret`)**
    *   **Calculation:** `math.log(close / close[1])`
    *   **Purpose:** This computes the continuously compounded return for each bar. Using log returns is standard in quantitative analysis as it makes returns time-additive and normalizes price changes, forming the fundamental data point for all subsequent calculations.

*   **"Prior" Mean Return (`p_mu`)**
    *   **Indicator:** Simple Moving Average (SMA).
    *   **Specific Configuration:**
        *   **Source:** Logarithmic Returns (`ret`).
        *   **Length:** `lookback_p` (default: 252 bars).
    *   **Functional Role:** Calculates the arithmetic mean of log returns over a long-term window (approximating one trading year). This serves as the "prior belief" or the historical baseline for the asset's expected return (drift).

*   **"Prior" Variance (`p_var`)**
    *   **Indicator:** Statistical Variance.
    *   **Specific Configuration:**
        *   **Source:** Logarithmic Returns (`ret`).
        *   **Length:** `lookback_p` (default: 252 bars).
    *   **Functional Role:** Measures the dispersion (volatility) of log returns around the `p_mu` over the long-term window.

*   **"Prior" Precision (`p_prec`)**
    *   **Indicator:** Custom Mathematical Operation.
    *   **Calculation:** `p_var > 0 ? 1 / p_var : 0`
    *   **Functional Modification:** This is a non-standard, "hacked" component that calculates the mathematical precision, defined as the reciprocal of variance. The ternary operator (`? :`) is a safeguard against division-by-zero errors.
    *   **Intended Effect:** To quantify the "credibility" or "confidence" in the long-term mean return (`p_mu`). A low-variance (stable) history yields high precision, while a high-variance (erratic) history yields low precision.

*   **"Evidence" Mean Return (`s_mu`)**
    *   **Indicator:** Simple Moving Average (SMA).
    *   **Specific Configuration:**
        *   **Source:** Logarithmic Returns (`ret`).
        *   **Length:** `lookback_e` (default: 60 bars).
    *   **Functional Role:** Calculates the mean of log returns over a shorter-term window (approximating one trading quarter). This represents the "new evidence" of the asset's recent performance.

*   **"Evidence" Variance (`s_var`)**
    *   **Indicator:** Statistical Variance.
    *   **Specific Configuration:**
        *   **Source:** Logarithmic Returns (`ret`).
        *   **Length:** `lookback_e` (default: 60 bars).
    *   **Functional Role:** Measures the volatility of log returns around the `s_mu` over the short-term window.

*   **"Evidence" Precision (`s_prec`)**
    *   **Indicator:** Custom Mathematical Operation.
    *   **Calculation:** `s_var > 0 ? 1 / s_var : 0`
    *   **Functional Modification:** Identical in logic to `p_prec`, but applied to the short-term "Evidence" data.
    *   **Intended Effect:** To quantify the confidence in the recent mean return (`s_mu`). A stable, low-volatility recent trend will be assigned a high precision value.

### 2. Logic Layering & Confluence

The script layers its statistical components to derive a single, risk-adjusted position sizing factor (`f_bayes`). This process is a form of hierarchical filtering.

*   **Interaction Dynamics: Precision-Weighted Confluence**
    *   The core of the logic is the formula `p_prec * p_mu + s_prec * s_mu`. This is not a simple average; it is a **precision-weighted average** of the long-term ("Prior") and short-term ("Evidence") mean returns.
    *   This mechanism dynamically adjusts the influence of each lookback period. If recent performance is stable and consistent (low `s_var`, high `s_prec`), the short-term mean (`s_mu`) will dominate the final calculation. Conversely, if recent performance is volatile and noisy (high `s_var`, low `s_prec`), its influence is automatically down-weighted, and the more stable long-term mean (`p_mu`) carries more weight.
    *   The engine is therefore looking for a **Confluence** of positive returns, but it intelligently filters the signal-to-noise ratio by giving more credence to the less volatile period.

*   **Hierarchical Filtering:**
    1.  **Primary Filter (Bayesian Blend):** The precision-weighted average (`p_prec * p_mu + s_prec * s_mu`) is the first layer, creating a blended forecast of expected return.
    2.  **Directional Gatekeeper:** The `math.max(..., 0)` function acts as a hard gate. It enforces a long-only bias by flooring any negative expected return at zero. If the blended forecast is negative, the target leverage becomes zero, effectively filtering out all bearish signals.
    3.  **Risk Management Filter (Kelly Fraction):** The result is then multiplied by `kelly_frac` (default 0.5). This acts as a risk-scaling filter, systematically reducing the position size suggested by the raw model to account for estimation errors and reduce portfolio volatility.
    4.  **Leverage Ceiling:** Finally, `math.min(..., max_lever)` serves as the ultimate risk-control gatekeeper. It caps the final calculated leverage at a user-defined maximum (`max_lever`, default 2.0), preventing the strategy from taking on unbounded risk, regardless of how strong the signal is.

### 3. The Execution Engine

The execution logic is systematic and periodic, governed by a single temporal trigger.

*   **Target Position Sizing:**
    *   `target = strategy.equity * f_bayes / close`: This formula translates the abstract leverage factor (`f_bayes`) into a concrete target quantity of the asset. It determines the total capital to allocate (`strategy.equity * f_bayes`) and divides by the asset's price to find the number of units to hold.
    *   `diff = target - strategy.position_size`: This calculates the delta between the desired position and the current position. This value is the exact quantity to be transacted.

*   **Boolean Logic & Trigger Condition:**
    *   **The Catalyst:** The entire execution block is wrapped in the condition `if bar_index % interval == 0`. The modulo operator (`%`) ensures that the logic only runs on a periodic basis (e.g., every 7th bar if `interval` is 7). This is the sole trigger for rebalancing.
    *   **Condition 1 (Increase Position):**
        *   **Logic:** `if diff > 0`
        *   **Action:** `strategy.order("Expand", strategy.long, diff)`
        *   **Execution:** If the calculated target position is greater than the current holding, a market order is sent to buy the exact difference (`diff`), thus increasing the position to match the target.
    *   **Condition 2 (Decrease Position):**
        *   **Logic:** `else if diff < 0`
        *   **Action:** `strategy.order("Reduce", strategy.short, -diff)`
        *   **Execution:** If the calculated target position is less than the current holding, a market order is sent to sell the absolute difference (`-diff`). This reduces the position size. The use of `strategy.short` in this context is Pine Script's syntax for placing a sell order to reduce or close a long position.

*   **Mathematical Constants and Risk Profile:**
    *   `interval = 7`: This hard-coded number defines the rebalancing frequency. It acts as a temporal filter, reducing transaction costs and preventing reactions to high-frequency market noise. It fundamentally impacts the strategy's lag and responsiveness.
    *   `max_lever = 2`: This constant imposes a strict ceiling on the maximum capital at risk, directly constraining the strategy's maximum potential gain and loss. It is a non-negotiable risk parameter.
    *   `kelly_frac = 0.5`: This multiplier directly governs the strategy's risk appetite. A value of 0.5 ("Half Kelly") creates a more conservative risk-to-reward profile than the full Kelly Criterion would, aiming for a less volatile equity curve by systematically reducing bet size.
    