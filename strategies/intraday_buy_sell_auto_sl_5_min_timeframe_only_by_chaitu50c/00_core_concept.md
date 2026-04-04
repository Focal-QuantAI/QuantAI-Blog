
# Core Concept

### 1. The Market Philosophy

This strategy operates on a **Momentum Reversal** philosophy, specifically tailored for intraday price action. It posits that significant intraday trend changes are not random but often follow a predictable, short-term pattern of exhaustion and confirmation. The core thesis is that a failed attempt by one side of the market (e.g., a bearish push) followed by a strong, confirmed counter-attack from the other side provides a high-probability entry to capture the initial thrust of a new micro-trend. It seeks to generate alpha by identifying the precise moment sentiment shifts, confirmed by a breakout from a recently established micro-structure, thereby front-running the subsequent momentum wave.

### 2. The Trade Narrative

The script is designed to remain dormant until a specific three-candle "story" unfolds over a 15-minute period (three 5-minute bars). For a long entry, the narrative begins with a bearish candle, signaling seller presence. This is immediately followed by a bullish candle, indicating a fight for control. The climax occurs with a third, consecutive bullish candle that not only confirms buyer dominance but also achieves a breakout by closing above the high of the initial bearish candle. This sequence tells a story of seller exhaustion, buyer absorption, and a confirmed momentum shift. The script interprets this as a validated setup to go long, anticipating further upside. The inverse narrative (Green-Red-Red with a breakdown) triggers a short entry.

### 3. Trigger Logic & Mechanics

The strategy’s elegance lies in its minimalist, price-action-based logic, eschewing traditional lagging indicators for raw candlestick analysis.

*   **Confluence of Pattern & Breakout:** The trigger is not merely a candle pattern (e.g., Red-Green-Green) but the **confluence** of this pattern with a decisive volatility breakout. The requirement for the third candle's close to exceed the first candle's high (for a buy) acts as a critical filter. This ensures the script isn't just trading a pattern but a pattern validated by demonstrable momentum, significantly improving the signal-to-noise ratio.

*   **Timing Filter:** The logic is hard-coded to evaluate only on the close of every third 5-minute candle. This temporal discipline forces patience, ensuring the full 15-minute narrative has concluded before a decision is made, which reduces noise from intra-bar fluctuations.

*   **Catalyst:** The catalyst that flips the script from "observing" to "executing" is the simultaneous fulfillment of the three-bar color sequence and the price breakout. This dual condition serves as the definitive signal that a tactical reversal is in play, warranting capital deployment with a structurally-defined stop-loss at the nadir of the three-bar formation.
