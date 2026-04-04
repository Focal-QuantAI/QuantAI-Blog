
# Core Concept

### 1. The Market Philosophy
This strategy is a classic **Volatility Breakout** model. It operates on the economic principle that markets cycle between periods of low-volatility consolidation (equilibrium or range-bound trading) and high-volatility expansion (trending). The core thesis is that a compression of price range signifies building market energy and order accumulation. When this balance is broken, the subsequent move is likely to be sharp and directional. The strategy aims to capture the alpha generated during this transition from indecision to a new, short-term consensus, exploiting the initial thrust of momentum as trapped traders are forced to liquidate and new participants chase the move.

### 2. The Trade Narrative
The script waits for a specific market story to unfold: a period of quiet consolidation. The ideal setup is a tight, sideways channel where price action is visibly "coiling." Volatility, as measured by the Average True Range (ATR), must be suppressed, indicating a temporary truce between buyers and sellers. Within this quiet environment, the script identifies the immediate structural boundaries—the most recent significant pivot high and pivot low. The narrative culminates when price decisively breaks and closes beyond one of these established levels, signaling that the equilibrium has been shattered and a new directional impulse is underway. This breakout must also form a higher low for a long entry (or a lower high for a short), confirming the structural integrity of the move.

### 3. Trigger Logic & Mechanics
The trigger is a confluence of three distinct logical conditions designed to enhance the signal-to-noise ratio.

*   **Structure & Volatility Confluence:** The strategy combines `pivots` with an `ATR`-based range filter. Pivots define the key structural levels to watch, while the ATR filter ensures the breakout originates from a low-volatility base. This tandem use is critical; it prevents the strategy from firing on random noise in a choppy, high-volatility environment and focuses its capital on "clean" breaks from consolidation.
*   **Confirmation Filter:** The entry logic requires a bullish breakout to be accompanied by a higher low (`low > lastPivotLow`), and vice-versa for shorts. This serves as a crucial confirmation, filtering out false breakouts or "head fakes" where price pierces a level but the underlying structure does not support a continued move.
*   **Catalyst:** The catalyst is the closing price crossing a previously established pivot high or low *after* the consolidation and confirmation conditions are met. This specific event, combined with a cooldown period to prevent over-trading, flips the script from passive observation to active execution, with a predefined risk-reward structure based on the breakout's volatility.
    