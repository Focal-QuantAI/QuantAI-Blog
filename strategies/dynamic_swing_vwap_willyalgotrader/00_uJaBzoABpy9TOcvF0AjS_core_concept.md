
# Core Concept

### 1. The Market Philosophy

This script operates on a **Momentum** philosophy, anchored by a dynamic measure of fair value. Its core thesis is that significant market turning points (swing highs and lows) represent a psychological and structural "reset" in order flow. Following such a pivot, a new directional bias, or trend, is established. The strategy aims to capture this emerging trend by tracking the volume-weighted average price from the moment of the turn. It posits that price will tend to respect this new, re-anchored VWAP as a dynamic support/resistance, and deviations from it offer opportunities to join the prevailing momentum. The underlying principle is trend persistence following a period of investor exhaustion or capitulation, which is marked by the swing pivot.

### 2. The Trade Narrative

The script is designed to engage after the market tells a story of reversal. The ideal setup begins with an established directional move that culminates in a clear, structurally significant swing high or low—a point that is the highest high or lowest low over a considerable lookback period. This pivot is not just a minor fluctuation; it is a definitive rejection of previous price levels. Once this anchor point is confirmed, the script hypothesizes that a new regime has begun. It then looks for price action to validate this new trend by holding above the newly calculated VWAP in an uptrend or below it in a downtrend, confirming that institutional flow is supporting the new direction.

### 3. Trigger Logic & Mechanics

The strategy’s engine is a confluence of three key components. First, it uses a classic pivot detector (`highestbars`/`lowestbars`) to identify structural turning points, which serve as the anchor for the entire analysis. Second, instead of a standard moving average, it employs an Exponentially Weighted Moving Average (EWMA) of both price and volume. This creates a "Dynamic Swing VWAP" that is not bound by exchange sessions.

The catalyst that flips the script from "observing" to "executing" is the confirmation of a new swing pivot. At this moment, the VWAP calculation is completely re-anchored to the pivot bar. This reset is the strategy's primary signal. To enhance the signal-to-noise ratio, the script incorporates an adaptive "alpha" based on ATR, allowing the VWAP to track price more aggressively in high-volatility environments and more smoothly in calm markets. Optional filters, like anchoring only on Lower-Lows or Higher-Highs, further refine the trigger to focus exclusively on high-conviction trend continuations, thereby improving the potential quality of each signal.
    