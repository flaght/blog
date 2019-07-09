## Alpha191分析

编号 | 因子公式| 因子含义|分析说明|
---|---|---|---|
alpha_1|(`-1 * CORR(RANK(DELTA(LOG(VOLUME), 1)),`<br>`RANK(((CLOSE - OPEN) / OPEN)), 6))`|刻画成交量变化排名与日内收益率排名的偏离程度，表现为成交量变化排名上升，日内收益率排名下降；成交量变化排名下降，日内收益率排名上升。|
alpha_2|`(-1 * DELTA((((CLOSE - LOW) - (`<br>`HIGH - CLOSE))/ (HIGH - LOW)), 1))`|刻画多空失衡变动情况，用(CLOSE - LOW) - (HIGH - CLOSE)) / (HIGH - LOW)表示多空力量不平衡度。|
alpha_3|`SUM((CLOSE=DELAY(CLOSE,1)?0:CLOSE-(`<br>`CLOSE>DELAY(CLOSE,1)?MIN(LOW,DELAY(`<br>`CLOSE,1)):MAX(HIGH,DELAY(CLOSE,1)))),6)`|刻画上升趋势中，当日收盘对比昨收盘价和最低价的涨幅；下降趋势中，当日收盘价对比昨收盘价和最高价的跌幅。|
alpha_4||区间突破因子。价格（2日均价）突破布林带（8日）上限取-1，价格突破布林带下限取1，价格在布林带区间内时，成交量放大或不变取1，成交量缩小取-1
alpha_5|`(-1 * TSMAX(CORR(TSRANK(VOLUME, 5),`<br>` TSRANK(HIGH, 5), 5), 3))`|短期（3天）内成交量、高点最大偏离程度。表现为短周期内（5天）成交量上升，最高价下降；或者成交量下降，最高价上升。|
alpha_6|`(RANK(SIGN(DELTA((((OPEN * 0.85) + (`<br>`HIGH * 0.15))), 4)))* -1)`|开盘价与最高值相对基期差别的负号的排序|
alpha_7|`((RANK(MAX((VWAP - CLOSE), 3)) + RANK(`<br>`MIN((VWAP - CLOSE), 3))) * RANK(`<br>`DELTA(VOLUME, 3)))`|VWAP与ClOSE差别的排名与VOLUME变动排名的差别|
alpha_8|`RANK(DELTA(((((HIGH + LOW) / 2`<br>`) * 0.2) + (VWAP * 0.8)), 4) * -1)`|最高价、最低价和均价的变化排名|
alpha_9|`SMA(((HIGH+LOW)/2-(DELAY(HIGH,1`<br>`)+DELAY(LOW,1))/2)*(HIGH-LOW`<br>`)/VOLUME,7,2)`|最高价和最低价均值的变动，日内波动与成交量之比的均值，均值方式为指数平滑法|
alpha_10|`(RANK(MAX(((RET < 0) ? STD(RET, `<br>`20) : CLOSE)^2),5)) `|收益率小于0比波动，收益率大于0比收盘价|
alpha_11|`SUM(((CLOSE-LOW)-(HIGH-CLOSE))/(`<br>`HIGH-LOW).*VOLUME,6)`|比较（CLOSE-LOW）与（HIGH-CLOSE），并对成交量进行加权再求和。该因子反映的是流入流出量。|
alpha_12|`(RANK((OPEN - (SUM(VWAP, 10) / 10)`<br>`))) * (-1 * (RANK(ABS((CLOSE - VWAP)))))`|OPEN与VWAP之差排名，乘上CLOSE与VWAP之差绝对值的排名|
alpha_13|`(((HIGH * LOW)^0.5) - VWAP)`|HIGH和LOW的几何平均数与VWAP之差|
alpha_14|`CLOSE-DELAY(CLOSE,5)`|5日收盘价之差||
alpha_15|`OPEN/DELAY(CLOSE,1)-1`|开盘价相对昨日收盘价涨幅|