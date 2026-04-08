
---

description: "Pine Script Source Code for Multi Timeframe Volume Profiles [TradingIQ]"

---

# Source Code

```pinescript
// This Pine Script® code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © KioseffTrading

//@version=6
indicator("Multi Timeframe Volume Profiles [TradingIQ]", overlay = true, max_boxes_count = 500, max_labels_count = 500, max_lines_count = 500, max_polylines_count = 100)

enum modelType 

    regVP   = "Traditional Volume Profile"
    deltaVP = "Delta Profile"

model          = input.enum(defval = modelType.regVP, title = "Model", options = [modelType.regVP, modelType.deltaVP])
rows           = input.int(defval = 20, minval = 5, maxval = 200, title = "Profile Rows")

htfUse1        = input.bool(defval = true, title = "", group = "HTF", inline = "1")
htf1           = input.timeframe("30", "HTF #1", group  = "HTF", inline = "1")
lowerTF1       = input.timeframe(defval = "1", title = "LTF Vol", group = "HTF", inline = "1")

htfUse2        = input.bool(defval = true, title = "", group = "HTF", inline = "2")
htf2           = input.timeframe("60", "HTF #2", group  = "HTF", inline = "2")
lowerTF2       = input.timeframe(defval = "1", title = "LTF Vol", group = "HTF", inline = "2")

htfUse3        = input.bool(defval = true, title = "", group = "HTF", inline = "3")
htf3           = input.timeframe("240", "HTF #3", group  = "HTF", inline = "3")
lowerTF3       = input.timeframe(defval = "1", title = "LTF Vol", group = "HTF", inline = "3")

htfUse4        = input.bool(defval = false, title = "", group = "HTF", inline = "4")
htf4           = input.timeframe("1D", "HTF #4", group  = "HTF", inline = "4")
lowerTF4       = input.timeframe(defval = "1", title = "LTF Vol", group = "HTF", inline = "4")

htfUse5        = input.bool(defval = false, title = "", group = "HTF", inline = "5")
htf5           = input.timeframe("1W", "HTF #5", group  = "HTF", inline = "5")
lowerTF5       = input.timeframe(defval = "1", title = "LTF Vol", group = "HTF", inline = "5")

showVals       = input.bool(defval = false, title = "Show Total Vol/Delta Labels")
upCol          = input.color(defval = #55ffda, title = "Up Color")
dnCol          = input.color(defval = color.rgb(255, 116, 116), title = "Down Color")
pocVaLab       = input.bool(defval = true, title = "POC + VA Labels")
type htfDraw 

    array<box>  body 
    array<line> wick
    array<label> keyPointsLabel 
    array<box>   valsBoxes

type dataStoreLTF

    float H 
    float L 
    float V

type profile 

    array<float>        levels
    array<float>        delta 
    array<float>        buyVol 
    array<float>        sellVol 
    array<float>        totalVol 
    array<dataStoreLTF> LTFvals
    array<chart.point>  normedValsBuy 
    array<chart.point>  normedValsSell
    array<polyline>     regVP 

direction(lowerTF) => 

    if lowerTF == "1T" 

        switch 

            close == bid => -1 
            close == ask =>  1
            =>               math.sign(close - close[1]) 
        
    else 

        math.sign(close - close[1])

method setVals(profile htfProfile, int dnLev, int upLev, float div) => 

    for x = dnLev to upLev

        absDiv = math.abs(div)

        htfProfile.delta   .set(x, htfProfile.delta.get(x) + div)
        htfProfile.totalVol.set(x, htfProfile.totalVol.get(x) + absDiv)

        switch math.sign(div) 

            1  => htfProfile.buyVol .set(x, htfProfile.buyVol .get(x) + absDiv)
            -1 => htfProfile.sellVol.set(x, htfProfile.sellVol.get(x) + absDiv)

method addPoints(array<chart.point> id, bool isBuy, int left, int normBuyEnd, int normSell, float level, float Range, float signedDelta, int right, int normSellEnd) => 

    tempPoints = 
    
     if model == modelType.regVP 

        switch isBuy

            true =>  array.from(
                                chart.point.from_index(left, level), 
                                chart.point.from_index(normBuyEnd, level), 
                                chart.point.from_index(normBuyEnd, level + Range), 
                                chart.point.from_index(left, level + Range)
                                )

            =>       array.from(
                                chart.point.from_index(normBuyEnd, level),
                                chart.point.from_index(normBuyEnd - normSell, level),
                                chart.point.from_index(normBuyEnd - normSell, level + Range),
                                chart.point.from_index(normBuyEnd, level + Range)
                                )

    else if model == modelType.deltaVP 

        switch 

            signedDelta == 1 and isBuy => array.from(

                     chart.point.from_index(left, level), 
                     chart.point.from_index(normBuyEnd, level), 
                     chart.point.from_index(normBuyEnd, level + Range), 
                     chart.point.from_index(left, level + Range)
                    
             )

            signedDelta == -1 and not isBuy => array.from(

                     chart.point.from_index(right, level), 
                     chart.point.from_index(normSellEnd, level), 
                     chart.point.from_index(normSellEnd, level + Range), 
                     chart.point.from_index(right, level + Range)
                    
             )      

            isBuy => array.from(chart.point.from_index(left, level), chart.point.from_index(left, level + Range))
            =>       array.from(chart.point.from_index(right, level), chart.point.from_index(right, level + Range))

    id.concat(tempPoints)

formTF(string HTF) =>

    isH   = str.contains(HTF, "D") or str.contains(HTF, "W") or str.contains(HTF, "M")
    tfNum = str.tonumber(HTF)

    switch 

        isH   => HTF 
        tfNum >= 60 and tfNum % 60 == 0 => str.tostring(tfNum / 60) + "h"
        =>       HTF

getHTFvals(HTF, left, right, lowerTF, use) => 

    if use

        var htfO = open, var htfH = high, var htfL = low, var htfT = time  

        var htfProfile  = profile.new(array.new<float>(rows), array.new<float>(rows, 0), array.new<float>(rows, 0), array.new<float>(rows, 0), array.new<float>(rows, 0), array.new<dataStoreLTF>(), array.new<chart.point>(), array.new<chart.point>(), array.new<polyline>(2))
        var miniProfile = profile.new(array.new<float>(20), array.new<float>(20, 0), array.new<float>(20, 0), array.new<float>(20, 0), array.new<float>(20, 0), array.new<dataStoreLTF>(), array.new<chart.point>(), array.new<chart.point>(), array.new<polyline>(2))

        [ltfH, ltfL, ltfV] = request.security_lower_tf("", lowerTF, [high, low, volume * direction(lowerTF)]) 


        if timeframe.change(HTF)

            htfO := open 
            htfH := high 
            htfL := low 
            htfT := time

        lastTime = last_bar_time - htfT <= timeframe.in_seconds(HTF) * 1000

        htfH := math.max(high, htfH)
        htfL := math.min(low, htfL)

        if lastTime

            Range   = (htfH - htfL) / rows
            Range20 = (htfH - htfL) / 20

            if timeframe.change(HTF)

                htfProfile.LTFvals.clear()

            if ltfH.size() > 0 

                for [i, data] in ltfV
    
                    htfProfile.LTFvals.push(dataStoreLTF.new(ltfH.get(i), ltfL.get(i), data))
            
            var dLevels = htfDraw.new(array.new<box>(), array.new<line>(), array.new<label>(), array.new<box>())

            if barstate.islast 

                bodySize = dLevels.body.size(), wickSize = dLevels.wick.size(), labelSize = dLevels.keyPointsLabel.size()
                valsBoxSize = dLevels.valsBoxes.size()

                if bodySize > 0 
                    for i = 0 to bodySize - 1
                        dLevels.body.shift().delete()

                if wickSize > 0 
                    for i = 0 to wickSize - 1
                        dLevels.wick.shift().delete()

                if labelSize > 0 
                    for i = 0 to labelSize - 1
                        dLevels.keyPointsLabel.shift().delete()

                if valsBoxSize > 0 
                    for i = 0 to valsBoxSize - 1 
                        dLevels.valsBoxes.shift().delete()

                for polys in htfProfile.regVP
                    polys.delete()

                htfProfile.normedValsBuy .clear()
                htfProfile.normedValsSell.clear()

                for i = 0 to rows - 1

                    htfProfile.levels.set(i, htfL + Range * i)

                for data in htfProfile.LTFvals

                    upLev = htfProfile.levels.binary_search_leftmost(data.H)
                    dnLev = htfProfile.levels.binary_search_leftmost(data.L)

                    div = data.V / (math.abs(upLev - dnLev) + 1)

                    htfProfile.setVals(dnLev, upLev, div)

                deltaAbs = htfProfile.delta.abs()

                [min, max] = switch model 

                    modelType.regVP   => [math.min(htfProfile.buyVol.min(), htfProfile.sellVol.min()), math.max(htfProfile.buyVol.max(), htfProfile.sellVol.max())]
                    modelType.deltaVP => [deltaAbs.min(), deltaAbs.max()]

                if model == modelType.regVP
                    htfProfile.normedValsSell.push(chart.point.from_index(left, htfL))

                for i = 0 to rows - 1

                    getBuy = htfProfile.buyVol.get(i), getSell  = htfProfile.sellVol.get(i)
                    level  = htfProfile.levels.get(i), getDelta = htfProfile.delta.get(i)

                    normBuy   = math.round(1 + (((getBuy - min) / (max - min)) * 15))
                    normSell  = math.round(1 + (((getSell - min) / (max - min)) * 15))
                    normDelta = math.round(1 + (((deltaAbs.get(i) - min) / (max - min)) * 15))

                    normBuyEnd  = switch model 

                        modelType.regVP   => left - normBuy
                        modelType.deltaVP => left - normDelta

                    normSellEnd = right + 2 + normDelta 
                    signedDelta = math.sign(getDelta)

                    htfProfile.normedValsBuy .addPoints(true, left, normBuyEnd, normSell, level, Range, signedDelta, right + 2, normSellEnd)
                    htfProfile.normedValsSell.addPoints(false, left, normBuyEnd, normSell, level, Range, signedDelta, right + 2, normSellEnd)

                    getTotalVol = htfProfile.totalVol.get(i)

                    if close >= level and close <= level + Range 

                        xPoint = switch model 
                            
                            modelType.regVP => normBuyEnd - normSell - 2
                            =>                 normBuyEnd - 2

                        dLevels.keyPointsLabel.push(label.new(xPoint, math.avg(level, level + Range), style = label.style_label_center, color = #00000000, textcolor = chart.fg_color, size = size.tiny, text = "●"))


                if showVals

                    for i = 0 to 19

                        miniProfile.levels.set(i, htfL + Range * i)

                    for data in htfProfile.LTFvals

                        upLev = miniProfile.levels.binary_search_leftmost(data.H)
                        dnLev = miniProfile.levels.binary_search_leftmost(data.L)

                        divMini = data.V / (math.abs(upLev - dnLev) + 1)

                        miniProfile.setVals(dnLev, upLev, divMini)

                    deltaAbsMini = miniProfile.delta.abs()

                    [minMini, maxMini] = switch model 

                        modelType.regVP   => [math.min(miniProfile.buyVol.min(), miniProfile.sellVol.min()), math.max(miniProfile.buyVol.max(), miniProfile.sellVol.max())]
                        modelType.deltaVP => [deltaAbsMini.min(), deltaAbsMini.max()]

                    if model == modelType.regVP
                        miniProfile.normedValsSell.push(chart.point.from_index(left, htfL))

                    for i = 0 to rows - 1

                        level  = miniProfile.levels.get(i)

                        seriesShow = switch model

                            modelType.regVP   => miniProfile.totalVol.get(i)
                            modelType.deltaVP => miniProfile.delta.get(i)

                        colNow = switch 

                            model == modelType.regVP =>                                  chart.fg_color
                            model == modelType.deltaVP and math.sign(seriesShow) == 1 => upCol
                            =>                                                           dnCol

                        dLevels.valsBoxes.push(box.new(left + 1, level, right + 1, level + Range20, text = str.tostring(seriesShow, format.volume), bgcolor = #00000000, text_color = colNow, border_color = #00000000))


                if model == modelType.regVP
                    htfProfile.normedValsSell.push(chart.point.from_index(left, htfH))

                htfProfile.regVP.set(1, polyline.new(htfProfile.normedValsSell, line_color = chart.bg_color, line_width = 1, fill_color = color.new(dnCol, 20)))
                htfProfile.regVP.set(0, polyline.new(htfProfile.normedValsBuy , line_color = chart.bg_color, line_width = 1, fill_color = color.new(upCol, 20)))

                pocIndex    = htfProfile.totalVol.indexof(htfProfile.totalVol.max())
                getPOClevel = htfProfile.levels.get(pocIndex)

                indexUp = pocIndex, indexDn = pocIndex, sum = 0.0

                target  = htfProfile.totalVol.sum() * 0.7

                loopSize = htfProfile.levels.size() 

                for i = 0 to loopSize - 1

                    if indexUp == indexDn 

                        sum     += htfProfile.totalVol.get(indexUp)
                        indexUp += 1
                        indexDn -= 1
                        if sum >= target 
                            break 

                    else 

                        if indexUp < loopSize 

                            sum     += htfProfile.totalVol.get(indexUp)
                            indexUp += 1
                            if sum >= target 
                                break

                        if indexDn > -1 

                            sum     += htfProfile.totalVol.get(indexDn) 
                            indexDn -= 1
                            if sum >= target 
                                break 
                            
                getVAHlevel = htfProfile.levels.get(math.min(indexUp, loopSize - 1))
                getVALlevel = htfProfile.levels.get(math.max(indexDn, 0))

                if pocVaLab

                    dLevels.keyPointsLabel.push(label.new(right + 8, math.avg(getPOClevel, getPOClevel + Range), style = label.style_label_center, color = #00000000, text = "◂ POC", textcolor = chart.fg_color, size = size.small))
                    dLevels.keyPointsLabel.push(label.new(right + 8, math.avg(getVAHlevel, getVAHlevel + Range), style = label.style_label_center, color = #00000000, text = "◂ VAH", textcolor = chart.fg_color, size = size.small))
                    dLevels.keyPointsLabel.push(label.new(right + 8, math.avg(getVALlevel, getVALlevel + Range), style = label.style_label_center, color = #00000000, text = "◂ VAL", textcolor = chart.fg_color, size = size.small))

                bodyCol = switch

                    close >= htfO => upCol
                    =>               dnCol

                mid = math.round(math.avg(left + 1, right + 1))

                dLevels.body.push(box.new(left + 1, math.max(close, htfO), right + 1, math.min(close, htfO), border_color = chart.bg_color, border_width = 1,bgcolor = color.new(bodyCol, showVals ? 80 : 0)))
                dLevels.wick.push(line.new(mid, htfH, mid, math.max(htfO, close) , color = color.gray, width = 2))
                dLevels.wick.push(line.new(mid, htfL, mid, math.min(htfO, close), color = color.gray, width = 2))
                dLevels.keyPointsLabel.push(label.new(mid, htfProfile.levels.last() + Range, style = label.style_label_down, color = #00000000, text = formTF(HTF), textcolor = chart.fg_color, size = size.small))

                y1 = math.max(close, htfO)
                y2 = math.min(close, htfO)

                pointArr = array.from(
                    chart.point.from_index(left  + 1, y1), 
                    chart.point.from_index(right + 1, y1), 
                    chart.point.from_index(left  + 1, y2), 
                    chart.point.from_index(right + 1, y2)  
                )



getHTFvals(htf1, bar_index + 49, bar_index + 59, lowerTF1, htfUse1)
getHTFvals(htf2, bar_index + 109, bar_index + 119, lowerTF2, htfUse2)
getHTFvals(htf3, bar_index + 169, bar_index + 179, lowerTF3, htfUse3)
getHTFvals(htf4, bar_index + 229, bar_index + 239, lowerTF4, htfUse4)
getHTFvals(htf5, bar_index + 289, bar_index + 299, lowerTF5, htfUse5)

```

