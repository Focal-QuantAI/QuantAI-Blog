
# Indicators Description

### 1. Component Deconstruction

#### **Pivot Low (`ta.pivotlow`)**
*   **Specific Configuration:**
    *   **Price Source:** `low` (implicit in the function).
    *   **Lookback Periods:** The `leftbars` and `rightbars` arguments are both set to the user-defined `pivLen` input, which defaults to `5`.
*   **Functional Modification:**
    *   This is a standard, unmodified implementation of the `pivotlow` function. Its purpose is purely structural: to identify a price low that is strictly lower than the lows of the `pivLen` preceding bars and the `pivLen` succeeding bars. The script stores the price level (`pivot`) and bar index (`pindex`) of the most recent confirmed pivot low for later use as a breakdown threshold.

#### **Adaptive Simple Moving Average (SMA) Band**
This is a composite, non-standard indicator constructed from three distinct components: an SMA, an ATR, and an adaptive length mechanism.

*   **A. Simple Moving Average (`ta.sma`)**
    *   **Specific Configuration:**
        *   **Price Source:** `high`. This is a critical choice, as it positions the average above the main body of the candlesticks, creating a natural ceiling.
        *   **Length (`len`):** This is the core of the adaptive mechanic. The length is not fixed. It is initialized at `10` upon a bullish trend confirmation and then increments by `1` on every subsequent bar that remains in a bearish state (`not direction`). This growth is capped by the `smaMax` input (default `50`).
    *   **Functional Modification:** The SMA's lookback period is dynamic. As a downtrend persists, the SMA's length increases, causing it to have more **lag** and become less sensitive to recent price action. This intentionally "raises the bar" for a bullish reversal, as a longer-term average requires a more substantial and sustained price move to be crossed.

*   **B. Average True Range (`ta.atr`)**
    *   **Specific Configuration:**
        *   **Length:** Hard-coded at `50`. This provides a smoothed, longer-term measure of market volatility.
    *   **Functional Modification:** The ATR is not used as a standalone volatility gauge. Its output is multiplied by the `smaMult` input (default `1.0`) and then added directly to the adaptive SMA. This transforms the SMA line into the lower boundary of a **volatility band**.

*   **C. The Composite Band (`sma`)**
    *   **Mathematical Formula:** `sma = ta.sma(high, len) + (ta.atr(50) * smaMult)`
    *   **Intended Effect:** This composite indicator creates a dynamic, adaptive resistance ceiling that is only active during a bearish trend. The band automatically widens in volatile markets (due to the ATR component) and becomes progressively harder to cross as the downtrend matures (due to the increasing SMA length). It is designed to filter out minor relief rallies and only signal a potential trend change on a high-momentum breakout.

### 2. Logic Layering & Confluence

The script's engine is built on a foundation of **hierarchical filtering** governed by a single boolean state variable: `direction`. This variable acts as a master switch, determining which set of rules the script is currently enforcing. The system is always in one of two mutually exclusive states.

*   **State 1: Bullish Regime (`direction == true`)**
    *   **Active Logic:** The script's sole focus is the detection of a structural breakdown. The adaptive SMA band logic is completely dormant.
    *   **Gatekeeper:** The `ta.pivotlow` function acts as the gatekeeper. It continuously identifies potential support levels. The trigger condition is a confirmed close *below* the most recently established `pivot` level.
    *   **Interaction Dynamics:** This is a pure **Threshold Cross** model. There is no confluence required with other indicators. The logic dictates that a break of a confirmed price structure is, by itself, a sufficient signal to flip the market state to bearish.

*   **State 2: Bearish Regime (`direction == false`)**
    *   **Active Logic:** The script activates the adaptive SMA band calculation. The `len` variable begins incrementing on each bar, increasing the SMA's lag.
    *   **Gatekeeper:** The adaptive SMA band (`sma`) is the gatekeeper for re-entering a bullish state. The pivot low detection continues, but a break of it has no logical effect, as the trend is already considered bearish.
    *   **Interaction Dynamics:** This is also a **Threshold Cross** model, but the threshold is dynamic and adaptive. The crossover of the `sma` band represents a confluence of:
        1.  **Price Action:** `close` moving above the band.
        2.  **Momentum:** The move must be strong enough to overcome a resistance level that has been systematically desensitized over time.
        3.  **Volatility:** The move must also exceed the volatility buffer provided by the `atr * smaMult` component.

This asymmetrical layering ensures the script is never looking for bullish and bearish signals simultaneously, dramatically reducing logical conflicts and improving the signal-to-noise ratio.

### 3. The Execution Engine

The script's execution is defined by two distinct state-flipping triggers. These are not "entry/exit" signals in the traditional sense, but rather the definitive moments that change the script's entire operational mode.

#### **Bearish State Trigger (Trend Initiation)**
*   **Boolean Logic:** A bearish state is triggered when the following conditions are met simultaneously:
    1.  `direction == true` (The script must currently be in a bullish regime).
    2.  `ta.crossunder(close, pivot)` (The closing price of the bar moves below the stored price level of the last confirmed pivot low).
    3.  `barstate.isconfirmed == true` (The signal is only valid on the close of the bar, preventing intra-bar whipsaws).
*   **Resulting Action:** The `direction` variable is set to `false`.

#### **Bullish State Trigger (Trend Reversal)**
*   **Boolean Logic:** A bullish state is triggered when the following conditions are met simultaneously:
    1.  `direction == false` (The script must currently be in a bearish regime).
    2.  `ta.crossover(close, sma)` (The closing price of the bar moves above the calculated adaptive SMA band).
    3.  `barstate.isconfirmed == true` (The signal is only valid on the close of the bar).
*   **Resulting Action:** The `direction` variable is set to `true`, and the adaptive length `len` is reset to `10`.

#### **Mathematical Constants & Their Influence**
*   **`pivLen` (Default: 5):** This constant directly controls the **significance** of the support structure. A higher value (e.g., 15) will identify fewer, but more structurally important, pivot lows, leading to fewer but higher-conviction bearish signals. A lower value increases sensitivity to minor swings.
*   **`smaMax` (Default: 50):** This defines the **maximum lag** or "inertia" of the downtrend resistance band. It sets the upper limit on how "mature" a downtrend can become before the script stops making it harder for a reversal to occur. It effectively caps the difficulty of a bullish trigger.
*   **`smaMult` (Default: 1.0):** This is a direct **volatility sensitivity multiplier**. It controls the risk profile of the bullish trigger.
    *   `smaMult > 1.0`: Increases the ATR offset, pushing the band further from price. This requires a more powerful momentum move to trigger a bullish reversal, filtering for higher-conviction signals at the cost of potentially later entries.
    *   `smaMult < 1.0`: Decreases the ATR offset, making the band easier to cross. This increases sensitivity and leads to earlier signals, but with a higher probability of capturing false breakouts (noise).
*   **`len` Reset Value (Hard-coded: 10):** Upon a bullish flip, the adaptive length is reset to 10. This is higher than the `smaMin` of 5. This choice ensures that immediately following a reversal, the resistance band (if it were to be calculated) would not be hyper-sensitive, providing a small buffer against an immediate failed rally.
    