
# Core Concept

### 1. The Market Philosophy
This strategy is rooted in **Momentum Continuation**, designed to capture alpha from the persistence of micro-trends on the 1-minute timeframe. It operates on the principle that a confirmed trend, after a brief and shallow pullback, is likely to resume with renewed force. The core economic thesis is that a momentary dip within a strong directional move often serves as a "shakeout," flushing out weak hands and creating a liquidity event for stronger participants to reload their positions. The script aims to identify this point of re-engagement, capitalizing on the subsequent acceleration in price as the primary trend reasserts itself.

### 2. The Trade Narrative
The ideal setup requires a market that is not just trending, but trending with structural integrity. For a long entry, the script demands a clear sequence of higher lows, established via non-repainting pivots. Within this established uptrend, the strategy waits for a specific three-bar "story" to unfold: a strong bullish candle, followed by a brief bearish pullback, and then a final, decisive bullish candle. This pattern signifies that buyers have successfully absorbed a counter-move and are now driving price to a new short-term high. The script is engineered to engage precisely at this moment of confirmed buyer dominance, entering after the pullback has failed and momentum has been re-established.

### 3. Trigger Logic & Mechanics
The strategy achieves a high signal-to-noise ratio through a sophisticated confluence of filters. The non-repainting pivots define the overarching trend, serving as the primary state filter. The "Strong Candle" logic qualifies the price action, ensuring entries are based on conviction, not just price movement. The specific `bull-bear-bull` (or `bear-bull-bear`) sequence is the key pattern recognition element, identifying the aforementioned shakeout.

The final trigger is a confluence event: the close of a confirmed bar that completes this three-bar pattern, breaks the recent five-bar high, and occurs within the context of the established macro trend. Crucially, an ATR-based filter prevents entries near immediate support or resistance, acting as a dynamic risk management constraint to avoid trades with a poor initial risk/reward profile. The script transitions from "observing" to "executing" only when this multi-factor alignment confirms the trend's resilience.
    