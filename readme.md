//@version=6
//==============================================================================
// PARANA PROJECT - DIRECTION AUDIT
//------------------------------------------------------------------------------
// Version : v0.1.0
// Purpose : Validate the directional-reading layer before using it as an
//           ingredient in a trading strategy.
//
// A directional call requires agreement between:
// - confirmed local swing structure,
// - local EMA regime,
// - two confirmed higher-timeframe regimes,
// - optional relative-volume participation.
//
// The script does NOT place orders. After a fixed number of chart bars it
// records whether each fresh bullish/bearish call was directionally correct.
//==============================================================================

indicator(
     title = "Parana Project Direction Audit v0.1.0",
     shorttitle = "PARANA DIRECTION v0.1",
     overlay = true,
     max_labels_count = 500
)

//==============================================================================
// 01. INPUTS
//==============================================================================

string cfg_groupLocal = "01 - Local Direction"
string cfg_groupHtf = "02 - Higher Timeframes"
string cfg_groupAudit = "03 - Audit"
string cfg_groupVisual = "04 - Visuals"

int cfg_pivotLength = input.int(5, "Pivot sensitivity", minval = 2, maxval = 50, group = cfg_groupLocal)
int cfg_fastEmaLength = input.int(50, "Fast EMA", minval = 5, maxval = 500, group = cfg_groupLocal)
int cfg_slowEmaLength = input.int(200, "Slow EMA", minval = 20, maxval = 500, group = cfg_groupLocal)
int cfg_volumeLength = input.int(20, "Relative volume lookback", minval = 5, maxval = 200, group = cfg_groupLocal)
float cfg_minRelativeVolume = input.float(1.0, "Minimum relative volume", minval = 0.1, step = 0.05, group = cfg_groupLocal)
bool cfg_useVolume = input.bool(true, "Use volume as confluence", group = cfg_groupLocal)

string cfg_htf1 = input.timeframe("D", "Higher timeframe 1", group = cfg_groupHtf, tooltip = "Recommended for a 4H chart: D.")
string cfg_htf2 = input.timeframe("W", "Higher timeframe 2", group = cfg_groupHtf, tooltip = "Recommended for a 4H chart: W.")
int cfg_htfEmaLength = input.int(50, "Higher-timeframe EMA", minval = 5, maxval = 500, group = cfg_groupHtf)
int cfg_minDirectionScore = input.int(70, "Minimum direction score", minval = 40, maxval = 100, group = cfg_groupHtf)

int cfg_evaluationBars = input.int(12, "Evaluation horizon (chart bars)", minval = 1, maxval = 500, group = cfg_groupAudit, tooltip = "On a 4H chart, 12 bars equal approximately two days.")
int cfg_minSamples = input.int(20, "Minimum calls before judging", minval = 5, maxval = 500, group = cfg_groupAudit)

bool cfg_showCalls = input.bool(true, "Show fresh directional calls", group = cfg_groupVisual)
bool cfg_showDashboard = input.bool(true, "Show direction dashboard", group = cfg_groupVisual)
bool cfg_showEma = input.bool(true, "Show local EMAs", group = cfg_groupVisual)

//==============================================================================
// 02. CONFIRMED LOCAL STRUCTURE
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

float fastEma = ta.ema(close, cfg_fastEmaLength)
float slowEma = ta.ema(close, cfg_slowEmaLength)
bool emaBull = close > fastEma and fastEma > slowEma
bool emaBear = close < fastEma and fastEma < slowEma

float volumeAverage = ta.sma(volume, cfg_volumeLength)
float relativeVolume = not na(volumeAverage) and volumeAverage > 0.0 ? volume / volumeAverage : na
bool volumePass = not cfg_useVolume or (not na(relativeVolume) and relativeVolume >= cfg_minRelativeVolume)

//==============================================================================
// 03. CONFIRMED HIGHER-TIMEFRAME DIRECTION
// The [1] expression uses only completed higher-timeframe bars. lookahead_on
// makes that confirmed value available across the following lower-timeframe bar.
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
// 04. DIRECTION SCORE AND FRESH CALLS
//==============================================================================

int bullScore = (structureBull ? 35 : 0) + (emaBull ? 15 : 0) + (htf1Direction == 1 ? 20 : 0) + (htf2Direction == 1 ? 20 : 0) + (volumePass ? 10 : 0)
int bearScore = (structureBear ? 35 : 0) + (emaBear ? 15 : 0) + (htf1Direction == -1 ? 20 : 0) + (htf2Direction == -1 ? 20 : 0) + (volumePass ? 10 : 0)

int directionState = bullScore >= cfg_minDirectionScore and bullScore > bearScore ? 1 : bearScore >= cfg_minDirectionScore and bearScore > bullScore ? -1 : 0
string directionText = directionState == 1 ? "BULLISH" : directionState == -1 ? "BEARISH" : "NEUTRAL"
color directionColor = directionState == 1 ? color.lime : directionState == -1 ? color.red : color.silver

bool bullCall = barstate.isconfirmed and directionState == 1 and nz(directionState[1]) != 1
bool bearCall = barstate.isconfirmed and directionState == -1 and nz(directionState[1]) != -1

//==============================================================================
// 05. DIRECTIONAL ACCURACY AUDIT
// A call is correct if price is in the called direction after the selected
// horizon. This measures direction only, independently from entries/stops/TP.
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
// 06. VISUALS
//==============================================================================

plot(cfg_showEma ? fastEma : na, "Fast EMA", color = color.orange, linewidth = 2)
plot(cfg_showEma ? slowEma : na, "Slow EMA", color = color.blue, linewidth = 2)
plotshape(cfg_showCalls and bullCall, title = "Bullish direction call", style = shape.labelup, location = location.belowbar, color = color.lime, textcolor = color.black, size = size.tiny, text = "DIR +")
plotshape(cfg_showCalls and bearCall, title = "Bearish direction call", style = shape.labeldown, location = location.abovebar, color = color.red, textcolor = color.white, size = size.tiny, text = "DIR -")

var table g_dashboard = table.new(position.top_right, 2, 10, border_width = 1)

f_cell(int _column, int _row, string _text, color _background, color _textColor) =>
    table.cell(g_dashboard, _column, _row, _text, bgcolor = _background, text_color = _textColor, text_size = size.tiny)

if barstate.islast and cfg_showDashboard
    color headerBg = color.rgb(20, 31, 45)
    color labelBg = color.rgb(38, 48, 61)
    color valueBg = color.rgb(28, 35, 45)
    f_cell(0, 0, "PARANA DIRECTION", headerBg, color.white)
    f_cell(1, 0, "v0.1 AUDIT", headerBg, color.white)
    f_cell(0, 1, "Current direction", labelBg, color.white)
    f_cell(1, 1, directionText + " " + str.tostring(math.max(bullScore, bearScore)), directionColor, color.black)
    f_cell(0, 2, "Local structure", labelBg, color.white)
    f_cell(1, 2, structureBull ? "BULLISH" : structureBear ? "BEARISH" : "NEUTRAL", valueBg, color.white)
    f_cell(0, 3, "HTF " + cfg_htf1, labelBg, color.white)
    f_cell(1, 3, htf1Direction == 1 ? "BULLISH" : htf1Direction == -1 ? "BEARISH" : "NEUTRAL", valueBg, color.white)
    f_cell(0, 4, "HTF " + cfg_htf2, labelBg, color.white)
    f_cell(1, 4, htf2Direction == 1 ? "BULLISH" : htf2Direction == -1 ? "BEARISH" : "NEUTRAL", valueBg, color.white)
    f_cell(0, 5, "Relative volume", labelBg, color.white)
    f_cell(1, 5, na(relativeVolume) ? "n/a" : str.tostring(relativeVolume, "#.##") + "x", valueBg, color.white)
    f_cell(0, 6, "Calls evaluated", labelBg, color.white)
    f_cell(1, 6, str.tostring(g_totalCalls), valueBg, color.white)
    f_cell(0, 7, "Overall accuracy", labelBg, color.white)
    f_cell(1, 7, na(accuracy) ? "n/a" : str.tostring(accuracy, "#.0") + "%", valueBg, color.white)
    f_cell(0, 8, "Bull / Bear hit rate", labelBg, color.white)
    f_cell(1, 8, (na(bullAccuracy) ? "n/a" : str.tostring(bullAccuracy, "#.0") + "%") + " / " + (na(bearAccuracy) ? "n/a" : str.tostring(bearAccuracy, "#.0") + "%"), valueBg, color.white)
    f_cell(0, 9, "Audit result", labelBg, color.white)
    f_cell(1, 9, auditStatus, auditStatus == "DIRECTION EDGE CANDIDATE" ? color.lime : auditStatus == "NO DIRECTION EDGE" ? color.red : color.orange, color.black)

//==============================================================================
// END OF v0.1.0
//==============================================================================
