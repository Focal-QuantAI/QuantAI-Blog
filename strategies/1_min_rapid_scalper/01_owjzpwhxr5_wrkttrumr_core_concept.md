
# Core Concept

### 1. The Market Philosophy

This strategy operates on a core philosophy of **Momentum Ignition**. It is designed to capture alpha not from an established trend, but from the very moment a new, aggressive micro-trend is born. The underlying principle is that the most powerful and predictable price movements occur immediately following a decisive shift in market sentiment. The script theorizes that a specific sequence of price action can signal this "ignition event" with a high degree of confidence. It aims to exploit the initial, explosive burst of directional activity before mean reversion or consolidation forces take hold, making it a pure momentum-hunting algorithm.

### 2. The Trade Narrative

The script is searching for a very specific market story: a violent reversal that immediately establishes a new, powerful direction. The ideal setup begins with a market moving in one direction, followed by a clear reversal candle. This reversal is not tentative; it is confirmed by a sequence of three consecutive Heikin Ashi candles that are not only directionally aligned but are also accelerating in strength (indicated by increasing body size). Crucially, these candles must show no signs of opposition (i.e., no counter-trend wicks), painting a picture of complete capitulation by the opposing side and a sudden, unified rush of buying or selling pressure.

### 3. Trigger Logic & Mechanics

The strategy’s engine is a confluence of Heikin Ashi candle morphology and a trend-strength filter.

*   **Heikin Ashi (HA) Sequencing:** Instead of raw price, the script uses HA candles to filter noise and isolate pure momentum. The trigger requires a strict three-candle pattern of accelerating, wick-less HA bars. This serves as the primary filter, confirming that a new directional impulse is not only present but gaining strength.
*   **ADX Regime Filter:** The Average Directional Index (ADX) acts as a gatekeeper. By requiring the ADX to be above a set threshold, the strategy ensures it only engages when the market possesses sufficient directional energy to sustain a move. This improves the signal-to-noise ratio by avoiding trades during listless, choppy conditions where momentum signals are unreliable.

The catalyst that flips the script from "observing" to "executing" is the successful completion of this three-candle HA sequence while the ADX confirms a trending environment. This confluence is the strategy's definition of a high-probability entry point for a rapid scalping trade.
    