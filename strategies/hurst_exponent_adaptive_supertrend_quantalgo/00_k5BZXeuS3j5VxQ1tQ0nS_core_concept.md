
# Core Concept

### 1. The Market Philosophy

This strategy operates on a **Regime-Adaptive Momentum** philosophy. Its core thesis is that financial markets are not monolithic; they cycle between periods of directional persistence (trending) and random, mean-reverting noise (choppy). The script attempts to exploit the alpha generated during trending phases while actively mitigating the risk of whipsaws and drawdowns inherent in consolidative environments. The underlying principle is that by quantitatively diagnosing the market's "memory" using the Hurst exponent, the strategy can dynamically adjust its sensitivity, participating aggressively only when trend persistence is statistically probable (H > 0.5) and becoming defensive when price action is likely anti-persistent (H < 0.5).

### 2. The Trade Narrative

The script is designed to ignore directionless noise and engage only when a statistically significant trend emerges. The ideal setup begins with the market transitioning from a period of low conviction, characterized by a low Hurst exponent (H < 0.5). The narrative the script waits for is a quantifiable shift in market character where the Hurst exponent climbs above 0.5, signaling that the asset's price action is developing memory and persistence. This statistical confirmation of a new trending regime is the prerequisite. The trade is not triggered by this alone; it requires a subsequent price action catalyst—a decisive break of a key volatility-based level—to confirm the new directional bias is in motion.

### 3. Trigger Logic & Mechanics

The strategy’s mechanics are a sophisticated confluence of three components designed to maximize the signal-to-noise ratio:

*   **Regime Diagnosis (Hurst Exponent):** The Hurst exponent acts as the strategic overlay, determining the system's overall posture. It quantifies whether the market is trending or range-bound.
*   **Signal Smoothing (Kalman Filter):** A low-lag Kalman filter is used instead of a traditional moving average to provide a cleaner, more responsive measure of the underlying price trajectory. Crucially, its tracking gain is *increased* in trending markets (high H) and *decreased* in choppy markets (low H), making it inherently adaptive.
*   **Dynamic Trigger (Adaptive Supertrend):** The classic Supertrend logic provides the entry/exit framework. However, its ATR multiplier is dynamically scaled by the Hurst exponent. In choppy markets (low H), the bands widen significantly to prevent false signals. In strong trends (high H), they tighten to allow for more responsive entries and exits.

The final trigger is the Kalman-smoothed price crossing this adaptive Supertrend band, an event that only occurs when the statistical regime and price action are in full alignment.
    