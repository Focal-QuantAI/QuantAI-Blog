
# Indicators Description

### 1. Component Deconstruction

This section dissects each technical component of the script, detailing its configuration and any custom mathematical logic.

*   **Non-Repainting Pivots (Zigzag Logic)**
    *   **Specific Configuration:** The core engine uses `ta.pivothigh(high, 5, 5)` and `ta.pivotlow(low, 5, 5)`.
        *   **Price Source:** `high` for pivot highs, `low` for pivot lows.
        *   **Lookback Periods:** The left and right lookback periods are both set to `5`. A pivot high is only confirmed after 5 subsequent bars have failed to make a new high. Symmetrically for a pivot low.
    *   **Functional Modification:** The script implements a non-repainting mechanism by storing confirmed pivot values in `var` variables (`last_high`, `prev_high`, `last_low`, `prev_low`). When the native `ta.pivothigh` or `ta.pivotlow` function returns a non-`na` value (indicating a pivot has just been confirmed `pivot_len` bars in the past), the script updates its historical state. This converts the repainting nature of the built-in functions into a stable, non-repainting series of the two most recent confirmed pivot points, which is essential for reliable backtesting and live signal generation.

*   **Trend Definition**
    *   **Specific Configuration:** This is a derived boolean (`trend_up`, `trend_down`) based on the non-repainting pivot values.
    *   **Functional Modification:** Trend is defined purely by the sequence of confirmed pivots.
        *   `trend_up` is `true` if the most recent confirmed pivot low (`last_low`) is greater than the previously confirmed pivot low (`prev_low`). This constitutes a higher low.
        *   `trend_down` is `true` if the most recent confirmed pivot high (`last_high`) is lower than the previously confirmed pivot high (`prev_high`). This constitutes a lower high.
        This is a classical Dow Theory definition of trend applied on a micro-scale.

*   **Support & Resistance Levels**
    *   **Specific Configuration:** `support = ta.lowest(low, 10)[1]` and `resistance = ta.highest(high, 10)[1]`.
        *   **Lookback Period:** 10 bars.
        *   **Price Source:** `low` for support, `high` for resistance.
        *   **Offset:** `[1]`.
    *   **Functional Modification:** The `[1]` offset is a critical design choice. It ensures that the S/R levels are calculated based on the 10 bars *preceding* the current, signal-generating bar. This prevents look-ahead bias by defining the S/R context *before* the entry signal is evaluated.

*   **ATR Proximity Filter**
    *   **Specific Configuration:** `atr = ta.atr(14)`. This is a standard 14-period Average True Range.
    *   **Functional Modification:** The ATR is not used for stop-loss calculation but as a dynamic buffer to define "nearness" to S/R.
        *   **Mathematical Logic:** `sr_distance = atr * 0.5`. A zone is created with a width of 50% of the current 14-period ATR value.
        *   **Application:** The condition `math.abs(close - support) < sr_distance` checks if the closing price is within this dynamic buffer zone of the support level (and symmetrically for resistance). This makes the filter adaptive; the "no-trade zone" expands in volatile markets and contracts in quiet ones, improving the signal-to-noise ratio.

*   **Strong Candle Filter**
    *   **Specific Configuration:** This is a custom boolean logic, not a standard indicator.
    *   **Functional Modification:** It quantifies a "strong" or "decisive" candle.
        *   **Mathematical Logic:** A bullish candle is "strong" if its body (`close - open`) is greater than 50% of its total range (`high - low`). A bearish candle is "strong" if its body (`open - close`) is greater than 50% of its total range.
        *   **Intended Effect:** This filters out candles of indecision, such as dojis and spinning tops, ensuring that the bars used in the pattern recognition represent significant momentum and not market noise.

*   **Breakout Confirmation**
    *   **Specific Configuration:** `breakout_up = high > ta.highest(high, 5)[1]` and `breakout_down = low < ta.lowest(low, 5)[1]`.
    *   **Functional Modification:** This functions as a short-term Donchian Channel breakout. It confirms renewed momentum by requiring the current bar's high (for longs) or low (for shorts) to exceed the price extreme of the previous 5 bars. The `[1]` offset ensures the breakout is relative to the *prior* range.

### 2. Logic Layering & Confluence

The script's engine achieves signal precision by stacking these components in a strict, hierarchical order. A signal is only generated if all layers of the filter return a permissive state.

*   **Interaction Dynamics:** The strategy is built on **Confluence** and **Hierarchical Filtering**. It does not use divergence. Every component must agree for a signal to be valid.

*   **Hierarchical Filtering:**
    1.  **Gatekeeper (Regime Filter):** The non-repainting pivot structure (`trend_up` / `trend_down`) acts as the highest-level filter. It first establishes the market's directional bias. If the trend condition is not met (e.g., `trend_up` is not `true`), all subsequent logic for a long signal is ignored.
    2.  **Pattern Filter:** The script then scans for a specific three-bar sequence using the **Strong Candle Filter**. For a long entry, it requires: Strong Bull Candle `[2]` -> Strong Bear Candle `[1]` -> Strong Bull Candle `[0]`. This identifies the "pullback-resumption" narrative. This is a form of pattern recognition based on **Threshold Crosses** (each candle must cross the 50% body-to-range threshold).
    3.  **Momentum Trigger:** The `breakout_up` / `breakout_down` condition serves as the final confirmation of momentum. The resumption candle must not only be strong but also powerful enough to break the immediate 5-bar price ceiling/floor.
    4.  **Risk Filter (Exclusionary Logic):** The `not near_resistance` / `not near_support` condition is the final check. It acts as a veto. Even if the trend, pattern, and momentum are perfectly aligned, the trade is blocked if the entry price is too close to a recent S/R level, preserving a viable risk-reward profile.

### 3. The Execution Engine

This section defines the precise boolean logic and mathematical constants that trigger an entry signal.

*   **Boolean Logic: `up_signal`**
    A `true` signal is returned only if the following conditions are met simultaneously on a confirmed bar close (`barstate.isconfirmed`):
    1.  `is_1m`: The chart timeframe is exactly 1 minute.
    2.  `trend_up`: The structural trend is up (last confirmed pivot low > previous pivot low).
    3.  `bull[2]`: The candle two bars ago was a strong bullish candle.
    4.  `bear[1]`: The candle one bar ago was a strong bearish candle (the pullback).
    5.  `bull`: The current, closing candle is a strong bullish candle (the resumption).
    6.  `breakout_up`: The high of the current candle broke above the highest high of the previous 5 candles.
    7.  `not near_resistance`: The closing price is not within the exclusionary zone (0.5 * ATR) of the 10-bar resistance level.

*   **Boolean Logic: `down_signal`**
    The logic is perfectly symmetrical for a short signal:
    1.  `is_1m`: The chart timeframe is exactly 1 minute.
    2.  `trend_down`: The structural trend is down (last confirmed pivot high < previous pivot high).
    3.  `bear[2]`: The candle two bars ago was a strong bearish candle.
    4.  `bull[1]`: The candle one bar ago was a strong bullish candle (the counter-trend bounce).
    5.  `bear`: The current, closing candle is a strong bearish candle (the resumption).
    6.  `breakout_down`: The low of the current candle broke below the lowest low of the previous 5 candles.
    7.  `not near_support`: The closing price is not within the exclusionary zone (0.5 * ATR) of the 10-bar support level.

*   **Mathematical Constants & Their Influence**
    *   **`pivot_len = 5`:** Defines the reactivity of the trend filter. A value of 5 on a 1M chart creates a sensitive micro-trend definition, ideal for scalping.
    *   **`S/R lookback = 10`:** Establishes a very short-term S/R horizon (10 minutes), focusing only on immediate price obstacles.
    *   **`ATR multiplier = 0.5`:** This is a key risk management constant. It dictates the "breathing room" required for a trade. A value of `0.5` provides a moderate filter, blocking trades that are entering directly into a potential reversal zone. Increasing this value would make the filter stricter, reducing signal frequency but potentially increasing the quality of the remaining signals.
    *   **`Strong Candle threshold = 0.5`:** The 50% body-to-range ratio is a strict definition of momentum. It ensures the pattern is composed of decisive price action, filtering out noise and indecision.
    *   **`Breakout lookback = 5`:** A 5-bar breakout is a confirmation of immediate momentum. It ensures the entry occurs as price is accelerating, not stalling.
    