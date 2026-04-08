
# Core Concept

### 1. The Market Philosophy

This script operates on a **Momentum Continuation** philosophy, specifically tailored for high-frequency, intraday scalping. Its core thesis is that established micro-trends on the 1-minute timeframe exhibit predictable persistence after brief, sharp pullbacks. The strategy aims to capture alpha by identifying moments of temporary counter-trend exhaustion within a larger, confirmed directional move. It presupposes that a one-bar "shakeout" is often a liquidity-gathering event that precedes the next leg of the trend, rather than a genuine reversal. The script is designed to thrive in trending, volatile environments, systematically fading one-bar pullbacks in anticipation of trend resumption.

### 2. The Trade Narrative

The ideal setup is a "breather" within a sprint. The script waits for the market to first establish a clear micro-trend, defined by a series of higher pivot lows (for an uptrend) or lower pivot highs (for a downtrend). Once this context is set, the narrative it seeks is a specific three-bar sequence: a strong candle in the trend's direction, followed by a sharp, single-bar pullback against the trend, and culminating in a powerful resumption candle that breaks the recent micro-high. This pattern tells the story of a healthy trend that has momentarily paused, shaken out weak hands, and is now re-accelerating with conviction. The strategy avoids entering when price is already hugging a recent support or resistance zone, ensuring there is perceived "room to run."

### 3. Trigger Logic & Mechanics

The strategy achieves its high signal-to-noise ratio through a strict confluence of filters. The non-repainting pivot structure provides the foundational trend direction, acting as the primary regime filter. The script then hunts for its specific three-bar pullback pattern, using a candle-body filter to ensure each bar represents decisive action, not indecision.

The "Why" behind this tandem is to isolate pullbacks that are both shallow and temporary. The final catalyst—the trigger that flips the script from "observing" to "executing"—is the confluence of two events on the closing bar: 1) The completion of the strong bullish (or bearish) resumption candle, and 2) A simultaneous breakout above the high (or below the low) of the previous five bars. This breakout acts as the final confirmation of renewed momentum, while the ATR-based filter ensures the entry isn't made directly into an obvious price ceiling or floor, thus preserving a viable risk-reward profile.
    