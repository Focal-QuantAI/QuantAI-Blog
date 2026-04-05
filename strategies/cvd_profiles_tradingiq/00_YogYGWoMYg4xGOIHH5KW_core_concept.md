
# Core Concept

### 1. The Market Philosophy
This script operates on a philosophy of **Order Flow Momentum**. It posits that aggregate price action is a lagging indicator, while the net flow of aggressive market orders—quantified by Cumulative Volume Delta (CVD)—is the true leading indicator of market intention. The underlying principle is that significant price moves are not random but are driven by the sustained pressure of either buyers or sellers overwhelming the other. By dissecting volume into its aggressive buying and selling components at each price level, the strategy aims to detect the footprints of institutional activity and front-run the resulting price discovery. It seeks to capture alpha by identifying where control is shifting between buyers and sellers before it is fully reflected in price.

### 2. The Trade Narrative
The script is designed to identify a narrative of **initiative versus absorption**. The ideal setup occurs when a clear imbalance of aggressive orders builds at a key structural level. For a long entry, the story would be: price consolidates or pulls back to a support level, but the CVD Profile reveals that aggressive sellers are being absorbed by passive buyers, with initiative buyers stepping in, causing a strong positive CVD. This indicates that the selling pressure is failing to push prices lower and a reversal is likely. The script is looking for this divergence—weakness in price action but strength in underlying order flow—to signal a high-probability entry.

### 3. Trigger Logic & Mechanics
The script’s genius lies in its fusion of two powerful concepts: Cumulative Volume Delta and Market/Volume Profiling.

*   **Why these indicators?** It doesn't just use CVD as a simple oscillator; it projects the CVD onto a vertical histogram, creating a "CVD Profile." This answers not only "who is in control?" (the CVD) but, critically, "*where* are they in control?" (the profile). This provides crucial context that a simple line chart lacks.
*   **How do filters serve?** The Value Area (VA) and Point of Control (POC) of the CVD Profile act as dynamic filters. They improve the signal-to-noise ratio by focusing the analyst’s attention on the price levels where the most significant battles between buyers and sellers are occurring. A breakout of the CVD Value Area, for instance, is a much stronger signal than random fluctuations.
*   **The Catalyst:** The script flips from "observing" to "executing" not on a simple crossover, but on visual confluence. The trigger is the observation of a significant CVD imbalance (e.g., a high-magnitude reading in the "Imbalance Ratio" model) occurring at a strategic location, such as the edge of the value area or outside of a consolidation range, signaling that the auction is ready to move directionally.
    