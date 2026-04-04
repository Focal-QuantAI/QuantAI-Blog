
# Indicators to Strategy Blueprint

Here is the transformation of the "Sigmoid Transition Trailing Stop" indicator into a production-ready algorithmic trading framework.

### 1. Execution Triggers (Entry & Direction)

The provided script's core strength is its `direction` variable, which flips between `1` (Bullish) and `-1` (Bearish). This state change is the ideal trigger for our strategy. We will treat this as a "reversal" system, where a signal to go Long simultaneously closes any existing Short position, and vice-versa.

*   **Long Entry Condition:** A Long trade is initiated when the trend direction flips from Bearish to Bullish.
    *   `longCondition = direction == 1 and direction[1] == -1`

*   **Short Entry Condition:** A Short trade is initiated when the trend direction flips from Bullish to Bearish.
    *   `shortCondition = direction == -1 and direction[1] == 1`

*   **Execution Nuances:**
    *   **Execution on Bar Close Confirmation:** The indicator's logic (`close < trailingStop` or `close > trailingStop`) confirms a trend flip *at the close of the current bar*. To avoid look-ahead bias and ensure the signal is locked in, all strategy orders should be executed on the **open of the next bar**. This is the default, non-repainting behavior of Pine Script's `strategy.entry()` function and reflects how a live system would operate (i.e., receive signal at 16:59:59, place market order for the 17:00:00 open).

*   **Signal Reversals:**
    *   The system is designed to be "always in" the market, flipping from Long to Short. By using the same ID string for both `strategy.entry()` calls (e.g., "SigTS"), Pine Script's engine will automatically handle the reversal. When `longCondition` becomes true, it will close the existing short position and open a new long one. This is the most efficient way to manage reversals and avoids having separate `strategy.close()` logic for entries.

### 2. Multi-Tiered Exit Logic

The indicator itself *is* a sophisticated trailing stop. We will adopt it as our primary exit mechanism and augment it with additional layers for robustness.

*   **Initial Stop Loss (Volatility-Based):**
    *   The indicator already provides this. When a trend flips, the new `trailingStop` is calculated based on the ATR (`upperBand` or `lowerBand`). This value, captured at the moment of entry, serves as our initial, volatility-adjusted stop loss. There is no need for arbitrary percentages.
    *   **Logic:** On the bar the `longCondition` is true, the current value of `trailingStop` is the stop loss for the new long position.

*   **Take Profit / Trailing Mechanism:**
    *   **Primary Trailing Stop:** The core Sigmoid Trailing Stop logic is our main profit-protection and trailing mechanism. Its unique feature is the "adjustment phase" (`isAdjusting`), where it aggressively moves closer to the price during strong, established trends. This dynamic behavior is superior to a simple fixed-step or percentage-based trail. We will use this `trailingStop` value to update our exit order on every bar.
    *   **Optional Multi-Stage Take Profit:** For strategies requiring partial profit-taking, we can introduce ATR-based targets.
        *   **TP1:** `entry_price + (atrAtEntry * 2.5)`
        *   **TP2:** `entry_price + (atrAtEntry * 4.0)`
        *   These would be implemented as `strategy.exit` limit orders to scale out of the position (e.g., sell 33% at TP1, 33% at TP2, and let the final 34% run with the Sigmoid Trailing Stop).

*   **Time-Based Exits:**
    *   **End of Day (EOD) Exit:** Essential for non-24/7 markets or day-trading strategies to avoid overnight risk. A simple time-based rule will square all positions before the session close.
        *   **Logic:** `if (time >= session_close_time) strategy.close_all()`
    *   **Trade Stagnation Exit:** A professional tactic to free up capital from non-performing trades. If a position has been open for a specified number of bars (`maxBarsInTrade`) and has not achieved a minimum profit target (e.g., 1R or 1x ATR), it is closed.
        *   **Logic:** `if (bar_index - strategy.opentrades.entry_bar_index(0) > maxBarsInTrade and strategy.opentrades.profit(0) < minProfitThreshold) strategy.close(...)`

### 3. Capital Allocation & Risk Management

Moving from signals to execution requires a strict mathematical framework for position sizing. We will implement a fixed-fractional risk model.

*   **Risk-Based Sizing:**
    *   The core principle is to risk a fixed percentage of account equity on every single trade. The position size is therefore a function of the distance to the initial stop loss.
    *   **Formula:**
        `StopLossDistance = abs(EntryPrice - InitialStopLoss)`
        `RiskAmountInCurrency = AccountEquity * RiskPercentage`
        `PositionSize = RiskAmountInCurrency / StopLossDistance`
    *   **Implementation:** This calculation must be performed just before placing the `strategy.entry` order. We capture the `trailingStop` value on the signal bar to determine the `InitialStopLoss`.

*   **Pyramiding & Scaling In:**
    *   The indicator's `isAdjusting` flag provides a powerful, logic-driven signal to add to a winning position. When the price has moved far enough from the stop to trigger the sigmoid adjustment, it signals a strong, confirmed trend. This is an ideal moment to pyramid.
    *   **Rule:** If a position is already open and the `isAdjusting` flag turns from `false` to `true`, a new, smaller unit can be added to the trade. The strategy's `pyramiding` parameter must be enabled and set to a maximum number of entries (e.g., 3). The stop for the entire combined position would continue to be the main `trailingStop`.

### 4. Implementation Snippet (Pine Logic)

This snippet demonstrates the conversion of the indicator into a fully-featured strategy, incorporating the principles above.

```pine
//@version=5
// 1. STRATEGY DECLARATION with realistic parameters
strategy("Automated Sigmoid TS Strategy", 
     overlay=true, 
     initial_capital=100000,
     commission_type=strategy.commission.percent,
     commission_value=0.07, // Example: 0.07% per trade
     slippage=2, // Example: 2 ticks of slippage per order
     pyramiding=3, // Allow up to 3 entries in the same direction
     default_qty_type=strategy.fixed, // We will calculate quantity manually
     process_orders_on_close=false // Execute on the open of the next bar
     )

// 2. INPUTS for Risk and Exit Management
riskPercent    = input.float(1.0, "Risk Per Trade %", minval=0.1, maxval=10) / 100
useEodExit     = input.bool(true, "Use End-of-Day Exit?")
eodHour        = input.int(15, "EOD Hour", minval=0, maxval=23)
eodMinute      = input.int(45, "EOD Minute", minval=0, maxval=59)

// =================================================================================
// PASTE THE ORIGINAL INDICATOR'S LOGIC HERE (from line 9 to 128)
// ... all inputs, functions, and logic for calculating 'direction' and 'trailingStop'
// =================================================================================
// This work is licensed under a Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0) https://creativecommons.org/licenses/by-nc-sa/4.0/
// © LuxAlgo

//---------------------------------------------------------------------------------------------------------------------}
// Constants
//---------------------------------------------------------------------------------------------------------------------{
COLOR_BULL = #089981
COLOR_BEAR = #f23645
TRANSP_90  = 90
TRANSP_70  = 70
TRANSP_50  = 50

//---------------------------------------------------------------------------------------------------------------------}
// Inputs
//---------------------------------------------------------------------------------------------------------------------{
atrLengthInput   = input.int(200,   "ATR Length",                  minval = 1,                 tooltip = "ATR lookback period.")
atrMultInput     = input.float(3.0, "ATR Multiplier",              minval = 0.1, step = 0.1,   tooltip = "Initial distance when the trend flips.")
sigLengthInput   = input.int(20,    "Sigmoid Length (Bars)",       minval = 2,                 tooltip = "Bars to complete the transition.")
sigAmpMultInput  = input.float(3.0, "Sigmoid Amplitude (ATR Units)",minval = 0.1, step = 0.1,   tooltip = "How much closer to price the TS moves during transition.")
minDistMultInput = input.float(0.5, "Min Distance (ATR Units)",    minval = 0.0, step = 0.1,   tooltip = "Adjustment halts if TS gets within this distance of price.")

//---------------------------------------------------------------------------------------------------------------------}
// Functions
//---------------------------------------------------------------------------------------------------------------------{
// @function Returns a value clamped between a minimum and maximum.
clamp(float val, float low, float high) => 
    math.min(math.max(val, low), high)

// @function Returns a sigmoid transition value (0..1) based on progress 't'.
sigmoid(float t) =>
    float x    = -6.0 + 12.0 * clamp(t, 0.0, 1.0)
    float sMin = 1.0 / (1.0 + math.exp(6.0))
    float sMax = 1.0 / (1.0 + math.exp(-6.0))
    float sig  = 1.0 / (1.0 + math.exp(-x))
    (sig - sMin) / (sMax - sMin)

//---------------------------------------------------------------------------------------------------------------------}
// Logic
//---------------------------------------------------------------------------------------------------------------------{
float atr = ta.atr(atrLengthInput)

// State Tracking
var float trailingStop = na
var int   direction    = 1 // 1: Bull, -1: Bear
var bool  isAdjusting  = false
var int   sigCounter   = 0
var float startLevel   = na
var float targetOffset = 0.0

// Base Bands (only used for flips)
float upperBand = high + atrMultInput * atr
float lowerBand = low - atrMultInput * atr

// Flip and Basic Persistence
if na(trailingStop) or na(atr)
    trailingStop := direction == 1 ? lowerBand : upperBand
else
    if direction == 1
        if close < trailingStop
            direction    := -1
            trailingStop := upperBand
            isAdjusting  := false
            sigCounter   := 0
    else
        if close > trailingStop
            direction    := 1
            trailingStop := lowerBand
            isAdjusting  := false
            sigCounter   := 0

// Adjustment Trigger
float currentDist = direction == 1 ? close - trailingStop : trailingStop - close
float kDist       = atrMultInput * (atr > 0 ? atr : 1.0)
float minDist     = minDistMultInput * (atr > 0 ? atr : 1.0)

// Start sigmoid transition if price-to-stop gap becomes too wide
if not isAdjusting and currentDist > kDist
    isAdjusting  := true
    sigCounter   := 0
    startLevel   := trailingStop
    targetOffset := sigAmpMultInput * atr

// Adjustment execution
if isAdjusting
    sigCounter += 1
    
    // Normalized progress (0 to 1)
    float t         = float(sigCounter) / float(sigLengthInput)
    float sigFactor = sigmoid(t)
    
    // Position candidate: starts at 'startLevel' and moves towards price by 'targetOffset'
    float adjustment = targetOffset * sigFactor
    float candidate  = direction == 1 ? startLevel + adjustment : startLevel - adjustment
    
    // Distance check to prevent stop getting too close
    float newDist = direction == 1 ? close - candidate : candidate - close
    
    if newDist < minDist or sigCounter >= sigLengthInput
        isAdjusting := false
    else
        // Converge only: Maintain trailing property (only move closer to price)
        if direction == 1
            trailingStop := math.max(trailingStop, candidate)
        else
            trailingStop := math.min(trailingStop, candidate)

// Visuals (can be kept for backtesting analysis)
color tsColor = color.new(direction == 1 ? COLOR_BULL : COLOR_BEAR, 70)
plot(trailingStop, "Trailing Stop", tsColor, 2)
// =================================================================================

// 3. STRATEGY EXECUTION LOGIC

// --- Entry Conditions ---
bool longCondition  = direction == 1 and direction[1] == -1
bool shortCondition = direction == -1 and direction[1] == 1

// --- Position Sizing ---
float riskAmount    = strategy.equity * riskPercent
float stopDistance  = math.abs(close - trailingStop) // Approx. distance on signal bar
float positionSize  = stopDistance > 0 ? riskAmount / stopDistance : 0

// --- Pyramiding Condition ---
bool pyramidCondition = isAdjusting and not isAdjusting[1]

// --- Execution ---
if (longCondition)
    // Using one ID handles reversals automatically
    strategy.entry("SigTS_Entry", strategy.long, qty=positionSize, comment="Long Entry")

if (shortCondition)
    strategy.entry("SigTS_Entry", strategy.short, qty=positionSize, comment="Short Entry")

// --- Pyramiding Logic ---
if (pyramidCondition and strategy.position_size > 0)
    // Add a smaller position, e.g., 50% of the initial size
    strategy.entry("SigTS_Pyr", strategy.long, qty=(positionSize * 0.5), comment="Pyramid Long")

if (pyramidCondition and strategy.position_size < 0)
    strategy.entry("SigTS_Pyr", strategy.short, qty=(positionSize * 0.5), comment="Pyramid Short")


// --- Exit Management ---
if (strategy.position_size != 0)
    // The core exit: A dynamic trailing stop that updates on every bar
    strategy.exit("Exit", stop=trailingStop, comment="Trailing Stop")

// --- Time-Based Exit ---
isEod = useEodExit and (hour(time, syminfo.timezone) == eodHour) and (minute(time, syminfo.timezone) >= eodMinute)
if (isEod)
    strategy.close_all(comment="EOD Close")

```
    