
    # Core Concept

    ### 1. The Market Philosophy

This strategy operates on a **Momentum** philosophy, specifically designed to capture the inception of new intraday trends. Its core thesis is that a failed reversal attempt, confirmed by a decisive breakout, signals a powerful shift in market sentiment and the beginning of a directional move. The script is not chasing existing trends but rather identifying the precise moment of trend birth following a period of conflict between buyers and sellers. The underlying principle is that the exhaustion of one side (e.g., sellers failing to push prices lower) followed by an aggressive counter-attack (buyers breaking a recent high) provides a high-probability entry point to ride the subsequent wave of momentum. This approach seeks to generate alpha by entering before the new trend is obvious to the broader market.

### 2. The Trade Narrative

The script is programmed to identify a specific three-act play on the 5-minute chart, unfolding over a 15-minute block. For a buy signal, the story begins with a bearish candle, indicating seller presence. This is immediately challenged by a bullish second candle, signaling a fight for control. The narrative culminates with a third, strong bullish candle that not only confirms buyer dominance but also closes decisively above the high of the initial bearish candle. This sequence tells a story of a failed bearish push being aggressively absorbed and reversed. The strategy remains passive until this entire narrative of "attempted move, struggle, and decisive reversal breakout" is complete, providing a clear setup for a long position.

### 3. Trigger Logic & Mechanics

The strategy’s execution logic is a study in confluence, using pure price action instead of lagging indicators to enhance its signal-to-noise ratio.

*   **Pattern Recognition:** The core is a three-candle reversal pattern (e.g., Red-Green-Green for a buy). This acts as the initial filter, identifying a potential shift in micro-structure.
*   **Timing Filter:** The logic is constrained to trigger only on the close of the third candle in a 15-minute sequence. This ensures the full "story" has unfolded, preventing premature entries during market indecision.
*   **Breakout Catalyst:** The final and most critical trigger is the breakout confirmation. A buy signal requires the close of the third candle to exceed the high of the first candle in the pattern. This is not just a change in color; it's a structural break that validates the new momentum and serves as the catalyst, flipping the script from "observing" to "executing." The automatically calculated stop-loss, placed at the low of the three-candle formation, provides a structurally sound and volatility-adjusted risk parameter, defining the strategy's drawdown profile on a per-trade basis.
    