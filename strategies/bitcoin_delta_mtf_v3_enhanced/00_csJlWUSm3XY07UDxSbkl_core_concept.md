
# Core Concept

### 1. The Market Philosophy

This script operates on a sophisticated **Order Flow Imbalance** philosophy. It is a hybrid model designed to capture alpha from both **Momentum** and **Mean Reversion** scenarios. The core thesis is that the "footprints" of institutional capital are visible in the relationship between price, volume, and delta (net buying/selling pressure). The strategy aims to identify moments of either overwhelming, trend-aligned order flow (Momentum) or significant divergence between price action and underlying flow, which signals participant exhaustion and an impending reversal (Mean Reversion). It profits by detecting when aggressive market participants are either firmly in control or are being absorbed by passive, larger players, creating high-probability inflection points.

### 2. The Trade Narrative

The script seeks two primary narratives. The first is a **Momentum Confluence** story: a higher timeframe (e.g., 60m) establishes a clear directional bias, which is then confirmed by strong, aligned delta on the primary and lower timeframes. This paints a picture of a healthy, validated trend with broad participation. The second narrative is one of **Counter-Execution**: price makes a new high, but the underlying delta is weak or negative, and intrabar analysis reveals high volume with little price change ("Absorption"). This tells a story of smart money distributing positions into retail FOMO, or accumulating at lows as panic selling is absorbed, setting the stage for a sharp reversal against the prevailing short-term move.

### 3. Trigger Logic & Mechanics

The strategy’s engine is a multi-timeframe (MTF) weighted delta calculation, which provides a hierarchical view of market conviction. This is not a simple indicator cross; it’s a system of **confluence and veto**.

*   **Why these indicators?** The MTF delta establishes context, while granular, intrabar Cumulative Volume Delta (CVD) and Point of Control (POC) analysis provide real-time insight into the current candle's auction process. This combination allows the script to differentiate between a genuine breakout and a terminal price spike that is being faded by large players. VWAP is used as an institutional benchmark to contextualize price as over-extended or reverting to a mean.

*   **How do filters work?** The script employs powerful filters to enhance its signal-to-noise ratio. For instance, a "Hidden Signal" (price moving opposite to volume-weighted pressure) or an "Absorption" signal can veto a primary momentum entry, preventing trades into bull or bear traps.

*   **The Catalyst:** The script flips from "observing" to "executing" when a primary delta signal surpasses a threshold *and* is validated by a high "Composite Confidence" score. This score synthesizes nine weighted factors—including trend alignment, momentum acceleration, and the absence of exhaustion signals—into a single, normalized percentage. A high score represents the critical mass of evidence needed to confirm the trade narrative and trigger an execution.
    