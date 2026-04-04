
# Core Concept

### 1. The Market Philosophy

This script's investment thesis is rooted in **Trend-Filtered Mean Reversion**. It operates on the principle that even within a strong, established trend, markets exhibit periods of exhaustion and temporary counter-trend movements (pullbacks or rallies). The strategy does not attempt to catch absolute market tops or bottoms; instead, it seeks to identify the end of these minor pullbacks to re-engage with the dominant, primary trend. The core economic principle is that while short-term sentiment can become overextended, institutional capital flow (represented by the 200 EMA) provides a gravitational pull. This strategy aims to capture the alpha generated when short-term price action snaps back into alignment with this long-term bias.

### 2. The Trade Narrative

The ideal setup is a story of qualified exhaustion. For a long entry, the script is looking for a market that is in a clear structural uptrend (price trading above its 200 EMA). Within this uptrend, a period of selling or consolidation occurs, pushing price to a local low. Crucially, as price makes this new low, the RSI fails to follow suit, printing a higher low. This bullish divergence signals that the selling momentum is waning and the pullback is likely losing steam. The asset is showing signs of being oversold on a short-term basis, presenting a high-probability entry point to rejoin the prevailing macro uptrend.

### 3. Trigger Logic & Mechanics

The script’s execution mechanism is a sophisticated **Confluence Engine** designed to elevate the signal-to-noise ratio. It avoids relying on any single indicator, instead demanding that multiple, distinct conditions align before generating a signal.

*   **Why these indicators?** The **RSI** is used for its primary function: identifying momentum exhaustion through overbought/oversold readings and, more importantly, **Divergence**. The **200 EMA** acts as a macro filter, providing a proxy for institutional bias and ensuring trades are only considered in the direction of the "smart money."
*   **How filters reduce noise:** The "Confluence Score" is the key. A simple RSI cross is a low-quality signal. However, by requiring a confluence of events—such as a bullish divergence (3 points) occurring while the RSI exits an oversold state (2 points) in a market above the 200 EMA (1 point)—the script synthesizes a high-conviction setup.
*   **The Catalyst:** The trigger is not a single event but the cumulative score of these weighted factors exceeding a user-defined threshold. This score-based system is the final arbiter, flipping the script from observation to execution only when a critical mass of evidence supports the trade thesis.
    