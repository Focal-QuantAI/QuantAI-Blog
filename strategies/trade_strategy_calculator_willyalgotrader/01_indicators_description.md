
# Indicators Description

### 1. Component Deconstruction

This script does not utilize standard time-series indicators (e.g., RSI, EMA). Instead, its "components" are user-defined statistical parameters and derived mathematical metrics that form a quantitative framework.

#### **A. Primary Input Components (User-Defined Parameters)**

These are the raw, static inputs that serve as the foundation for all subsequent calculations.

*   **`depositInput` (Float):**
    *   **Specific Configuration:** Default `1000.0`, `minval = 1`.
    *   **Functional Role:** Represents the total account equity. It is the principal amount upon which all risk and position sizing calculations are based.

*   **`riskPctInput` (Float):**
    *   **Specific Configuration:** Default `1.0`, `minval = 0.1`, `maxval = 100`.
    *   **Functional Role:** Defines the percentage of `depositInput` the user is willing to lose on a single trade. This is the core risk management parameter.

*   **`winrateInput` (Float):**
    *   **Specific Configuration:** Default `55.0`, `minval = 1`, `maxval = 99`.
    *   **Functional Role:** Represents the historical probability of a trade being profitable. It is a key variable in calculating expectancy and the Kelly Criterion.

*   **`avgRrInput` (Float):**
    *   **Specific Configuration:** Default `2.0`, `minval = 0.1`, `maxval = 50`.
    *   **Functional Role:** The average reward-to-risk ratio achieved on *winning trades only*. It quantifies the average magnitude of a win relative to the amount risked.

*   **`slPctInput` (Float):**
    *   **Specific Configuration:** Default `1.0`, `minval = 0.01`, `maxval = 50`.
    *   **Functional Role:** The stop-loss distance as a percentage of the entry price. This directly influences the calculated position size.

*   **`commissionInput` (Float):**
    *   **Specific Configuration:** Default `0.04`, `minval = 0`, `maxval = 5`.
    *   **Functional Role:** The percentage-based commission charged by the exchange for a single leg of a trade (either entry or exit).

*   **`slippageInput` (Float):**
    *   **Specific Configuration:** Default `0.05`, `minval = 0`, `maxval = 5`.
    *   **Functional Role:** An estimated percentage for order execution slippage. It is used to create a more conservative and realistic stop-loss distance.

*   **`leverageInput` (Integer):**
    *   **Specific Configuration:** Default `1`, `minval = 1`, `maxval = 125`.
    *   **Functional Role:** The leverage multiplier. Used to calculate the required margin and estimate the liquidation price for derivatives trading.

#### **B. Derived Computational Components (The Mathematical Engine)**

These are metrics calculated by combining the primary inputs. They represent the script's "hacked" or custom indicators.

*   **`riskAmount`:**
    *   **Mathematical Logic:** `depositInput * (riskPctInput / 100.0)`
    *   **Intended Effect:** Translates the abstract risk percentage (`riskPctInput`) into a concrete, absolute dollar amount. This value is the maximum permissible loss for the trade.

*   **`slPctAdj` (Adjusted Stop Loss Percentage):**
    *   **Mathematical Logic:** `slPctInput * (1.0 + slippageInput / 100.0)`
    *   **Intended Effect:** This is a non-standard modification that buffers the user-defined stop-loss percentage. It pre-emptively accounts for slippage by widening the effective stop distance used for position sizing, leading to a slightly smaller, more conservative position.

*   **`commFactor` (Commission Factor):**
    *   **Mathematical Logic:** `commissionInput / 100.0 * 2.0`
    *   **Intended Effect:** Converts the one-way commission percentage into a decimal factor and multiplies it by `2.0` to account for a full round-trip (entry and exit). This ensures total trading costs are correctly factored into position sizing and net profit calculations.

*   **`positionUsd` (Position Size in USD):**
    *   **Mathematical Logic:** `safeDiv(riskAmount, slPctAdj / 100.0 + commFactor)`
    *   **Intended Effect:** This is the core position sizing formula. It calculates the exact position size where a move to the stop-loss price results in a loss equal to `riskAmount`.
        *   **Denominator Breakdown:** The denominator `(slPctAdj / 100.0 + commFactor)` represents the *total percentage cost* of the trade if it is stopped out. It sums the slippage-adjusted stop-loss percentage and the round-trip commission percentage.
        *   **Mechanism:** By dividing the fixed dollar risk by the total percentage cost, the formula derives the notional value that aligns the two.

*   **`evPerTrade` (Expectancy Value per Trade):**
    *   **Mathematical Logic:** `(winrateInput / 100.0) * riskAmount * avgRrInput - (1.0 - winrateInput / 100.0) * riskAmount - totalComm`
    *   **Intended Effect:** This formula calculates the statistical average profit or loss per trade over a large sample size. It models the strategy's profitability by subtracting the expected loss and commissions from the expected gain. A positive EV indicates a profitable strategy.

*   **`kellyPct` (Kelly Criterion Percentage):**
    *   **Mathematical Logic:** `safeDiv(wr * avgRrInput - (1.0 - wr), avgRrInput) * 100.0` where `wr` is `winrateInput / 100.0`.
    *   **Intended Effect:** Implements the Kelly Criterion formula to determine the theoretically optimal fraction of the bankroll to risk per trade to maximize the long-term logarithmic growth of capital. The script uses this as a benchmark against the user's `riskPctInput`.

*   **`dRuinProb` (Probability of Ruin):**
    *   **Mathematical Logic:** `math.pow(safeDiv(1.0 - dWr / 100.0, (dWr / 100.0) * dRr), dBankrollUnits) * 100.0`
    *   **Intended Effect:** An implementation of the formula for Risk of Ruin. It estimates the probability of losing the entire bankroll, given the strategy's edge (`dEdge`) and the number of betting units the bankroll is divided into (`dBankrollUnits`, derived from `dRisk`). This serves as a critical portfolio stress test.

### 2. Logic Layering & Confluence

The script layers its calculations in a distinct hierarchy, where each layer builds upon the previous one to filter the user's decision-making process.

*   **Layer 1: Foundational Risk Definition**
    *   **Components:** `depositInput`, `riskPctInput`.
    *   **Interaction Dynamics:** These inputs establish the absolute risk capital (`riskAmount`). This is the foundational "gatekeeper" for the entire system. All subsequent calculations are constrained by this value.

*   **Layer 2: Trade-Specific Parameterization**
    *   **Components:** `slPctInput`, `slippageInput`, `commissionInput`, `entryPrice`.
    *   **Interaction Dynamics:** This layer takes the absolute risk from Layer 1 and combines it with the specific parameters of the proposed trade. The confluence of `riskAmount`, `slPctAdj`, and `commFactor` is used to compute the `positionUsd`. This is a **Hierarchical Filter**; the position size is entirely dependent on the risk defined in Layer 1.

*   **Layer 3: Strategy Viability Analysis (The "Go/No-Go" Filter)**
    *   **Components:** `winrateInput`, `avgRrInput`, `evPerTrade`, `kellyPct`, `beWinrate`.
    *   **Interaction Dynamics:** This layer assesses the mathematical soundness of the underlying strategy itself, independent of the single trade.
        *   **Threshold Cross:** The primary check is whether `evPerTrade > 0`. If expectancy is negative, the system signals a non-viable strategy.
        *   **Threshold Cross:** A secondary check is `winrateInput > beWinrate`. This confirms if the user's win rate is sufficient to overcome costs at the given R:R.
        *   **Confluence/Divergence:** The script analyzes the **Divergence** between the user's chosen `riskPctInput` and the calculated optimal `kellyPct`. This divergence is categorized ("Over-bet", "Optimal", "Conservative") to filter for improper bet sizing.

*   **Layer 4: Portfolio Stress Testing & Growth Projection**
    *   **Components:** `prob10Losses`, `dRuinProb`, `dFinalCompound`, `dMaxDdPct`.
    *   **Interaction Dynamics:** This final layer projects the consequences of the strategy over time and under duress. It acts as a psychological and strategic filter.
        *   **Threshold Cross:** The `dRuinProb` is compared against acceptable levels (e.g., < 5%). A high probability of ruin flags a strategy as too dangerous, even if it has a positive expectancy.
        *   **Stochastic Modeling:** The "Stress Test" section calculates the probability and drawdown of consecutive losses (`prob5Losses`, `ddAfter5`, etc.). This filters decisions by forcing the user to confront the statistical reality of drawdowns inherent in their strategy's `winrateInput`.

### 3. The Execution Engine

The script's "Execution Engine" is not automated; it is a computational core that processes manual inputs to produce analytical outputs. The "triggers" are the mathematical conditions that generate specific data points and visual warnings in the dashboard.

*   **Trigger Condition:** The entire engine is encapsulated within an `if barstate.islast` block. This ensures the full suite of calculations runs only once per script load/update, making it highly efficient. The primary trigger is any change to a user `input`.

*   **Boolean Logic & Signal Generation:**

    *   **Insufficient Margin Warning:**
        *   **Logic:** `marginOk = (marginRequired <= depositInput)` returns `false`.
        *   **Condition:** `safeDiv(positionUsd, leverageInput * 1.0) > depositInput`.
        *   **Effect:** Renders a prominent red banner with the text "🚨 INSUFFICIENT MARGIN".

    *   **Kelly Criterion Status:**
        *   **Logic:** A series of nested conditional checks on `kellyPct` and `riskPctInput`.
        *   **Conditions:**
            1.  `if kellyPct <= 0`: Returns "🔴 No edge".
            2.  `else if riskPctInput > kellyPct * 2.0`: Returns "🚨 >2x Kelly".
            3.  `else if riskPctInput > kellyPct`: Returns "🔴 Over-bet".
            4.  `else if riskPctInput >= halfKelly`: Returns "🟢 Optimal".
            5.  `else`: Returns "🟡 Conservative".
        *   **Effect:** Provides a color-coded status that immediately classifies the user's risk level relative to the mathematically optimal bet size.

    *   **Negative Expectancy Warning (Growth Section):**
        *   **Logic:** `dEvPct <= 0`.
        *   **Condition:** `(dWr / 100.0) * (dRisk / 100.0 * dRr) - (1.0 - dWr / 100.0) * (dRisk / 100.0) <= 0`.
        *   **Effect:** Prevents calculation of positive growth metrics (e.g., "Days to x2") and instead displays a "⚠️ Negative EV!" warning, acting as a hard stop for growth forecasting.

*   **Mathematical Constants & Their Influence:**

    *   **`2.0` (in `commFactor`):**
        *   **Significance:** A hard-coded multiplier representing a **round-trip trade** (one entry, one exit).
        *   **Influence:** Ensures that the total cost of trading is not underestimated. Doubling the one-way commission provides a more accurate denominator for the `positionUsd` calculation and a more realistic `evPerTrade`, directly impacting the risk profile.

    *   **`0.95` (in `liqPrice`):**
        *   **Significance:** A hard-coded multiplier used to approximate the liquidation price. The formula is `entryPrice * (1.0 - 0.95 / leverageInput)`.
        *   **Influence:** This constant assumes that liquidation occurs when losses equal ~95% of the margin used, leaving a small buffer (5%) for the exchange's maintenance margin requirements. It provides a conservative, though not exchange-guaranteed, estimate of the liquidation price, directly influencing the trader's assessment of leverage risk.

    *   **`1.5` (in `dWorstStreak`):**
        *   **Significance:** A heuristic multiplier applied to the expected number of losses per day (`dExpLossPerDay`).
        *   **Influence:** It creates an estimate for a "worst-case" losing streak that is 50% larger than the statistical average. This is not a rigorous calculation but a rule-of-thumb designed to force a more conservative estimation of maximum potential drawdown (`dMaxDdPct`), thereby influencing the user's perception of the strategy's volatility and risk.

    *   **`23.0` (in `dLogGrowth` check):**
        *   **Significance:** A hard-coded threshold to check for potential floating-point overflow errors within Pine Script. `math.pow` can return `na` for very large exponents, and `math.log(very_large_number)` can exceed this approximate limit.
        *   **Influence:** This is a technical constraint handler. If the projected logarithmic growth is too extreme, it prevents the `dFinalCompound` calculation from failing. Instead of an error, it displays "∞ (overflow)", managing the user experience and acknowledging the limitations of the platform's mathematical precision for exponential projections.
