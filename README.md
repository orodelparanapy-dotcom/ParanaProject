//@version=6
//==============================================================================
// PARANA PROJECT
// Institutional Market Intelligence
//------------------------------------------------------------------------------
// Version : v0.2.0 Swing Engine
// Type    : Indicator
//
// This release detects confirmed price pivots, applies an initial noise filter,
// enforces High/Low alternation, stores accepted swings, and optionally draws
// non-repainting SH / SL labels. It does not issue trade signals.
//==============================================================================

indicator(
     title = "Parana Project v0.2.0 - Swing Engine",
     shorttitle = "PARANA v0.2",
     overlay = true,
     max_labels_count = 200,
     max_lines_count = 200,
     max_boxes_count = 50
)

//==============================================================================
// 01. PROJECT CONSTANTS
//==============================================================================

const string c_PROJECT_NAME = "PARANA PROJECT"
const string c_VERSION = "v0.2.0 Swing Engine"
const string c_ENGINE_NAME = "Parana Swing Engine"
const string c_STATUS_RUNNING = "RUNNING"

const int c_SWING_HIGH = 1
const int c_SWING_LOW = -1
const int c_MAX_SWINGS = 200
const int c_MAX_RENDERED_LABELS = 190

const int c_DASHBOARD_COLUMNS = 2
const int c_DASHBOARD_ROWS = 9

//==============================================================================
// 02. USER CONFIGURATION
//==============================================================================

string cfg_groupGeneral = "01 - General"
string cfg_groupSwing = "02 - Swing Engine"
string cfg_groupDev = "03 - Development"

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
        f_swingShortText(lastSwing.kind) + " @ " + str.tostring(lastSwing.price, format.mintick)

//==============================================================================
// 06. SWING ENGINE
//==============================================================================

f_storeSwing(Swing _swing) =>
    if array.size(g_swings) >= c_MAX_SWINGS
        array.shift(g_swings)
    array.push(g_swings, _swing)

f_newSwing(int _kind, float _price, int _barIndex, int _timestamp, float _strength) =>
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
         "confirmed"
    )

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
        string text = f_swingShortText(_swing.kind)
        if cfg_showStrength
            text += " " + str.tostring(_swing.strength, "#.0")

        color swingColor = _swing.kind == c_SWING_HIGH ? color.red : color.lime
        labelStyle = _swing.kind == c_SWING_HIGH ? label.style_label_down : label.style_label_up
        label newLabel = label.new(
             _swing.barIndex,
             _swing.price,
             text,
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

            f_setDashboardCell(0, 6, "Last swing", labelBackground, color.silver)
            f_setDashboardCell(1, 6, f_lastSwingText(), valueBackground, color.white)

            f_setDashboardCell(0, 7, "Pivot length", labelBackground, color.silver)
            f_setDashboardCell(1, 7, str.tostring(cfg_pivotLength), valueBackground, color.white)

            string logText = cfg_developerMode ? g_lastLog : "Enable Developer mode for log"
            f_setDashboardCell(0, 8, "Log", labelBackground, color.silver)
            f_setDashboardCell(1, 8, logText, valueBackground, color.white)

//==============================================================================
// 08. MAIN LOOP
//==============================================================================

if sw_newHigh
    float strength = f_swingStrength(sw_pivotHigh)
    if f_isSwingAccepted(c_SWING_HIGH, sw_pivotHigh)
        Swing newSwing = f_newSwing(c_SWING_HIGH, sw_pivotHigh, bar_index - cfg_pivotLength, time[cfg_pivotLength], strength)
        f_storeSwing(newSwing)
        f_drawSwing(newSwing)
        g_nextSwingId += 1
        g_lastLog := "Accepted SH: " + str.tostring(sw_pivotHigh, format.mintick)
    else
        g_lastLog := "Rejected SH: alternation or distance filter"

else if sw_newLow
    float strength = f_swingStrength(sw_pivotLow)
    if f_isSwingAccepted(c_SWING_LOW, sw_pivotLow)
        Swing newSwing = f_newSwing(c_SWING_LOW, sw_pivotLow, bar_index - cfg_pivotLength, time[cfg_pivotLength], strength)
        f_storeSwing(newSwing)
        f_drawSwing(newSwing)
        g_nextSwingId += 1
        g_lastLog := "Accepted SL: " + str.tostring(sw_pivotLow, format.mintick)
    else
        g_lastLog := "Rejected SL: alternation or distance filter"

f_renderDashboard()

//==============================================================================
// END OF v0.2.0 SWING ENGINE
// Next planned increment: v0.3.0 Swing classification (HH, HL, LH, LL).
//==============================================================================
