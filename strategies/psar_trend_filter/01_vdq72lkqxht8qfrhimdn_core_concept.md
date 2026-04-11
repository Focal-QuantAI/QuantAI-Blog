
# Core Concept

### 1. The Market Philosophy
This script operates on a **Filtered Momentum** philosophy. Its core thesis is that while the Parabolic SAR (PSAR) is effective at identifying trend direction, its raw signals are prone to generating significant noise and false positives in non-trending or weakly trending markets. The strategy aims to generate alpha not by inventing a new signal, but by systematically filtering out the low-probability trades inherent in a basic PSAR system. It presupposes that profitable trends are not only directional but also possess sufficient strength and duration. By layering filters for long-term trend alignment (EMA) and trend intensity (ADX), the script seeks to engage only when the probability of trend persistence is statistically high, thereby improving the signal-to-noise ratio.

### 2. The Trade Narrative
The script is designed to wait for a specific market story to unfold before acting. This narrative begins with the establishment of a clear, macro-directional bias, where price is trading decisively above or below a 50-period EMA. This sets the stage. The second act requires the market to demonstrate conviction; the ADX must rise above a minimum threshold, signaling that the trend is not lazy or choppy but is backed by genuine momentum. Only when this environment of a strong, confirmed trend is present does the script look for its entry cue: a PSAR flip in the direction of the macro trend. This flip represents a tactical opportunity to join a move that is already in progress and has demonstrated its validity.

### 3. Trigger Logic & Mechanics
The strategy’s execution logic is a cascade of confirmations designed to achieve high-conviction entries.

*   **Core Signal & Context:** The PSAR flip provides the initial, raw entry signal. However, this signal is immediately qualified by a 50-period EMA. This confluence ensures the trade is aligned with the broader market current, preventing the system from fighting powerful underlying trends.
*   **Strength & Regime Filter:** The ADX filter is the most critical component for managing the drawdown profile. By requiring an ADX value above 15, the script acts as a regime filter, effectively deactivating itself during low-volatility, range-bound conditions where momentum strategies typically underperform.
*   **Final Catalyst:** The confluence of a PSAR flip, alignment with the EMA, and confirmation of trend strength from the ADX serves as the final catalyst. This multi-factor agreement transforms the script from a passive observer into an active participant, executing a trade only when a tactical entry point appears within a strategically favorable market structure.
    