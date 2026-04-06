
# Core Concept

### 1. The Market Philosophy
This script's primary investment thesis is rooted in **Adaptive Mean Reversion**. It operates on the principle of trend exhaustion, positing that established trends eventually become overextended and are statistically likely to reverse. The "Reversal" mode specifically targets these turning points, seeking to capture alpha from the violent snap-back that often follows a period of one-sided price action. The underlying psychological driver is investor capitulation and profit-taking at extremes.

However, its core innovation is the belief that the parameters defining an "exhausted" trend are not static. The script's philosophy is that market regimes (trending vs. ranging, high vs. low volatility) are fluid. Therefore, a successful strategy must continuously learn and adapt its definition of a valid entry. It is not just a mean reversion strategy; it is a meta-strategy designed to find the optimal mean reversion parameters for the current market context.

### 2. The Trade Narrative
The script waits for a specific story to unfold. For a bullish reversal, the narrative begins with a clear, established downtrend, defined by price trading below an adaptive SuperTrend band. During this descent, two critical events must occur: first, the price must print a legitimate new low, confirming the trend's validity (`Require Fresh Pivot`). Second, the RSI must dip into oversold territory, signaling that downside momentum is waning and sellers are becoming exhausted. The setup is now primed. The market is telling a story of a powerful trend that is running out of steam. The final chapter is the reversal itself: price must then rally with enough force to cross back above the SuperTrend line, flipping its state from bearish to bullish. This structural break, confirmed by prior momentum decay, completes the trade narrative.

### 3. Trigger Logic & Mechanics
The engine's logic is a sophisticated confluence of structure, momentum, and (optionally) volume.

*   **Core Structure:** An adaptive SuperTrend, built on ATR, serves as the dynamic trend-defining framework. Its sensitivity and width are not fixed but are constantly tuned by a back-testing optimizer to improve the signal-to-noise ratio.
*   **Confirmation Filter:** The RSI acts as a crucial gatekeeper. It ensures the script only considers reversals that are preceded by genuine momentum exhaustion (overbought/oversold conditions). This prevents entries on directionless, choppy price action where the SuperTrend might flip without any underlying conviction.
*   **The Catalyst:** The trigger is the SuperTrend flip itself. The script observes the market as it meets the filter criteria (e.g., a new low during a downtrend with an oversold RSI). The definitive catalyst that shifts the script from "observing" to "executing" is the price closing across the SuperTrend band. This event validates that the structural trend has broken, providing the final, high-conviction entry signal. The entire system is designed to dynamically adjust the thresholds for these components, seeking to maintain a consistent edge as market behavior evolves.
    