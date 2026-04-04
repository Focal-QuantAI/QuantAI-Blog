
# Core Concept

### 1. The Market Philosophy
This script's core philosophy is rooted in **Momentum** and the principle of **trend persistence**. It operates on the thesis that once a directional move gains sufficient velocity, it is more likely to continue than to reverse. The script’s *raison d’être* is not merely to identify a trend but to dynamically optimize the risk-reward profile within it. It addresses a classic weakness of passive trailing stops: their tendency to lag significantly in fast-moving markets, giving back substantial profit on a reversal. By adaptively tightening the stop only when momentum is confirmed, it seeks to protect unrealized gains more efficiently, thus improving the strategy's overall alpha.

### 2. The Trade Narrative
The script is designed to react to a specific market story: the **acceleration phase of a maturing trend**. The initial setup is a standard trend flip, where price crosses the prior stop level. However, its unique logic triggers when the market demonstrates exceptional strength. The narrative it seeks is, "A new trend is not just holding; it's accelerating with conviction." This is identified when price rapidly pulls away from the trailing stop, creating a volatility-adjusted gap that is larger than normal. This widening gap is the explicit setup—interpreted not as risk, but as a signal of a high-quality trend, prompting the script to shift from passive following to active management.

### 3. Trigger Logic & Mechanics
The script’s intelligence lies in its dual-state mechanism. Initially, it functions as a standard ATR-based trailing stop for baseline trend definition. The **catalyst** that flips it into its active, "transition" state is when the price-to-stop distance exceeds a predefined volatility threshold. This event signifies a high signal-to-noise ratio, confirming the trend's strength.

Why the sigmoid function? Instead of a crude linear adjustment, the S-curve provides a smooth, non-disruptive tightening. It begins slowly (to avoid reacting to a minor spike), accelerates during the mid-point of the transition, and then decelerates as it approaches its new, tighter level. This sophisticated confluence—using ATR for volatility context and a sigmoid curve for dynamic adjustment—allows the strategy to aggressively defend profits during high-conviction moves without being prematurely stopped out by normal market breathing.
    