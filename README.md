//@version=6
//==============================================================================
// PARANA PROJECT
// Institutional Market Intelligence
//------------------------------------------------------------------------------
// Version : v0.5.0 CHoCH
// Type    : Indicator
//
// This release distinguishes a continuation break (BOS) from a break against
// the active trend (CHoCH). Events are still diagnostic; no trade signals are
// generated.
//==============================================================================

indicator(
     title = "Parana Project v0.5.0 - CHoCH",
     shorttitle = "PARANA v0.5",
     overlay = true,
     max_labels_count = 200,
     max_lines_count = 200,
     max_boxes_count = 50
)

//==============================================================================
// 01. PROJECT CONSTANTS
//==============================================================================

const string c_PROJECT_NAME = "PARANA PROJECT"
const string c_VERSION = "v0.5.0 CHoCH"
const string c_ENGINE_NAME = "Parana Structure Engine"
const string c_STATUS_RUNNING = "RUNNING"

const int c_SWING_HIGH = 1
const int c_SWING_LOW = -1
const int c_MAX_SWINGS = 200
const int c_MAX_RENDERED_LABELS = 190

const int c_DASHBOARD_COLUMNS = 2
const int c_DASHBOARD_ROWS = 10

//==============================================================================
// 02. USER CONFIGURATION
//==============================================================================

string cfg_groupGeneral = "01 - General"
string cfg_groupSwing = "02 - Swing Engine"
string cfg_groupStructure = "03 - Structure Breaks"
string cfg_groupDev = "04 - Development"

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

            string logText = cfg_developerMode ? g_lastLog : "Enable Developer mode for log"
            f_setDashboardCell(0, 9, "Last event", labelBackground, color.silver)
            f_setDashboardCell(1, 9, cfg_developerMode ? logText : g_lastStructureEvent, valueBackground, color.white)

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
// END OF v0.5.0 CHoCH
// Next planned increment: v0.6.0 Structure score and multi-timeframe context.
//==============================================================================
