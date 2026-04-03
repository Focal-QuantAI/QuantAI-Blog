
# Core Concept

### 1. The Market Philosophy

This script operates on a philosophy of **Structural Analysis and Price Action Context**. It eschews predictive indicators in favor of identifying the most fundamental battlefield on any chart: the immediate, visible range of price. The underlying principle is that the highest high and lowest low within a trader's current field of view represent critical psychological inflection points. These levels are imbued with the memory of recent peak optimism (the high) and peak pessimism (the low). They naturally attract liquidity and serve as reference points for profit-taking, stop-loss placement, and new entries, making them potent zones for potential mean reversion or breakout activity. The script’s raison d’être is not to predict direction but to objectively frame the current market structure for a discretionary trader.

### 2. The Trade Narrative

The script is designed to activate when a trader is assessing the immediate tactical landscape. The "story" it highlights is the formation of a localized trading range. It visually answers the question: "Within the price action I can currently see, where have buyers been definitively rejected, and where have sellers been exhausted?" The ideal setup is a chart where price is oscillating between two clear extremes. By drawing these boundaries, the script creates a visual narrative of containment. It tells the trader that, for the observable period, all market activity has been confined within these two price points, setting the stage for strategies that either fade these levels or anticipate a violent break from them.

### 3. Trigger Logic & Mechanics

This script is a decision-support tool, not an automated execution system; its "trigger" is the trader's own analysis, informed by the script's output.

Its core mechanic is a highly efficient, event-driven loop that executes only on the last bar of the chart (`barstate.islast`). This minimizes computational overhead. The script then iterates backward through the price bars, checking only those within the visible screen area (`chart.left_visible_bar_time` to `chart.right_visible_bar_time`). It performs a direct price comparison to find the absolute high and low within this dynamic window. The catalyst that flips the script from "observing" to "drawing" is this completed scan. It then dynamically plots horizontal lines at these discovered price levels, providing a real-time, adaptive framework of support and resistance that automatically adjusts as the user pans or zooms the chart.
