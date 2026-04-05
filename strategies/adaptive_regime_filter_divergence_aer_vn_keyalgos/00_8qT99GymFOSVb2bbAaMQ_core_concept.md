
# Core Concept

### 1. The Market Philosophy

This investment thesis is predicated on exploiting **trend exhaustion and momentum decay**. The core philosophy is a sophisticated form of mean reversion, but one that requires a mature trend to exist first. It operates on the principle that as a directional move persists, its underlying efficiency diminishes. The "effort" (cumulative price travel) required to achieve new highs or lows increases, while the "result" (net displacement) weakens. This divergence between price and momentum efficiency often precedes a period of consolidation or a significant reversal, as the dominant market participants become exhausted and contrarian pressure builds. The strategy aims to capture alpha by identifying these precise inflection points where a trend is most vulnerable.

### 2. The Trade Narrative

The script is designed to act on a specific market story: an established, efficient trend that is beginning to show signs of internal weakness. The ideal setup begins with the market in a clear uptrend or downtrend, characterized by a high Efficiency Ratio (ER), indicating smooth, directional price action. The narrative unfolds as price pushes to a new extreme (e.g., a higher high). However, this new peak is achieved with less conviction than the last. The underlying ER oscillator fails to confirm the new price high, instead printing a lower high. This creates a "regular bearish divergence," telling a story of a trend running on fumes. The market is outwardly strong, but its internal engine is sputtering, creating a high-probability setup for a counter-trend trade.

### 3. Trigger Logic & Mechanics

The strategy’s engine is a confluence of the Kaufman Efficiency Ratio (ER) and an adaptive volatility filter.

*   **Why these indicators?** The ER is used as the primary measure of momentum because it directly quantifies the signal-to-noise ratio of price movement, distinguishing clean trends from choppy price action. This is then filtered by an **adaptive threshold** based on the Average True Range (ATR). This dynamic element ensures the strategy adjusts its definition of a "trend" to the current volatility regime, reducing false signals in either placid or chaotic markets.

*   **The Catalyst:** The script transitions from "observing" to "executing" upon the confirmation of a divergence. The trigger is not merely a crossover or a threshold break, but the specific pattern of price making a new swing high/low while the ER fails to do so. This confluence—a mature trend regime combined with a confirmed divergence between price and its directional efficiency—serves as the high-conviction catalyst, signaling that the prevailing trend's risk/reward profile has unfavorably shifted.
    