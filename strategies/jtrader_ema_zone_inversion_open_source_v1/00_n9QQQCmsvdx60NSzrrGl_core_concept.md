
# Core Concept

### 1. The Market Philosophy
This strategy operates on a **Momentum Continuation** philosophy, specifically designed to capture alpha from **Liquidity Grab** events within an established trend. The core thesis is that powerful trends do not move in a straight line; they are punctuated by sharp pullbacks that hunt for stop-loss orders resting in predictable zones of value. The script theorizes that the most potent continuation signals occur immediately after the market has successfully "raided" this liquidity (e.g., below a swing low in an uptrend) and then aggressively invalidates the bearish structure created during that pullback. It exploits the psychological principle of trend persistence, betting that a failed attempt to reverse the market will result in an even stronger move in the original direction.

### 2. The Trade Narrative
For a long entry, the script demands a specific story. First, the market must be in a robust, institutional uptrend, evidenced by bullishly stacked 33, 50, and 200 EMAs. Within this context, a sharp, aggressive sell-off occurs, creating a price inefficiency known as a Fair Value Gap (FVG). This pullback must form a swing low that dips its wick into the dynamic 33/50 EMA zone—the "liquidity raid"—before closing higher. The narrative's climax is not the dip itself, but the subsequent price action: a powerful surge that closes *above* the high of the bearish FVG. This "inversion" of a bearish pattern signals that the liquidity hunt is complete and the dominant bullish order flow has violently reasserted control.

### 3. Trigger Logic & Mechanics
The strategy’s elegance lies in its confluence of filters to achieve a high signal-to-noise ratio.

*   **Why these indicators?** The 200 EMA acts as a macro regime filter, restricting trades to the side of the dominant trend. The 33/50 EMA zone is the designated "battleground" for liquidity. The FVG pinpoints the exact moment of imbalance the strategy seeks to see invalidated.
*   **How do filters reduce noise?** The script doesn't trade the FVG formation or the dip into the EMA zone. Instead, it waits for the FVG to be decisively *violated* in the direction of the primary trend. This confirms the pullback was a trap, not a reversal. The additional check that a prior pivot wicked into the EMA zone ensures the setup was preceded by a probable liquidity grab, increasing its validity.
*   **The Catalyst:** The trigger flips from "observing" to "executing" when price closes through the boundary of a recent, counter-trend FVG, provided the broader trend structure (stacked EMAs) is intact and a preceding liquidity raid into the EMA zone can be identified. This invalidation is the definitive signal that the path of least resistance has resumed.
    