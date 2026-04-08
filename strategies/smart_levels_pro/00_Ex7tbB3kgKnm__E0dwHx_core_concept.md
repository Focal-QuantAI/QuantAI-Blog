
# Core Concept

### 1. The Market Philosophy
This script operates on a foundational principle of **Mean Reversion and Volatility Breakout** centered on structural price levels. Its core philosophy is that past highs and lows of significant timeframes (daily, weekly, monthly) and trading sessions (Asia, London, New York) act as powerful psychological and algorithmic inflection points. These levels represent areas of prior order flow imbalance and serve as magnets for future price action. The strategy aims to exploit the market's "memory," capitalizing on two primary reactions: either a rejection from these levels as liquidity is defended (mean reversion) or a powerful continuation once these liquidity pools are decisively breached (breakout).

### 2. The Trade Narrative
The script does not generate signals but rather sets the stage for a specific trade narrative. It seeks to identify when current price action interacts with a historically significant barrier. The ideal setup is a clean approach to a previous high or low (e.g., Previous Day's High - PDH) or a session extreme. For a mean reversion play, the narrative is one of "failed auction," where price probes a key level but fails to find acceptance, signaling exhaustion and a likely return toward an equilibrium point. Conversely, for a breakout, the story is one of "price discovery," where a high-volume breach of a key level triggers a cascade of stop orders, fueling a momentum-driven move into new territory.

### 3. Trigger Logic & Mechanics
This indicator is a decision-support tool, not an automated execution system. Its "trigger" is the visualization of confluence.

*   **Why these indicators in tandem?** The script forgoes traditional oscillators for pure price structure. By overlaying daily, weekly, and monthly levels with intraday session ranges, it creates a map of potential support and resistance zones. The true power lies in their confluence; for instance, a Previous Weekly High that aligns with a New York session high presents a far more formidable resistance level, increasing the signal-to-noise ratio for any trade taken there.

*   **How do filters serve to reduce noise?** The script’s primary filter is timeframe hierarchy. A monthly level holds more structural weight than a daily one. By presenting all levels, it allows the strategist to assess the context and avoid taking trades at minor levels when a major one is nearby. The session ranges act as intraday filters, framing price action within the context of the most active market participants.

*   **What is the catalyst?** The catalyst is the interaction of price with one of these pre-defined levels. The script transitions from "observing" to "informing" when price enters a "hot zone," empowering the trader to look for specific price action patterns (e.g., pin bars, engulfing candles) to confirm the level's influence and execute with higher probability.
    