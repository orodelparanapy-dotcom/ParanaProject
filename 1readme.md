//@version=6
//==============================================================================
// PARANA PROJECT - DIRECTION AUDIT
//------------------------------------------------------------------------------
// Version : v0.2.0 Regime First
// Purpose : Audit direction only after classifying the market as TREND, RANGE
//           or TRANSITION. This is a diagnostic indicator, not a strategy.
//
// Design principle
// A direction is meaningful only when the market first proves it is trending.
// In a range or transition, the correct system response is WAIT / OBSERVE.
//==============================================================================

indicator(
     title = "Parana Project Direction Audit v0.2.0 - Regime First",
     shorttitle = "PARANA DIRECTION v0.2",
     overlay = true,
     max_labels_count = 500
)

//==============================================================================
// 01. INPUTS
//==============================================================================

string cfg_groupRegime = "01 - Regime Classification"
string cfg_groupStructure = "02 - Local Structure"
string cfg_groupHtf = "03 - Higher Timeframes"
string cfg_groupAudit = "04 - Audit"
string cfg_groupVisual = "05 - Visuals"

int cfg_fastEmaLength = input.int(50, "Fast EMA", minval = 5, maxval = 500, group = cfg_groupRegime)
int cfg_slowEmaLength = input.int(200, "Slow EMA", minval = 20, maxval = 500, group = cfg_groupRegime)
int cfg_atrLength = input.int(14, "ATR length", minval = 2, maxval = 100, group = cfg_groupRegime)
int cfg_diLength = input.int(14, "DI length", minval = 2, maxval = 100, group = cfg_groupRegime)
int cfg_adxSmoothing = input.int(14, "ADX smoothing", minval = 2, maxval = 100, group = cfg_groupRegime)
float cfg_trendAdx = input.float(22.0, "Minimum ADX for trend", minval = 5.0, step = 0.5, group = cfg_groupRegime)
float cfg_rangeAdx = input.float(17.0, "Maximum ADX for range", minval = 1.0, step = 0.5, group = cfg_groupRegime)
float cfg_minEmaSpreadAtr = input.float(0.50, "Minimum EMA spread (ATR)", minval = 0.05, step = 0.05, group = cfg_groupRegime)
int cfg_slopeLookback = input.int(10, "Slow EMA slope lookback", minval = 1, maxval = 100, group = cfg_groupRegime)
float cfg_minSlopeAtr = input.float(0.15, "Minimum slow EMA slope (ATR)", minval = 0.01, step = 0.01, group = cfg_groupRegime)

int cfg_pivotLength = input.int(5, "Pivot sensitivity", minval = 2, maxval = 50, group = cfg_groupStructure)
int cfg_volumeLength = input.int(20, "Relative volume lookback", minval = 5, maxval = 200, group = cfg_groupStructure)
float cfg_minRelativeVolume = input.float(1.0, "Minimum relative volume", minval = 0.1, step = 0.05, group = cfg_groupStructure)
bool cfg_useVolume = input.bool(true, "Use volume as confluence", group = cfg_groupStructure)

string cfg_htf1 = input.timeframe("D", "Higher timeframe 1", group = cfg_groupHtf, tooltip = "Recommended for a 4H chart: D.")
string cfg_htf2 = input.timeframe("W", "Higher timeframe 2", group = cfg_groupHtf, tooltip = "Recommended for a 4H chart: W.")
int cfg_htfEmaLength = input.int(50, "Higher-timeframe EMA", minval = 5, maxval = 500, group = cfg_groupHtf)
int cfg_minDirectionScore = input.int(75, "Minimum direction score", minval = 50, maxval = 100, group = cfg_groupHtf)

int cfg_evaluationBars = input.int(12, "Evaluation horizon (chart bars)", minval = 1, maxval = 500, group = cfg_groupAudit)
int cfg_minSamples = input.int(30, "Minimum calls before judging", minval = 5, maxval = 500, group = cfg_groupAudit)

bool cfg_showCalls = input.bool(true, "Show fresh directional calls", group = cfg_groupVisual)
bool cfg_showDashboard = input.bool(true, "Show direction dashboard", group = cfg_groupVisual)
bool cfg_showEma = input.bool(true, "Show local EMAs", group = cfg_groupVisual)
bool cfg_showRegimeBackground = input.bool(false, "Color range / transition background", group = cfg_groupVisual)

//==============================================================================
// 02. LOCAL MARKET DATA
//==============================================================================

float fastEma = ta.ema(close, cfg_fastEmaLength)
float slowEma = ta.ema(close, cfg_slowEmaLength)
float atr = ta.atr(cfg_atrLength)
[plusDi, minusDi, adx] = ta.dmi(cfg_diLength, cfg_adxSmoothing)

float emaSpreadAtr = atr > 0.0 ? math.abs(fastEma - slowEma) / atr : 0.0
float slowSlopeAtr = atr > 0.0 ? (slowEma - slowEma[cfg_slopeLookback]) / atr : 0.0
bool emaBull = close > fastEma and fastEma > slowEma
bool emaBear = close < fastEma and fastEma < slowEma
bool trendStrength = adx >= cfg_trendAdx and emaSpreadAtr >= cfg_minEmaSpreadAtr
bool rangeCondition = adx <= cfg_rangeAdx or emaSpreadAtr < cfg_minEmaSpreadAtr

//==============================================================================
// 03. CONFIRMED SWING STRUCTURE
//==============================================================================

float pivotHigh = ta.pivothigh(high, cfg_pivotLength, cfg_pivotLength)
float pivotLow = ta.pivotlow(low, cfg_pivotLength, cfg_pivotLength)

var float g_lastSwingHigh = na
var float g_previousSwingHigh = na
var float g_lastSwingLow = na
var float g_previousSwingLow = na

if not na(pivotHigh)
    g_previousSwingHigh := g_lastSwingHigh
    g_lastSwingHigh := pivotHigh

if not na(pivotLow)
    g_previousSwingLow := g_lastSwingLow
    g_lastSwingLow := pivotLow

bool structureBull = not na(g_previousSwingHigh) and not na(g_previousSwingLow) and g_lastSwingHigh > g_previousSwingHigh and g_lastSwingLow > g_previousSwingLow
bool structureBear = not na(g_previousSwingHigh) and not na(g_previousSwingLow) and g_lastSwingHigh < g_previousSwingHigh and g_lastSwingLow < g_previousSwingLow

float volumeAverage = ta.sma(volume, cfg_volumeLength)
float relativeVolume = not na(volumeAverage) and volumeAverage > 0.0 ? volume / volumeAverage : na
bool volumePass = not cfg_useVolume or (not na(relativeVolume) and relativeVolume >= cfg_minRelativeVolume)

//==============================================================================
// 04. CONFIRMED HIGHER-TIMEFRAME REGIME
// [1] prevents a current, still-forming higher-timeframe candle from changing
// a historical direction call.
//==============================================================================

f_htfDirection(string _timeframe, int _emaLength) =>
    request.security(
         syminfo.tickerid,
         _timeframe,
         close[1] > ta.ema(close, _emaLength)[1] ? 1 : close[1] < ta.ema(close, _emaLength)[1] ? -1 : 0,
         gaps = barmerge.gaps_off,
         lookahead = barmerge.lookahead_on
    )

int htf1Direction = f_htfDirection(cfg_htf1, cfg_htfEmaLength)
int htf2Direction = f_htfDirection(cfg_htf2, cfg_htfEmaLength)

//==============================================================================
// 05. REGIME FIRST, DIRECTION SECOND
//==============================================================================

bool localBullTrend = trendStrength and emaBull and slowSlopeAtr >= cfg_minSlopeAtr and structureBull
bool localBearTrend = trendStrength and emaBear and slowSlopeAtr <= -cfg_minSlopeAtr and structureBear

string regime = rangeCondition ? "RANGE" : localBullTrend or localBearTrend ? "TREND" : "TRANSITION"
color regimeColor = regime == "TREND" ? color.lime : regime == "RANGE" ? color.orange : color.silver

int bullScore = (structureBull ? 30 : 0) + (emaBull ? 20 : 0) + (slowSlopeAtr >= cfg_minSlopeAtr ? 10 : 0) + (htf1Direction == 1 ? 15 : 0) + (htf2Direction == 1 ? 15 : 0) + (volumePass ? 10 : 0)
int bearScore = (structureBear ? 30 : 0) + (emaBear ? 20 : 0) + (slowSlopeAtr <= -cfg_minSlopeAtr ? 10 : 0) + (htf1Direction == -1 ? 15 : 0) + (htf2Direction == -1 ? 15 : 0) + (volumePass ? 10 : 0)

int directionState = regime == "TREND" and bullScore >= cfg_minDirectionScore and bullScore > bearScore ? 1 : regime == "TREND" and bearScore >= cfg_minDirectionScore and bearScore > bullScore ? -1 : 0
string directionText = directionState == 1 ? "BULLISH" : directionState == -1 ? "BEARISH" : "WAIT"
color directionColor = directionState == 1 ? color.lime : directionState == -1 ? color.red : color.silver

bool bullCall = barstate.isconfirmed and directionState == 1 and nz(directionState[1]) != 1
bool bearCall = barstate.isconfirmed and directionState == -1 and nz(directionState[1]) != -1

//==============================================================================
// 06. DIRECTIONAL ACCURACY AUDIT
//==============================================================================

var int g_totalCalls = 0
var int g_correctCalls = 0
var int g_bullCalls = 0
var int g_bullCorrect = 0
var int g_bearCalls = 0
var int g_bearCorrect = 0

if barstate.isconfirmed and bullCall[cfg_evaluationBars]
    g_totalCalls += 1
    g_bullCalls += 1
    if close > close[cfg_evaluationBars]
        g_correctCalls += 1
        g_bullCorrect += 1

if barstate.isconfirmed and bearCall[cfg_evaluationBars]
    g_totalCalls += 1
    g_bearCalls += 1
    if close < close[cfg_evaluationBars]
        g_correctCalls += 1
        g_bearCorrect += 1

float accuracy = g_totalCalls > 0 ? g_correctCalls * 100.0 / g_totalCalls : na
float bullAccuracy = g_bullCalls > 0 ? g_bullCorrect * 100.0 / g_bullCalls : na
float bearAccuracy = g_bearCalls > 0 ? g_bearCorrect * 100.0 / g_bearCalls : na
string auditStatus = g_totalCalls < cfg_minSamples ? "MORE SAMPLES NEEDED" : accuracy >= 55.0 ? "DIRECTION EDGE CANDIDATE" : "NO DIRECTION EDGE"

//==============================================================================
// 07. VISUALS
//==============================================================================

plot(cfg_showEma ? fastEma : na, "Fast EMA", color = color.orange, linewidth = 2)
plot(cfg_showEma ? slowEma : na, "Slow EMA", color = color.blue, linewidth = 2)
plotshape(cfg_showCalls and bullCall, title = "Bullish direction call", style = shape.labelup, location = location.belowbar, color = color.lime, textcolor = color.black, size = size.tiny, text = "DIR +")
plotshape(cfg_showCalls and bearCall, title = "Bearish direction call", style = shape.labeldown, location = location.abovebar, color = color.red, textcolor = color.white, size = size.tiny, text = "DIR -")
bgcolor(cfg_showRegimeBackground ? regime == "RANGE" ? color.new(color.orange, 90) : regime == "TRANSITION" ? color.new(color.silver, 92) : na : na, title = "Regime background")

var table g_dashboard = table.new(position.top_right, 2, 12, border_width = 1)

f_cell(int _column, int _row, string _text, color _background, color _textColor) =>
    table.cell(g_dashboard, _column, _row, _text, bgcolor = _background, text_color = _textColor, text_size = size.tiny)

if barstate.islast and cfg_showDashboard
    color headerBg = color.rgb(20, 31, 45)
    color labelBg = color.rgb(38, 48, 61)
    color valueBg = color.rgb(28, 35, 45)
    f_cell(0, 0, "PARANA DIRECTION", headerBg, color.white)
    f_cell(1, 0, "v0.2 REGIME", headerBg, color.white)
    f_cell(0, 1, "Market regime", labelBg, color.white)
    f_cell(1, 1, regime, regimeColor, color.black)
    f_cell(0, 2, "Current direction", labelBg, color.white)
    f_cell(1, 2, directionText + " " + str.tostring(math.max(bullScore, bearScore)), directionColor, color.black)
    f_cell(0, 3, "Local structure", labelBg, color.white)
    f_cell(1, 3, structureBull ? "BULLISH" : structureBear ? "BEARISH" : "NEUTRAL", valueBg, color.white)
    f_cell(0, 4, "ADX / EMA spread", labelBg, color.white)
    f_cell(1, 4, str.tostring(adx, "#.0") + " / " + str.tostring(emaSpreadAtr, "#.00") + " ATR", valueBg, color.white)
    f_cell(0, 5, "HTF " + cfg_htf1, labelBg, color.white)
    f_cell(1, 5, htf1Direction == 1 ? "BULLISH" : htf1Direction == -1 ? "BEARISH" : "NEUTRAL", valueBg, color.white)
    f_cell(0, 6, "HTF " + cfg_htf2, labelBg, color.white)
    f_cell(1, 6, htf2Direction == 1 ? "BULLISH" : htf2Direction == -1 ? "BEARISH" : "NEUTRAL", valueBg, color.white)
    f_cell(0, 7, "Relative volume", labelBg, color.white)
    f_cell(1, 7, na(relativeVolume) ? "n/a" : str.tostring(relativeVolume, "#.##") + "x", valueBg, color.white)
    f_cell(0, 8, "Calls evaluated", labelBg, color.white)
    f_cell(1, 8, str.tostring(g_totalCalls), valueBg, color.white)
    f_cell(0, 9, "Overall accuracy", labelBg, color.white)
    f_cell(1, 9, na(accuracy) ? "n/a" : str.tostring(accuracy, "#.0") + "%", valueBg, color.white)
    f_cell(0, 10, "Bull / Bear hit rate", labelBg, color.white)
    f_cell(1, 10, (na(bullAccuracy) ? "n/a" : str.tostring(bullAccuracy, "#.0") + "%") + " / " + (na(bearAccuracy) ? "n/a" : str.tostring(bearAccuracy, "#.0") + "%"), valueBg, color.white)
    f_cell(0, 11, "Audit result", labelBg, color.white)
    f_cell(1, 11, auditStatus, auditStatus == "DIRECTION EDGE CANDIDATE" ? color.lime : auditStatus == "NO DIRECTION EDGE" ? color.red : color.orange, color.black)

//==============================================================================
// END OF v0.2.0
//==============================================================================
