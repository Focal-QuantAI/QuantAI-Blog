
# Core Concept

### 1. The Market Philosophy

This script’s investment thesis is rooted in the principles of **Liquidity Hunting and Mean Reversion from Imbalances**, a framework often associated with "Smart Money Concepts" (SMC). The core philosophy posits that market price is not random but is deliberately engineered by institutional players to trigger liquidity events. It operates on the economic principle that large orders require substantial counter-party liquidity to be filled efficiently. This liquidity is predictably found where retail stop-losses cluster—primarily above recent highs and below recent lows. The strategy aims to identify these liquidity zones and subsequent price imbalances (Fair Value Gaps, Order Blocks) left in the wake of aggressive institutional activity, anticipating that price will revert to these areas to mitigate positions and rebalance.

### 2. The Trade Narrative

The script seeks a narrative of confirmed trend continuation followed by a discounted or premium entry. The ideal setup begins with a clear "Break of Structure" (BoS), where price decisively closes beyond a prior swing high or low, validating the prevailing trend. This impulsive move often leaves behind inefficiencies—namely, Fair Value Gaps (FVGs) and Order Blocks (the last opposing candle before the impulse). The "story" is one of patience: after the market shows its hand with a powerful trend move, the script waits for price to pull back into these pre-identified zones of institutional interest. A high-probability trade is signaled not by the pullback itself, but by price entering a confluence of these zones—for instance, mitigating a bullish Order Block that resides within a larger Fair Value Gap.

### 3. Trigger Logic & Mechanics

The script’s logic is built on **confluence** to enhance its signal-to-noise ratio. It is not a standalone signal generator but a sophisticated map of institutional footprints.

*   **Why these indicators?** The ZigZag defines market structure (highs/lows), which is the foundational layer. Pivot-based liquidity levels then mark the "fuel" (stop-losses) for future moves. Finally, Order Blocks and FVGs pinpoint the precise, imbalanced price zones from which institutional moves originated. Using them in tandem allows a trader to see the full sequence: structure, the liquidity that was taken, and the resulting imbalance to which price may return.

*   **How do filters work?** The script’s primary filter is its focus on price action context. It ignores ranging, "dead" price action and only highlights areas created by significant momentum. The labels "BoS" (Break of Structure) and "CHoCH" (Change of Character) act as a contextual filter, distinguishing between trend continuation and potential reversals.

*   **The Catalyst:** The script flips from "observing" to "highlighting opportunity" when live price interacts with a drawn zone. The catalyst for a trader is seeing price enter a historical Order Block or FVG. This is the cue to watch for a reaction—a rejection or acceptance at that level—which forms the basis for a discretionary trade execution, seeking to ride the next institutional wave.
    