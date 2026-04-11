
# Source Code

```pinescript

//@version=6
indicator("PSAR Trend Filter", shorttitle="PSAR Trend Filter", overlay=true, max_labels_count=500)

//---------------------------------------------------------
// INPUTS
//---------------------------------------------------------
const string calcGroup   = "Calculation"
const string filterGroup = "Signal Filters"
const string visualGroup = "Visuals"

start     = input.float(0.02, step=0.001, title="PSAR Start", group=calcGroup)
increment = input.float(0.02, step=0.001, title="PSAR Increment", group=calcGroup)
maximum   = input.float(0.20, step=0.01,  title="PSAR Maximum", group=calcGroup)

confirmOnClose  = input.bool(true,  title="Confirm Signals On Bar Close", group=filterGroup)
useEMAFilter    = input.bool(true,  title="Use EMA Trend Filter", group=filterGroup)
emaLength       = input.int(50, minval=1, title="EMA Length", group=filterGroup)
useADXFilter    = input.bool(true,  title="Use ADX Strength Filter", group=filterGroup)
adxLength       = input.int(14, minval=1, title="ADX Length", group=filterGroup)
adxMin          = input.float(15.0, step=0.5, title="Minimum ADX", group=filterGroup)
useCandleFilter = input.bool(false, title="Use Candle Confirmation", group=filterGroup)
useCooldown     = input.bool(true,  title="Use Cooldown Between Signals", group=filterGroup)
cooldownBars    = input.int(1, minval=0, title="Cooldown Bars", group=filterGroup)

psarWidth            = input.int(2, minval=1, maxval=4, title="PSAR Point Width", group=visualGroup)
highlightStartPoints = input.bool(false, title="Highlight Start Points", group=visualGroup)
highlightState       = input.bool(true,  title="Highlight Trend State", group=visualGroup)
showEMA              = input.bool(true,  title="Show EMA Filter", group=visualGroup)

//---------------------------------------------------------
// CORE CALCULATION
//---------------------------------------------------------
psar = ta.sar(start, increment, maximum)
dir  = close > psar ? 1 : -1

rawBuySignal  = ta.crossover(close, psar)
rawSellSignal = ta.crossunder(close, psar)

emaValue = ta.ema(close, emaLength)

// Правильное получение ADX
[_, _, adxValue] = ta.dmi(adxLength, adxLength)

//---------------------------------------------------------
// FILTERS
//---------------------------------------------------------
trendLongOk  = not useEMAFilter or close > emaValue
trendShortOk = not useEMAFilter or close < emaValue

// Проверка на na(), чтобы индикатор работал корректно с самого начала истории
strengthOk = not useADXFilter or (not na(adxValue) and adxValue >= adxMin)

candleLongOk  = not useCandleFilter or close > open
candleShortOk = not useCandleFilter or close < open
barConfirmed = not confirmOnClose or barstate.isconfirmed

var int lastSignalBar = 0
cooldownOk = not useCooldown or (bar_index - lastSignalBar > cooldownBars)

buySignal  = rawBuySignal and trendLongOk and strengthOk and candleLongOk and cooldownOk and barConfirmed
sellSignal = rawSellSignal and trendShortOk and strengthOk and candleShortOk and cooldownOk and barConfirmed

if buySignal or sellSignal
    lastSignalBar := bar_index

//---------------------------------------------------------
// COLORS
//---------------------------------------------------------
bullColor      = color.rgb(0, 184, 148)
bearColor      = color.rgb(214, 48, 49)
emaColor       = color.rgb(120, 120, 120)
bullFillColor  = color.new(bullColor, 88)
bearFillColor  = color.new(bearColor, 88)
startBullColor = color.new(bullColor, 0)
startBearColor = color.new(bearColor, 0)

psarColor = dir == 1 ? bullColor : bearColor

//---------------------------------------------------------
// PLOTS
//---------------------------------------------------------
psarPlot = plot(psar, title="PSAR", style=plot.style_circles, linewidth=psarWidth, color=psarColor)
emaPlot  = plot(showEMA and useEMAFilter ? emaValue : na, title="EMA Filter", color=emaColor, linewidth=2)

// Исправленный блок заливки фона
midPricePlot = plot(ohlc4, title="Mid Price", display=display.none, editable=false)
c_fill = highlightState ? (dir == 1 ? bullFillColor : bearFillColor) : color(na)
fill(midPricePlot, psarPlot, title="Trend State Fill", color=c_fill)

plotshape(rawBuySignal and highlightStartPoints ? psar : na, title="Raw Long Start", location=location.absolute, style=shape.circle, size=size.small, color=startBullColor)
plotshape(rawSellSignal and highlightStartPoints ? psar : na, title="Raw Short Start", location=location.absolute, style=shape.circle, size=size.small, color=startBearColor)

//---------------------------------------------------------
// ALERTS
//---------------------------------------------------------
alertcondition(rawBuySignal or rawSellSignal, title="PSAR Raw Direction Change", message="PSAR raw direction changed on {{exchange}}:{{ticker}}")
alertcondition(buySignal, title="PSAR Filtered Long", message="MarketStructureLab PSAR Trend Filter FREE LONG on {{exchange}}:{{ticker}}")
alertcondition(sellSignal, title="PSAR Filtered Short", message="MarketStructureLab PSAR Trend Filter FREE SHORT on {{exchange}}:{{ticker}}")

```

