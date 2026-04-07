
# Core Concept

### 1. The Market Philosophy

This script's investment thesis is rooted in a hybrid model of **Trend Following** and **Mean Reversion**. It operates on the philosophy that while markets trend, price action perpetually oscillates around a "fair value" mean, driven by institutional volume. The strategy's alpha is derived from identifying the inception of a new directional trend, as defined by a smoothed, volume-weighted average price (VWAP), and then capitalizing on statistically significant deviations from that moving average. The underlying principle is that the T3-smoothed VWAP represents the market's current equilibrium, and any extreme extension away from it is likely a temporary overreaction, presenting either an exhaustion point (for exits) or a potential re-entry point in the direction of the primary trend.

### 2. The Trade Narrative

The script is designed to engage after a period of directional ambiguity resolves. The ideal "story" begins with the market establishing a clear directional bias, confirmed not just by price but by significant volume flow. This is visualized as the T3-smoothed VWAP line—the strategy's core basis—inflecting and beginning a sustained slope upwards or downwards. The script remains passive until this "regime shift" is confirmed. Once the trend is established (e.g., the T3-VWAP is rising), the script anticipates pullbacks towards this dynamic mean. The narrative it seeks is: "The institutional trend is now bullish; we will look to enter long positions as price temporarily dips to levels of dynamic support defined by volatility."

### 3. Trigger Logic & Mechanics

The strategy's mechanics are a sophisticated confluence of three core components designed to enhance the signal-to-noise ratio:

*   **VWAP & T3 Smoothing:** The script first calculates a raw VWAP, anchoring its analysis to institutional "fair value." It then applies a T3 moving average—a low-lag, triple-smoothed EMA—to this VWAP. The "Why" is to create an exceptionally smooth yet responsive dynamic mean, filtering out the noise and whipsaw inherent in raw price or standard moving averages.
*   **ATR Volatility Bands:** The bands are not based on standard deviation but on the Average True Range (ATR). This makes the expected price boundaries adaptive to current market volatility, providing a more realistic risk and opportunity framework than fixed-percentage bands.
*   **Catalyst:** The primary entry trigger is the crossover of the T3-VWAP line itself (`t3 > t3[1]`), signaling the official start of a new trend. This flips the script from "observing" to "executing." Subsequent take-profit signals are triggered when price overextends and crosses the outermost ATR band *against* the primary trend, indicating potential exhaustion and a reversion back to the mean.
    