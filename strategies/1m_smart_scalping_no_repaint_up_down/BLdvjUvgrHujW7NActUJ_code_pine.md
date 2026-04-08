
# Source Code

```pinescript

//@version=5
indicator("1M Smart Scalping (No Repaint) - UP/DOWN", overlay=true)

// =======================
// ⏱️ TIMEFRAME CHECK (1 MIN)
// =======================
is_1m = timeframe.period == "1"

// =======================
// 🔁 NO-REPAINT ZIGZAG (PIVOTS)
// =======================
pivot_len = 5

ph = ta.pivothigh(high, pivot_len, pivot_len)
pl = ta.pivotlow(low, pivot_len, pivot_len)

var float last_high = na
var float prev_high = na
var float last_low = na
var float prev_low = na

if not na(ph)
    prev_high := last_high
    last_high := ph

if not na(pl)
    prev_low := last_low
    last_low := pl

// =======================
// 📈 TREND (CONFIRMED)
// =======================
trend_up = not na(prev_low) and last_low > prev_low
trend_down = not na(prev_high) and last_high < prev_high

// =======================
// 📊 SUPPORT & RESISTANCE
// =======================
support = ta.lowest(low, 10)[1]
resistance = ta.highest(high, 10)[1]

// ATR FILTER (Dynamic distance)
atr = ta.atr(14)
sr_distance = atr * 0.5

near_support = math.abs(close - support) < sr_distance
near_resistance = math.abs(close - resistance) < sr_distance

// =======================
// 🕯️ STRONG CANDLE FILTER
// =======================
bull = close > open and (close - open) > (high - low) * 0.5
bear = open > close and (open - close) > (high - low) * 0.5

// =======================
// 💥 STRONG BREAKOUT
// =======================
breakout_up = high > ta.highest(high, 5)[1]
breakout_down = low < ta.lowest(low, 5)[1]

// =======================
// 🎯 FINAL SIGNAL (NO REPAINT)
// =======================
up_signal = is_1m and barstate.isconfirmed and trend_up and bull[2] and bear[1] and bull and breakout_up and not near_resistance

down_signal = is_1m and barstate.isconfirmed and trend_down and bear[2] and bull[1] and bear and breakout_down and not near_support

// =======================
// 📍 PLOT SIGNALS
// =======================
plotshape(up_signal, title="BUY", style=shape.labelup, text="UP", location=location.belowbar, color=color.green, textcolor=color.white)

plotshape(down_signal, title="SELL", style=shape.labeldown, text="DOWN", location=location.abovebar, color=color.red, textcolor=color.white)

// =======================
// 🔔 ALERT SYSTEM (BOT READY)
// =======================
entry_time = str.format("{0,time,HH:mm}", time)

if up_signal
    alert("SIGNAL=UP|PRICE=" + str.tostring(close) + "|TIME=" + entry_time, alert.freq_once_per_bar_close)

if down_signal
    alert("SIGNAL=DOWN|PRICE=" + str.tostring(close) + "|TIME=" + entry_time, alert.freq_once_per_bar_close)

// =======================
// 📊 OPTIONAL: BACKGROUND TREND COLOR
// =======================
bgcolor(trend_up ? color.new(color.green, 90) : trend_down ? color.new(color.red, 90) : na)


```

