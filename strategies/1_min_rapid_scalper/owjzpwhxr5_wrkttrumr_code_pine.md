
# Source Code

```pinescript

// This Pine Script® code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © BVL-Crypto

//@version=6
strategy('1-Min Rapid Scalper', overlay = true, process_orders_on_close = false, initial_capital = 10000, default_qty_type = strategy.percent_of_equity, default_qty_value = 10)

// ==============================================================================
// 1. OVERARCHING SETTINGS & INPUTS
// ==============================================================================
grp_main = 'Main Direction & Logic'
tradeDir = input.string('Both', title = 'Trade Direction Toggle', options = ['Long', 'Short', 'Both'], group = grp_main)

grp_ind = 'Indicator Settings (ADX)'
adxLen = input.int(7, title = 'ADX Length', minval = 1, group = grp_ind)
adxThresh = input.float(20.0, title = 'ADX Threshold', minval = 0.0, group = grp_ind)

grp_exit = 'Exit Settings'
useAtr = input.bool(false, title = 'Enable ATR Multiplier Exit (Overrides 1-Bar Exit)', group = grp_exit)
atrLen = input.int(14, title = 'ATR Length', minval = 1, group = grp_exit)
tpMult = input.float(1.0, title = 'Take Profit (ATR Multiplier)', step = 0.1, group = grp_exit)
slMult = input.float(2.0, title = 'Stop Loss (ATR Multiplier)', step = 0.1, group = grp_exit)
thePlug = input.float(1.0, title = 'The Plug (Hard SL %)', step = 0.1, group = grp_exit)

// ==============================================================================
// 2. DATA ACQUISITION: HEIKIN ASHI & ADX
// ==============================================================================
// Pulling precise Heikin Ashi values for the active timeframe
ticker_id = ticker.heikinashi(syminfo.tickerid)
[haOpen, haHigh, haLow, haClose] = request.security(ticker_id, timeframe.period, [open, high, low, close])

// Calculating ADX
[diPlus, diMinus, adxValue] = ta.dmi(adxLen, adxLen)

// ==============================================================================
// 3. CANDLE MORPHOLOGY & LOGIC
// ==============================================================================
// Calculate HA Body Size
haSize = math.abs(haClose - haOpen)

// Direction and Wick Evaluations
isHaBull = haClose > haOpen
isHaBear = haClose < haOpen
noWickBull = haOpen == haLow
noWickBear = haOpen == haHigh

// Sequencer: Long
bull1 = isHaBull[2] and noWickBull[2]
bull2 = isHaBull[1] and noWickBull[1] and haSize[1] > haSize[2]
bull3 = isHaBull and noWickBull and haSize > haSize[1]
colorFlipBull = isHaBear[3] // Previous candle must be bearish for the flip

// Sequencer: Short
bear1 = isHaBear[2] and noWickBear[2]
bear2 = isHaBear[1] and noWickBear[1] and haSize[1] > haSize[2]
bear3 = isHaBear and noWickBear and haSize > haSize[1]
colorFlipBear = isHaBull[3] // Previous candle must be bullish for the flip

// Final Triggers combined with ADX Filter
longTrigger = colorFlipBull and bull1 and bull2 and bull3 and adxValue > adxThresh
shortTrigger = colorFlipBear and bear1 and bear2 and bear3 and adxValue > adxThresh

// ==============================================================================
// 4. ORDER EXECUTION & ROUTING
// ==============================================================================
// Variables for dynamic ATR target tracking
var float longTP = na
var float longSL = na
var float shortTP = na
var float shortSL = na

atrValue = ta.atr(atrLen)

// -- ENTRY LOGIC --
// Enters immediately at the open of the immediate next candle
if longTrigger and (tradeDir == 'Long' or tradeDir == 'Both')
    strategy.entry('Long', strategy.long, comment = 'Long')

if shortTrigger and (tradeDir == 'Short' or tradeDir == 'Both')
    strategy.entry('Short', strategy.short, comment = 'Short')

// -- EXIT LOGIC --
if useAtr
    // Lock in ATR targets exactly at the moment of entry
    if strategy.position_size > 0 and strategy.position_size[1] == 0
        float atrStop = strategy.opentrades.entry_price(0) - atrValue[1] * slMult
        float plugStop = strategy.opentrades.entry_price(0) * (1 - thePlug / 100)

        longTP := strategy.opentrades.entry_price(0) + atrValue[1] * tpMult
        longSL := math.max(atrStop, plugStop) // The Plug caps the maximum allowable loss distance
        longSL

    if strategy.position_size < 0 and strategy.position_size[1] == 0
        float atrStop = strategy.opentrades.entry_price(0) + atrValue[1] * slMult
        float plugStop = strategy.opentrades.entry_price(0) * (1 + thePlug / 100)

        shortTP := strategy.opentrades.entry_price(0) - atrValue[1] * tpMult
        shortSL := math.min(atrStop, plugStop) // The Plug caps the maximum allowable loss distance
        shortSL

    // Execute ATR Exits
    if strategy.position_size > 0
        strategy.exit('Exit Long', 'Long', limit = longTP, stop = longSL, comment = 'TP')
    if strategy.position_size < 0
        strategy.exit('Exit Short', 'Short', limit = shortTP, stop = shortSL, comment = 'TP')
        // 1-Bar Exit Logic (Exit on close of entry candle)
        // By issuing a close command during the entry candle, the execution engine
else // triggers it at the immediate next open (which is equivalent to the entry candle's close)
    if strategy.position_size > 0
        strategy.close('Long', comment = 'TP')
    if strategy.position_size < 0
        strategy.close('Short', comment = 'TP')

// Optional Visual Debugging: Highlight the 3rd trigger candle
bgcolor(longTrigger ? color.new(color.green, 85) : na, title = 'Long Trigger Background')
bgcolor(shortTrigger ? color.new(color.red, 85) : na, title = 'Short Trigger Background')


```

