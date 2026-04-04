
# Indicators Description

### 1. Component Deconstruction

The script's engine is built upon a custom pattern recognition framework rather than traditional oscillating studies. Its core components are a dual-layer pivot detection system and a set of mathematical constants used for validation.

#### **Pivot Detection Engine**

The script utilizes two instances of the built-in `ta.pivothigh` and `ta.pivotlow` functions to identify significant swing points in the price structure. These serve as the foundational data points for all subsequent wave construction.

*   **Primary Pivots (`primaryPivots`)**
    *   **Specific Configuration:**
        *   **Indicator:** `ta.pivothigh(high, 13, 13)` and `ta.pivotlow(low, 13, 13)`.
        *   **Lookback Period:** A `primarySwingLen` of **13 bars** to the left and right is required to confirm a pivot. This configuration is designed to identify major, structurally significant swing points.
        *   **Price Source:** `high` for pivot highs, `low` for pivot lows.
    *   **Functional Modification:** Raw pivots are subjected to a rigorous filtering mechanism before being stored in the `primaryPivots` array.
        *   **Alternation Filter:** A new pivot is only considered valid if it is of the opposite type to the preceding pivot (e.g., a high can only follow a low).
        *   **Volatility Filter:** A new pivot must represent a price movement of at least `minSwingPct` (**5.0%** by default) relative to the previous pivot's price. The function `pctMove(fromPrice, toPrice)` calculates this. This filter ensures that only swings of a minimum magnitude are considered, effectively filtering out minor price chop and adapting the analysis to the asset's volatility profile.
        *   **Repainting Mitigation:** The script includes logic to update the price and index of the last pivot if a new, higher high (or lower low) forms before an opposing pivot is confirmed. This ensures the pivot marks the true extremum of the swing.

*   **Secondary Pivots (`secondaryPivots`)**
    *   **Specific Configuration:**
        *   **Indicator:** `ta.pivothigh(high, 5, 5)` and `ta.pivotlow(low, 5, 5)`.
        *   **Lookback Period:** A `secondarySwingLen` of **5 bars** is used. This shorter period is designed to detect smaller, internal swings that constitute the sub-waves of the primary waves.
        *   **Price Source:** `high` and `low`.
    *   **Functional Modification:**
        *   **Volatility Filter:** A `minSubSwingPct` of **2.0%** is applied to filter noise from these more sensitive pivots.
        *   **Purpose:** The `secondaryPivots` array is not used for primary pattern construction. Its sole purpose is to serve the `countSubWaves` function, which provides a quantitative measure of a primary wave's internal complexity. This data is then used as a guideline (`subWaveCountValid`) to increase or decrease the confidence score of a detected pattern.

#### **Mathematical & Logical Constants**

The script does not use standard indicators but relies on a set of hard-coded mathematical ratios and rules derived from Elliott Wave theory.

*   **Fibonacci Ratios:**
    *   **`FIB_RETRACE`:** An array `[0.236, 0.382, 0.500, 0.618, 0.786]` used to validate corrective waves and project potential support/resistance zones.
    *   **`FIB_EXTEND`:** An array `[1.000, 1.272, 1.618, 2.000, 2.618, 4.236]` used to validate the length of impulse waves (e.g., Wave 3 relative to Wave 1) and project price targets.
*   **Cardinal Rules (Impulse Waves):** These are implemented as boolean functions that act as non-negotiable logical gates.
    *   `rule1Valid`: Wave 2 cannot retrace beyond the starting price of Wave 1.
    *   `rule2Valid`: The price length of Wave 3 cannot be the shortest among Waves 1, 3, and 5.
    *   `rule3Valid`: The price level of Wave 4's termination cannot enter the price territory of Wave 1 (overlap is disallowed).
*   **Guideline Heuristics:** These are softer rules that influence a pattern's confidence score.
    *   `checkAlternation`: Compares the character of Wave 2 and Wave 4. It classifies each correction as "sharp" (deep and fast) or "sideways" (shallow and time-consuming) and rewards patterns where the two are different.
    *   `subWaveCountValid`: Checks if the number of secondary pivots within a primary wave aligns with theory (e.g., motive waves should have more sub-pivots than corrective waves).

### 2. Logic Layering & Confluence

The script's intelligence lies in its multi-layered filtering process, which transforms raw price data into high-probability, validated patterns.

*   **Layer 1: Structural Data Generation (Pivots)**
    The foundational layer converts the continuous price stream into a discrete series of structurally significant swing points (`primaryPivots`) using the configured lookback periods and volatility filters. This dramatically reduces the signal-to-noise ratio from the outset.

*   **Layer 2: Geometric Template Matching**
    The engine takes the most recent sequence of 4 or 6 primary pivots and attempts to match them against predefined geometric templates:
    *   **6-Pivot Patterns:** Impulse, Diagonal, Triangle.
    *   **4-Pivot Patterns:** Zigzag, Flat.
    This is a purely structural check to see if the sequence of highs and lows conforms to the expected shape of a given pattern (e.g., an impulse wave requires an alternating sequence of `low-high-low-high-low-high` for a bullish move).

*   **Layer 3: Hierarchical Filtering & Validation**
    This is the core of the logic engine, where a geometrically plausible pattern is subjected to a hierarchy of tests.
    1.  **Cardinal Rule Gatekeeper:** The pattern is first tested against the three non-negotiable Elliott Wave rules (`validateImpulseFull`). If any rule is violated (e.g., Wave 4 overlaps Wave 1 in a non-diagonal), the pattern is immediately invalidated and rejected. This is the primary filter.
    2.  **Fibonacci Confluence:** If the cardinal rules pass, the script measures the Fibonacci relationships between the constituent waves. It checks for **confluence** with ideal ratios (e.g., does Wave 2 retrace to near 61.8% of Wave 1? Is Wave 3 a 1.618 extension of Wave 1?).
    3.  **Guideline Verification:** The script further validates the pattern against softer guidelines like wave alternation (`checkAlternation`) and sub-wave structure (`subWaveCountValid`).

*   **Layer 4: Confidence Scoring**
    Instead of a simple pass/fail, the script generates a quantitative `confidence` score.
    *   A pattern that passes only the cardinal rules receives a base score (e.g., `0.40` for an impulse).
    *   This score is then incrementally increased for each guideline and ideal Fibonacci ratio it satisfies. For example, a Wave 2 retracing to 61.8% adds `0.10` to the score, while a textbook Wave 3 extension to 1.618 adds `0.12`.
    *   This creates a robust system where "textbook" patterns that exhibit strong confluence across multiple metrics achieve a high confidence score (approaching 1.0), while technically valid but imperfect patterns receive a lower score.

### 3. The Execution Engine

The script's trade signals are not triggered by simple indicator crosses but are gated by the output of the entire pattern validation engine.

*   **Trigger Conditions**
    *   **Wave 3 Entry Signal (`newLongSignal` / `newShortSignal`):**
        *   **Boolean Logic:** The primary trigger is the function `detectWave3Entry`, which identifies a three-pivot sequence (p0, p1, p2) forming a potential Wave 1 and Wave 2. The core condition is that the retracement of the second leg (`p2.price`) must fall within the classic range of **38.2% to 78.6%** of the first leg.
        *   **Gatekeeper Logic:** This raw signal is then filtered by a crucial gate: `confirmedCtx.confidence >= 0.35`. A Wave 3 entry signal will **only** be plotted if the broader wave count has achieved a minimum confidence score of 35%. This ensures that the trade is taken within a context that the engine has already validated as a plausible Elliott Wave structure.

    *   **Wave C Reversal Signal:**
        *   **Boolean Logic:** This signal is triggered upon the completion of a confirmed corrective pattern (`zigzag` or `flat`). The logic is `confirmedCtx.confirmedPhase == "corrective"`.
        *   **Directional Logic:** The signal direction is opposite to the direction of the completed correction. If a bearish correction (`confirmedCtx.isBullish == false`) terminates at a low pivot (`!lastP.isHigh`), a `newLongSignal` is generated. This is a pure reversal trigger.

*   **Exit Conditions**
    *   **Wave 5 Exit Signal (`wave5ExitSignal`):**
        *   **Boolean Logic:** This signal requires two conditions to be met:
            1.  A 5-wave impulse pattern must be confirmed (`confirmedCtx.confirmedPattern == "impulse"`).
            2.  The length of the last move (Wave 5) must be less than 30% of the length of the move from the start of Wave 3 to the end of Wave 5 (`lastMove < totalMove * 0.3`).
        *   **Purpose:** This logic is designed to detect momentum exhaustion. A significantly shorter Wave 5 relative to the powerful Wave 3 is a classic sign that the impulse trend is weakening, providing a structural basis for taking profit.

*   **Mathematical Constants & Risk Profile**
    *   **`primarySwingLen = 13`:** This directly influences the time scale of the analysis. A larger value would identify longer-term waves, leading to fewer but more significant signals. A smaller value would increase sensitivity and signal frequency, likely at the cost of reliability.
    *   **`minSwingPct = 5.0`:** This is a critical risk and volatility parameter. It defines the minimum market "event" the script will pay attention to. On a stable asset, this value might be too high, causing the script to ignore valid patterns. On a volatile asset, it effectively filters out noise. It forces the strategy to focus on moves of a certain magnitude, implicitly affecting the potential risk-to-reward profile of the trades it identifies.
    *   **`confidence >= 0.35`:** This hard-coded threshold is the final arbiter of signal quality. It dictates the minimum level of "textbook perfection" required before the script will issue an alert. A higher threshold would result in very few, but theoretically very high-quality, signals. The default value of 0.35 represents a balance between signal frequency and strict pattern adherence.
    