
# Indicators Description

### 1. Component Deconstruction

This section deconstructs the individual mathematical and logical components that form the foundation of the `RSI Elite Toolkit`.

#### **Relative Strength Index (RSI)**

*   **Specific Configuration:**
    *   **Price Source:** `input.source(close, "RSI Source")` - Defaults to the bar's closing price.
    *   **Length:** `input.int(21, "RSI Period")` - A lookback period of 21 bars, slightly longer than the common 14-period default, aiming to smooth the oscillator and focus on more established momentum shifts.
*   **Functional Modification:**
    *   **Custom Function:** The RSI is calculated within a dedicated function `calc_rsi()`. This is a standard implementation using `ta.rma` (Wilder's Moving Average) for smoothing the `up` and `down` components, which is the correct mathematical definition of the RSI. It is not a "hacked" version but rather a clean encapsulation.
    *   **Non-Repainting MTF Engine:** The script's primary modification is its Multi-Timeframe (MTF) capability. When `enable_mtf` is true, it uses `request.security()` to fetch the RSI value from a higher timeframe (`mtf_timeframe`). Crucially, it employs `barmerge.lookahead_off` to prevent the engine from using future data, ensuring the signals are non-repainting and historically accurate.

#### **RSI Signal Smoothing (Moving Average of RSI)**

*   **Specific Configuration:**
    *   **Type:** `input.string("SMA", "Smoothing Type", options = ["SMA", "EMA", "WMA", "RMA"])` - A user-selectable moving average type, defaulting to a Simple Moving Average (SMA).
    *   **Length:** `input.int(14, "Smoothing Period")` - A 14-period lookback.
*   **Functional Modification:**
    *   This is not a moving average of price, but a moving average of the `rsi_val` itself. Its purpose is to act as a "signal line" for the RSI oscillator. The mathematical effect is a second layer of smoothing applied to the momentum calculation. Crossovers between the RSI and its own moving average are used to generate signals with reduced lag compared to price crossing a moving average, but with a better signal-to-noise ratio than raw RSI threshold crosses.

#### **Institutional Bias Filter (Exponential Moving Average)**

*   **Specific Configuration:**
    *   **Price Source:** Hard-coded to `close`.
    *   **Length:** `input.int(200, "Bias Period")` - A fixed 200-period EMA.
*   **Functional Modification:**
    *   This is a standard 200 EMA applied to the main chart's price. It is not mathematically modified. Its function is purely contextual, serving as a binary filter to define the macro market structure (Bullish if `close > 200 EMA`, Bearish if `close < 200 EMA`).

#### **Volatility Matrix (Average True Range - ATR)**

*   **Specific Configuration:**
    *   **Length:** `input.int(14, "Volatility Matrix (ATR)")` - A standard 14-period ATR.
*   **Functional Modification:**
    *   This is a standard ATR calculation. It is not used as a signal generator but as a core component of the risk management engine. Its value provides a dynamic, volatility-adjusted unit of measurement for calculating Stop Loss and Take Profit distances.

#### **Divergence Intelligence Engine**

*   **Specific Configuration:**
    *   **Pivot Lookback:** `div_pivot_left` (5 bars), `div_pivot_right` (5 bars) - Defines the window for identifying a pivot high/low on the RSI. A pivot low, for example, is a point where the RSI is lower than the 5 bars before it and the 5 bars after it.
    *   **Analysis Range:** `div_max_range` (60 bars), `div_min_range` (5 bars) - Constrains the search for a previous pivot. A divergence is only considered valid if the distance between the two pivots is between 5 and 60 bars.
*   **Functional Modification:**

    *   This is a custom-built logical engine, not a standard indicator. Its mechanics are as follows:
        1.  **Pivot Identification:** It uses `ta.pivotlow()` and `ta.pivothigh()` on the `rsi_val` to locate significant turning points in momentum.
        2.  **State Capture:** Upon finding a pivot, `ta.valuewhen()` is used to capture the RSI value and the corresponding `low` or `high` price at that exact bar. It stores both the most recent pivot (`[0]`) and the one prior (`[1]`).
        3.  **Distance Validation:** `ta.barssince()` calculates the number of bars between the two pivots. This distance is checked against `div_min_range` and `div_max_range` to filter out divergences that are either too close together (noise) or too far apart (irrelevant).
        4.  **Logical Matrix:** It applies the classic definitions of divergence:
            *   **Regular Bullish:** `price_pl < price_pl_prev` (Lower Low in Price) AND `rsi_pl > rsi_pl_prev` (Higher Low in RSI).
            *   **Regular Bearish:** `price_ph > price_ph_prev` (Higher High in Price) AND `rsi_ph < rsi_ph_prev` (Lower High in RSI).

### 2. Logic Layering & Confluence

The script's primary innovation is its departure from single-trigger logic in favor of a weighted, score-based confluence engine. It does not use a strict hierarchical filter but rather assigns evidentiary weight to different market phenomena.

*   **Interaction Dynamics:** The engine synthesizes signals from three distinct types of events:
    1.  **Threshold Crosses:** The RSI crossing above the `level_os` (Oversold) or below the `level_ob` (Overbought) threshold. This is the most basic form of mean-reversion signal.
    2.  **Crossovers:** The `rsi_val` crossing its own `smoothing_ma`. This is a momentum-shift signal.
    3.  **Divergences:** The sophisticated price-momentum decoupling identified by the Divergence Engine. This is a momentum-exhaustion signal.

*   **Hierarchical Filtering (via Weighted Scoring):** The script achieves filtering not by a sequence of gates, but by assigning points to each condition. This creates a flexible hierarchy where a high-value event can trigger a signal on its own, while lower-value events must occur in combination.
    *   **Divergence (3 points):** Given the highest weight, as it represents a significant structural failure in momentum. A confirmed divergence is the most powerful single component.
    *   **RSI Extreme Exit (2 points):** A strong confirmation that short-term pressure is abating.
    *   **Trend Bias (1 point):** The 200 EMA acts as a contextual modifier. A trade aligned with the macro trend is considered higher probability and is awarded a point.
    *   **RSI MA Cross (1 point):** The lowest weight, representing an early, and potentially noisy, shift in momentum.

A signal is generated only when the cumulative `long_score` or `short_score` meets or exceeds the `min_confluence` threshold (defaulting to 3). This means a signal can be triggered by `Divergence` alone (3 pts), or by `RSI Extreme Exit` + `Trend Bias` (2+1=3 pts), or by `RSI Extreme Exit` + `RSI MA Cross` (2+1=3 pts).

### 3. The Execution Engine

#### **Trigger Conditions**
*   **Boolean Logic:** The final trigger is a simple Boolean comparison:
    *   `buy_signal = long_score >= min_confluence`
    *   `sell_signal = short_score >= min_confluence`
*   The script also includes a state machine (`var int active_state`) which prevents new signals from being generated while a trade visualization is already active on the chart (`active_state != 0`). A new signal can only trigger if `active_state == 0`.

#### **Exit Conditions & Mathematical Constants**

The script does not generate dynamic exit signals. Instead, it projects a static Stop Loss (SL) and Take Profit (TP) zone based on volatility at the time of the trigger.

*   **Mathematical Constants:**
    *   `sl_multiplier`: A float (default 3.0) that scales the ATR value to determine the stop loss distance.
    *   `tp_multiplier`: A float (default 8.0) that scales the ATR value to determine the take profit distance.

*   **Risk-to-Reward Profile:** The script's design hard-codes a specific risk-to-reward ratio based on the input multipliers.
    *   **Stop Loss Calculation:**
        *   Long: `entry_px - (ta.atr(atr_period) * sl_multiplier)`
        *   Short: `entry_px + (ta.atr(atr_period) * sl_multiplier)`
    *   **Take Profit Calculation:**
        *   Long: `entry_px + (ta.atr(atr_period) * tp_multiplier)`
        *   Short: `entry_px - (ta.atr(atr_period) * tp_multiplier)`
    *   **Implied R:R:** With default settings, the script visualizes a trade with a fixed Risk:Reward ratio of `tp_multiplier / sl_multiplier` = 8.0 / 3.0 ≈ **2.67:1**. The use of ATR makes this calculation dynamic to market volatility; the stop and target are wider in volatile markets and tighter in calm markets, but their ratio remains constant.

*   **Trade Termination (Visualization Only):** The `active_state` is reset to `0` (allowing for a new signal) if price touches the calculated SL or TP levels.
    *   Long Exit: `low <= stop_level` or `high >= target_level`
    *   Short Exit: `high >= stop_level` or `low <= target_level`
    This logic governs the termination of the visual boxes on the chart, it does not represent a backtested strategy exit.
    