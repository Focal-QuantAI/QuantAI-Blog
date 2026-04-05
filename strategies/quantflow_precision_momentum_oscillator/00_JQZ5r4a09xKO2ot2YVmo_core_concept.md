
# Core Concept

### 1. The Market Philosophy

This script operates on a sophisticated **Mean Reversion** philosophy, augmented with momentum-based timing. Its core thesis is that price action is gravitationally bound to a volume-weighted average price (the MIDAS VWAP), which acts as the "true" market equilibrium for a given session or period. The strategy posits that significant deviations from this anchor are statistically unsustainable, driven by temporary emotional excess (greed or fear). It seeks to generate alpha by identifying the point of peak deviation—the moment of trend exhaustion—and positioning for the inevitable reversion to the mean, where institutional value is perceived to lie.

### 2. The Trade Narrative

The ideal trade setup unfolds as a story of overextension. The script waits for the market to make a decisive, high-velocity move away from its session anchor. As price trends, the oscillator value stretches into an "extreme" zone, defined by a statistical standard deviation and marked by a Fibonacci threshold (0.618). This signifies that the current price is becoming "expensive" or "cheap" relative to the volume-weighted consensus. The narrative the script is looking for is this: a market that has sprinted too far, too fast, and is now showing signs of fatigue. The presence of momentum divergence or volatility compression ("coiling") provides powerful contextual confirmation that the prevailing trend is losing its underlying support, setting the stage for a sharp reversal.

### 3. Trigger Logic & Mechanics

The strategy’s engine is a fusion of statistical and momentum analysis. It first establishes an objective measure of value using a cumulative VWAP, reset at a strategic anchor point (e.g., daily open). It then calculates a Z-Score to quantify how far the current price has deviated in standard deviation terms.

*   **Why these indicators?** The VWAP provides an institutional-grade anchor, while the Z-Score normalizes the deviation, creating a universal measure of "overbought/oversold." This raw signal is then smoothed with a zero-lag Double EMA (DEMA) to create a highly responsive oscillator, improving the signal-to-noise ratio for precise entry timing.

*   **How do filters work?** The Fibonacci thresholds act as the primary filter, defining the zones where mean reversion is statistically probable. The "Predictive Markers" (Sparks) serve as an advanced filter, detecting a change in the oscillator's *velocity* (momentum of momentum) to front-run a reversal before it is obvious.

*   **The Catalyst:** The script transitions from "observing" to "executing" upon a confluence of events. The primary catalyst is the oscillator entering an extreme Fibonacci zone and then exhibiting a "Spark" or a classic momentum divergence. This confluence signals that not only is the price statistically overextended, but the force driving it has also begun to wane, providing a high-probability trigger for a counter-trend trade.
    