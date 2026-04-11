
# Indicators Description

### 1. Component Deconstruction

#### **Heikin Ashi (HA) Data**
*   **Specific Configuration:** The script does not calculate Heikin Ashi values from standard `ohlc` data. Instead, it uses a direct data request via `ticker.heikinashi(syminfo.tickerid)` and `request.security()`. This ensures it retrieves the broker's native HA values for the current symbol and timeframe, which are then deconstructed into `haOpen`, `haHigh`, `haLow`, and `haClose`.
*   **Functional Modification:** The script performs a **morphological analysis** of the HA candles. It does not use them for visualization but as a source for quantitative data points. The core logic is built upon derived boolean values and numerical properties of these candles, such as:
    *   `haSize`: The absolute body size (`math.abs(haClose - haOpen)`).
    *   `isHaBull`/`isHaBear`: The direction of the candle body.
    *   `noWickBull`/`noWickBear`: A boolean test for the absence of a lower wick on a bullish candle (`haOpen == haLow`) or an upper wick on a bearish candle (`haOpen == haHigh`), indicating strong, unopposed momentum.

#### **Average Directional Index (ADX)**
*   **Specific Configuration:**
    *   **Indicator:** `ta.dmi` which returns the Positive Directional Indicator (+DI), Negative Directional Indicator (-DI), and the ADX.
    *   **Lookback Period:** The length for both the DI calculation and the subsequent ADX smoothing is set by `adxLen`, with a default of **7**. This is a short period, making the ADX highly reactive to recent changes in trend strength.
    *   **Price Source:** The underlying `ta.dmi` function implicitly uses standard price inputs for its True Range and Directional Movement calculations.
*   **Functional Modification:** There is no mathematical modification to the ADX formula itself. Its function is purely as a **binary state filter**. The script does not analyze the ADX's slope or its relationship with the +/-DI lines; it only checks if `adxValue` is above a static threshold (`adxThresh`).

#### **Average True Range (ATR)**
*   **Specific Configuration:**
    *   **Indicator:** `ta.atr`.
    *   **Lookback Period:** The smoothing period is defined by `atrLen`, with a default of **14**.
    *   **Smoothing Method:** The standard Pine Script `ta.atr` uses a Running Moving Average (RMA).
*   **Functional Modification:** The ATR value is not modified. It is used as a dynamic unit of volatility to calculate exit price targets. Its value is captured at the bar *preceding* the entry (`atrValue[1]`) to establish fixed stop-loss and take-profit levels, preventing the targets from shifting with market volatility after a position is opened.

---

### 2. Logic Layering & Confluence

The script's engine is a hierarchical filtering system designed to isolate a specific "momentum ignition" event. The logic is layered to progressively increase signal confidence.

*   **Layer 1: Reversal Prerequisite (The Flip)**
    *   **Interaction Dynamic:** The engine first looks for a change of state. The `colorFlipBull` and `colorFlipBear` variables require the candle at `[3]` (three bars prior to the trigger candle) to be of the opposite color.
    *   **Hierarchical Filtering:** This acts as the initial gatekeeper. It ensures the subsequent pattern is a *reversal* and not merely a continuation of an existing trend, focusing the strategy on inflection points.

*   **Layer 2: Trend Regime Filter (The Gatekeeper)**
    *   **Interaction Dynamic:** A **Threshold Cross**. The `adxValue` must be greater than the `adxThresh` input (default: 20.0).
    *   **Hierarchical Filtering:** The ADX acts as a macro-level "regime filter." If the market lacks sufficient directional energy (i.e., is in a chop or consolidation phase), the ADX will be below the threshold, and the entire HA pattern analysis is disregarded. This is designed to improve the signal-to-noise ratio by only permitting trades in environments capable of supporting a momentum burst.

*   **Layer 3: Momentum Ignition Pattern (The Trigger Sequence)**
    *   **Interaction Dynamic:** This is a strict **Pattern Recognition** sequence based on the confluence of three distinct HA candle properties across three consecutive bars.
        1.  **Directional Unison:** All three candles in the sequence (`[2]`, `[1]`, and the current bar `[0]`) must be bullish (for a long) or bearish (for a short).
        2.  **Momentum Confirmation:** All three candles must be "marubozu-style" in the direction of the trend (no lower wicks for longs, no upper wicks for shorts). This signals a complete lack of opposition.
        3.  **Momentum Acceleration:** The body size of each candle must be greater than the previous one (`haSize[1] > haSize[2]` and `haSize > haSize[1]`). This confirms that the new momentum is not only present but actively strengthening.

The final signal is a product of **confluence**: the reversal prerequisite is met, the ADX confirms a trending regime, and the three-bar HA sequence confirms an accelerating, unopposed momentum ignition.

---

### 3. The Execution Engine

#### **Trigger Conditions**

*   **Boolean Logic (Long Trigger):** A `true` signal is generated on the current bar if and only if all the following conditions are met:
    1.  The HA candle 3 bars ago was bearish (`isHaBear[3]`).
    2.  The HA candle 2 bars ago was bullish AND had no lower wick (`bull1`).
    3.  The HA candle 1 bar ago was bullish, had no lower wick, AND its body was larger than the candle from 2 bars ago (`bull2`).
    4.  The current HA candle is bullish, has no lower wick, AND its body is larger than the candle from 1 bar ago (`bull3`).
    5.  The current ADX(7) value is greater than `adxThresh` (20.0).

*   **Boolean Logic (Short Trigger):** A `true` signal is generated on the current bar if and only if all the following conditions are met:
    1.  The HA candle 3 bars ago was bullish (`isHaBull[3]`).
    2.  The HA candle 2 bars ago was bearish AND had no upper wick (`bear1`).
    3.  The HA candle 1 bar ago was bearish, had no upper wick, AND its body was larger than the candle from 2 bars ago (`bear2`).
    4.  The current HA candle is bearish, has no upper wick, AND its body is larger than the candle from 1 bar ago (`bear3`).
    5.  The current ADX(7) value is greater than `adxThresh` (20.0).

#### **Exit Conditions & Mathematical Constants**

The script employs two mutually exclusive exit mechanisms, controlled by the `useAtr` boolean input.

*   **Mechanism 1: ATR-Based Exits (`useAtr = true`)**
    *   **Take Profit:** Calculated as `Entry Price + (ATR[1] * tpMult)`. The default `tpMult` of **1.0** sets the take profit at a distance of 1 ATR from the entry price.
    *   **Stop Loss:** The effective stop loss is the *tighter* of two calculations:
        1.  **Volatility Stop:** `Entry Price - (ATR[1] * slMult)`. The default `slMult` of **2.0** sets a volatility-based stop at a distance of 2 ATRs, establishing a default **1:2 Reward-to-Risk ratio**.
        2.  **Catastrophe Stop ("The Plug"):** `Entry Price * (1 - thePlug / 100)`. The default `thePlug` of **1.0** sets a hard stop loss at 1% below the entry price.
    *   **Logic:** The final stop loss (`longSL`) is determined by `math.max(atrStop, plugStop)`. This means the stop is placed at the price level *closest* to the entry, effectively using "The Plug" as a maximum risk cap that overrides the ATR calculation if market volatility is exceptionally high.

*   **Mechanism 2: Fixed 1-Bar Exit (`useAtr = false`)**
    *   **Logic:** If a position is open, the `strategy.close()` command is issued. Due to the setting `process_orders_on_close = false`, an order generated during a bar is executed at the open of the next bar.
    *   **Execution:** When a long entry is triggered on Bar X, the `strategy.close()` command is also processed on Bar X. This places a market order to close the position at the open of Bar X+1. This results in a fixed holding period of exactly one bar.
    *   **Mathematical Constants:** This mechanism does not use any multipliers. The exit is purely time-based.
    