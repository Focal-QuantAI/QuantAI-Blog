
# Core Concept

### 1. The Market Philosophy
This script operates on a **Volatility Breakout** philosophy. Its core thesis is that periods of price consolidation and contracting volatility, specifically in the form of wedge patterns, represent a temporary equilibrium or market indecision. This state is inherently unstable. The strategy aims to capture the alpha generated when this compression resolves into a high-velocity, directional move. The underlying principle is one of pent-up energy release; as buyers and sellers battle into a tightening range, the eventual victory of one side leads to a rapid price cascade as the losing side capitulates and stop-losses are triggered.

### 2. The Trade Narrative
The ideal market "story" for this script begins with a period of structured, converging price action. For a bullish setup (falling wedge), the market is making a series of lower highs and lower lows, but the slope of the lows is shallower than the highs, indicating that selling pressure is waning. Conversely, for a bearish setup (rising wedge), the market posts higher highs and higher lows, but the buying momentum is weakening, evidenced by the converging trendlines. The script is looking for this "coiling spring" environment—a clear, multi-touch consolidation pattern where volatility diminishes as the apex of the wedge approaches.

### 3. Trigger Logic & Mechanics
The strategy’s mechanics are built on a confluence of structure, price action, and momentum.

*   **Structural Identification:** Instead of relying on lagging oscillators, the script builds its own model of market structure using `pivothigh` and `pivotlow` points. By requiring a minimum number of touches, it enhances the statistical validity and signal-to-noise ratio of the identified trendlines.
*   **Noise Filtration:** The script employs two key filters. First, it invalidates any pattern if the price violates the boundaries *before* a valid breakout, ensuring the integrity of the consolidation phase. Second, the `Minimum Breakout Body %` input serves as a conviction filter. It demands that the breakout candle shows decisive momentum, ignoring weak or indecisive "wicks" that could signify a false move or liquidity hunt.
*   **Execution Catalyst:** The script transitions from "observing" to "executing" only when a candle closes decisively outside the wedge boundary *in the expected direction* (upside for a falling wedge, downside for a rising wedge) and meets the body-size threshold. This confluence is the trigger, signaling that the period of indecision is over and a new directional impulse has likely begun.
    