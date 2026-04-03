
# Core Concept

### 1. The Market Philosophy

This script operates on a foundational **Mean Reversion** and **Volatility Breakout** philosophy. Its core purpose is to dynamically identify the absolute high and low within the trader's current field of view, effectively framing the localized market structure. The underlying principle is that these price extremes represent significant psychological levels—points of maximum optimism (the high) and maximum pessimism (the low). These levels act as natural liquidity zones where either profit-taking will induce a reversal (mean reversion) or a decisive breach will signal a powerful shift in market conviction, leading to a new leg (breakout). The script provides an agnostic framework to exploit either outcome by making these critical inflection points explicit and objective.

### 2. The Trade Narrative

The script does not seek a complex narrative but rather establishes the stage for one. The "story" it highlights is the formation of a temporary, relevant trading range defined by the visible price action on the chart. It assumes that the most recent, visible history is the most pertinent for near-term decision-making. The ideal setup for a user of this tool is a market approaching one of these dynamically drawn boundaries. The narrative culminates as price interacts with the identified high or low, forcing a decision: is this a point of exhaustion where a fade trade offers high probability, or is it a point of capitulation where a breakout entry will capture the start of a new impulse?

### 3. Trigger Logic & Mechanics

This tool eschews traditional indicators for pure price action analysis. Its genius lies in its adaptive lookback period.

*   **Why these indicators?** It uses the most fundamental indicators: `high` and `low` prices. The primary filter is not a mathematical derivative like an RSI but the chart's visible time window (`chart.left_visible_bar_time`). This ensures the identified support and resistance levels are always relevant to the trader's current analysis, dramatically improving the signal-to-noise ratio by ignoring stale, irrelevant price history.

*   **How it reduces noise:** By confining its search to the visible bars, the script automatically filters out distant highs and lows that have no bearing on the current market structure, preventing analysis paralysis.

*   **The Catalyst:** The script itself does not execute trades. Its catalyst is the visual confluence of live price action approaching the automatically rendered high or low line. It flips from "observing" to "acting" (by drawing the lines) only on the last bar, an efficient method to provide real-time, non-repainting levels. This transforms subjective level-drawing into an objective, data-driven process, empowering the discretionary trader.
