
# Source Code

```pinescript

// This work is licensed under Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International  
// https://creativecommons.org/licenses/by-nc-sa/4.0/
// © BigBeluga

//@version=6
indicator("Trend Pulse [BigBeluga]", overlay = true, max_lines_count = 500)


// ＩＮＰＵＴＳ ――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――{

grpSettings = "Settings"
pivLen = input.int(
     5,
     "Pivot Detection Length",
     tooltip = "Defines how many bars are required on each side to confirm a pivot low. Higher values produce fewer but more reliable structural pivots, while lower values react faster but may introduce noise.\n\nThe break of a confirmed pivot low is used to detect a shift into a bearish trend."
     , group = grpSettings
)
smaMax = input.int(
     50,
     "Adaptive Band Max Length",
     minval = 5,
     tooltip = "Sets the maximum lookback length for the adaptive trend band. During bearish phases, the band gradually expands up to this value, making trend flips harder to trigger as the trend matures.\n\nA bullish trend shift is detected when price breaks above the adaptive band."
     , group = grpSettings
)

smaMult = input.float(
     1,
     "Adaptive Band Multiplier",
     minval = 0,
     step = 0.1,
     tooltip = "Controls the vertical offset of the adaptive trend band. Higher values place the band further away from price, requiring a stronger move to trigger a bullish break, while lower values make trend shifts more sensitive."
     , group = grpSettings
)

smaMin = 5

grpStl   = "Trend Style"
trendLen = input.int(
     100,
     "Trend Aging Length",
     tooltip = "Controls how quickly the trend color transitions as the trend progresses. This is based on time since the last trend flip, not momentum or strength. Longer trends gradually move toward a mature/exhausted state.",
     group = grpStl
)
upColor = input.color(
     #18d0bd,
     "Bullish Trend (Fresh)",
     tooltip = "Color used immediately after a bullish trend flip. Represents a newly established trend with lower exhaustion risk.",
     inline = "Bull",
     group = grpStl
)
upColor1 = input.color(
     color.rgb(13, 169, 75),
     "Bullish Trend (Mature)",
     tooltip = "Color used as a bullish trend ages over time. Represents a later-stage trend that may be more prone to pullbacks or reversals.",
     inline = "Bull1",
     group = grpStl
)
dnColor = input.color(
     #cf1d3a,
     "Bearish Trend (Fresh)",
     tooltip = "Color used immediately after a bearish trend flip. Represents a newly established downtrend with lower exhaustion risk.",
     inline = "Bear",
     group = grpStl
)
dnColor1 = input.color(
     color.orange,
     "Bearish Trend (Mature)",
     tooltip = "Color used as a bearish trend ages over time. Represents a later-stage downtrend where momentum may start weakening.",
     inline = "Bear1",
     group = grpStl
)

var pivot  = float(na)
var pindex = int(na)

var Pline  = line(na)
var direction = bool(na)

var sma   = float(na)
var start = 0 

var len = smaMin
// }


// ＣＡＬＣＵＬＡＴＩＯＮＳ――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――{

pl = ta.pivotlow(pivLen, pivLen)
var plLine = line(na)

if not na(pl)

    plLine.delete()

    pivot  := pl 
    pindex := bar_index[pivLen]

    if direction
        plLine := line.new(pindex, pivot, pindex, pivot, color = color.new(chart.fg_color, 50), style = line.style_dashed)

plLine.set_x2(bar_index)

if ta.crossunder(close, pivot) and barstate.isconfirmed

    if direction
        Pline := line.new(pindex, pivot, bar_index, pivot, color = chart.fg_color, style = line.style_dashed)

    direction := false 


if not direction and len <= smaMax
    len += 1
    
atr = ta.atr(50) * smaMult
sma := ta.sma(high, len) + atr

if ta.crossover(close, sma) and barstate.isconfirmed
    direction := true
    sma := float(na)
    len := 10


if direction != direction[1]
    start := bar_index

tLength = bar_index - start


downCol = color.from_gradient(tLength, 5, trendLen, color.new(dnColor, 0), dnColor1)
uppCol =  color.from_gradient(tLength, 5, trendLen, color.new(upColor, 0), upColor1)

tcol = direction and barstate.isconfirmed ? uppCol : not direction and barstate.isconfirmed ? downCol : color(na)
// }


// ＰＬＯＴ ――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――{

psma= plot(not direction ? sma : float(na), "Down Trend Band"
    , color = downCol
    , style = plot.style_linebr
    , linewidth = 3)

pp = plot(hl2, "", display = display.none, editable = false)

fill(psma, pp, not direction ? sma : float(na), hl2, color.new(downCol, 85), color(na))

barcolor(tcol, title = "Bar Color")
plotcandle(open, high, low, close, title='Candles Color', color = tcol, wickcolor=tcol, bordercolor = tcol)

plotshape(not na(pl) and barstate.isconfirmed ? pl : float(na), "Pivot Low"
    , shape.circle, location.absolute
    , offset = -pivLen
    , size = size.tiny
    , color = color.new(tcol, 50))

plotshape(not direction and direction[1] ? sma : float(na), "Break Down"
    , shape.circle, location.absolute
    , offset = 0
    , size = size.tiny
    , color = color.new(tcol, 50))
    
plotshape(not direction[1] and direction ? sma[1] : float(na), "Break Up"
    , shape.circle, location.absolute
    , offset = -1
    , size = size.tiny
    , color = color.new(tcol, 50))
// }


```

