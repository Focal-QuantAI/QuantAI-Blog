
# Indicators Description

### 1. Component Deconstruction

#### **Non-Repainting Pivots (Custom ZigZag)**
*   **Core Functions:** `ta.pivothigh(source, leftbars, rightbars)` and `ta.pivotlow(source, leftbars, rightbars)`.
*   **Specific Configuration:**
    *   `source`: `high` for pivots high, `low` for pivots low.
    *   `leftbars`: `pivot_len` = 5.
    *   `rightbars`: `pivot_len` = 5.
    *   A pivot high is confirmed only when a bar's `high` is greater than the `high` of the 5 preceding and 5 succeeding bars. This introduces a 5-bar lag, as the script must wait for 5 bars to close to the right of a potential pivot before confirming it.
*   **Functional Modification (Non-Repainting Logic):**
    *   The script uses `var` declared variables (`last_high`, `prev_high`, `last_low`, `prev_low`) to create a persistent state machine for trend detection.
    *   When `ta.pivothigh` returns a valid price (not `na`), it signifies a confirmed pivot from 5 bars ago. The script then updates its state: the existing `last_high` is shifted to `prev_high`, and the newly confirmed pivot value becomes the new `last_high`.
    *   **Intended Effect:** This architecture transforms the repainting nature of `ta.pivothigh` into a non-repainting, lagging trend structure. The trend is defined by the relationship between the two most recently *confirmed* pivots (`last_low > prev_low` for an uptrend), providing a robust, albeit delayed, assessment of market structure.

#### **Support & Resistance Levels**
*   **Core Functions:** `ta.lowest(source, length)` and `ta.highest(source, length)`.
*   **Specific Configuration:**
    *   `source`: `low` for support, `high` for resistance.
    *   `length`: 10 bars.
    *   `offset`: `[1]`. The calculation is performed on the *previous* 10 bars, excluding the current, developing bar.
*   **Functional Modification:** These are not used as direct entry or exit triggers. They serve as reference points for the ATR-based proximity filter. They define the immediate "danger zones" where a trade might face immediate opposition.

#### **Average True Range (ATR) Proximity Filter**
*   **Core Function:** `ta.atr(length)`.
*   **Specific Configuration:**
    *   `length`: 14 periods. This is a standard lookback for volatility measurement.
*   **Functional Modification:** The raw ATR value is multiplied by a constant to create a dynamic buffer zone.
    *   **Mathematical Logic:** `sr_distance = atr * 0.5`. This calculates a distance equivalent to 50% of the current 14-period ATR.
    *   **Intended Effect:** This creates a volatility-adjusted "no-trade zone" around the 10-bar support and resistance levels. In volatile markets, the ATR is larger, thus the buffer zone expands, requiring price to be further away from S/R to trigger a signal. In quiet markets, the zone contracts. This dynamically adjusts the risk profile by preventing entries with insufficient room for price to move before hitting a potential reversal point.

#### **Strong Candle Filter**
*   **Core Function:** Custom boolean logic.
*   **Specific Configuration:**
    *   **Bullish:** `(close - open) > (high - low) * 0.5`. The candle's body must be larger than 50% of its total range (from high to low).
    *   **Bearish:** `(open - close) > (high - low) * 0.5`. Same logic for a bearish candle.
*   **Functional Modification:** This is a custom-built filter to quantify "conviction." It isolates candles that demonstrate strong directional pressure and filters out indecisive price action (e.g., Dojis, spinning tops) where the body is small relative to the wicks. It improves the signal-to-noise ratio by ensuring the pattern components are based on decisive moves, not random fluctuations.

#### **Breakout Filter**
*   **Core Functions:** `ta.highest(source, length)` and `ta.lowest(source, length)`.
*   **Specific Configuration:**
    *   `source`: `high` for upside, `low` for downside.
    *   `length`: 5 bars.
    *   `offset`: `[1]`. The comparison is against the highest high or lowest low of the *previous* 5 bars.
*   **Functional Modification:** This acts as an immediate momentum confirmation. The condition `high > ta.highest(high, 5)[1]` ensures that the entry candle is not just a strong candle, but one that is actively breaking a short-term price ceiling, confirming buyer aggression at the point of signal generation.

---

### 2. Logic Layering & Confluence

The script's engine is a hierarchical filter cascade where each layer must be passed before the next is evaluated. This creates a high-confluence setup designed to minimize false signals.

*   **Interaction Dynamics:** The primary dynamic is **Confluence**. The engine does not look for divergences; it demands that trend, pattern, and momentum align in the same direction.

*   **Hierarchical Filtering:**
    1.  **State Filter (Macro-Trend):** The non-repainting pivot structure (`trend_up` / `trend_down`) is the highest-level gatekeeper. It defines the permissible trade direction. If the market structure is not showing confirmed higher lows (`trend_up = false`), the entire long-side logic is bypassed, regardless of any bullish patterns.
    2.  **Pattern Filter (Micro-Sequence):** Once the macro-trend is confirmed, the engine looks for the specific three-bar sequence:
        *   **Long:** `bull[2]` (initial push) -> `bear[1]` (pullback) -> `bull` (resumption).
        *   This is a pattern recognition module that identifies the "shakeout and re-engagement" narrative.
    3.  **Confirmation Filter (Immediate Momentum):** The `breakout_up` / `breakout_down` condition acts as the subsequent check. The three-bar pattern may be present, but the script waits for the final candle to break the immediate 5-bar range. This confirms that the resumption move has enough force to establish a new short-term high/low.
    4.  **Risk Management Filter (Veto Power):** The `not near_resistance` / `not near_support` condition is the final gatekeeper. It has veto power over the entire signal. If all other conditions are met, but the entry price is within the ATR-defined buffer of a recent S/R level, the signal is suppressed. This layer prioritizes trade location and initial risk-to-reward potential over the signal pattern itself.

---

### 3. The Execution Engine

The trigger is a boolean `true` value returned only when a precise set of conditions are met on a confirmed, closed bar.

#### **Long Signal (`up_signal`)**
*   **Boolean Logic:** A `true` signal is generated if and only if all the following conditions are met simultaneously:
    1.  `is_1m`: The chart is on the 1-minute timeframe.
    2.  `barstate.isconfirmed`: The signal is evaluated only on the close of the bar, preventing intra-bar repainting.
    3.  `trend_up`: The two most recently confirmed pivot lows are forming a higher low (`last_low > prev_low`).
    4.  `bull[2]`: The bar two periods ago was a "strong" bullish candle (body > 50% of range).
    5.  `bear[1]`: The previous bar was a "strong" bearish candle (the pullback).
    6.  `bull`: The current, closing bar is a "strong" bullish candle (the resumption).
    7.  `breakout_up`: The high of the current bar is greater than the highest high of the previous 5 bars.
    8.  `not near_resistance`: The closing price is *not* within 0.5 * ATR(14) of the 10-bar high.

#### **Short Signal (`down_signal`)**
*   **Boolean Logic:** The logic is a mirror image of the long signal:
    1.  `is_1m`: True.
    2.  `barstate.isconfirmed`: True.
    3.  `trend_down`: The two most recently confirmed pivot highs are forming a lower high (`last_high < prev_high`).
    4.  `bear[2]`: The bar two periods ago was a "strong" bearish candle.
    5.  `bull[1]`: The previous bar was a "strong" bullish candle (the counter-trend bounce).
    6.  `bear`: The current, closing bar is a "strong" bearish candle.
    7.  `breakout_down`: The low of the current bar is less than the lowest low of the previous 5 bars.
    8.  `not near_support`: The closing price is *not* within 0.5 * ATR(14) of the 10-bar low.

#### **Mathematical Constants**
*   **`pivot_len = 5`:** Defines the sensitivity of the trend filter. A value of 5 requires a significant swing to be confirmed, filtering out minor chop and focusing on more established structural shifts. It imposes a 5-bar lag on trend confirmation.
*   **`10` (S/R Lookback):** Defines a very short-term horizon for support and resistance. This is appropriate for a 1-minute scalping strategy, as it focuses only on the most immediate price obstacles.
*   **`0.5` (ATR Multiplier):** This is a critical risk management constant. A value of `0.5` creates a moderately sized "no-trade zone."
    *   **Influence on R:R:** By forcing entries to occur away from immediate resistance, it ensures there is, at minimum, a predefined amount of "clear air" for the trade to move into profit before hitting a likely point of friction. This mechanically attempts to improve the initial risk-to-reward profile of each setup. A higher multiplier would increase this effect at the cost of fewer signals.
    