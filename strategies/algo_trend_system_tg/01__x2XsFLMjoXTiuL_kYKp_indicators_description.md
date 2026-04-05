
# Indicators Description

### 1. Component Deconstruction

This section deconstructs the primary mathematical components of the "Algo Trend System TG" script.

#### **Supertrend Indicator**
*   **Specific Configuration:**
    *   **Function:** `ta.supertrend()`
    *   **ATR Lookback Period (`atrPeriod`):** `10`. This defines the period for the underlying Average True Range (ATR) calculation, which measures volatility.
    *   **ATR Multiplier (`factor`):** `3.0`. The calculated ATR value is multiplied by this factor to determine the offset of the trend bands from the price's median.
    *   **Price Source:** The `ta.supertrend` function internally uses `(high + low) / 2` as the median price from which the ATR-based offset is calculated.
*   **Functional Modification:**
    *   This is a standard, unmodified implementation of the Supertrend indicator. Its primary role is not as a standalone signal generator but as a tactical, volatility-adjusted trigger mechanism. The script does not use the plotted `supertrend` line for its logic, but rather the internal `direction` series returned by the function. This series holds a value of `-1` for a bullish trend and `1` for a bearish trend.

#### **Exponential Moving Average (EMA) Cloud**
*   **Specific Configuration:**
    *   **Function:** `ta.ema(source, length)`
    *   **Fast EMA (`emaF`):** A `50`-period EMA calculated from the `close` price.
    *   **Slow EMA (`emaS`):** A `150`-period EMA calculated from the `close` price.
*   **Functional Modification:**
    *   The two EMAs are not used as independent crossover signals. Instead, their relative position (`emaF > emaS` or `emaF < emaS`) is used to establish a persistent "state" or "regime." This transforms the EMAs from lagging indicators into a single, binary regime filter that defines the permissible trade direction over a macro timeframe. The visual `fill` between the two lines reinforces this concept of a continuous trend environment.

---

### 2. Logic Layering & Confluence

The script's intelligence derives from its hierarchical filtering process, where signals must pass through sequential logical gates to be considered valid.

*   **Hierarchical Filtering:** The engine employs a two-tier filtering architecture:
    1.  **Gatekeeper (Regime Filter):** The **EMA Cloud (50/150)** serves as the primary filter. It establishes the strategic directional bias. A buy signal is only considered if the market is in a bullish regime (`emaF > emaS`), and a sell signal is only considered in a bearish regime (`emaF < emaS`). This layer is designed to filter out all trades that run counter to the dominant, long-term trend, significantly reducing the number of potential signals.
    2.  **Trigger (Tactical Filter):** The **Supertrend (10, 3.0)** acts as the secondary, more sensitive filter. It is not used to define the overall trend but to identify a specific event: the resumption of momentum after a pullback.

*   **Interaction Dynamics:** The core logic is built on **Confluence** following a specific narrative: "a pullback to the mean and its subsequent rejection."
    *   **For a Long Trade:** The system does not trigger when the Supertrend is simply bullish. It waits for a precise sequence:
        1.  The EMA Cloud must be bullish (`emaF > emaS`).
        2.  The price must have pulled back sufficiently to flip the more sensitive Supertrend indicator to a bearish state.
        3.  The trigger event is the **resumption of strength**, identified by the exact bar on which the Supertrend flips *back* to a bullish state. This is captured by `ta.crossunder(direction, 0)`.
    *   This layering ensures the script only engages when short-term momentum (Supertrend) realigns with the long-term trend (EMA Cloud), aiming to capture the continuation move.

*   **Ancillary Filters:**
    *   **Temporal Filter:** An optional time filter (`inAllowedTime`) acts as an additional gate, allowing signals only within specified trading sessions. This reduces noise by focusing on periods of expected higher liquidity or volatility.
    *   **State Filter:** A state machine (`inTrade`, `tradeDirection`) prevents the system from taking a new trade in the same direction while one is already active, effectively enforcing a "one trade at a time" rule per direction.

---

### 3. The Execution Engine

This section details the precise boolean logic for trade execution and the mathematical constants governing risk management.

#### **Trigger Conditions**

*   **Buy Signal (`buySignal`):** A `true` value is returned if the following four conditions are met on the same bar:
    1.  `ta.crossunder(direction, 0)`: The internal Supertrend state variable crosses from `1` (bearish) to `-1` (bullish).
    2.  `emaF > emaS`: The 50-period EMA is above the 150-period EMA.
    3.  `inAllowedTime == true`: The bar's time is within a valid session (if enabled).
    4.  `tradeDirection != 1`: The system is not currently in a long trade.

*   **Sell Signal (`sellSignal`):** A `true` value is returned if the following four conditions are met on the same bar:
    1.  `ta.crossover(direction, 0)`: The internal Supertrend state variable crosses from `-1` (bullish) to `1` (bearish).
    2.  `emaF < emaS`: The 50-period EMA is below the 150-period EMA.
    3.  `inAllowedTime == true`: The bar's time is within a valid session (if enabled).
    4.  `tradeDirection != -1`: The system is not currently in a short trade.

#### **Exit Conditions & Mathematical Constants**

The script utilizes a fixed-target exit mechanism, not a dynamic or indicator-based one. All profit and loss levels are calculated from the `close` of the signal bar.

*   **Mathematical Constants:**
    *   **Stop Loss (`sl_dist`):** A fixed point/dollar value (default: `20.0`). This value is subtracted from a long entry's `close` or added to a short entry's `close`. The risk per trade is therefore static in terms of price points, not percentage or volatility.
    *   **Take Profits (`tp1_dist`, `tp2_dist`, `tp3_dist`):** Fixed point/dollar values (defaults: `10.0`, `20.0`, `30.0`). These are added to a long entry's `close` or subtracted from a short entry's `close`.

*   **Risk-to-Reward Profile:**
    *   The script's R:R is entirely dependent on the user-defined inputs. With default settings, the profile is as follows:
        *   **R:R to TP1:** 10:20 = **0.5:1**
        *   **R:R to TP2:** 20:20 = **1:1**
        *   **R:R to TP3:** 30:20 = **1.5:1**

*   **Trade Invalidation (for Statistical Tracking):**
    *   The internal dashboard logic tracks a single "win" condition based on the `closing_tp` input.
    *   A trade is marked as a **Loss** if price touches the `stopLoss` level (`low <= stopLoss` for longs; `high >= stopLoss` for shorts).
    *   A trade is marked as a **Win** if price touches the user-selected final take profit level (`high >= finalTPTarget` for longs; `low <= finalTPTarget` for shorts).
    *   This is a "first-touch" model; the simulation does not account for partial profits or complex trade management, treating the entire position as closed once one of these two levels is breached.
    