
# Core Concept

### 1. The Market Philosophy

This algorithm operates on a classic **Momentum** philosophy, specifically designed to capture continuations within an established, macro-level trend. Its core thesis is built on the principle of **Trend Persistence**, which posits that an asset's price is more likely to continue in its current direction than to reverse. The strategy aims to generate alpha by identifying periods where short-term price action realigns with the dominant, longer-term market flow. It systematically filters out counter-trend noise, seeking to engage only when there is a high degree of directional agreement across multiple timeframes, thereby exploiting the behavioral tendency of market participants to pile into a strengthening move.

### 2. The Trade Narrative

The script is engineered to act on a specific market story: "a pullback to the mean within a confirmed trend." For a long entry, the narrative begins with the market already in a clear uptrend, defined by a "golden cross" of a fast and slow EMA cloud. Within this bullish environment, the price experiences a temporary dip or consolidation, enough to flip the more sensitive Supertrend indicator bearish. The ideal setup is this brief period of weakness. The trade is not triggered on the dip itself, but on the *resumption* of strength—the precise moment the Supertrend flips back to bullish, signaling that the pullback has likely concluded and the primary uptrend is reasserting control.

### 3. Trigger Logic & Mechanics

The strategy’s execution logic relies on a confluence of two distinct trend-following indicators to enhance its signal-to-noise ratio.

*   **The EMA Cloud (50/150):** This serves as the foundational "regime filter." By requiring the fast EMA to be above the slow EMA for buys (and vice-versa for sells), it establishes the strategic, long-term directional bias. It answers the question, "What is the market's dominant intention?"
*   **The Supertrend (ATR-based):** This acts as the tactical entry trigger. Its flip provides a volatility-adjusted signal confirming a potential shift in short-term momentum.

The catalyst is the **confluence** of these two elements. A Supertrend flip is ignored unless it aligns with the broader EMA cloud regime. This dual-filter approach is designed to prevent entries on minor counter-trend bounces that often lead to whipsaws and increase the drawdown profile. The entry is taken at the close of the signal bar, with fixed dollar-based targets, suggesting an application for instruments with consistent point values, like futures.
    