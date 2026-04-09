
# Source Code

```pinescript

// This Pine Script® code is subject to the Terms of Use at https://www.tradingview.com/pine-script-docs/concepts/licensing/
// © InstitutionalFlowScalper

//@version=6
indicator("Institutional Flow Scalper [IFS] v4", shorttitle="IFS v4", overlay=true, max_lines_count=500, max_labels_count=500, max_boxes_count=500)

// ─────────────────────────────────────────────────────────────────────────────
// ██ INPUTS
// ─────────────────────────────────────────────────────────────────────────────

// ── Signal Engine ──
grpSignal = "Signal Engine"
sensitivity     = input.string("Medium", "Sensitivity", options=["Low", "Medium", "High", "Adaptive"], group=grpSignal, tooltip="Adaptive mode auto-adjusts based on current volatility regime")
minConfidence   = input.int(30, "Min Confidence Score (0-100)", minval=0, maxval=100, group=grpSignal)
showSignals     = input.bool(true, "Show Entry Signals", group=grpSignal)
showExits       = input.bool(true, "Show Exit Signals", group=grpSignal)

// ── Volume Delta (CVD) ──
grpCVD = "Volume Delta Engine"
cvdLength       = input.int(14, "CVD Lookback", minval=5, maxval=50, group=grpCVD)
cvdSmoothing    = input.int(5, "CVD Smoothing", minval=1, maxval=20, group=grpCVD)
showCVDDiv      = input.bool(true, "Show CVD Divergences", group=grpCVD)

// ── Liquidity Sweep ──
grpLiq = "Liquidity Sweep Detection"
swingLookback   = input.int(10, "Swing Point Lookback", minval=3, maxval=30, group=grpLiq)
sweepThreshold  = input.float(1.5, "Volume Spike Threshold (x avg)", minval=1.0, maxval=5.0, step=0.1, group=grpLiq)
showSweeps      = input.bool(true, "Show Liquidity Sweeps", group=grpLiq)

// ── VWAP Institutional ──
grpVWAP = "Institutional VWAP"
showVWAP        = input.bool(true, "Show VWAP", group=grpVWAP)
showVWAPBands   = input.bool(true, "Show VWAP Bands (1σ, 2σ)", group=grpVWAP)
vwapBandMult1   = input.float(1.0, "Band 1 Multiplier", minval=0.5, maxval=3.0, step=0.1, group=grpVWAP)
vwapBandMult2   = input.float(2.0, "Band 2 Multiplier", minval=1.0, maxval=5.0, step=0.1, group=grpVWAP)
pocLength       = input.int(20, "POC Lookback", minval=10, maxval=50, group=grpVWAP)

// ── Absorption Detection ──
grpAbsorb = "Order Absorption"
absorbVolMult   = input.float(2.0, "Absorption Volume Multiple", minval=1.5, maxval=5.0, step=0.1, group=grpAbsorb)
absorbBodyRatio = input.float(0.3, "Max Body/Range Ratio", minval=0.1, maxval=0.5, step=0.05, group=grpAbsorb)
showAbsorption  = input.bool(true, "Show Absorption Zones", group=grpAbsorb)

// ── Anti-Chop Filter ──
grpChop = "Anti-Chop Filter"
chopLength      = input.int(14, "Chop Index Length", minval=5, maxval=30, group=grpChop)
chopThreshold   = input.float(65.0, "Chop Threshold", minval=50.0, maxval=75.0, step=0.1, group=grpChop, tooltip="Above this value = choppy market, signals disabled")
showChopZone    = input.bool(true, "Show Chop Zone Background", group=grpChop)

// ── Session Awareness ──
grpSession = "Session Awareness"
showSessionBG   = input.bool(false, "Show Session Background", group=grpSession)
nySession       = input.session("0930-1100", "NY Morning Session", group=grpSession)
pmSession       = input.session("1400-1600", "NY Afternoon Session", group=grpSession)
ldnSession      = input.session("0300-0500", "London Session", group=grpSession)

// ── Risk Management ──
grpRisk = "Risk Management"
atrLength       = input.int(14, "ATR Length", minval=5, maxval=30, group=grpRisk)
atrMultSL       = input.float(1.5, "Stop Loss (ATR Multiple)", minval=0.5, maxval=5.0, step=0.1, group=grpRisk)
atrMultTP      = input.float(1.0, "Take Profit (ATR Multiple)", minval=0.5, maxval=5.0, step=0.1, group=grpRisk)
showSLTP        = input.bool(true, "Show SL/TP Levels", group=grpRisk)
sltp_extend     = input.int(30, "SL/TP Line Length (bars)", minval=10, maxval=100, group=grpRisk)

// ── Dashboard ──
grpDash = "Dashboard"
showDashboard   = input.bool(true, "Show Dashboard", group=grpDash)
dashPosition    = input.string("Top Right", "Dashboard Position", options=["Top Left", "Top Right", "Bottom Left", "Bottom Right"], group=grpDash)
dashSize        = input.string("Small", "Dashboard Size", options=["Tiny", "Small", "Normal"], group=grpDash)

// ── Visual Style ──
grpVis = "Visual Style"
bullColor       = input.color(color.new(#00E676, 0), "Bull Color", group=grpVis)
bearColor       = input.color(color.new(#FF1744, 0), "Bear Color", group=grpVis)
neutralColor    = input.color(color.new(#FFD740, 0), "Neutral/Warning Color", group=grpVis)
vwapColor       = input.color(color.new(#2196F3, 0), "VWAP Color", group=grpVis)
entryColor      = input.color(color.new(#FFFFFF, 0), "Entry Line Color", group=grpVis)
slColor         = input.color(color.new(#FF1744, 0), "Stop Loss Color", group=grpVis)
tpColor        = input.color(color.new(#00E676, 0), "TP Color", group=grpVis)

// ─────────────────────────────────────────────────────────────────────────────
// ██ CORE CALCULATIONS
// ─────────────────────────────────────────────────────────────────────────────

atrVal = ta.atr(atrLength)

atrPctRank = ta.percentrank(atrVal, 100)
adaptiveMult = switch sensitivity
    "Low"      => 0.7
    "High"     => 1.3
    "Adaptive" => atrPctRank > 70 ? 0.7 : atrPctRank < 30 ? 1.3 : 1.0
    => 1.0

// ─────────────────────────────────────────────────────────────────────────────
// ██ PILLAR 1: SYNTHETIC VOLUME DELTA (CVD)
// ─────────────────────────────────────────────────────────────────────────────

barRange    = high - low
bodySize    = math.abs(close - open)
upperWick   = high - math.max(close, open)
lowerWick   = math.min(close, open) - low

closePosition = barRange > 0 ? (close - low) / barRange : 0.5
buyVolume     = volume * closePosition
sellVolume    = volume * (1.0 - closePosition)
volumeDelta   = buyVolume - sellVolume

var float cvd = 0.0
cvd := cvd + volumeDelta

var float sessionCVD = 0.0
isNewSession = ta.change(time("D")) != 0
sessionCVD := isNewSession ? volumeDelta : sessionCVD + volumeDelta

cvdSmoothed = ta.ema(sessionCVD, cvdSmoothing)
cvdRateOfChange = ta.roc(cvdSmoothed, cvdLength)

// CVD Divergence
pivotLowPrice  = ta.pivotlow(low, swingLookback, swingLookback)
pivotHighPrice = ta.pivotlow(-high, swingLookback, swingLookback)

var float prevPivotLowPrice  = na
var float prevPivotLowCVD    = na
var float prevPivotHighPrice = na
var float prevPivotHighCVD   = na

bullishCVDDiv = false
bearishCVDDiv = false

if not na(pivotLowPrice)
    currLowCVD = cvdSmoothed[swingLookback]
    if not na(prevPivotLowPrice) and not na(prevPivotLowCVD)
        if pivotLowPrice < prevPivotLowPrice and currLowCVD > prevPivotLowCVD
            bullishCVDDiv := true
    prevPivotLowPrice := pivotLowPrice
    prevPivotLowCVD   := currLowCVD

if not na(pivotHighPrice)
    currHighPrice = -pivotHighPrice
    currHighCVD   = cvdSmoothed[swingLookback]
    if not na(prevPivotHighPrice) and not na(prevPivotHighCVD)
        if currHighPrice > prevPivotHighPrice and currHighCVD < prevPivotHighCVD
            bearishCVDDiv := true
    prevPivotHighPrice := currHighPrice
    prevPivotHighCVD   := currHighCVD

// ─────────────────────────────────────────────────────────────────────────────
// ██ PILLAR 2: LIQUIDITY SWEEP DETECTION
// ─────────────────────────────────────────────────────────────────────────────

swingHigh = ta.pivothigh(high, swingLookback, 1)
swingLow  = ta.pivotlow(low, swingLookback, 1)

var float recentSwingHigh = na
var float recentSwingLow  = na

if not na(swingHigh)
    recentSwingHigh := swingHigh
if not na(swingLow)
    recentSwingLow := swingLow

avgVolume   = ta.sma(volume, 20)
volumeSpike = volume > avgVolume * sweepThreshold

bearishSweep = not na(recentSwingHigh) and high > recentSwingHigh and close < recentSwingHigh and close < open and volumeSpike
bullishSweep = not na(recentSwingLow) and low < recentSwingLow and close > recentSwingLow and close > open and volumeSpike

// ─────────────────────────────────────────────────────────────────────────────
// ██ PILLAR 3: MICRO MOMENTUM SHIFT
// ─────────────────────────────────────────────────────────────────────────────

priceROC   = ta.roc(close, cvdLength)
cvdROC     = ta.roc(cvdSmoothed, cvdLength)

momentumBullish = priceROC < 0 and cvdROC > 0
momentumBearish = priceROC > 0 and cvdROC < 0

pressureIndex = ta.ema(volumeDelta / math.max(volume, 1) * 100, cvdSmoothing)

// ─────────────────────────────────────────────────────────────────────────────
// ██ ORDER ABSORPTION
// ─────────────────────────────────────────────────────────────────────────────

isAbsorption      = volume > avgVolume * absorbVolMult and barRange > 0 and (bodySize / barRange) < absorbBodyRatio
bullishAbsorption = isAbsorption and close > open
bearishAbsorption = isAbsorption and close < open

// ─────────────────────────────────────────────────────────────────────────────
// ██ INSTITUTIONAL VWAP
// ─────────────────────────────────────────────────────────────────────────────

var float sumPV  = 0.0
var float sumVol = 0.0
var float sumPV2 = 0.0

if isNewSession
    sumPV  := 0.0
    sumVol := 0.0
    sumPV2 := 0.0

typicalPrice = (high + low + close) / 3.0
sumPV  += typicalPrice * volume
sumVol += volume
sumPV2 += typicalPrice * typicalPrice * volume

vwapVal    = sumVol > 0 ? sumPV / sumVol : close
vwapVar    = sumVol > 0 ? sumPV2 / sumVol - vwapVal * vwapVal : 0.0
vwapStdDev = math.sqrt(math.max(vwapVar, 0))

vwapUpper1 = vwapVal + vwapStdDev * vwapBandMult1
vwapLower1 = vwapVal - vwapStdDev * vwapBandMult1
vwapUpper2 = vwapVal + vwapStdDev * vwapBandMult2
vwapLower2 = vwapVal - vwapStdDev * vwapBandMult2

vwapDistance = vwapStdDev > 0 ? (close - vwapVal) / vwapStdDev : 0.0

// ── POC ──
var float pocPrice = na
pocSumPV  = ta.sma(typicalPrice * volume, pocLength) * pocLength
pocSumVol = ta.sma(volume, pocLength) * pocLength
pocPrice := pocSumVol > 0 ? pocSumPV / pocSumVol : close

// ─────────────────────────────────────────────────────────────────────────────
// ██ CHOPPINESS INDEX
// ─────────────────────────────────────────────────────────────────────────────

highestHigh = ta.highest(high, chopLength)
lowestLow   = ta.lowest(low, chopLength)
atrSum      = ta.sma(atrVal, chopLength) * chopLength
rangeHL     = highestHigh - lowestLow

chopIndex = rangeHL > 0 ? 100.0 * math.log10(atrSum / rangeHL) / math.log10(chopLength) : 50.0
isChoppy  = chopIndex > chopThreshold

// ─────────────────────────────────────────────────────────────────────────────
// ██ SESSION AWARENESS
// ─────────────────────────────────────────────────────────────────────────────

inNYMorning  = not na(time(timeframe.period, nySession, "America/New_York"))
inNYAfternoon = not na(time(timeframe.period, pmSession, "America/New_York"))
inLondon     = not na(time(timeframe.period, ldnSession, "America/New_York"))
inKillZone   = inNYMorning or inNYAfternoon or inLondon
sessionMult  = inKillZone ? 1.2 : 1.0

// ─────────────────────────────────────────────────────────────────────────────
// ██ CONFIDENCE SCORE (0-100)
// ─────────────────────────────────────────────────────────────────────────────

deltaScore   = math.min(math.abs(pressureIndex) / 2.0, 20.0)
cvdDivScore  = (bullishCVDDiv or bearishCVDDiv) ? 20.0 : 0.0
sweepScore   = (bullishSweep or bearishSweep) ? 20.0 : 0.0
momScore     = math.min(math.abs(cvdROC - priceROC), 20.0)
vwapScore    = math.abs(vwapDistance) > 1.5 ? 8.0 : math.abs(vwapDistance) > 1.0 ? 5.0 : 2.0
absorbScore  = isAbsorption ? 6.0 : 0.0
contextScore = math.min(vwapScore + absorbScore + 6.0, 20.0)

rawConfidence = deltaScore + cvdDivScore + sweepScore + momScore + contextScore
confidence    = math.min(rawConfidence * adaptiveMult * sessionMult, 100.0)

// ─────────────────────────────────────────────────────────────────────────────
// ██ SIGNAL GENERATION (v3 - Enhanced for Scalping Frequency)
// ─────────────────────────────────────────────────────────────────────────────

// ── Additional Common Pillars ──
// EMA trend alignment (fast crosses slow)
emaFast = ta.ema(close, 9)
emaSlow = ta.ema(close, 21)
emaBullish = emaFast > emaSlow
emaBearish = emaFast < emaSlow
emaCrossUp   = ta.crossover(emaFast, emaSlow)
emaCrossDown = ta.crossunder(emaFast, emaSlow)

// Volume spike (current bar volume > 1.3x average)
volStrong = volume > avgVolume * 1.3

// Strong candle (body > 60% of range, directional conviction)
strongBullCandle = barRange > 0 and (close - open) / barRange > 0.6 and close > open and volStrong
strongBearCandle = barRange > 0 and (open - close) / barRange > 0.6 and close < open and volStrong

// Price rejection from VWAP bands (mean reversion signal)
vwapBounceUp   = low <= vwapLower1 and close > vwapLower1 and close > open
vwapBounceDown = high >= vwapUpper1 and close < vwapUpper1 and close < open

// VWAP cross
vwapBullish = close > vwapVal and close[1] <= vwapVal
vwapBearish = close < vwapVal and close[1] >= vwapVal

// ── Pillar Count (now with 8 possible pillars, need 2) ──
bullPillarCount = 0
bullPillarCount += volumeDelta > 0 and deltaScore > 3 ? 1 : 0    // 1. Strong buy delta
bullPillarCount += bullishCVDDiv or momentumBullish ? 1 : 0        // 2. CVD divergence or momentum shift
bullPillarCount += bullishSweep ? 1 : 0                            // 3. Liquidity sweep
bullPillarCount += bullishAbsorption ? 1 : 0                       // 4. Order absorption
bullPillarCount += vwapBullish or vwapBounceUp ? 1 : 0             // 5. VWAP cross or bounce
bullPillarCount += emaCrossUp or (emaBullish and strongBullCandle) ? 1 : 0  // 6. EMA alignment + strong candle
bullPillarCount += close > pocPrice and close[1] <= pocPrice ? 1 : 0  // 7. POC breakout

bearPillarCount = 0
bearPillarCount += volumeDelta < 0 and deltaScore > 3 ? 1 : 0
bearPillarCount += bearishCVDDiv or momentumBearish ? 1 : 0
bearPillarCount += bearishSweep ? 1 : 0
bearPillarCount += bearishAbsorption ? 1 : 0
bearPillarCount += vwapBearish or vwapBounceDown ? 1 : 0
bearPillarCount += emaCrossDown or (emaBearish and strongBearCandle) ? 1 : 0
bearPillarCount += close < pocPrice and close[1] >= pocPrice ? 1 : 0

// ── Bias ──
bullishBias = (volumeDelta > 0 ? 1 : 0) + (momentumBullish ? 1 : 0) + (bullishCVDDiv ? 2 : 0) + (bullishSweep ? 2 : 0) + (bullishAbsorption ? 1 : 0) + (close > vwapVal ? 1 : 0) + (emaBullish ? 1 : 0)
bearishBias = (volumeDelta < 0 ? 1 : 0) + (momentumBearish ? 1 : 0) + (bearishCVDDiv ? 2 : 0) + (bearishSweep ? 2 : 0) + (bearishAbsorption ? 1 : 0) + (close < vwapVal ? 1 : 0) + (emaBearish ? 1 : 0)

passesConfidence = confidence >= minConfidence
passesChop       = not isChoppy
passesSession    = inKillZone

longSignal  = bullishBias > bearishBias and bullPillarCount >= 2 and passesConfidence and passesChop and passesSession and barstate.isconfirmed
shortSignal = bearishBias > bullishBias and bearPillarCount >= 2 and passesConfidence and passesChop and passesSession and barstate.isconfirmed

var int lastSignalBar = 0
signalCooldown = bar_index - lastSignalBar >= 3

// Position state declared here so it can be used in entry conditions
// State: 0=flat, 1=long, -1=short
var int   posState    = 0
var float posEntry    = na
var float posSL       = na
var float posTP      = na
var int   posEntryBar = na

// CRITICAL: Only allow new entry when position is FLAT (previous trade closed by TP or SL)
longEntry  = longSignal and signalCooldown and posState == 0
shortEntry = shortSignal and signalCooldown and posState == 0

if longEntry or shortEntry
    lastSignalBar := bar_index

// ─────────────────────────────────────────────────────────────────────────────
// ██ POSITION TRACKER
// ─────────────────────────────────────────────────────────────────────────────

// ── Open Long ──
if longEntry
    posState    := 1
    posEntry    := close
    posSL       := close - atrVal * atrMultSL
    posTP      := close + atrVal * atrMultTP
    posEntryBar := bar_index

// ── Open Short ──
if shortEntry
    posState    := -1
    posEntry    := close
    posSL       := close + atrVal * atrMultSL
    posTP      := close - atrVal * atrMultTP
    posEntryBar := bar_index

// ── Exit Conditions ──

longSLHit   = posState == 1 and not na(posSL) and low <= posSL
shortSLHit  = posState == -1 and not na(posSL) and high >= posSL

longTPHit   = posState == 1 and not na(posTP) and high >= posTP
shortTPHit  = posState == -1 and not na(posTP) and low <= posTP

// Momentum exit (only before TP reached)
longMomExit  = posState == 1 and momentumBearish and pressureIndex < -5
shortMomExit = posState == -1 and momentumBullish and pressureIndex > 5

longExit  = longSLHit or longTPHit or longMomExit
shortExit = shortSLHit or shortTPHit or shortMomExit

// Determine exit reason
var string exitReason = ""
if longExit and posState == 1
    if longTPHit
        exitReason := "TP HIT ✓"
    else if longSLHit
        exitReason := "SL HIT ✗"
    else
        exitReason := "MOM EXIT"
    posState   := 0
    posEntry   := na
    posSL      := na
    posTP     := na

if shortExit and posState == -1
    if shortTPHit
        exitReason := "TP HIT ✓"
    else if shortSLHit
        exitReason := "SL HIT ✗"
    else
        exitReason := "MOM EXIT"
    posState   := 0
    posEntry   := na
    posSL      := na
    posTP     := na

// ─────────────────────────────────────────────────────────────────────────────
// ██ PLOTTING - VWAP & OVERLAYS
// ─────────────────────────────────────────────────────────────────────────────

plot(showVWAP ? vwapVal : na, "VWAP", vwapColor, linewidth=2)
plot(emaFast, "EMA 9", color.new(#FF9800, 40), linewidth=1)
plot(emaSlow, "EMA 21", color.new(#9C27B0, 40), linewidth=1)
p_vu1 = plot(showVWAPBands ? vwapUpper1 : na, "VWAP +1σ", color.new(vwapColor, 60))
p_vl1 = plot(showVWAPBands ? vwapLower1 : na, "VWAP -1σ", color.new(vwapColor, 60))
p_vu2 = plot(showVWAPBands ? vwapUpper2 : na, "VWAP +2σ", color.new(vwapColor, 80), style=plot.style_circles)
p_vl2 = plot(showVWAPBands ? vwapLower2 : na, "VWAP -2σ", color.new(vwapColor, 80), style=plot.style_circles)
fill(p_vu1, p_vl1, color.new(vwapColor, 92))
plot(showVWAP ? pocPrice : na, "Dynamic POC", color.new(#FF9800, 30), style=plot.style_cross)

bgcolor(showChopZone and isChoppy ? color.new(neutralColor, 90) : na, title="Chop Zone")
bgcolor(showSessionBG and inNYMorning ? color.new(#2196F3, 93) : na, title="NY Morning")
bgcolor(showSessionBG and inNYAfternoon ? color.new(#4CAF50, 93) : na, title="NY Afternoon")
bgcolor(showSessionBG and inLondon ? color.new(#9C27B0, 93) : na, title="London")

// ─────────────────────────────────────────────────────────────────────────────
// ██ PLOTTING - ENTRIES & EXITS (Today only to keep chart clean)
// ─────────────────────────────────────────────────────────────────────────────

// Only show labels/lines/boxes from today's session
isToday = ta.change(time("D")) == 0 and time >= timenow - 86400000

plotshape(showSignals and longEntry, "Long Entry", shape.labelup, location.belowbar, bullColor, size=size.large, text="LONG", textcolor=color.white)
plotshape(showSignals and shortEntry, "Short Entry", shape.labeldown, location.abovebar, bearColor, size=size.large, text="SHORT", textcolor=color.white)

barcolor(longEntry ? bullColor : shortEntry ? bearColor : longExit ? bearColor : shortExit ? bullColor : na)

// Exit labels (today only)
if showExits and longExit and isToday
    color exitCol = exitReason == "TP HIT ✓" ? color.new(tpColor, 10) : exitReason == "SL HIT ✗" ? color.new(bearColor, 10) : color.new(neutralColor, 10)
    label.new(bar_index, high, exitReason, color=exitCol, textcolor=color.white, style=label.style_label_down, size=size.normal)

if showExits and shortExit and isToday
    color exitCol = exitReason == "TP HIT ✓" ? color.new(tpColor, 10) : exitReason == "SL HIT ✗" ? color.new(bearColor, 10) : color.new(neutralColor, 10)
    label.new(bar_index, low, exitReason, color=exitCol, textcolor=color.white, style=label.style_label_up, size=size.normal)

// CVD Divergence (today only)
if showCVDDiv and bullishCVDDiv and isToday
    label.new(bar_index - swingLookback, low[swingLookback], "Bull\nDiv", color=color.new(bullColor, 20), textcolor=color.white, style=label.style_label_up, size=size.small)
if showCVDDiv and bearishCVDDiv and isToday
    label.new(bar_index - swingLookback, high[swingLookback], "Bear\nDiv", color=color.new(bearColor, 20), textcolor=color.white, style=label.style_label_down, size=size.small)

// Sweeps (today only)
if showSweeps and bullishSweep and isToday
    label.new(bar_index, low, "SWEEP ▲", color=color.new(bullColor, 10), textcolor=color.white, style=label.style_label_up, size=size.normal)
if showSweeps and bearishSweep and isToday
    label.new(bar_index, high, "SWEEP ▼", color=color.new(bearColor, 10), textcolor=color.white, style=label.style_label_down, size=size.normal)

// Absorption (plotshape handles all bars but these are small diamonds, keeping them)
plotshape(showAbsorption and bullishAbsorption and isToday, "Bull Absorb", shape.diamond, location.belowbar, color.new(bullColor, 30), size=size.tiny)
plotshape(showAbsorption and bearishAbsorption and isToday, "Bear Absorb", shape.diamond, location.abovebar, color.new(bearColor, 30), size=size.tiny)

// ─────────────────────────────────────────────────────────────────────────────
// ██ PLOTTING - SL/TP/ENTRY VISUALS (ALL TRADES - HISTORICAL)
// ─────────────────────────────────────────────────────────────────────────────

// Instead of showing all historical trades, only show today's to keep chart clean

if showSLTP and (longEntry or shortEntry) and isToday
    float ep  = close
    float slP = na
    float tpP = na

    if longEntry
        slP := ep - atrVal * atrMultSL
        tpP := ep + atrVal * atrMultTP
    else
        slP := ep + atrVal * atrMultSL
        tpP := ep - atrVal * atrMultTP

    int bS = bar_index
    int bE = bar_index + sltp_extend

    // Entry line
    line.new(bS, ep, bE, ep, color=entryColor, style=line.style_solid, width=2)
    label.new(bE + 1, ep, "ENTRY " + str.tostring(ep, "#.##"), color=color.new(entryColor, 20), textcolor=color.new(#000000, 0), style=label.style_label_left, size=size.normal)

    // SL line
    line.new(bS, slP, bE, slP, color=slColor, style=line.style_solid, width=2)
    label.new(bE + 1, slP, "SL " + str.tostring(slP, "#.##") + "\n" + str.tostring(math.abs(ep - slP), "#.##") + " pts", color=color.new(slColor, 20), textcolor=color.white, style=label.style_label_left, size=size.normal)

    // TP line
    line.new(bS, tpP, bE, tpP, color=tpColor, style=line.style_solid, width=2)
    label.new(bE + 1, tpP, "TP " + str.tostring(tpP, "#.##") + "\n" + str.tostring(math.abs(tpP - ep), "#.##") + " pts", color=color.new(tpColor, 20), textcolor=color.white, style=label.style_label_left, size=size.normal)


    // Colored zones
    if longEntry
        box.new(bS, ep, bE, slP, border_color=color.new(slColor, 80), bgcolor=color.new(slColor, 88), border_width=0)
        box.new(bS, tpP, bE, ep, border_color=color.new(tpColor, 80), bgcolor=color.new(tpColor, 88), border_width=0)
    else
        box.new(bS, slP, bE, ep, border_color=color.new(slColor, 80), bgcolor=color.new(slColor, 88), border_width=0)
        box.new(bS, ep, bE, tpP, border_color=color.new(tpColor, 80), bgcolor=color.new(tpColor, 88), border_width=0)

    // Confidence badge
    string dirText = longEntry ? "▲ LONG ENTRY" : "▼ SHORT ENTRY"
    string pText   = ""
    if longEntry
        pText := (volumeDelta > 0 and deltaScore > 3 ? "Delta " : "") + (bullishCVDDiv or momentumBullish ? "Mom " : "") + (bullishSweep ? "Sweep " : "") + (bullishAbsorption ? "Absorb " : "") + (vwapBullish or vwapBounceUp ? "VWAP " : "") + (emaCrossUp or (emaBullish and strongBullCandle) ? "EMA " : "")
    else
        pText := (volumeDelta < 0 and deltaScore > 3 ? "Delta " : "") + (bearishCVDDiv or momentumBearish ? "Mom " : "") + (bearishSweep ? "Sweep " : "") + (bearishAbsorption ? "Absorb " : "") + (vwapBearish or vwapBounceDown ? "VWAP " : "") + (emaCrossDown or (emaBearish and strongBearCandle) ? "EMA " : "")

    color cBg = confidence >= 80 ? color.new(bullColor, 10) : confidence >= 65 ? color.new(#2196F3, 10) : color.new(neutralColor, 10)

    if longEntry
        label.new(bS, low - atrVal * 0.5, dirText + "\nConfidence: " + str.tostring(math.round(confidence)) + "%\nPillars: " + pText, color=cBg, textcolor=color.white, style=label.style_label_up, size=size.normal)
    else
        label.new(bS, high + atrVal * 0.5, dirText + "\nConfidence: " + str.tostring(math.round(confidence)) + "%\nPillars: " + pText, color=cBg, textcolor=color.white, style=label.style_label_down, size=size.normal)


// ─────────────────────────────────────────────────────────────────────────────
// ██ DASHBOARD
// ─────────────────────────────────────────────────────────────────────────────

if showDashboard and barstate.islast
    var table dash = na

    tblPos = switch dashPosition
        "Top Left"     => position.top_left
        "Top Right"    => position.top_right
        "Bottom Left"  => position.bottom_left
        "Bottom Right" => position.bottom_right
        => position.top_right

    tblSize = switch dashSize
        "Tiny"   => size.tiny
        "Small"  => size.small
        "Normal" => size.normal
        => size.small

    dash := table.new(tblPos, 2, 10, bgcolor=color.new(#1a1a2e, 10), border_width=1, border_color=color.new(#333355, 0), frame_width=2, frame_color=color.new(#333355, 0))

    table.cell(dash, 0, 0, "IFS v4", text_color=color.white, text_size=tblSize, bgcolor=color.new(#16213e, 0), text_formatting=text.format_bold)
    table.cell(dash, 1, 0, "Dashboard", text_color=color.new(color.white, 40), text_size=tblSize, bgcolor=color.new(#16213e, 0))

    confColor = confidence >= 80 ? bullColor : confidence >= 60 ? neutralColor : bearColor
    table.cell(dash, 0, 1, "Confidence", text_color=color.new(color.white, 30), text_size=tblSize)
    table.cell(dash, 1, 1, str.tostring(math.round(confidence)) + "%", text_color=confColor, text_size=tblSize, text_formatting=text.format_bold)

    deltaCol = volumeDelta > 0 ? bullColor : bearColor
    table.cell(dash, 0, 2, "Vol Delta", text_color=color.new(color.white, 30), text_size=tblSize)
    table.cell(dash, 1, 2, volumeDelta > 0 ? "BUY ▲" : "SELL ▼", text_color=deltaCol, text_size=tblSize)

    pressCol = pressureIndex > 5 ? bullColor : pressureIndex < -5 ? bearColor : neutralColor
    table.cell(dash, 0, 3, "Pressure", text_color=color.new(color.white, 30), text_size=tblSize)
    table.cell(dash, 1, 3, str.tostring(math.round(pressureIndex, 1)), text_color=pressCol, text_size=tblSize)

    vwapDistCol = math.abs(vwapDistance) > 2 ? bearColor : math.abs(vwapDistance) > 1 ? neutralColor : bullColor
    table.cell(dash, 0, 4, "VWAP Dist", text_color=color.new(color.white, 30), text_size=tblSize)
    table.cell(dash, 1, 4, (vwapDistance > 0 ? "+" : "") + str.tostring(math.round(vwapDistance, 2)) + "σ", text_color=vwapDistCol, text_size=tblSize)

    table.cell(dash, 0, 5, "ATR", text_color=color.new(color.white, 30), text_size=tblSize)
    table.cell(dash, 1, 5, str.tostring(math.round(atrVal, 2)), text_color=color.new(color.white, 50), text_size=tblSize)

    sessStr = inNYMorning ? "NY AM 🔥" : inNYAfternoon ? "NY PM 🔥" : inLondon ? "LDN 🔥" : "OFF-HOURS"
    sessCol = inKillZone ? bullColor : color.new(color.white, 50)
    table.cell(dash, 0, 6, "Session", text_color=color.new(color.white, 30), text_size=tblSize)
    table.cell(dash, 1, 6, sessStr, text_color=sessCol, text_size=tblSize)

    chopCol = isChoppy ? bearColor : bullColor
    table.cell(dash, 0, 7, "Chop Idx", text_color=color.new(color.white, 30), text_size=tblSize)
    table.cell(dash, 1, 7, str.tostring(math.round(chopIndex, 1)), text_color=chopCol, text_size=tblSize)

    biasCol = bullishBias > bearishBias ? bullColor : bearishBias > bullishBias ? bearColor : neutralColor
    table.cell(dash, 0, 8, "Bias", text_color=color.new(color.white, 30), text_size=tblSize)
    table.cell(dash, 1, 8, bullishBias > bearishBias ? "BULLISH" : bearishBias > bullishBias ? "BEARISH" : "NEUTRAL", text_color=biasCol, text_size=tblSize, text_formatting=text.format_bold)

    // Position status row
    posStr = posState == 1 ? "LONG ACTIVE" : posState == -1 ? "SHORT ACTIVE" : "FLAT"
    posCol = posState == 1 ? bullColor : posState == -1 ? bearColor : color.new(color.white, 50)
    table.cell(dash, 0, 9, "Position", text_color=color.new(color.white, 30), text_size=tblSize)
    table.cell(dash, 1, 9, posStr, text_color=posCol, text_size=tblSize, text_formatting=text.format_bold)

// ─────────────────────────────────────────────────────────────────────────────
// ██ ALERTS
// ─────────────────────────────────────────────────────────────────────────────

alertcondition(longEntry, "IFS Long Entry", "IFS: LONG signal | Confidence: {{plot_0}}% | Entry: {{close}}")
alertcondition(shortEntry, "IFS Short Entry", "IFS: SHORT signal | Confidence: {{plot_0}}% | Entry: {{close}}")
alertcondition(longExit, "IFS Long Exit", "IFS: Exit LONG")
alertcondition(shortExit, "IFS Short Exit", "IFS: Exit SHORT")
alertcondition(bullishSweep, "Bullish Sweep", "IFS: Bullish liquidity sweep")
alertcondition(bearishSweep, "Bearish Sweep", "IFS: Bearish liquidity sweep")
alertcondition(isAbsorption, "Absorption", "IFS: Order absorption detected")

// ─────────────────────────────────────────────────────────────────────────────
// ██ DATA WINDOW
// ─────────────────────────────────────────────────────────────────────────────

plot(confidence, "Confidence Score", display=display.data_window, color=color.new(color.white, 100))
plot(pressureIndex, "Pressure Index", display=display.data_window, color=color.new(color.white, 100))
plot(sessionCVD, "Session CVD", display=display.data_window, color=color.new(color.white, 100))
plot(chopIndex, "Chop Index", display=display.data_window, color=color.new(color.white, 100))
plot(vwapDistance, "VWAP Distance", display=display.data_window, color=color.new(color.white, 100))

```

