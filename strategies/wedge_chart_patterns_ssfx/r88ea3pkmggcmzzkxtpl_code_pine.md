
# Source Code

```pinescript

// This work is licensed under Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International
// https://creativecommons.org/licenses/by-nc-sa/4.0/
// © SimplySafeFx QuantTrading

//@version=6
indicator("Wedge Chart Patterns [SSFX]", overlay = true, max_lines_count = 500, max_labels_count = 500)

// -----------------------------------------------------------------------------
// Inputs
// -----------------------------------------------------------------------------
grp_detect = "Pattern Detection Rules"
int INPUT_PIVOT_LEFT      = input.int(5, "Pivot Left", minval = 2, group = grp_detect)
int INPUT_PIVOT_RIGHT     = input.int(5, "Pivot Right", minval = 2, group = grp_detect)
int MIN_TOUCHES_PER_LINE  = input.int(3, "Minimum Touches per Line", minval = 2, group = grp_detect)

grp_entry = "Entry & Risk Management"
float MIN_BODY_PCT        = input.float(50.0, "Minimum Breakout Body %", minval = 1, maxval = 100, step = 1, group = grp_entry)
string SL_MODE            = input.string("Breakout Candle", "SL Mode", options = ["Breakout Candle", "Previous Swing", "Fixed Points"], group = grp_entry)
string TP_MODE            = input.string("Risk Reward", "TP Mode", options = ["Risk Reward", "Fixed Points"], group = grp_entry)
float RR_RATIO            = input.float(1.0, "Risk Reward Ratio", minval = 1.0, step = 0.1, group = grp_entry)
float FIXED_SL_POINTS     = input.float(300.0, "Fixed SL Points", minval = 0.1, step = 0.1, group = grp_entry)
float FIXED_TP_POINTS     = input.float(300.0, "Fixed TP Points", minval = 0.1, step = 0.1, group = grp_entry)
int LEVEL_EXTEND_BARS     = input.int(25, "Extend Entry / SL / TP (bars)", minval = 5, group = grp_entry)
int MAX_TRADE_HISTORY     = input.int(10, "Keep Trades", minval = 1, maxval = 20, group = grp_entry)

grp_visual = "Visual Design"
color INPUT_COL_FALLING   = input.color(color.new(color.lime, 0), "Falling Wedge Color", group = grp_visual)
color INPUT_COL_RISING    = input.color(color.new(color.red, 0), "Rising Wedge Color", group = grp_visual)
color LONG_COLOR          = input.color(color.new(color.lime, 0), "Long Arrow Color", group = grp_visual)
color SHORT_COLOR         = input.color(color.new(color.red, 0), "Short Arrow Color", group = grp_visual)
color ENTRY_COLOR         = input.color(color.new(color.aqua, 0), "Entry Line Color", group = grp_visual)
color SL_COLOR            = input.color(color.new(color.red, 0), "SL Line Color", group = grp_visual)
color TP_COLOR            = input.color(color.new(color.green, 0), "TP Line Color", group = grp_visual)
color SL_FILL_COLOR       = input.color(color.new(color.red, 82), "SL Fill Color", group = grp_visual)
color TP_FILL_COLOR       = input.color(color.new(color.green, 82), "TP Fill Color", group = grp_visual)
bool SHOW_WEDGE_LINES     = input.bool(true, "Show Wedge Lines", group = grp_visual)
bool SHOW_TRADE_LEVELS    = input.bool(true, "Show Entry / SL / TP", group = grp_visual)
bool SHOW_SIGNAL_MARKERS  = input.bool(true, "Show Entry Arrows", group = grp_visual)
bool SHOW_PRICE_LABELS    = input.bool(true, "Show Entry / SL / TP Price Labels", group = grp_visual)

// -----------------------------------------------------------------------------
// Types
// -----------------------------------------------------------------------------
type Coordinate
    int   index
    float price

type Trendline
    Coordinate start
    Coordinate end
    float      slope
    line       line_id

type Wedge
    Trendline upper
    Trendline lower
    bool      is_rising
    linefill  fill_id
    bool      is_broken = false
    bool      is_failed = false

// -----------------------------------------------------------------------------
// Methods
// -----------------------------------------------------------------------------
method calc_slope(Trendline this) =>
    float denom = math.max(this.end.index - this.start.index, 1)
    (this.end.price - this.start.price) / denom

method get_price_at(Trendline this, int bar_idx) =>
    this.start.price + this.slope * (bar_idx - this.start.index)

method get_apex_index(Wedge this) =>
    float m1 = this.upper.slope
    float m2 = this.lower.slope
    float y1 = this.upper.start.price
    float y2 = this.lower.start.price
    int x1   = this.upper.start.index
    int x2   = this.lower.start.index
    float denom = m1 - m2
    denom == 0.0 ? na : (y2 - y1 + m1 * x1 - m2 * x2) / denom

method check_violation(Wedge this, int from_idx, int to_idx) =>
    bool violated = false
    for i = from_idx to to_idx
        float up_p = this.upper.get_price_at(i)
        float lo_p = this.lower.get_price_at(i)
        float c_p  = close[bar_index - i]
        if c_p > up_p + syminfo.mintick or c_p < lo_p - syminfo.mintick
            violated := true
            break
    violated

method delete_drawings(Wedge this) =>
    if not na(this.fill_id)
        linefill.delete(this.fill_id)
    if not na(this.upper.line_id)
        line.delete(this.upper.line_id)
    if not na(this.lower.line_id)
        line.delete(this.lower.line_id)

method draw_pattern(Wedge this) =>
    color c = this.is_rising ? INPUT_COL_RISING : INPUT_COL_FALLING
    color lineC = SHOW_WEDGE_LINES ? c : color.new(c, 100)
    color fillC = SHOW_WEDGE_LINES ? color.new(c, 90) : color.new(c, 100)

    this.upper.line_id := line.new(
         this.upper.start.index, this.upper.start.price,
         this.upper.end.index,   this.upper.end.price,
         xloc = xloc.bar_index, color = lineC, width = 1)

    this.lower.line_id := line.new(
         this.lower.start.index, this.lower.start.price,
         this.lower.end.index,   this.lower.end.price,
         xloc = xloc.bar_index, color = lineC, width = 1)

    this.fill_id := linefill.new(this.upper.line_id, this.lower.line_id, fillC)

// -----------------------------------------------------------------------------
// State
// -----------------------------------------------------------------------------
var array<Coordinate> pivot_highs = array.new<Coordinate>()
var array<Coordinate> pivot_lows  = array.new<Coordinate>()

var Wedge active_rising  = na
var Wedge active_falling = na

var array<line> entry_lines   = array.new<line>()
var array<line> sl_lines      = array.new<line>()
var array<line> tp_lines      = array.new<line>()
var array<linefill> sl_fills  = array.new<linefill>()
var array<linefill> tp_fills  = array.new<linefill>()

var array<label> entry_labels = array.new<label>()
var array<label> sl_labels    = array.new<label>()
var array<label> tp_labels    = array.new<label>()

float hiddenEntry = na
float hiddenSL    = na
float hiddenTP    = na

// -----------------------------------------------------------------------------
// Helper functions
// -----------------------------------------------------------------------------
trim_trade_history() =>
    while array.size(entry_lines) > MAX_TRADE_HISTORY
        line oldEntry = array.shift(entry_lines)
        line oldSL    = array.shift(sl_lines)
        line oldTP    = array.shift(tp_lines)

        linefill oldSLFill = array.shift(sl_fills)
        linefill oldTPFill = array.shift(tp_fills)

        label oldEntryLabel = array.shift(entry_labels)
        label oldSLLabel    = array.shift(sl_labels)
        label oldTPLabel    = array.shift(tp_labels)

        if not na(oldSLFill)
            linefill.delete(oldSLFill)
        if not na(oldTPFill)
            linefill.delete(oldTPFill)

        if not na(oldEntry)
            line.delete(oldEntry)
        if not na(oldSL)
            line.delete(oldSL)
        if not na(oldTP)
            line.delete(oldTP)

        if not na(oldEntryLabel)
            label.delete(oldEntryLabel)
        if not na(oldSLLabel)
            label.delete(oldSLLabel)
        if not na(oldTPLabel)
            label.delete(oldTPLabel)

store_trade_drawings(float entryPrice, float slPrice, float tpPrice) =>
    if SHOW_TRADE_LEVELS
        int x2 = bar_index + LEVEL_EXTEND_BARS

        line entryLine = line.new(bar_index, entryPrice, x2, entryPrice, xloc = xloc.bar_index, color = ENTRY_COLOR, width = 2)
        line slLine    = line.new(bar_index, slPrice,    x2, slPrice,    xloc = xloc.bar_index, color = SL_COLOR,    width = 2)
        line tpLine    = line.new(bar_index, tpPrice,    x2, tpPrice,    xloc = xloc.bar_index, color = TP_COLOR,    width = 2)

        linefill slFill = linefill.new(entryLine, slLine, SL_FILL_COLOR)
        linefill tpFill = linefill.new(entryLine, tpLine, TP_FILL_COLOR)

        label entryLabel = na
        label slLabel    = na
        label tpLabel    = na

        if SHOW_PRICE_LABELS
            entryLabel := label.new(
                 x2, entryPrice,
                 "Entry " + str.tostring(entryPrice, format.mintick),
                 xloc = xloc.bar_index,
                 style = label.style_label_left,
                 color = ENTRY_COLOR,
                 textcolor = color.white,
                 size = size.small)

            slLabel := label.new(
                 x2, slPrice,
                 "SL " + str.tostring(slPrice, format.mintick),
                 xloc = xloc.bar_index,
                 style = label.style_label_left,
                 color = SL_COLOR,
                 textcolor = color.white,
                 size = size.small)

            tpLabel := label.new(
                 x2, tpPrice,
                 "TP " + str.tostring(tpPrice, format.mintick),
                 xloc = xloc.bar_index,
                 style = label.style_label_left,
                 color = TP_COLOR,
                 textcolor = color.white,
                 size = size.small)

        array.push(entry_lines, entryLine)
        array.push(sl_lines, slLine)
        array.push(tp_lines, tpLine)

        array.push(sl_fills, slFill)
        array.push(tp_fills, tpFill)

        array.push(entry_labels, entryLabel)
        array.push(sl_labels, slLabel)
        array.push(tp_labels, tpLabel)

        trim_trade_history()

process_wedge(Wedge w) =>
    int signalDir = 0
    float bodyPct = 0.0

    if not na(w) and not w.is_broken and not w.is_failed
        line.set_x2(w.upper.line_id, bar_index)
        line.set_y2(w.upper.line_id, w.upper.get_price_at(bar_index))
        line.set_x2(w.lower.line_id, bar_index)
        line.set_y2(w.lower.line_id, w.lower.get_price_at(bar_index))

        float u_p = w.upper.get_price_at(bar_index)
        float l_p = w.lower.get_price_at(bar_index)

        float candleRange = math.max(high - low, syminfo.mintick)
        bodyPct := math.abs(close - open) / candleRange * 100.0

        bool b_up = close > u_p
        bool b_dn = close < l_p

        bool correctBreakout = (w.is_rising and b_dn) or (not w.is_rising and b_up)
        bool invalidation    = (w.is_rising and b_up) or (not w.is_rising and b_dn) or (u_p <= l_p)
        bool strongBody      = bodyPct >= MIN_BODY_PCT

        if correctBreakout and strongBody
            w.is_broken := true
            signalDir := w.is_rising ? -1 : 1
        else if correctBreakout and not strongBody
            w.is_failed := true
        else if invalidation
            w.is_failed := true

    [signalDir, bodyPct]

// -----------------------------------------------------------------------------
// Pivot collection
// -----------------------------------------------------------------------------
float ph = ta.pivothigh(high, INPUT_PIVOT_LEFT, INPUT_PIVOT_RIGHT)
float pl = ta.pivotlow(low, INPUT_PIVOT_LEFT, INPUT_PIVOT_RIGHT)

if not na(ph)
    array.push(pivot_highs, Coordinate.new(bar_index - INPUT_PIVOT_RIGHT, ph))
if not na(pl)
    array.push(pivot_lows, Coordinate.new(bar_index - INPUT_PIVOT_RIGHT, pl))

if array.size(pivot_highs) > 20
    array.shift(pivot_highs)
if array.size(pivot_lows) > 20
    array.shift(pivot_lows)

// -----------------------------------------------------------------------------
// Pattern detection
// -----------------------------------------------------------------------------
if (not na(ph) or not na(pl)) and array.size(pivot_highs) >= MIN_TOUCHES_PER_LINE and array.size(pivot_lows) >= MIN_TOUCHES_PER_LINE
    Coordinate p1h = array.get(pivot_highs, array.size(pivot_highs) - MIN_TOUCHES_PER_LINE)
    Coordinate pNh = array.get(pivot_highs, array.size(pivot_highs) - 1)
    Coordinate p1l = array.get(pivot_lows,  array.size(pivot_lows)  - MIN_TOUCHES_PER_LINE)
    Coordinate pNl = array.get(pivot_lows,  array.size(pivot_lows)  - 1)

    Trendline tl_up = Trendline.new(p1h, pNh, 0.0, na)
    Trendline tl_lo = Trendline.new(p1l, pNl, 0.0, na)

    tl_up.slope := tl_up.calc_slope()
    tl_lo.slope := tl_lo.calc_slope()

    int w_type = 0
    if tl_up.slope > 0 and tl_lo.slope > 0 and tl_lo.slope > tl_up.slope
        w_type := 1
    if tl_up.slope < 0 and tl_lo.slope < 0 and tl_up.slope < tl_lo.slope
        w_type := 2

    if w_type != 0
        Wedge new_w = Wedge.new(tl_up, tl_lo, w_type == 1, na)
        float apex_idx = new_w.get_apex_index()
        int start_idx = math.min(p1h.index, p1l.index)

        if not na(apex_idx) and apex_idx > bar_index and not new_w.check_violation(start_idx, bar_index)
            if w_type == 1
                if not na(active_rising) and not active_rising.is_broken and not active_rising.is_failed
                    active_rising.delete_drawings()
                active_rising := new_w
            else
                if not na(active_falling) and not active_falling.is_broken and not active_falling.is_failed
                    active_falling.delete_drawings()
                active_falling := new_w

            new_w.draw_pattern()

// -----------------------------------------------------------------------------
// Signal evaluation
// -----------------------------------------------------------------------------
[risingSig, risingBody] = process_wedge(active_rising)
[fallingSig, fallingBody] = process_wedge(active_falling)

int signalDir = risingSig != 0 ? risingSig : fallingSig
bool longSignal  = signalDir == 1
bool shortSignal = signalDir == -1

hiddenEntry := na
hiddenSL := na
hiddenTP := na

// -----------------------------------------------------------------------------
// Trade logic
// -----------------------------------------------------------------------------
float recentSwingLow  = array.size(pivot_lows)  > 0 ? array.get(pivot_lows,  array.size(pivot_lows)  - 1).price : low
float recentSwingHigh = array.size(pivot_highs) > 0 ? array.get(pivot_highs, array.size(pivot_highs) - 1).price : high

if signalDir != 0
    float entryPrice   = close
    float fixedSLDist  = FIXED_SL_POINTS * syminfo.mintick
    float fixedTPDist  = FIXED_TP_POINTS * syminfo.mintick

    float slPrice      = na
    float tpPrice      = na
    float riskDistance = na

    if signalDir == 1
        slPrice :=
             SL_MODE == "Breakout Candle" ? low :
             SL_MODE == "Previous Swing"  ? recentSwingLow :
             entryPrice - fixedSLDist

        riskDistance := entryPrice - slPrice
        tpPrice := TP_MODE == "Fixed Points" ? entryPrice + fixedTPDist : entryPrice + riskDistance * RR_RATIO

    if signalDir == -1
        slPrice :=
             SL_MODE == "Breakout Candle" ? high :
             SL_MODE == "Previous Swing"  ? recentSwingHigh :
             entryPrice + fixedSLDist

        riskDistance := slPrice - entryPrice
        tpPrice := TP_MODE == "Fixed Points" ? entryPrice - fixedTPDist : entryPrice - riskDistance * RR_RATIO

    bool validTrade = signalDir == 1 ? (slPrice < entryPrice and tpPrice > entryPrice) : (slPrice > entryPrice and tpPrice < entryPrice)

    if validTrade
        store_trade_drawings(entryPrice, slPrice, tpPrice)
        hiddenEntry := entryPrice
        hiddenSL := slPrice
        hiddenTP := tpPrice

// -----------------------------------------------------------------------------
// Visual signals
// -----------------------------------------------------------------------------
plotshape(SHOW_SIGNAL_MARKERS and longSignal,  title = "Long Signal",  style = shape.triangleup,   location = location.belowbar, color = LONG_COLOR,  size = size.small)
plotshape(SHOW_SIGNAL_MARKERS and shortSignal, title = "Short Signal", style = shape.triangledown, location = location.abovebar, color = SHORT_COLOR, size = size.small)

// Hidden plots for alert placeholders
plot(hiddenEntry, "Entry", display = display.none)
plot(hiddenSL, "SL", display = display.none)
plot(hiddenTP, "TP", display = display.none)

alertcondition(longSignal,  title = "Bullish Wedge Breakout", message = "Bullish wedge breakout detected | Entry: {{plot(\"Entry\")}} | SL: {{plot(\"SL\")}} | TP: {{plot(\"TP\")}}")
alertcondition(shortSignal, title = "Bearish Wedge Breakout", message = "Bearish wedge breakout detected | Entry: {{plot(\"Entry\")}} | SL: {{plot(\"SL\")}} | TP: {{plot(\"TP\")}}")

```

