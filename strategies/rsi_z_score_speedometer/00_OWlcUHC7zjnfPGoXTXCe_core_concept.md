
# Core Concept

### 1. The Market Philosophy
This script is fundamentally rooted in a **Mean Reversion** philosophy. It operates on the principle that asset prices, while prone to trending, exhibit a strong tendency to revert to a statistical equilibrium after extreme deviations. The strategy seeks to generate alpha by identifying points of investor exhaustion or speculative excess. The underlying thesis is that moves that are both rapid (high RSI) and statistically unusual (high Z-Score) are unsustainable. These conditions create a "stretched rubber band" effect, where the probability of a corrective price swing back towards the mean becomes significantly elevated.

### 2. The Trade Narrative
The ideal setup for this tool is a market that has just experienced a sharp, directional impulse, creating a potential climax condition. The narrative is one of overextension: price has moved significantly and rapidly away from its recent average, pushing momentum indicators to their limits. This could be a parabolic rise in an uptrend or a capitulatory sell-off in a downtrend. The script is not looking for a trend to join; it is hunting for the moment a trend becomes exhausted and vulnerable to a reversal. It visualizes the precise moment when price is statistically "far from home," suggesting an imminent pullback.

### 3. Trigger Logic & Mechanics
This script functions as a sophisticated dashboard, providing a confluence of two distinct but complementary measures of market extremity.

*   **The RSI Speedometer** acts as a momentum gauge, answering the question: "How fast and aggressive was the recent price move?" It uses the classic Relative Strength Index to quantify momentum exhaustion, with readings above 70 or below 30 signaling overbought or oversold conditions.
*   **The Z-Score Speedometer** provides statistical context, answering: "How unusual is the current price relative to its recent history?" By calculating the number of standard deviations the price is from its moving average, it filters for statistically significant deviations that are likely unsustainable.

The "trigger" is not an automated signal but a visual confluence for the discretionary trader. When both needles point to an extreme (e.g., RSI in "Strong Sell" and Z-Score above +2σ), it provides a high-conviction signal that the market is overextended. This dual-filter approach dramatically improves the signal-to-noise ratio, helping to distinguish genuine exhaustion from simple, healthy trend momentum.
    