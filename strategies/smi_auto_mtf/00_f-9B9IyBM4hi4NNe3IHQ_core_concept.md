
# Core Concept

### 1. The Market Philosophy
This script operates on a **Mean Reversion** philosophy, specifically targeting momentum exhaustion. Its core thesis is that financial markets exhibit cyclical behavior, and periods of extreme directional momentum (overbought or oversold conditions) are unsustainable. The underlying principle is that after a strong price extension, profit-taking and counter-trend pressure will cause the asset's price to revert toward its recent statistical mean. The script's unique "alpha" is its attempt to systematize this concept across different timeframes by automatically adjusting its sensitivity, theorizing that the nature of momentum cycles is fractal but requires different parameters for optimal capture at each scale.

### 2. The Trade Narrative
The ideal setup for this strategy is not a nascent trend, but a mature one showing signs of fatigue. The market "story" begins with a sustained directional move that pushes price significantly away from its recent equilibrium. This action drives the SMI indicator deep into an extreme zone (overbought >40 or oversold <-40). The script remains in an observational state during this trend, patiently waiting for the momentum to peak. The narrative it seeks to capitalize on is the precise moment this momentum begins to wane—the first sign of reversal—signaling that the dominant pressure is subsiding and a corrective move is probable.

### 3. Trigger Logic & Mechanics
The strategy’s engine is the Stochastic Momentum Index (SMI), a refined oscillator chosen for its double exponential smoothing. This technique is deliberately employed over a standard Stochastic or RSI to filter out market noise and reduce false signals, thereby improving the signal-to-noise ratio at the cost of some lag.

The script’s primary innovation is its adaptive parameterization; it automatically lengthens its lookback periods on higher timeframes (e.g., Daily, Weekly) and shortens them on lower ones (e.g., 15m, 1h). This serves as an intelligent, built-in filter, ensuring the indicator's sensitivity is appropriate for the inherent volatility and cycle length of the timeframe being analyzed.

The catalyst that flips the script from "observing" to "executing" is the confluence of two events: the SMI residing in an extreme zone (overbought/oversold) and its subsequent crossover of its signal line. This crossover confirms a definitive shift in momentum, providing a high-conviction trigger for a mean reversion trade.
    