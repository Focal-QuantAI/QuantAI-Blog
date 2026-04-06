
# Core Concept

### 1. The Market Philosophy

This strategy's "raison d'être" is rooted in **Smart Money Concepts (SMC)**, a framework that attempts to model the behavior of institutional market participants. It operates on the theory of order flow and liquidity hunting, blending elements of both **Momentum** and **Mean Reversion**. The core philosophy is that significant market moves are preceded by a "Change of Character" (a structural break), and that price will often revert to areas of inefficiency (Fair Value Gaps) or unmitigated orders (Order Blocks) within favorable pricing zones (Premium/Discount) before continuing in its new direction. The strategy aims to generate alpha by identifying these institutional footprints and entering not on the initial breakout, but on the subsequent, higher-probability pullback.

### 2. The Trade Narrative

The script is waiting for a specific story to unfold: a clear trend, defined by a series of swing highs and lows, must first show signs of exhaustion or reversal. This is signaled by a **Market Structure Shift (MSS)**, where price decisively breaks a prior swing point against the prevailing trend. However, the script does not chase this initial momentum. Instead, it patiently observes for a corrective pullback. For a bullish setup, it requires price to retrace into a "Discount" zone (the lower 50% of a defined range). For a bearish setup, it waits for a rally into a "Premium" zone. The ideal narrative is this pullback targeting a recently formed Fair Value Gap (FVG), signaling a move to rebalance the market before the next impulsive leg begins.

### 3. Trigger Logic & Mechanics

The strategy’s execution is a masterclass in confluence. It uses a hierarchy of filters to improve its signal-to-noise ratio.

*   **Primary Catalyst:** The foundational trigger is the **Market Structure Shift (MSS)**, which identifies a potential change in directional control. This is the script’s initial point of interest.
*   **Value Filter:** The **Premium & Discount Zone** logic acts as a crucial discipline mechanism. By requiring buys to occur only in discount and sells only in premium, it systematically prevents chasing extended moves and improves the potential risk-reward profile of each trade.
*   **Precision Filter:** The requirement for a **Fair Value Gap (FVG)** serves to pinpoint the entry. An FVG represents a price inefficiency or imbalance. By demanding a trade trigger in or near an FVG, the strategy hypothesizes it is entering at a point where the market is most likely to react strongly.

The script flips from "observing" to "executing" only when these three core conditions align: a structural break has occurred, price has pulled back to a value zone, and an inefficiency is present to be filled. The addition of volume analysis further ranks the signal's strength, providing a final layer of conviction.
    