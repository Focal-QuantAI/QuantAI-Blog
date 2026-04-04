
# Core Concept

### 1. The Market Philosophy
This strategy is a sophisticated expression of **Mean Reversion**, designed to capitalize on investor exhaustion at price extremes. Its core thesis is that when an asset’s price aggressively overshoots multiple, distinct measures of its typical trading range, it is likely the result of a liquidity hunt or a stop-loss cascade, not a new trend. Such moves are often unsustainable. The script wagers that this overextension will trigger a sharp "snap-back" reaction as counter-trend participants step in. It seeks to capture alpha not from following a trend, but from systematically fading the climactic, failed attempts to break established support or resistance.

### 2. The Trade Narrative
The script is engineered to identify a specific market story: the "failed breakdown" or "failed breakout." The ideal setup is a period where price action leads to a confluence of three distinct support or resistance zones—a historical price level, a statistical deviation band (Bollinger), and a dynamic volatility channel. The narrative begins when a single candle exhibits a sudden, high-velocity spike that pierces through this entire trifecta of boundaries. The critical plot twist, however, is the rejection. The candle must fail to hold these new levels and instead close back inside the established zones, signaling that the aggressive move has run out of momentum and a reversal is imminent.

### 3. Trigger Logic & Mechanics
The strategy’s precision comes from its multi-layered confirmation process, which dramatically improves the signal-to-noise ratio.

*   **Why these indicators?** The use of static Support/Resistance, Bollinger Bands, and the custom "Quantum" channel creates a powerful confluence. A signal is only generated when price violates all three simultaneously, filtering out weaker, less significant price fluctuations that might only breach a single indicator.
*   **How do filters work?** The primary filter is the strict requirement for a "touch and reject" on a single bar. The `low` must pierce the support confluence, but the `close` must be back above it. This confirms both the overextension and the immediate reversal pressure. An optional higher-timeframe (MTF) trend filter adds a strategic overlay, ensuring these mean-reversion scalps are taken in alignment with the broader market direction, thus avoiding attempts to catch a falling knife in a strong downtrend.
*   **The Catalyst:** The script transitions from "observing" to "executing" at the moment a candle closes. The piercing of the bands during the candle's formation is the setup; the close back inside the bands is the definitive trigger, confirming the failure of the move and validating the entry.
    