
# Indicators Description

### 1. Component Deconstruction

This section provides a granular analysis of each mathematical and logical component used in the script's engine.

#### **Core Indicators & Studies**

*   **Average True Range (ATR)**
    *   **Specific Configuration:** `ta.atr(atrLen)` with a default `atrLen` of `9`.
    *   **Functional Modification:** Standard implementation. It calculates the average volatility over the last 9 periods. Its primary function is to dynamically size the stop-loss distance based on recent market volatility.

*   **Simple Moving Average (SMA) of Volume**
    *   **Specific Configuration:** `ta.sma(volume, 20)`.
    *   **Functional Modification:** Standard implementation. It computes a 20-period SMA of volume, establishing a baseline for "average" volume. This is used as a benchmark to identify high-volume, high-conviction trigger candles.

*   **Relative Strength Index (RSI)**
    *   **Specific Configuration:** `ta.rsi(close, 14)`.
    *   **Functional Modification:** Standard 14-period RSI based on `close` price. It is not used as a primary trigger but as an optional filter to detect price-momentum divergences.

*   **Exponential Moving Average (EMA)**
    *   **Specific Configuration:** `ta.ema(close, 200)`.
    *   **Functional Modification:** Standard 200-period EMA. It is not part of the entry logic but is used post-entry within the "Confluence Meter" to score the trade's alignment with the long-term trend.

*   **Bollinger Bands (BB)**
    *   **Specific Configuration:** `ta.bb(close, 20, 2.0)`.
    *   **Functional Modification:** The script exclusively extracts the middle band (`bbMid`), which is a 20-period SMA of the `close` price. The upper and lower bands are discarded. This component serves as an optional, short-term trend filter.

#### **Custom-Engineered Components**

*   **Premium & Discount (PD) Zones**
    *   **Specific Configuration:** Utilizes a `pdLookback` of `100` periods. It identifies the `highest(high)` and `lowest(low)` over this lookback period.
    *   **Functional Modification:** This is a custom calculation that defines a trading range.
        *   `rangeHigh = ta.highest(high, 100)`
        *   `rangeLow = ta.lowest(low, 100)`
        *   `equilibrium = (rangeHigh + rangeLow) / 2`
        *   The engine defines a "Discount" zone as any price below the `equilibrium` and a "Premium" zone as any price above it. This is a non-standard, range-based value filter derived from Smart Money Concepts.

*   **Market Structure Shift (MSS) Engine**
    *   **Specific Configuration:** Employs two distinct lookback periods: `internalLookback` (default `9`) and `swingLookback` (default `50`).
    *   **Functional Modification:** This is a custom-built pivot detection and break-of-structure engine.
        1.  **Pivot Identification:** It identifies swing points using the logic `high[N] == ta.highest(high, N * 2 + 1)`, where `N` is the lookback. This confirms a candle's high is the absolute highest in a window of `2N+1` bars. A similar logic applies to swing lows.
        2.  **State-Holding:** It uses `var` variables (`lastISH`, `lastISL`, `lastSSH`, `lastSSL`) to store the price level of the *most recent* valid swing high or low.
        3.  **Shift Detection:** The core "shift" signal (`mssL` or `mssS`) is generated when the `close` price executes a `ta.crossover` or `ta.crossunder` of these stored swing point levels. The `structureMode` input determines whether to use the faster "Internal" structure, the slower "Swing" structure, or both.

*   **Fair Value Gap (FVG) Detector**
    *   **Specific Configuration:** This is a fixed 3-bar pattern recognition module.
    *   **Functional Modification:** It uses pure boolean logic to identify price inefficiencies.
        *   **Bullish FVG (`bFVG`):** `low > high[2]`. This identifies a gap between the current bar's low and the high from two bars prior. The condition `close[1] > high[2]` is an additional qualifier, ensuring the middle candle of the pattern showed strength.
        *   **Bearish FVG (`sFVG`):** `high < low[2]`. This identifies a gap between the current bar's high and the low from two bars prior. The condition `close[1] < low[2]` adds a similar confirmation for bearish momentum.

*   **Volume Momentum Check**
    *   **Specific Configuration:** Iterates over the last `volCandles` (default `3`).
    *   **Functional Modification:** This is a custom loop designed to validate volume strength. The variable `volIncreasing` remains `true` only if for the last 3 bars, `volume[i] >= volume[i+1]`. This confirms a pattern of non-decreasing volume leading into the trigger candle, indicating building momentum.

### 2. Logic Layering & Confluence

The script's engine is built on a hierarchical filtering model where a primary signal is qualified by a series of subsequent conditions.

*   **Hierarchical Filtering:** The logic operates as a sequential gatekeeping system.
    1.  **Primary Gatekeeper (The Event):** The **Market Structure Shift (MSS)** is the foundational trigger. No signal can be generated without a `mssL` (for longs) or `mssS` (for shorts) event occurring first. This is the script's core signal for a change in order flow.
    2.  **Secondary Gatekeeper (The Location):** The **Premium & Discount Zone** acts as the next filter. If `requirePDZone` is enabled, a bullish MSS is only considered valid if it occurs while the price is `inDiscount`. A bearish MSS is only valid if it occurs `inPremium`. This layer filters out trades in "expensive" areas, enforcing a "buy low, sell high" discipline.
    3.  **Tertiary Gatekeepers (The Catalysts):** Several optional filters provide final confirmation, creating a high-degree of confluence. These are all conjunctive (`AND`) conditions.
        *   **FVG Filter:** If `useFVGConfluence` is enabled, the MSS must occur concurrently with or one bar after a **Fair Value Gap**. This adds a liquidity-based rationale for the price move.
        *   **Divergence Filter:** If `useDivFilter` is enabled, a classic **RSI Divergence** must be present, signaling underlying momentum exhaustion of the prior micro-trend.
        *   **BB Filter:** If `useBBFilter` is enabled, the price must be above the 20-period SMA for longs (`close > bbMid`) or below it for shorts, acting as a simple trend-momentum check.

*   **Interaction Dynamics:** The primary dynamic is **Confluence**. The engine is not looking for divergence between indicators as a primary signal (unless the optional filter is on), but rather for a convergence of conditions: a structural break (`MSS`) happening in a value location (`PD Zone`) with a potential liquidity reason (`FVG`). Each enabled filter systematically increases the specificity of the setup, thereby aiming to improve the signal-to-noise ratio at the cost of signal frequency.

### 3. The Execution Engine

This section defines the precise boolean logic and mathematical constants that govern trade entry and risk management.

#### **Trigger Conditions**

*   **Long Entry (`bTrigger`):** A long signal is `true` if and only if all the following conditions are met:
    1.  `mssL` is true (a bullish Market Structure Shift has occurred).
    2.  AND (`requirePDZone` is false OR `inDiscount` is true).
    3.  AND (`useFVGConfluence` is false OR a bullish FVG exists on the current or prior bar).
    4.  AND (`useDivFilter` is false OR `bullDiv` is true).
    5.  AND (`useBBFilter` is false OR `close > bbMid`).
    6.  AND `strategy.position_size == 0` (no existing position).
    7.  AND `tradeDirection` is "Both" or "Long Only".

*   **Short Entry (`sTrigger`):** A short signal is `true` if and only if all the following conditions are met:
    1.  `mssS` is true (a bearish Market Structure Shift has occurred).
    2.  AND (`requirePDZone` is false OR `inPremium` is true).
    3.  AND (`useFVGConfluence` is false OR a bearish FVG exists on the current or prior bar).
    4.  AND (`useDivFilter` is false OR `bearDiv` is true).
    5.  AND (`useBBFilter` is false OR `close < bbMid`).
    6.  AND `strategy.position_size == 0` (no existing position).
    7.  AND `tradeDirection` is "Both" or "Short Only".

#### **Risk & Trade Management Mechanics**

*   **Mathematical Constants:**
    *   **`atrMult = 3.0`:** This multiplier defines the stop-loss width. The stop is placed `3.0 * ATR` away from the trigger bar's low (for longs) or high (for shorts). This relatively large multiplier creates a wide, volatility-adjusted stop designed to withstand market noise.
    *   **`tp1RR = 1.5`:** This constant sets the first take-profit target at a fixed **1.5:1 Risk-to-Reward ratio**. The profit target distance is calculated as `1.5 * (entry_price - stop_loss_price)`.
    *   **`volMult = 1.2`:** A trigger candle is classified as "STRONG" if its volume is greater than 120% of the 20-period volume SMA, adding a layer of volume confirmation.

*   **Exit Logic & Position Management:**
    1.  **Initial Stop:** An ATR-based stop is placed immediately upon entry.
    2.  **Partial Profit-Taking:** Upon reaching the `1.5R` target (`tp1`), the strategy exits **50%** of the position.
    3.  **Stop to Breakeven:** Critically, after the first partial profit is taken, the stop-loss for the remaining position is moved to the average entry price (`strategy.position_avg_price`). This immediately removes risk from the remainder of the trade.
    4.  **Trailing Stop:** After moving to breakeven, the stop-loss becomes a trailing stop. On each new bar, it is recalculated as `low - (atr * atrMult)` for longs. If this new value is higher than the current stop-loss (which is at breakeven or higher), the stop is trailed up, locking in profit while allowing the remainder of the position to run.
    