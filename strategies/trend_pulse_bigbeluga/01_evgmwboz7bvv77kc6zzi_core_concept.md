
# Core Concept

### 1. The Market Philosophy

This script’s core philosophy is rooted in **asymmetrical momentum and trend persistence**. It operates on the theory that trends are not created equal; the conditions that initiate a bearish regime are structurally different from those that signal a bullish reversal. The strategy posits that a downtrend begins with a clear, objective break of market structure (a failed support), a moment of capitulation. Conversely, a new uptrend requires overcoming a dynamic, adaptive resistance that grows stronger as the preceding downtrend matures. This captures the psychological principle that fear (selling) is often sudden and sharp, while confidence (buying) rebuilds more gradually against persistent selling pressure. The model seeks to generate alpha by correctly identifying these distinct phases and avoiding premature entries against an established trend.

### 2. The Trade Narrative

The script anticipates two distinct market stories. The bearish narrative begins after a period of upward or sideways movement establishes a confirmed swing low—a clear floor of support. The "setup" is the erosion of buying pressure, culminating in a decisive, confirmed close *below* this structural pivot. This event is interpreted not as a mere dip, but as a fundamental break in market structure, signaling a high probability of a sustained move lower. The bullish narrative unfolds during the subsequent downtrend. As prices fall, an adaptive resistance band forms overhead. The setup for a long entry is a powerful resurgence of buying momentum, strong enough to break *above* this dynamic ceiling, which has become progressively harder to breach over time.

### 3. Trigger Logic & Mechanics

The strategy employs a dual-trigger mechanism to define its asymmetrical approach.

*   **Bearish Trigger:** A `pivotlow()` break serves as the catalyst. This is a pure price action signal, chosen for its objectivity. By waiting for a confirmed structural failure, the script aims to filter out noise and only engage when there is a high-conviction shift in control from buyers to sellers.

*   **Bullish Trigger:** A crossover of an adaptive Simple Moving Average (SMA) combined with an ATR-based offset. The "Why" here is crucial: the SMA's lookback period *increases* the longer the downtrend persists. This makes the resistance band less sensitive over time, effectively raising the bar for a valid bullish reversal. This mechanic is designed to improve the signal-to-noise ratio by ignoring minor relief rallies and only recognizing a trend change when momentum is significant enough to overcome a mature, established downtrend's inertia. The flip from "observing" to "executing" occurs precisely at these distinct breakout and breakdown points.
    