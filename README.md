//@version=6
//==============================================================================
// PARANA PROJECT
// Institutional Market Intelligence
//------------------------------------------------------------------------------
// Version : v0.8.0 Trade Horizon
// Type    : Indicator
//
// This release translates timeframe and confirmed HTF alignment into an
// operational horizon. It remains a diagnostic tool and never emits entries,
// exits, or buy/sell advice.
//==============================================================================

indicator(
     title = "Parana Project v0.8.0 - Trade Horizon",
     shorttitle = "PARANA v0.8",
     overlay = true,
     max_labels_count = 200,
     max_lines_count = 200,
     max_boxes_count = 50
)

//==============================================================================
// 01. PROJECT CONSTANTS
//==============================================================================

const string c_PROJECT_NAME = "PARANA PROJECT"
const string c_VERSION = "v0.8.0 Trade Horizon"
const string c_ENGINE_NAME = "Parana Structure Engine"
const string c_STATUS_RUNNING = "RUNNING"

const int c_SWING_HIGH = 1
const int c_SWING_LOW = -1
const int c_MAX_SWINGS = 200
const int c_MAX_RENDERED_LABELS = 190

const int c_DASHBOARD_COLUMNS = 2
const int c_DASHBOARD_ROWS = 21

//==============================================================================
// 02. USER CONFIGURATION
//==============================================================================

string cfg_groupGeneral = "01 - General"
string cfg_groupSwing = "02 - Swing Engine"
string cfg_groupStructure = "03 - Structure Breaks"
string cfg_groupMtf = "04 - Higher-Timeframe Context"
string cfg_groupHorizon = "05 - Trade Horizon"
string cfg_groupDev = "06 - Development"

bool cfg_showDashboard = input.bool(
     defval = true,
     title = "Show dashboard",
     group = cfg_groupGeneral,
     tooltip = "Shows the Parana status panel in the top-right corner."
)

bool cfg_showSwings = input.bool(
     defval = true,
     title = "Show confirmed swings",
     group = cfg_groupGeneral,
     tooltip = "Draws SH and SL only after pivot confirmation."
)

int cfg_pivotLength = input.int(
     defval = 5,
     title = "Pivot sensitivity",
     minval = 2,
     maxval = 50,
     group = cfg_groupSwing,
     tooltip = "Bars required on each side to confirm a pivot. Higher values show fewer, more significant swings."
)

float cfg_minSwingDistancePct = input.float(
     defval = 0.40,
     title = "Minimum distance (%)",
     minval = 0.01,
     step = 0.05,
     group = cfg_groupSwing,
     tooltip = "Minimum percentage distance from the previous accepted swing."
)

bool cfg_showStrength = input.bool(
     defval = false,
     title = "Show swing strength",
     group = cfg_groupSwing,
     tooltip = "Adds the initial 0-100 strength estimate to swing labels."
)

bool cfg_showStructureBreaks = input.bool(
     defval = true,
     title = "Show confirmed structure events",
     group = cfg_groupStructure,
     tooltip = "Shows BOS and CHoCH only after a candle closes beyond the latest confirmed swing level."
)

bool cfg_showHtfContext = input.bool(
     defval = true,
     title = "Show higher-timeframe context",
     group = cfg_groupMtf,
     tooltip = "Shows confirmed structural direction from three configurable timeframes."
)

string cfg_htfOne = input.timeframe(
     defval = "60",
     title = "Context timeframe 1",
     group = cfg_groupMtf,
     tooltip = "Use a timeframe above the chart timeframe for higher-timeframe context."
)

string cfg_htfTwo = input.timeframe(
     defval = "240",
     title = "Context timeframe 2",
     group = cfg_groupMtf,
     tooltip = "Common setting for a 15-minute execution chart: 4H."
)

string cfg_htfThree = input.timeframe(
     defval = "D",
     title = "Context timeframe 3",
     group = cfg_groupMtf,
     tooltip = "Common setting for the daily higher-timeframe market context."
)

bool cfg_showTradeHorizon = input.bool(
     defval = true,
     title = "Show trade horizon",
     group = cfg_groupHorizon,
     tooltip = "Classifies the natural management horizon from the chart timeframe and HTF alignment."
)

bool cfg_developerMode = input.bool(
     defval = false,
     title = "Developer mode",
     group = cfg_groupDev,
     tooltip = "Displays the latest accepted or rejected pivot event in the dashboard."
)

//==============================================================================
// 03. CORE DATA TYPES
//==============================================================================

// Swing is the durable contract consumed by future Structure, Liquidity, and
// Wyckoff engines. Fields not used in this release are intentionally present.
type Swing
    int id
    int kind
    float price
    int barIndex
    int timestamp
    bool confirmed
    bool broken
    float strength
    float atrDistance
    float volumeStrength
    float quality
    string state

//==============================================================================
// 04. CORE STATE AND MEMORY
//==============================================================================

var array<Swing> g_swings = array.new<Swing>(0)
var array<label> g_swingLabels = array.new<label>(0)
var int g_nextSwingId = 1
var string g_lastLog = "Engine initialized"
var string g_engineStatus = c_STATUS_RUNNING
var string g_trendState = "NEUTRAL"
var string g_lastStructureEvent = "None"
var int g_lastBullBreakSwingId = na
var int g_lastBearBreakSwingId = na

var table g_dashboard = table.new(
     position.top_right,
     c_DASHBOARD_COLUMNS,
     c_DASHBOARD_ROWS,
     border_width = 1,
     frame_width = 1,
     border_color = color.new(color.gray, 65),
     frame_color = color.new(color.gray, 65)
)

//==============================================================================
// 05. CORE UTILITY FUNCTIONS
//==============================================================================

f_boolText(bool _value) =>
    _value ? "Yes" : "No"

f_swingKindText(int _kind) =>
    _kind == c_SWING_HIGH ? "HIGH" : _kind == c_SWING_LOW ? "LOW" : "UNKNOWN"

f_swingShortText(int _kind) =>
    _kind == c_SWING_HIGH ? "SH" : _kind == c_SWING_LOW ? "SL" : "?"

f_statusColor(string _status) =>
    _status == c_STATUS_RUNNING ? color.new(color.lime, 78) : color.new(color.orange, 78)

f_classSupportsTrend(string _classification, string _trend) =>
    (_trend == "BULLISH" and (_classification == "HH" or _classification == "HL")) or
     (_trend == "BEARISH" and (_classification == "LH" or _classification == "LL"))

f_scoreLabel(float _score) =>
    _score >= 85.0 ? "STRONG" : _score >= 70.0 ? "ESTABLISHED" : _score >= 50.0 ? "DEVELOPING" : "WEAK"

f_directionText(int _direction) =>
    _direction == 1 ? "BULLISH" : _direction == -1 ? "BEARISH" : "NEUTRAL"

f_localDirection() =>
    g_trendState == "BULLISH" ? 1 : g_trendState == "BEARISH" ? -1 : 0

f_contextLabel(float _alignment) =>
    _alignment >= 100.0 ? "FULLY ALIGNED" : _alignment >= 66.0 ? "PARTIALLY ALIGNED" : _alignment > 0.0 ? "CONFLICTED" : "NO HTF BIAS"

f_tradeHorizon() =>
    float chartSeconds = timeframe.in_seconds()
    chartSeconds <= 300.0 ? "SCALPING" : chartSeconds <= 3600.0 ? "INTRADAY" : chartSeconds <= 86400.0 ? "SWING" : "POSITIONAL"

f_expectedDuration() =>
    float chartSeconds = timeframe.in_seconds()
    chartSeconds <= 300.0 ? "5-30 minutes" :
     chartSeconds <= 3600.0 ? "1-8 hours" :
     chartSeconds <= 14400.0 ? "2-7 days" :
     chartSeconds <= 86400.0 ? "1-6 weeks" : "Weeks to months"

f_horizonSupport(float _alignment) =>
    _alignment >= 100.0 ? "HTF SUPPORTIVE" : _alignment >= 66.0 ? "HTF PARTIAL" : _alignment > 0.0 ? "HTF CONFLICT" : "HTF NEUTRAL"

f_managementGuidance(float _alignment) =>
    _alignment >= 100.0 ? "Manage within stated horizon" :
     _alignment >= 66.0 ? "Use conservative duration" :
     _alignment > 0.0 ? "Do not extend the setup" : "Wait for HTF context"

// This compact, stateful structure model runs inside request.security(). It is
// intentionally display-only in v0.8.0; the full local PSE remains the source
// of the local score and visual structure events.
f_htfDirection(int _pivotLength) =>
    var float lastHigh = na
    var float lastLow = na
    var int trend = 0

    float pivotHigh = ta.pivothigh(high, _pivotLength, _pivotLength)
    float pivotLow = ta.pivotlow(low, _pivotLength, _pivotLength)

    if not na(pivotHigh)
        lastHigh := pivotHigh
    if not na(pivotLow)
        lastLow := pivotLow

    if not na(lastHigh) and close > lastHigh
        trend := 1
    else if not na(lastLow) and close < lastLow
        trend := -1
    trend

f_htfAlignment(int _local, int _one, int _two, int _three) =>
    float alignment = 0.0
    if _local != 0
        int evaluated = 0
        int aligned = 0
        if _one != 0
            evaluated += 1
            aligned += _one == _local ? 1 : 0
        if _two != 0
            evaluated += 1
            aligned += _two == _local ? 1 : 0
        if _three != 0
            evaluated += 1
            aligned += _three == _local ? 1 : 0
        alignment := evaluated > 0 ? aligned / evaluated * 100.0 : 0.0
    alignment

// The score is intentionally decomposable:
// 35 points: a directional state exists.
// 25 points: the most recent swing supports that state.
// 25 points: the latest event is a BOS in that direction.
// 15 points: recent swings are structurally consistent with that direction.
f_structureScore() =>
    float score = 0.0

    if g_trendState != "NEUTRAL"
        score += 35.0

        if array.size(g_swings) > 0
            Swing lastSwing = array.get(g_swings, array.size(g_swings) - 1)
            if f_classSupportsTrend(lastSwing.state, g_trendState)
                score += 25.0

        bool lastEventSupportsTrend =
             (g_trendState == "BULLISH" and str.contains(g_lastStructureEvent, "BOS UP")) or
             (g_trendState == "BEARISH" and str.contains(g_lastStructureEvent, "BOS DOWN"))
        if lastEventSupportsTrend
            score += 25.0

        int swingCount = array.size(g_swings)
        int sampleSize = math.min(swingCount, 4)
        int supportingSwings = 0
        if sampleSize > 0
            for i = 0 to sampleSize - 1
                Swing sampledSwing = array.get(g_swings, swingCount - 1 - i)
                if f_classSupportsTrend(sampledSwing.state, g_trendState)
                    supportingSwings += 1
            score += supportingSwings / sampleSize * 15.0

    math.min(score, 100.0)

f_lastSwingText() =>
    if array.size(g_swings) == 0
        "None"
    else
        Swing lastSwing = array.get(g_swings, array.size(g_swings) - 1)
        lastSwing.state + " @ " + str.tostring(lastSwing.price, format.mintick)

//==============================================================================
// 06. SWING ENGINE
//==============================================================================

f_storeSwing(Swing _swing) =>
    if array.size(g_swings) >= c_MAX_SWINGS
        array.shift(g_swings)
    array.push(g_swings, _swing)

f_newSwing(int _kind, float _price, int _barIndex, int _timestamp, float _strength, string _classification) =>
    Swing.new(
         g_nextSwingId,
         _kind,
         _price,
         _barIndex,
         _timestamp,
         true,
         false,
         _strength,
         na,
         na,
         _strength,
         _classification
    )

// Finds the most recent accepted swing of the requested type. The new swing
// is classified before storage, so this always returns the prior comparison.
f_lastPriceByKind(int _kind) =>
    float lastPrice = na
    int swingCount = array.size(g_swings)
    if swingCount > 0
        for i = swingCount - 1 to 0
            Swing savedSwing = array.get(g_swings, i)
            if savedSwing.kind == _kind
                lastPrice := savedSwing.price
                break
    lastPrice

f_lastIdByKind(int _kind) =>
    int lastId = na
    int swingCount = array.size(g_swings)
    if swingCount > 0
        for i = swingCount - 1 to 0
            Swing savedSwing = array.get(g_swings, i)
            if savedSwing.kind == _kind
                lastId := savedSwing.id
                break
    lastId

// Market-structure classification v1. Equal highs/lows remain classified as
// LH/HL in this release. Dedicated equal-high/low liquidity logic comes later.
f_classifySwing(int _kind, float _price) =>
    float previousSameKindPrice = f_lastPriceByKind(_kind)
    string classification = f_swingShortText(_kind)

    if not na(previousSameKindPrice)
        if _kind == c_SWING_HIGH
            classification := _price > previousSameKindPrice ? "HH" : "LH"
        else
            classification := _price > previousSameKindPrice ? "HL" : "LL"
    classification

// An accepted swing must alternate direction and move far enough from the
// previous accepted one. The first confirmed pivot always initializes memory.
f_isSwingAccepted(int _kind, float _price) =>
    bool accepted = true
    if array.size(g_swings) > 0
        Swing lastSwing = array.get(g_swings, array.size(g_swings) - 1)
        bool alternates = _kind != lastSwing.kind
        float distancePct = lastSwing.price != 0.0 ? math.abs(_price - lastSwing.price) / lastSwing.price * 100.0 : 0.0
        accepted := alternates and distancePct >= cfg_minSwingDistancePct
    accepted

// The first version of strength measures only price separation. ATR and volume
// components will be added in later releases without changing the 0-100 scale.
f_swingStrength(float _price) =>
    float strength = 50.0
    if array.size(g_swings) > 0
        Swing lastSwing = array.get(g_swings, array.size(g_swings) - 1)
        float distancePct = lastSwing.price != 0.0 ? math.abs(_price - lastSwing.price) / lastSwing.price * 100.0 : 0.0
        float minimum = math.max(cfg_minSwingDistancePct, 0.01)
        strength := math.min(100.0, 50.0 + distancePct / minimum * 10.0)
    strength

f_drawSwing(Swing _swing) =>
    if cfg_showSwings
        string labelText = _swing.state
        if cfg_showStrength
            labelText += " " + str.tostring(_swing.strength, "#.0")

        color swingColor = _swing.kind == c_SWING_HIGH ? color.red : color.lime
        labelStyle = _swing.kind == c_SWING_HIGH ? label.style_label_down : label.style_label_up
        label newLabel = label.new(
             _swing.barIndex,
             _swing.price,
             labelText,
             xloc = xloc.bar_index,
             yloc = yloc.price,
             color = swingColor,
             style = labelStyle,
             textcolor = color.white,
             size = size.tiny
        )

        if array.size(g_swingLabels) >= c_MAX_RENDERED_LABELS
            label.delete(array.shift(g_swingLabels))
        array.push(g_swingLabels, newLabel)

f_drawStructureBreak(bool _isBullish, float _level, string _eventText) =>
    if cfg_showStructureBreaks
        bool isChoch = str.contains(_eventText, "CHoCH")
        color breakColor = isChoch ? color.fuchsia : _isBullish ? color.aqua : color.orange
        labelStyle = _isBullish ? label.style_label_up : label.style_label_down
        label newLabel = label.new(
             bar_index,
             _level,
             _eventText,
             xloc = xloc.bar_index,
             yloc = yloc.price,
             color = breakColor,
             style = labelStyle,
             textcolor = color.black,
             size = size.small
        )

        if array.size(g_swingLabels) >= c_MAX_RENDERED_LABELS
            label.delete(array.shift(g_swingLabels))
        array.push(g_swingLabels, newLabel)

// A pivot is available only cfg_pivotLength bars after it occurred. Once this
// code receives it, the corresponding SH/SL label cannot repaint.
float sw_pivotHigh = ta.pivothigh(high, cfg_pivotLength, cfg_pivotLength)
float sw_pivotLow = ta.pivotlow(low, cfg_pivotLength, cfg_pivotLength)

bool sw_newHigh = not na(sw_pivotHigh)
bool sw_newLow = not na(sw_pivotLow)

// Offset by one requested bar and use lookahead_on so the panel receives only
// completed higher-timeframe states. This intentionally delays HTF changes by
// one HTF bar in exchange for stable, non-repainting context.
int mtf_directionOne = request.security(syminfo.tickerid, cfg_htfOne, f_htfDirection(cfg_pivotLength)[1], gaps = barmerge.gaps_off, lookahead = barmerge.lookahead_on)
int mtf_directionTwo = request.security(syminfo.tickerid, cfg_htfTwo, f_htfDirection(cfg_pivotLength)[1], gaps = barmerge.gaps_off, lookahead = barmerge.lookahead_on)
int mtf_directionThree = request.security(syminfo.tickerid, cfg_htfThree, f_htfDirection(cfg_pivotLength)[1], gaps = barmerge.gaps_off, lookahead = barmerge.lookahead_on)

//==============================================================================
// 07. DASHBOARD
//==============================================================================

f_setDashboardCell(int _column, int _row, string _text, color _background, color _textColor) =>
    table.cell(
         g_dashboard,
         _column,
         _row,
         _text,
         bgcolor = _background,
         text_color = _textColor,
         text_size = size.small
    )

f_renderDashboard() =>
    if barstate.islast
        table.clear(g_dashboard, 0, 0, c_DASHBOARD_COLUMNS - 1, c_DASHBOARD_ROWS - 1)

        if cfg_showDashboard
            color headerBackground = color.rgb(23, 35, 52)
            color labelBackground = color.new(color.rgb(23, 35, 52), 45)
            color valueBackground = color.new(color.black, 15)
            int localDirection = f_localDirection()
            float htfAlignment = f_htfAlignment(localDirection, mtf_directionOne, mtf_directionTwo, mtf_directionThree)

            f_setDashboardCell(0, 0, c_PROJECT_NAME, headerBackground, color.white)
            f_setDashboardCell(1, 0, c_VERSION, headerBackground, color.white)

            f_setDashboardCell(0, 1, "Engine", labelBackground, color.silver)
            f_setDashboardCell(1, 1, c_ENGINE_NAME, valueBackground, color.white)

            f_setDashboardCell(0, 2, "Status", labelBackground, color.silver)
            f_setDashboardCell(1, 2, g_engineStatus, f_statusColor(g_engineStatus), color.white)

            f_setDashboardCell(0, 3, "Symbol", labelBackground, color.silver)
            f_setDashboardCell(1, 3, syminfo.ticker, valueBackground, color.white)

            f_setDashboardCell(0, 4, "Timeframe", labelBackground, color.silver)
            f_setDashboardCell(1, 4, timeframe.period, valueBackground, color.white)

            f_setDashboardCell(0, 5, "Stored swings", labelBackground, color.silver)
            f_setDashboardCell(1, 5, str.tostring(array.size(g_swings)), valueBackground, color.white)

            f_setDashboardCell(0, 6, "Last structure", labelBackground, color.silver)
            f_setDashboardCell(1, 6, f_lastSwingText(), valueBackground, color.white)

            f_setDashboardCell(0, 7, "Pivot length", labelBackground, color.silver)
            f_setDashboardCell(1, 7, str.tostring(cfg_pivotLength), valueBackground, color.white)

            f_setDashboardCell(0, 8, "Trend state", labelBackground, color.silver)
            f_setDashboardCell(1, 8, g_trendState, valueBackground, color.white)

            float structureScore = f_structureScore()
            f_setDashboardCell(0, 9, "Structure score", labelBackground, color.silver)
            f_setDashboardCell(1, 9, str.tostring(structureScore, "#.0") + " / 100", valueBackground, color.white)

            f_setDashboardCell(0, 10, "Score quality", labelBackground, color.silver)
            f_setDashboardCell(1, 10, f_scoreLabel(structureScore), valueBackground, color.white)

            string logText = cfg_developerMode ? g_lastLog : "Enable Developer mode for log"
            f_setDashboardCell(0, 11, "Last event", labelBackground, color.silver)
            f_setDashboardCell(1, 11, cfg_developerMode ? logText : g_lastStructureEvent, valueBackground, color.white)

            if cfg_showHtfContext
                f_setDashboardCell(0, 12, "HTF " + cfg_htfOne, labelBackground, color.silver)
                f_setDashboardCell(1, 12, f_directionText(mtf_directionOne), valueBackground, color.white)

                f_setDashboardCell(0, 13, "HTF " + cfg_htfTwo, labelBackground, color.silver)
                f_setDashboardCell(1, 13, f_directionText(mtf_directionTwo), valueBackground, color.white)

                f_setDashboardCell(0, 14, "HTF " + cfg_htfThree, labelBackground, color.silver)
                f_setDashboardCell(1, 14, f_directionText(mtf_directionThree), valueBackground, color.white)

                f_setDashboardCell(0, 15, "HTF alignment", labelBackground, color.silver)
                f_setDashboardCell(1, 15, str.tostring(htfAlignment, "#.0") + " %", valueBackground, color.white)

                f_setDashboardCell(0, 16, "Context", labelBackground, color.silver)
                f_setDashboardCell(1, 16, f_contextLabel(htfAlignment), valueBackground, color.white)

            if cfg_showTradeHorizon
                f_setDashboardCell(0, 17, "Trade horizon", labelBackground, color.silver)
                f_setDashboardCell(1, 17, f_tradeHorizon(), valueBackground, color.white)

                f_setDashboardCell(0, 18, "Expected duration", labelBackground, color.silver)
                f_setDashboardCell(1, 18, f_expectedDuration(), valueBackground, color.white)

                f_setDashboardCell(0, 19, "HTF support", labelBackground, color.silver)
                f_setDashboardCell(1, 19, f_horizonSupport(htfAlignment), valueBackground, color.white)

                f_setDashboardCell(0, 20, "Management", labelBackground, color.silver)
                f_setDashboardCell(1, 20, f_managementGuidance(htfAlignment), valueBackground, color.white)

//==============================================================================
// 08. MAIN LOOP
//==============================================================================

if sw_newHigh
    float strength = f_swingStrength(sw_pivotHigh)
    if f_isSwingAccepted(c_SWING_HIGH, sw_pivotHigh)
        string classification = f_classifySwing(c_SWING_HIGH, sw_pivotHigh)
        Swing newSwing = f_newSwing(c_SWING_HIGH, sw_pivotHigh, bar_index - cfg_pivotLength, time[cfg_pivotLength], strength, classification)
        f_storeSwing(newSwing)
        f_drawSwing(newSwing)
        g_nextSwingId += 1
        g_lastLog := "Accepted " + classification + ": " + str.tostring(sw_pivotHigh, format.mintick)
    else
        g_lastLog := "Rejected SH: alternation or distance filter"

else if sw_newLow
    float strength = f_swingStrength(sw_pivotLow)
    if f_isSwingAccepted(c_SWING_LOW, sw_pivotLow)
        string classification = f_classifySwing(c_SWING_LOW, sw_pivotLow)
        Swing newSwing = f_newSwing(c_SWING_LOW, sw_pivotLow, bar_index - cfg_pivotLength, time[cfg_pivotLength], strength, classification)
        f_storeSwing(newSwing)
        f_drawSwing(newSwing)
        g_nextSwingId += 1
        g_lastLog := "Accepted " + classification + ": " + str.tostring(sw_pivotLow, format.mintick)
    else
        g_lastLog := "Rejected SL: alternation or distance filter"

// Breaks use only the current, fully closed candle and a level belonging to a
// previously confirmed pivot. This prevents intrabar BOS labels from repainting.
float str_lastHighPrice = f_lastPriceByKind(c_SWING_HIGH)
float str_lastLowPrice = f_lastPriceByKind(c_SWING_LOW)
int str_lastHighId = f_lastIdByKind(c_SWING_HIGH)
int str_lastLowId = f_lastIdByKind(c_SWING_LOW)

bool str_crossedAboveLastHigh = ta.crossover(close, str_lastHighPrice)
bool str_crossedBelowLastLow = ta.crossunder(close, str_lastLowPrice)
bool str_bullBreak = barstate.isconfirmed and not na(str_lastHighPrice) and str_crossedAboveLastHigh and (na(g_lastBullBreakSwingId) or str_lastHighId != g_lastBullBreakSwingId)
bool str_bearBreak = barstate.isconfirmed and not na(str_lastLowPrice) and str_crossedBelowLastLow and (na(g_lastBearBreakSwingId) or str_lastLowId != g_lastBearBreakSwingId)

if str_bullBreak
    bool isChoch = g_trendState == "BEARISH"
    string eventText = isChoch ? "CHoCH UP" : "BOS UP"
    g_trendState := "BULLISH"
    g_lastBullBreakSwingId := str_lastHighId
    g_lastStructureEvent := eventText + " @ " + str.tostring(str_lastHighPrice, format.mintick)
    g_lastLog := g_lastStructureEvent
    f_drawStructureBreak(true, str_lastHighPrice, eventText)

else if str_bearBreak
    bool isChoch = g_trendState == "BULLISH"
    string eventText = isChoch ? "CHoCH DOWN" : "BOS DOWN"
    g_trendState := "BEARISH"
    g_lastBearBreakSwingId := str_lastLowId
    g_lastStructureEvent := eventText + " @ " + str.tostring(str_lastLowPrice, format.mintick)
    g_lastLog := g_lastStructureEvent
    f_drawStructureBreak(false, str_lastLowPrice, eventText)

f_renderDashboard()

//==============================================================================
// END OF v0.8.0 TRADE HORIZON
// Next planned increment: v0.9.0 Relative volume and participation context.
//==============================================================================
