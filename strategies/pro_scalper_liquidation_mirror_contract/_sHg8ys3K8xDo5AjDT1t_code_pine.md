
---
description: "Pine Script Source Code for Pro Scalper: Liquidation Mirror (Contract)"
---

# Source Code

```pinescript
//@version=5
indicator("Pro Scalper: Liquidation Mirror (Contract)", overlay=true, max_labels_count=500)

// ============================================================================
// 1. INPUTS (Configurable)
// ============================================================================
grp_trend    = "Trend Filter"
emaLength    = input.int(50, "Macro EMA Length", group=grp_trend)
stFactor     = input.float(3.0, "Supertrend Factor", group=grp_trend)
stPeriod     = input.int(10, "Supertrend Period", group=grp_trend)

grp_rev      = "Reversal Logic"
breakoutLen  = input.int(20, "Lookback Length (High/Low)", group=grp_rev)
volLimit     = input.float(1.5, "Liquidation Vol Multiplier", step=0.1, tooltip="Volume multiplier for spike detection. Lower = More Signals.", group=grp_rev)
adxMin       = input.int(30, "Min ADX (Trend Strength)", group=grp_rev)

grp_disp     = "Display"
showEma      = input.bool(true, "Show Macro EMA", group=grp_disp)
showLiqBg    = input.bool(true, "Highlight Liquidation Bars", group=grp_disp)

// ============================================================================
// 2. CORE CALCULATIONS
// ============================================================================
// Trend Baseline
macroEma = ta.ema(close, emaLength)
[_, direction] = ta.supertrend(stFactor, stPeriod)

// Momentum & Exhaustion
[_, _, adx] = ta.dmi(14, 14)
[_, _, hist] = ta.macd(close, 12, 26, 9)

// Volume Spike (Liquidation Proxy)
volSma = ta.sma(volume, 20)
isLiquidation = volume > (volSma * volLimit) 

// Exhaustion Check
momentumFade = (hist > 0 and hist < hist[1] * 1.02) or (hist < 0 and hist > hist[1] * 1.02)
adxHigh      = adx > adxMin

// Price Extremes
recentHigh = ta.highest(high[1], breakoutLen)
recentLow  = ta.lowest(low[1], breakoutLen)

// ============================================================================
// 3. ENTRY SIGNALS (MIRROR LOGIC)
// ============================================================================
// TOP ESCAPING (Short): Price hits New High + Vol Spike + Bullish Exhaustion
topEscaping = (direction < 0) and (high >= recentHigh) and isLiquidation and adxHigh and momentumFade

// BOTTOM FISHING (Long): Price hits New Low + Vol Spike + Bearish Exhaustion
bottomFishing = (direction > 0) and (low <= recentLow) and isLiquidation and adxHigh and momentumFade

// ============================================================================
// 4. VISUALS & ALERTS
// ============================================================================
plot(showEma ? macroEma : na, color=color.new(color.white, 80), title="Macro EMA")

// Signal Labels with New Terminology
if bottomFishing
    label.new(bar_index, low, "BOTTOM FISHING\n(LONG)", color=color.new(color.lime, 0), textcolor=color.black, style=label.style_label_up, yloc=yloc.belowbar, size=size.small)

if topEscaping
    label.new(bar_index, high, "TOP ESCAPING\n(SHORT)", color=color.new(color.red, 0), textcolor=color.white, style=label.style_label_down, yloc=yloc.abovebar, size=size.small)

// Bar Highlighting
barcolor(showLiqBg and isLiquidation ? color.new(color.aqua, 50) : na)

// Alerts
alertcondition(bottomFishing, "Bottom Fishing (Long)", "Liquidation found at bottom - Go Long")
alertcondition(topEscaping, "Top Escaping (Short)", "Liquidation found at top - Go Short")
```

