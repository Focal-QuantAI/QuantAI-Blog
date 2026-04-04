
# Indicators Description

### 1. Component Deconstruction

#### **A. Support/Resistance (SR) Retest Module**
*   **Core Function:** `ta.highest(high, lookback)` and `ta.lowest(low, lookback)`
*   **Specific Configuration:**
    *   **Lookback Period:** `lookback = 50` bars.
    *   **Price Source:** `high` and `low`.
    *   **Confirmation Bars:** `confirmBars = 2` bars.
*   **Functional Modification:** The module employs a historical offset via `[confirmBars]`. The plotted support (`srLow_fixed`) and resistance (`srHigh_fixed`) levels are not the highest/lowest values of the most recent 50 bars. Instead, they are the highest/lowest values from the 50-bar window that concluded **two bars ago**.
    *   **Mathematical Logic:** `srHigh_fixed` on the current bar is equal to the value of `ta.highest(high, 50)` from two bars prior.
    *   **Intended Effect:** This creates static, non-repainting price levels for the current and subsequent bar to interact with. By "fixing" the S/R levels from two bars ago, the script establishes a stable benchmark, preventing the S/R lines from shifting if the current bar makes a new high or low within the lookback period. This is critical for the "touch and reject" logic, as it provides a constant value to test against.

#### **B. Bollinger Bands (BB)**
*   **Core Function:** `ta.sma()` and `ta.stdev()`
*   **Specific Configuration:**
    *   **Length:** `bb_length = 15` bars.
    *   **Price Source:** `close`.
    *   **Standard Deviation Multiplier:** `bb_mult = 2.0`.
*   **Functional Modification:** This is a standard, unmodified implementation of Bollinger Bands.
    *   **Basis:** A 15-period Simple Moving Average of the closing price.
    *   **Deviation:** The 15-period standard deviation of the closing price, multiplied by 2.0.
    *   **Bands:** The upper and lower bands are calculated by adding/subtracting the deviation from the SMA basis.

#### **C. Quantum Trend Flow (QTF) Channel**
*   **Core Function:** A custom-built volatility channel.
*   **Specific Configuration:**
    *   **Channel Length:** `qtf_len = 20` bars.
    *   **Volatility Multiplier:** `qtf_mult = 2.0`.
    *   **Price Source:** `close`.
*   **Functional Modification:** This is a non-standard, "hacked" volatility channel that synthesizes two distinct volatility metrics.
    *   **Basis:** `ta.ema(close, 20)`. A 20-period Exponential Moving Average serves as the channel's centerline, providing a faster-reacting trend baseline than the BB's SMA.
    *   **Mathematical Logic (Deviation):** The channel width is determined by a composite volatility value: `(ta.atr(20) + (2.0 * ta.stdev(close, 20))) / 2`.
        1.  `ta.atr(20)`: The 20-period Average True Range measures volatility based on the full high-low range of the candles.
        2.  `2.0 * ta.stdev(close, 20)`: A 20-period, 2.0 multiplier Standard Deviation measures volatility based on the dispersion of closing prices around the mean.
    *   **Intended Effect:** By averaging ATR and Standard Deviation, the channel creates a hybrid volatility measurement. It is sensitive to both the intraday price range (from ATR) and the closing price volatility (from STDEV). This aims to create a more robust and adaptive channel that captures different facets of market volatility compared to a single-metric indicator like Bollinger Bands.

#### **D. Multi-Timeframe (MTF) Trend Filter**
*   **Core Function:** `request.security()`
*   **Specific Configuration:**
    *   **Higher Timeframe:** `mtf_res = "15"` (15-minute chart by default).
    *   **Trend Indicator:** A 50-period Exponential Moving Average (`ta.ema(close, 50)`).
*   **Functional Modification:** This is a standard implementation of an MTF trend filter.
    *   **Logic:** It fetches the 15-minute closing price and the 15-minute 50 EMA. The trend direction (`mtf_trend_dir`) is set to `1` (bullish) if the 15-minute close is above its 50 EMA, and `-1` (bearish) otherwise. This value is then used as a gatekeeper for trade signals.

### 2. Logic Layering & Confluence

The script's engine is built on a strict, hierarchical filtering system designed to identify high-probability mean-reversion setups by demanding a confluence of conditions.

*   **Interaction Dynamics:** The core dynamic is **Confluence of Threshold Crosses**. The engine does not look for divergences or complex patterns. Its primary function is to detect the single-bar event where price pierces a trifecta of support or resistance boundaries.

*   **Hierarchical Filtering:** The logic is layered to progressively increase signal quality and reduce noise.
    1.  **Layer 1: The Confluence Zone:** The script first establishes three distinct support/resistance zones:
        *   **Static Price Zone:** The `srHigh_fixed` / `srLow_fixed` levels.
        *   **Statistical Deviation Zone:** The `bb_upper` / `bb_lower` bands.
        *   **Hybrid Volatility Zone:** The `qtf_upper` / `qtf_lower` channel.
    2.  **Layer 2: The "Touch & Reject" Filter:** This is the foundational filter and the heart of the strategy. It requires a two-part condition on a single candle:
        *   **The Touch (`buyTouch`/`sellTouch`):** The `low` of the candle must pierce *all three* support boundaries, or the `high` must pierce *all three* resistance boundaries. This confirms the price overextension.
        *   **The Reject (`buyReject`/`sellReject`):** The `close` of that same candle must be back inside *all three* of the corresponding boundaries. This confirms the failure of the move and the immediate reversal pressure. A candle that pierces all three but closes outside even one of them will not generate a signal.
    3.  **Layer 3: The "Gatekeeper" Trend Filter:** The optional MTF filter (`mtf_check_long`/`mtf_check_short`) acts as a strategic overlay. If enabled, it functions as a master gatekeeper. A valid "Touch & Reject" signal will be discarded if it does not align with the broader trend on the higher timeframe (e.g., a BUY signal will be blocked if the 15-minute trend is bearish).
    4.  **Layer 4: The Temporal Filter:** The `last_signal` variable acts as a simple state machine to prevent signal spam. It ensures that a new `BUY` signal cannot be generated if the immediately preceding signal was also a `BUY`, and vice-versa for `SELL` signals. This forces the script to alternate between long and short signals, improving chart clarity.

### 3. The Execution Engine

#### **A. Boolean Logic (Trigger Conditions)**

The final trigger for a signal is a boolean `true` value returned by `finalLong` or `finalShort`.

*   **Long Signal (`finalLong`):** A `true` signal is generated if and only if all four of the following conditions are met on the same bar:
    1.  `low <= srLow_fixed AND low <= bb_lower AND low <= qtf_lower`
    2.  `close > srLow_fixed AND close > bb_lower AND close > qtf_lower`
    3.  `mtf_check_long == true` (This condition is always true if the MTF filter is disabled).
    4.  `last_signal != 1` (The previous signal was not a BUY).

*   **Short Signal (`finalShort`):** A `true` signal is generated if and only if all four of the following conditions are met on the same bar:
    1.  `high >= srHigh_fixed AND high >= bb_upper AND high >= qtf_upper`
    2.  `close < srHigh_fixed AND close < bb_upper AND close < qtf_upper`
    3.  `mtf_check_short == true` (This condition is always true if the MTF filter is disabled).
    4.  `last_signal != -1` (The previous signal was not a SELL).

#### **B. Mathematical Constants (Risk Profile)**

The script does not contain programmatic exit logic (e.g., `strategy.exit`). Instead, it visualizes a potential trade outcome using the "Dynamic RR Box" feature, which is governed by hard-coded percentage-based inputs.

*   **Stop Loss:** `rr_sl_pct = 0.5%`
*   **Take Profit:** `rr_tp_pct = 1.0%`

*   **Significance:**
    *   These constants define a fixed, non-adaptive risk-to-reward profile. By default, the script suggests a **1:2 Risk-to-Reward Ratio** (risking 0.5% to gain 1.0%).
    *   The Stop Loss and Take Profit levels are calculated as a static percentage of the entry price (`close` of the signal bar). This means the risk is not adjusted for market volatility. A 0.5% stop will represent a different number of points/pips depending on the asset's price and will not widen during periods of high ATR, unlike a volatility-based stop. This design choice prioritizes a consistent R:R structure over a volatility-adaptive one. The `TradeTracker` object and array manage the drawing of these boxes, extending them bar-by-bar until the `high` or `low` of a subsequent candle touches the calculated TP or SL price level.
    