
# Indicators Description

### 1. Component Deconstruction

The script's engine is built upon raw price action data and a custom temporal filter, rather than traditional library indicators.

*   **Raw Price Data (`open`, `high`, `low`, `close`)**
    *   **Specific Configuration:** The script directly accesses the OHLC values of the current bar (`[0]`), the previous bar (`[1]`), and the bar two periods prior (`[2]`). This constitutes a fixed 3-bar lookback period.
    *   **Functional Modification:** There are no modifications to the price data itself. It is used in its raw form to define candlestick color and structural price points (highs/lows).

*   **Custom Temporal Filter (`blockComplete`)**
    *   **Specific Configuration:** This is a bespoke logical component designed to synchronize the script's execution with a 15-minute cycle, despite operating on a 5-minute chart.
        *   **Timeframe Multiplier:** `tfMin = timeframe.multiplier` (evaluates to `5` on a 5-min chart).
        *   **Timeframe Check:** `is5min = timeframe.period == '5'`. This hard-codes the logic to only function on the 5-minute timeframe.
        *   **Block Cadence:** The core calculation uses a mathematical constant of `15 * 60 * 1000` (milliseconds in 15 minutes) to establish a fixed 15-minute grid over the trading day.
    *   **Functional Modification:** This component acts as a non-standard timing mechanism. Its mathematical logic is as follows:
        1.  `blockStartMs = math.floor(time / (15 * 60 * 1000)) * 15 * 60 * 1000`: This calculation takes the current bar's UNIX timestamp (`time`), performs integer division by the number of milliseconds in 15 minutes, and multiplies it back. The effect is to "floor" or "snap" the current time to the start of the most recent 15-minute interval (e.g., 9:30:00, 9:45:00, 10:00:00).
        2.  `idxInBlk = math.round((time - blockStartMs) / (tfMin * 60 * 1000)) + 1`: This determines the bar's position within the 15-minute block. It calculates the elapsed time since the block started and divides by the duration of a single 5-minute bar, resulting in an index of 1, 2, or 3.
        3.  `blockComplete = is5min and idxInBlk % 3 == 0`: The final boolean is `true` only if the script is on a 5-minute chart AND the bar is the third and final bar of the 15-minute block. This effectively throttles the execution logic to once every 15 minutes.

*   **Structural Extremes (`blkLow`, `blkHigh`)**
    *   **Specific Configuration:** These values are derived to define the stop-loss levels.
        *   `blkLow = math.min(math.min(low[2], low[1]), low)`
        *   `blkHigh = math.max(math.max(high[2], high[1]), high)`
    *   **Functional Modification:** This is a custom calculation that finds the absolute lowest low or highest high of the 3-bar pattern. It is not a volatility-based measure (like ATR) but a purely structural one, defining the risk boundary as the pattern's total price range.

### 2. Logic Layering & Confluence

The script employs a strict hierarchical filtering system to achieve a high signal-to-noise ratio. It does not use indicator convergence or divergence but relies on a sequential confluence of time, pattern, and price breakout.

*   **Interaction Dynamics:** The engine's primary dynamic is **Confluence**. A valid signal is only generated when three distinct conditions are met simultaneously: a temporal condition, a pattern condition, and a breakout condition.

*   **Hierarchical Filtering:** The logic is layered in a specific, non-negotiable order, with each layer acting as a "Gatekeeper" for the next.
    1.  **Primary Gatekeeper (Temporal Filter):** The `blockComplete` variable is the master filter. The entire signal generation logic is encapsulated within this check. If the current bar is not the third 5-minute bar of a 15-minute block, no further analysis occurs. This drastically reduces noise by forcing the script to evaluate a complete 15-minute "narrative" rather than reacting to intra-bar noise.
    2.  **Secondary Filter (Pattern Recognition):** If the temporal gate is passed, the script then checks for a specific 3-bar candlestick color sequence.
        *   For a **Buy Signal:** Red-Green-Green (`c0r and c1g and c2g`).
        *   For a **Sell Signal:** Green-Red-Red (`c0g and c1r and c2r`).
        This layer filters for the specific "failed push and reversal" pattern the strategy is designed to capture.
    3.  **Tertiary Filter (Breakout Confirmation):** If the pattern is confirmed, the final and most critical filter is applied. This is a **Threshold Cross**.
        *   For a **Buy Signal:** The close of the third candle must be greater than the high of the first candle (`close > high[2]`).
        *   For a **Sell Signal:** The close of the third candle must be less than the low of the first candle (`close < low[2]`).
        This confirms that the reversal has immediate momentum and has broken the micro-structure established by the initial candle of the pattern.

### 3. The Execution Engine

The trigger and exit conditions are defined by precise boolean logic and structural price points.

*   **Trigger Logic (Buy Signal)**
    A `buySignal` returns `true` if and only if all the following conditions are met on the same bar:
    *   `showSignals == true`: The user has enabled signals in the settings.
    *   `blockComplete == true`: It is the closing tick of the third 5-minute bar in a 15-minute sequence.
    *   `close[2] < open[2]`: The candle two bars ago was bearish (Red).
    *   `close[1] > open[1]`: The candle one bar ago was bullish (Green).
    *   `close > open`: The current candle is bullish (Green).
    *   `close > high[2]`: The current close has broken above the high of the first candle in the 3-bar sequence.

*   **Trigger Logic (Sell Signal)**
    A `sellSignal` returns `true` if and only if all the following conditions are met on the same bar:
    *   `showSignals == true`: The user has enabled signals in the settings.
    *   `blockComplete == true`: It is the closing tick of the third 5-minute bar in a 15-minute sequence.
    *   `close[2] > open[2]`: The candle two bars ago was bullish (Green).
    *   `close[1] < open[1]`: The candle one bar ago was bearish (Red).
    *   `close < open`: The current candle is bearish (Red).
    *   `close < low[2]`: The current close has broken below the low of the first candle in the 3-bar sequence.

*   **Exit Logic (Stop-Loss Hit)**
    The exit is not a pattern-based reversal but a simple stop-loss breach.
    *   **Boolean Logic:** The condition `hit = isB ? close <= lvl : close >= lvl` is evaluated on every bar for each active trade.
        *   If the trade is a buy (`isB == true`), a `hit` occurs if the bar's `close` is less than or equal to the pre-calculated stop-loss level (`lvl`).
        *   If the trade is a sell (`isB == false`), a `hit` occurs if the bar's `close` is greater than or equal to the pre-calculated stop-loss level (`lvl`).
    *   **Stop-Loss Level (`lvl`):** This level is not based on a multiplier (like ATR). It is a hard structural level defined at the moment of entry:
        *   For a buy, it is the absolute lowest low of the 3-bar trigger pattern (`blkLow`).
        *   For a sell, it is the absolute highest high of the 3-bar trigger pattern (`blkHigh`).
        This ties the risk profile directly to the volatility expressed during the formation of the signal pattern itself.

*   **Mathematical Constants**
    *   **`3` (in `idxInBlk % 3 == 0`):** This hard-codes the length of the pattern to exactly three bars. It defines the fixed window of analysis.
    *   **`15 * 60 * 1000`:** This constant (900,000 milliseconds) defines the script's decision-making cycle. It forces the 3-bar pattern to resolve within a 15-minute window, making it a core component of the strategy's temporal discipline.
    *   **`10` (in `titleOffset`):** This is a purely cosmetic constant that dictates the horizontal offset of the stop-loss label from the current bar, measured in bars. It has no impact on signal generation or risk management but affects visual presentation.
