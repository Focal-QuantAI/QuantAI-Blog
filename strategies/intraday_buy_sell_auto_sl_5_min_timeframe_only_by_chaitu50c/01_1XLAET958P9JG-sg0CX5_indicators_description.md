
    # Indicators Description

    ### 1. Component Deconstruction

The script's engine is built exclusively on raw price and time data, eschewing traditional lagging indicators for a pure price action approach.

#### **Time-Based Components**

*   **Timeframe Quantizer:** The script's core timing mechanism is designed to partition the 5-minute chart into discrete 15-minute blocks.
    *   **Specific Configuration:** The logic is hard-coded to a 15-minute interval using the constant `15 * 60 * 1000` (milliseconds in 15 minutes).
    *   **Mathematical Logic:** The expression `math.floor(time / (15 * 60 * 1000)) * 15 * 60 * 1000` calculates the Unix timestamp for the start of the current 15-minute block. It effectively "snaps" the current bar's time to the most recent 15-minute boundary (e.g., 09:15:00, 09:30:00, 09:45:00).
*   **Block Indexer:** This component identifies the position of a 5-minute candle within its 15-minute block.
    *   **Specific Configuration:** The index is calculated relative to the 5-minute timeframe (`tfMin = timeframe.multiplier` which is 5).
    *   **Mathematical Logic:** `idxInBlk = math.round((time - blockStartMs) / (tfMin * 60 * 1000)) + 1` calculates an index of 1, 2, or 3. It finds the elapsed milliseconds since the block started and divides by the number of milliseconds in a 5-minute bar to determine its ordinal position. The `+ 1` converts it from a zero-based to a one-based index.

#### **Price Action Components**

*   **Candle Body Direction:** The script transforms raw `open` and `close` data into boolean flags representing the direction of the candle's body.
    *   **Specific Configuration:** It analyzes the current bar (`close`, `open`), the previous bar (`close[1]`, `open[1]`), and the bar two periods prior (`close[2]`, `open[2]`).
    *   **Functional Modification:**
        *   `c0g = close[2] > open[2]` // True if candle 2 bars ago was green (bullish body)
        *   `c1r = close[1] < open[1]` // True if candle 1 bar ago was red (bearish body)
        *   And so on for `c2r`, `c0r`, `c1g`, `c2g`. This is a form of data quantization, converting continuous price data into a binary state (bullish/bearish) for pattern recognition.

#### **Volatility & Risk Component**

*   **Pattern-Based Range:** The script does not use a statistical volatility measure like ATR. Instead, it defines risk based on the absolute price range of the three-candle pattern itself.
    *   **Specific Configuration:** It uses the `high` and `low` of the three most recent bars.
    *   **Mathematical Logic:**
        *   `blkLow = math.min(low[2], low[1], low)`: This function finds the absolute lowest price point reached during the three-candle sequence. This value serves as the stop-loss level for a buy signal.
        *   `blkHigh = math.max(high[2], high[1], high)`: This function finds the absolute highest price point reached during the three-candle sequence. This serves as the stop-loss level for a sell signal.
    *   **Intended Effect:** This creates a structurally-defined stop-loss. The risk on any given trade is directly proportional to the volatility expressed during the formation of the setup pattern, rather than a longer-term average.

### 2. Logic Layering & Confluence

The script employs a strict hierarchical filtering system to achieve a high signal-to-noise ratio. A signal is only generated if a sequence of progressively finer conditions is met.

*   **Hierarchical Filtering:**
    1.  **Primary Gatekeeper (Timeframe):** The variable `is5min = timeframe.period == '5'` acts as the master switch. If the chart is not on a 5-minute timeframe, the entire execution engine is bypassed.
    2.  **Secondary Gatekeeper (Timing):** The condition `blockComplete = is5min and idxInBlk % 3 == 0` is the next filter. It ensures the logic only executes on the close of the *third* and final candle of a 15-minute block. This imposes a rigid timing discipline, preventing analysis during the pattern's formation.
    3.  **Tertiary Filter (Pattern Structure):** The script then checks for a specific sequence of candle colors. For a buy signal, it requires `c0r and c1g and c2g` (Red-Green-Green). This is the core pattern recognition layer.
    4.  **Final Confirmation (Breakout):** The last condition is a threshold cross. For a buy, `close > high[2]` confirms that the momentum of the third candle was strong enough to break the high of the first candle in the pattern. This acts as the final validation of the reversal's strength.

*   **Interaction Dynamics:**
    *   **Confluence:** The engine is a pure confluence model. A `buySignal` or `sellSignal` can only be `true` if the Timeframe, Timing, Pattern, and Breakout filters are all satisfied simultaneously on the same bar.
    *   **Threshold Cross:** The breakout condition (`close > high[2]` or `close < low[2]`) is a dynamic threshold cross. The threshold (`high[2]` or `low[2]`) is not a fixed value but is set by the price action of the pattern itself, making the trigger adaptive to recent volatility.

### 3. The Execution Engine

The trigger and exit mechanics are defined with mathematical precision, leaving no room for ambiguity.

#### **Trigger Conditions**

*   **Boolean Logic (Buy Signal):** A buy signal is generated when the following expression evaluates to `true`:
    `showSignals and blockComplete and c0r and c1g and c2g and close > high[2]`
    *   `showSignals`: User input must be enabled.
    *   `blockComplete`: It must be the close of the third 5-minute bar in a 15-minute block.
    *   `c0r`: The candle 2 bars ago must have been bearish.
    *   `c1g`: The candle 1 bar ago must have been bullish.
    *   `c2g`: The current candle must be bullish.
    *   `close > high[2]`: The closing price of the current candle must be greater than the high of the candle 2 bars ago.

*   **Boolean Logic (Sell Signal):** A sell signal is generated when the following expression evaluates to `true`:
    `showSignals and blockComplete and c0g and c1r and c2r and close < low[2]`
    *   `showSignals`: User input must be enabled.
    *   `blockComplete`: It must be the close of the third 5-minute bar in a 15-minute block.
    *   `c0g`: The candle 2 bars ago must have been bullish.
    *   `c1r`: The candle 1 bar ago must have been bearish.
    *   `c2r`: The current candle must be bearish.
    *   `close < low[2]`: The closing price of the current candle must be less than the low of the candle 2 bars ago.

#### **Exit Conditions (Stop-Loss Management)**

*   **Stop-Loss Placement:** The stop-loss is not calculated with a multiplier. It is placed directly at the extreme of the three-candle pattern.
    *   For a `buySignal`, the stop-loss is set at `blkLow`.
    *   For a `sellSignal`, the stop-loss is set at `blkHigh`.
*   **Stop-Loss Trigger:** The exit is triggered by a simple price cross.
    *   For a long position, the stop is hit if `close <= lvl` (where `lvl` is the `blkLow` at the time of entry).
    *   For a short position, the stop is hit if `close >= lvl` (where `lvl` is the `blkHigh` at the time of entry).
*   **Mathematical Constants:**
    *   `3`: Used in `idxInBlk % 3 == 0`, this hard-codes the pattern length to exactly three bars.
    *   `15`: Used in the time block calculation, this hard-codes the analysis window to 15 minutes.
    *   `10`: The `titleOffset` constant is purely for visual formatting of the stop-loss label on the chart and has no bearing on the mathematical logic of the trade engine.
    *   **Risk-to-Reward Profile:** The absence of any multipliers (e.g., 1.5x ATR) means the initial risk is defined absolutely by the pattern's height. The script does not define a take-profit mechanism, implying that profit-taking is discretionary or managed by an external system. The defined risk is therefore the price distance from the entry `close` to the pattern's extreme (`blkLow` or `blkHigh`).
    