
# Core Concept

### 1. The Market Philosophy
This script's investment thesis is rooted in a hybrid **Momentum and Mean Reversion** philosophy, specifically targeting the price action principle of **Structure Retesting**. It operates on the theory that after a significant market structure level is broken (a momentum event), price will often revert to this "broken" level to test it as new support/resistance before continuing in the direction of the breakout. This behavior is driven by liquidity dynamics; the retest zone represents an area where unfilled orders may reside and where traders who missed the initial move seek a second-chance entry, thus fueling the next leg of the trend.

### 2. The Trade Narrative
The script seeks a specific market story: a "Change of Character" (CHoCH). The narrative begins with the market breaking a previously established swing high or low, signaling a potential shift in control from sellers to buyers, or vice versa. This break is the initial plot point. The script then patiently waits for the subsequent chapter: a pullback. It requires price to move away from the broken level and then return, creating a classic "break-and-retest" setup. This pullback demonstrates that the initial breakout was not a runaway move, offering a more calculated entry point at a level of proven significance.

### 3. Trigger Logic & Mechanics
The engine's elegance lies in its confluence of price action and quantitative filtering.
*   **Why these indicators?** Pivot points are used to objectively define market structure. An ATR-based buffer creates a dynamic, volatility-aware retest zone, rather than a static line.
*   **How do filters reduce noise?** The script moves from "observing" to "executing" only when price, having left the level, returns and prints a specific confirmation signal (e.g., an engulfing candle or a close back outside the zone). This filters out weak or indecisive price action. The optional volume filter further improves the signal-to-noise ratio by requiring that the confirmation is backed by significant market interest.
*   **The Catalyst:** The "Delta Hybrid" component is the core innovation. It uses a volume-weighted proxy for order flow to gauge the *conviction* of the initial breakout and the *sentiment* during the retest. A setup with a strong breakout delta and supportive retest delta provides the highest quality signal, confirming that institutional momentum is likely behind the move.
    