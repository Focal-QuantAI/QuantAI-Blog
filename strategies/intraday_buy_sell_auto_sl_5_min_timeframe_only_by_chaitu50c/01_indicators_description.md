
# Indicators Description

### 1. Component Deconstruction

The script's engine is built upon custom, price-action-derived components rather than standard library indicators.

*   **Component 1: Temporal Block Filter**
    *   **Specific Configuration:** This component is hard-coded to operate exclusively on a **5-minute (`'5'`) timeframe**. It uses millisecond epoch time (`time`) to mathematically partition the trading session into discrete **15-minute blocks**.
    *   **Functional Modification:** This is a non-standard timing mechanism designed to enforce a specific analytical cadence.
        *   **Block Calculation:** `blockStartMs = math.floor(time / (15 * 60 * 1000)) * 15 * 60 * 1000` calculates the start time of the current 15-minute block by using integer division on the bar's timestamp.
        *   **Bar Indexing:** `idxInBlk = math.round((time - blockStartMs) / (tfMin * 60 * 1000)) + 1` determines if the current 5-minute bar is the first, second, or third within its 15-minute block.
        *   **Trigger Condition:** The variable `blockComplete` evaluates to `true` only on the close of the third 5-minute bar in the sequence (`idxInBlk % 3 == 0`). This acts as the primary gatekeeper for the entire logical engine, ensuring analysis only occurs once the full three-candle "narrative" has concluded.

*   **Component 2: Three-Bar Reversal Pattern Analyzer**
    *   **Specific Configuration:** This component analyzes a fixed lookback period of three bars (`[2]`, `[1]`, and the current bar `[0]`). It uses the `open` and `close` of each bar as its price source.
    *   **Functional Modification:** This is a custom pattern recognition module that defines two specific sequences:
        *   **Bullish Reversal Pattern:** A red candle followed by two consecutive green candles (`c0r and c1g and c2g`).
        *   **Bearish Reversal Pattern:** A green candle followed by two consecutive red candles (`c0g and c1r and c2r`).
        The component's output is a simple boolean state indicating whether one of these two rigid patterns has formed over the preceding three bars.

*   **Component 3: Structural Breakout Validator**
    *   **Specific Configuration:** This component uses the `close` of the current bar (`[0]`) and compares it against the `high` or `low` of the first bar in the three-bar sequence (`high[2]` or `low[2]`).
    *   **Functional Modification:** This is not a standard volatility breakout (e.g., ATR or Donchian Channel). It is a *pattern-relative* breakout. Its function is to validate the momentum of the reversal pattern.
        *   **Buy Validation:** `close > high[2]` requires the closing price of the third candle to break above the high of the *initial* bearish candle. This confirms that buyers have not only absorbed selling pressure but have established a new, higher price territory relative to the pattern's origin.
        *   **Sell Validation:** `close < low[2]` requires the close to break below the low of the initial bullish candle, confirming seller dominance.

*   **Component 4: Dynamic Stop-Loss Calculator**
    *   **Specific Configuration:** This component uses the `low` and `high` prices from all three bars in the signal pattern (`[2]`, `[1]`, `[0]`).
    *   **Functional Modification:** The stop-loss is not a fixed value or a multiple of a volatility metric. It is dynamically calculated based on the price structure of the setup itself.
        *   **Buy Signal Stop:** `blkLow = math.min(low[2], low[1], low)` sets the stop-loss at the absolute lowest point of the three-bar formation.
        *   **Sell Signal Stop:** `blkHigh = math.max(high[2], high[1], high)` sets the stop-loss at the absolute highest point of the three-bar formation.
        This method intrinsically links the initial risk of the trade to the volatility expressed during the pattern's formation.

### 2. Logic Layering & Confluence

The script employs a strict, hierarchical filtering system to maximize its signal-to-noise ratio. One component acts as a gatekeeper for the next, creating a cascade of conditions that must be met in sequence.

*   **Interaction Dynamics:** The core of the engine relies on **Confluence**. A signal is only generated when the *Three-Bar Reversal Pattern* occurs in confluence with a *Structural Breakout*. The script does not look for divergences.

*   **Hierarchical Filtering:**
    1.  **Primary Gatekeeper (Temporal Filter):** The `blockComplete` variable is the master switch. The entire execution logic is dormant until the close of the third 5-minute bar within a 15-minute window. This is the first and most rigid filter, eliminating all intra-block noise.
    2.  **Secondary Filter (Pattern Recognition):** If the temporal condition is met, the script then checks for the presence of the required three-candle color sequence (e.g., Red-Green-Green for a buy). If this pattern is not present, the logic chain is broken, and no signal is considered.
    3.  **Tertiary Filter (Momentum Validation):** Only if both the temporal and pattern filters are passed does the script evaluate the final condition: the *Structural Breakout*. The `close > high[2]` (for a buy) or `close < low[2]` (for a sell) acts as the definitive confirmation that the pattern has been validated by sufficient momentum to warrant a trade.

This layered approach ensures that the script is not just trading a simple candle pattern but a pattern that occurs at a specific cadence and is validated by a decisive price thrust.

### 3. The Execution Engine

The trigger and exit mechanisms are defined by precise boolean logic and structure-derived price levels.

*   **Trigger Conditions (Boolean Logic):**
    *   **`buySignal`:** A `true` signal is returned if and only if all the following conditions are met on the same bar:
        1.  `showSignals == true` (Input enabled)
        2.  `timeframe.period == '5'` (Chart is on the 5-minute timeframe)
        3.  `idxInBlk % 3 == 0` (It is the 3rd bar of a 15-minute block)
        4.  `close[2] < open[2]` (The first bar of the block was red)
        5.  `close[1] > open[1]` (The second bar of the block was green)
        6.  `close > open` (The current bar is green)
        7.  `close > high[2]` (The current close is above the high of the first bar)

    *   **`sellSignal`:** A `true` signal is returned if and only if all the following conditions are met on the same bar:
        1.  `showSignals == true`
        2.  `timeframe.period == '5'`
        3.  `idxInBlk % 3 == 0`
        4.  `close[2] > open[2]` (The first bar of the block was green)
        5.  `close[1] < open[1]` (The second bar of the block was red)
        6.  `close < open` (The current bar is red)
        7.  `close < low[2]` (The current close is below the low of the first bar)

*   **Exit Conditions (Stop-Loss):**
    *   The exit is not a dynamic trailing stop but a fixed stop-loss determined at the time of entry. The script maintains an array of active trades (`slActive`).
    *   For each active trade, the exit condition is checked on every bar close:
        *   **For a Buy Trade:** `hit = close <= lvl` (where `lvl` is the `blkLow` calculated at entry).
        *   **For a Sell Trade:** `hit = close >= lvl` (where `lvl` is the `blkHigh` calculated at entry).
    *   There is no take-profit logic; the exit is solely managed by the initial stop-loss level being breached.

*   **Mathematical Constants & Risk Profile:**
    *   **`15` / `3` / `5`:** These hard-coded numbers define the strategy's entire operational rhythm: evaluate a **3**-bar pattern on a **5**-minute chart every **15** minutes. They are fundamental to its design.
    *   **`[2]` (Lookback Index):** The choice to use `high[2]` and `low[2]` for the breakout validation is a critical design decision. It forces the breakout to be more significant than a simple one-bar breakout, requiring it to overcome the price level established at the beginning of the pattern. This increases the robustness of the signal.
    *   **Risk-to-Reward Influence:** The script does not use a fixed risk multiplier (like ATR). The risk is defined by the price range of the three-bar setup (`blkHigh - blkLow`). A volatile, wide-ranging setup pattern results in a larger stop-loss distance (higher initial risk), while a tight, low-volatility pattern results in a smaller stop-loss distance (lower initial risk). The R:R profile is therefore entirely dependent on the market conditions at the time of the signal.
