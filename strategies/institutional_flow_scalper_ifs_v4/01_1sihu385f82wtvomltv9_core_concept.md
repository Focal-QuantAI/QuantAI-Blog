
# Core Concept

### 1. The Market Philosophy

The "Institutional Flow Scalper" operates on a **Counter-Momentum / Liquidity Hunting** philosophy. Its core thesis is that institutional players frequently engineer short-term price moves to hunt liquidity resting above recent swing highs or below swing lows. This action, known as a "liquidity sweep" or "stop hunt," is designed to trigger stop-loss orders and entice breakout traders, creating the necessary volume for institutions to enter or exit large positions. The script aims to identify the climax of this manufactured momentum, betting on the subsequent sharp reversal as the engineered move exhausts itself. It seeks to exploit the psychological principle of retail trader capitulation, entering precisely when weak hands are forced out and smart money reveals its true intention.

### 2. The Trade Narrative

This strategy waits for a specific story of a failed breakout. The ideal setup begins with price action pushing aggressively beyond a well-defined recent swing high or low. This move creates the illusion of a strong trend continuation, drawing in momentum traders. However, the script identifies this as a potential trap by cross-referencing the price action with underlying volume dynamics. The narrative it seeks is this: price makes a new extreme, but the underlying buying or selling pressure (measured by Cumulative Volume Delta) fails to confirm the move, creating a divergence. Alternatively, the price spike is met with immense volume but fails to close with conviction, indicating large-scale order absorption. This confluence of events signals that the initial move was a liquidity grab, not a genuine breakout, setting the stage for a rapid reversal.

### 3. Trigger Logic & Mechanics

The script’s trigger mechanism is built on a principle of **confluence**, using a "Confidence Score" to quantify the strength of a setup. It avoids relying on a single indicator, instead requiring multiple "pillars" of evidence to align.

*   **Core Catalyst:** The primary signal is the combination of a **Liquidity Sweep** with either a **CVD Divergence** or an **Order Absorption** event. The sweep identifies the *location* of a potential reversal, while the volume-based indicators confirm the *weakness* of the move, suggesting it's a trap.
*   **Contextual Filters:** The **VWAP** and its standard deviation bands act as a gravitational guide, providing a mean-reversion target and contextualizing the price's location relative to the session's average. A sweep far from the VWAP that shows signs of reversing back towards it is a high-conviction setup.
*   **Noise Reduction:** An **Anti-Chop Filter** and **Session Awareness** act as critical regime filters. They improve the signal-to-noise ratio by deactivating the strategy during low-liquidity hours or in directionless, choppy markets where these patterns are unreliable. The final trigger requires at least two pillars to be present and the overall confidence score to exceed a minimum threshold, flipping the script from observation to execution.
    