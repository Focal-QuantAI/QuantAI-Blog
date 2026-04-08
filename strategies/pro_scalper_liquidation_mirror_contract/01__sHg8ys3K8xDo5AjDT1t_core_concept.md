
# Core Concept

### 1. The Market Philosophy

This strategy operates on a principle of **Trend Continuation via Mean Reversion**. It is not a classic range-bound mean reversion play; instead, it hypothesizes that within a strong, established trend, sharp counter-trend moves are often unsustainable exhaustion spikes caused by liquidity hunting or stop-loss cascades. The script's core philosophy is that these violent, high-volume price extensions represent the final capitulation of counter-trend participants. By identifying this point of maximum emotional duress, the strategy aims to enter precisely as the market "snaps back" to its dominant trajectory, effectively capturing alpha from the failure of the short-term breakout.

### 2. The Trade Narrative

The script is searching for a very specific market story. First, the market must be in a clear, high-conviction macro trend, confirmed by both a long-term EMA and a Supertrend filter. Within this established flow, the narrative requires a sudden, aggressive price thrust *against* the primary trend, breaking a recent short-term high or low. This is not a gentle pullback; it is a sharp, climactic move accompanied by an anomalous surge in volume—the script's proxy for a liquidation event. The ideal setup is a picture of failed aggression: a violent counter-trend spike that immediately shows signs of momentum exhaustion, creating the perfect vacuum for the primary trend to reassert control.

### 3. Trigger Logic & Mechanics

The script’s elegance lies in its confluence of filters designed to maximize the signal-to-noise ratio.

*   **The "Why":** The EMA and Supertrend establish the strategic directional bias, ensuring the algorithm only "fishes" for longs in a bull market and "escapes" tops in a bear market. The ADX filter is critical; by requiring a high reading, it mandates that the strategy only engages during periods of strong, directional momentum, avoiding whipsaws in choppy conditions. The volume spike logic is the event detector, pinpointing the abnormal activity, while the MACD Histogram fade acts as the final confirmation, verifying that the force behind the spike is already dissipating.

*   **The "How":** The script remains passive until a price spike breaks the recent `breakoutLen` high/low. This is the initial alert. It then cross-references this event with the filters. The catalyst that flips the script from "observing" to "executing" is the simultaneous occurrence of this price break on massive volume (`isLiquidation`), within a strong trend (`adxHigh`), and with confirmed momentum decay (`momentumFade`). This multi-factor trigger is designed to isolate high-probability reversal points from mere noise.
    