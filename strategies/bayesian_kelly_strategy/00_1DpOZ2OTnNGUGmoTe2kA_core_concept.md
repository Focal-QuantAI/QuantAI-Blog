
# Core Concept

### 1. The Market Philosophy

This strategy operates on a sophisticated **Statistical Momentum** philosophy, grounded in Bayesian inference. Its core thesis is that an asset's future return can be more accurately predicted by blending its long-term historical performance (the "prior belief") with its more recent performance (the "evidence"). The strategy seeks to exploit the persistence of trends, but not in a naive way. Instead of simple price-based momentum, it models the asset's return distribution. The underlying principle is that markets are not perfectly efficient; recent information (the last quarter's returns) provides a statistically significant update to our understanding of an asset's current drift, allowing for the capture of alpha by systematically allocating capital towards assets with a strengthening return profile.

### 2. The Trade Narrative

The script is not looking for a classic chart pattern but for a specific statistical "story" to unfold. The ideal setup is a market where an asset's recent returns (e.g., over the last 60 days) become demonstrably positive and, crucially, exhibit relatively low variance. This stability is key, as it lends higher "credibility" or "precision" to the recent evidence. The narrative is one of emerging, non-chaotic strength. The strategy engages when this new, positive evidence is strong enough to favorably update the asset's long-term (e.g., 252-day) return profile. It is designed to systematically increase exposure as statistical conviction in a positive future return grows.

### 3. Trigger Logic & Mechanics

The strategy’s engine is a precision-weighted forecast of returns. It eschews traditional indicators for first-principle statistics:

*   **Indicator Confluence:** It combines a long-term Simple Moving Average (SMA) and Variance of log-returns (the "Prior") with a short-term SMA and Variance (the "Evidence"). The "why" is to create a blended forecast where the weight given to each period's average return is inversely proportional to its variance. A stable, low-variance trend is trusted more than a volatile, noisy one.
*   **Noise Filtration:** The Bayesian weighting itself is the primary noise filter. A volatile, high-variance period of returns will have its influence automatically down-weighted, improving the signal-to-noise ratio. Furthermore, the periodic rebalancing (`interval`) acts as a temporal filter, preventing the strategy from overreacting to daily noise and reducing transaction costs.
*   **Execution Catalyst:** The script is always observing, continuously calculating the optimal leverage (`f_bayes`) based on the Kelly Criterion. However, the catalyst that flips it from "observing" to "executing" is the periodic rebalance trigger (`bar_index % interval == 0`). Only on these specific bars does it compare its target position to its actual position and issue orders to close the gap, ensuring a disciplined, systematic implementation.
    