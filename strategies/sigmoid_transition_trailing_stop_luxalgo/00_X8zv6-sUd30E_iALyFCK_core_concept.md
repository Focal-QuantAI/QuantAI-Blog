
# Core Concept

### 1. The Market Philosophy

This script operates on a **Momentum** philosophy, specifically designed for trend persistence and optimized risk management. Its core thesis is that once a directional trend is established, it is more likely to continue than to reverse. The strategy aims to capture the majority of this move by employing a dynamic, volatility-adaptive trailing stop. The "raison d'être" is not merely to identify a trend but to intelligently manage the trade *within* the trend. It seeks to solve the classic momentum trader's dilemma: how to let profits run without giving back too much during a sharp reversal. The script’s unique sigmoid adjustment mechanism is an attempt to dynamically optimize the risk/reward profile as a trend matures and accelerates.

### 2. The Trade Narrative

The script engages when a market regime change is confirmed. The "story" begins with the definitive failure of the prior trend. For a long entry, the narrative is one of price grinding through a period of bearish control, defined by a descending trailing stop positioned above the price. The setup culminates when price action gathers sufficient force to close decisively *above* this bearish stop level. This crossover is not just a signal; it's the narrative climax, signifying a transfer of power from sellers to buyers. At this point, the script declares a new bull trend, establishes a fresh trailing stop below the price, and prepares to ride the anticipated upward momentum.

### 3. Trigger Logic & Mechanics

The strategy’s engine is a volatility-adaptive trailing stop based on the Average True Range (ATR). Using ATR allows the stop distance to expand in volatile markets (reducing whipsaws) and contract in quiet ones (protecting gains), thus improving the signal-to-noise ratio.

The primary catalyst for a trend flip is simple: a price close across the established stop line. However, the script’s alpha lies in its secondary trigger: the **Sigmoid Transition**. When price accelerates strongly and the gap between price and the trailing stop exceeds a predefined ATR multiple, the script initiates a "catch-up" phase. Instead of a linear step, it uses a sigmoid function to smoothly and rapidly move the stop closer to the price over a fixed number of bars. This non-linear adjustment aggressively locks in profits during periods of high momentum, aiming to create a more favorable drawdown profile by reducing the distance a profitable trade can retrace before being stopped out.
    