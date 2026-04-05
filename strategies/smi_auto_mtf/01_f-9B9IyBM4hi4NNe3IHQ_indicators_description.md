
# Indicators Description

### 1. Component Deconstruction

The script's engine is built around a single, modified oscillating study: the Stochastic Momentum Index (SMI).

#### Stochastic Momentum Index (SMI)

*   **Specific Configuration:** The SMI's parameters are determined by a conditional logic layer.
    *   **Control Flag:** `i_override` (boolean). If `false` (default), the script uses automatic, timeframe-based parameters. If `true`, it uses the manual inputs.
    *   **Price Source:** The calculation is derived from `high`, `low`, and `close`. It does not use a single source like `close` or `hlc3`.
    *   **Automatic Parameters (Default):** The script queries the chart's `timeframe.period` and assigns parameters as follows:
        *   **15-minute (`"15"`):** K Period = 5, Smooth Period = 3, Signal Period = 3
        *   **1-hour (`"60"`):** K Period = 9, Smooth Period = 3, Signal Period = 3
        *   **4-hour (`"240"`):** K Period = 13, Smooth Period = 5, Signal Period = 5
        *   **Daily (`"D"`):** K Period = 13, Smooth Period = 5, Signal Period = 5
        *   **Weekly (`"W"`):** K Period = 20, Smooth Period = 7, Signal Period = 7
        *   **All Others:** K Period = 13, Smooth Period = 5, Signal Period = 5 (Fallback default)
    *   **Manual Parameters (Override):**
        *   `i_k`: K Period (default: 13)
        *   `i_smooth`: Smooth Period (default: 5)
        *   `i_signal`: Signal Period (default: 5)
    *   **Thresholds:**
        *   Overbought (`i_ob`): `40.0`
        *   Oversold (`i_os`): `-40.0`

*   **Functional Modification:** The script employs the William Blau version of the SMI, which deviates significantly from a standard Stochastic Oscillator.
    1.  **Stochastic Component:** Instead of calculating the close's position relative to the high-low range (like `%K`), it calculates the close's position relative to the *midpoint* of the high-low range (`diff = close - (hh + ll) / 2.0`). This centers the initial calculation around a zero point, measuring deviation from the mean price of the lookback period.
    2.  **Double Exponential Smoothing:** This is the key modification. A standard Stochastic uses a Simple Moving Average (SMA) for its `%D` signal line. The SMI applies a **double-smoothed Exponential Moving Average** (`ta.ema(ta.ema(value, period))`) to both the numerator (`diff`) and the denominator (`hl_range`).
        *   **Mathematical Effect:** Applying an EMA twice results in a significantly smoother output than a single EMA and vastly smoother than an SMA of the same period. This technique, known as Double Exponential Moving Average (DEMA) smoothing (though not the DEMA indicator itself), heavily reduces lag compared to two separate EMAs while aggressively filtering high-frequency noise.
        *   **Intended Effect:** This modification is designed to improve the signal-to-noise ratio. It deliberately sacrifices sensitivity to minor price fluctuations to produce a cleaner, more reliable momentum wave, making it better suited for identifying sustained momentum shifts rather than rapid, noisy oscillations.
    3.  **Normalization:** The final SMI value is calculated as `100.0 * s_diff / (0.5 * s_range)`. The `0.5` multiplier in the denominator effectively doubles the final value, scaling the output to a range that is typically bounded by -100 and +100, similar to other oscillators.

### 2. Logic Layering & Confluence

The script's logic is not based on a complex confluence of multiple indicators but rather on the internal state and behavior of the SMI itself.

*   **Interaction Dynamics:** The primary interaction is a **Threshold Cross** followed by a **Signal Line Crossover**.
    1.  **State Identification (The "Zone"):** The script first identifies a market state of momentum exhaustion. This is achieved when the `smi` line crosses a predefined threshold (`i_ob` or `i_os`). The `bgcolor` function visually confirms when the SMI is in this "actionable" zone. This is the first layer of filtering; the script is looking for a market that is statistically over-extended.
    2.  **Momentum Shift (The "Confirmation"):** The script does not trigger on entry into the zone. Instead, it waits for a definitive shift in momentum. This confirmation is provided by the `smi` line crossing its own EMA-based signal line (`sig`). This crossover event is the mechanical trigger, signifying that the rate of change of momentum has inflected.

*   **Hierarchical Filtering:**
    *   **Primary Filter (Implicit):** The automatic, multi-timeframe parameter adjustment acts as the highest level of filtering. By using longer, less sensitive lookback periods on higher timeframes (e.g., Weekly `k=20`), the script inherently filters out the noise of lower timeframes and focuses only on major, multi-week momentum cycles. Conversely, on a 15-minute chart (`k=5`), it becomes highly sensitive to capture short-term intraday mean reversion opportunities. This adaptive nature is the script's core filtering mechanism.
    *   **Secondary Filter (Explicit):** The overbought/oversold levels (`+40/-40`) act as a "gatekeeper" from a trader's perspective. While the script plots all crossovers (`bull_x`, `bear_x`), the intended use case described in the trade narrative implies that only crossovers occurring *while the SMI is within the OB/OS zones* are considered high-probability signals. The script provides the tools (`bgcolor`, `hline`) for a user to apply this filter visually or programmatically.

### 3. The Execution Engine

The script's triggers are discrete boolean events based on the SMI's behavior.

*   **Boolean Logic:**
    *   **Bullish Trigger (`bull_x`):** Defined by `ta.crossover(smi, sig)`. This condition evaluates to `true` for a single bar when the current `smi` value is greater than the current `sig` value, AND the previous bar's `smi` value was less than or equal to the previous bar's `sig` value. This marks the exact moment the momentum line overtakes its slower-moving average.
    *   **Bearish Trigger (`bear_x`):** Defined by `ta.crossunder(smi, sig)`. This condition evaluates to `true` for a single bar when the current `smi` value is less than the current `sig` value, AND the previous bar's `smi` value was greater than or equal to the previous bar's `sig` value. This marks the momentum line falling below its average.
    *   **Zone Entry Triggers (for Alerts):**
        *   `ta.crossover(smi, i_ob)`: Enters the overbought zone.
        *   `ta.crossunder(smi, i_os)`: Enters the oversold zone.

*   **Mathematical Constants:**
    *   **`40.0` / `-40.0`:** These are the default thresholds for overbought and oversold conditions. They define the "zone of interest" for mean reversion trades. Setting them at `40` instead of a more traditional `80/20` (like a standard Stochastic) is a direct consequence of the SMI's construction and normalization, which produces less extreme peaks and troughs. These values are critical as they define the level of momentum exhaustion the script considers significant.
    *   **`0.5`:** This constant in the SMI formula (`... / (0.5 * s_range)`) is a core component of William Blau's original specification. Its function is purely for scaling. By dividing by half the range, it normalizes the output to a wider, more readable +/- 100 range, amplifying the visual oscillation.
    *   **`0.0`:** The zero line is a crucial equilibrium point. A crossover of the zero line indicates a shift from positive (bullish) to negative (bearish) momentum, or vice-versa. It represents the point where the current price (`close`) is exactly at the midpoint of the high-low range over the smoothed lookback period.
    