//@version=6
//==============================================================================
// PARANÁ PROJECT
// Institutional Market Intelligence
//------------------------------------------------------------------------------
// Version : v0.1.0 Foundation
// Script  : Indicator scaffold
// Engine  : Paraná Swing Engine (PSE) - foundation only
//
// Purpose
// This release establishes the stable framework used by future Paraná engines.
// It intentionally does NOT generate trade signals, scores, HH/HL labels,
// BOS, CHoCH, or entries.  The Swing Engine is represented by its data model
// and its integration points, ready for the next sprint.
//
// Development rule: a new engine may consume Core data, but must not change
// the meaning of a previously confirmed data structure without a new version.
//==============================================================================

indicator(
     title = "Paraná Project v0.1.0 — Foundation",
     shorttitle = "PARANÁ v0.1",
     overlay = true,
     max_labels_count = 200,
     max_lines_count = 200,
     max_boxes_count = 50
)

//==============================================================================
// 01. PROJECT CONSTANTS
//==============================================================================

const string c_PROJECT_NAME = "PARANÁ PROJECT"
const string c_VERSION      = "v0.1.0 Foundation"
const string c_ENGINE_NAME  = "Core Framework"
const string c_STATUS_READY = "READY"

const int c_SWING_HIGH = 1
const int c_SWING_LOW  = -1
const int c_MAX_SWINGS = 200

const int c_DASHBOARD_COLUMNS = 2
const int c_DASHBOARD_ROWS    = 9

//==============================================================================
// 02. USER CONFIGURATION
//==============================================================================

string cfg_groupGeneral = "01 · General"
string cfg_groupSwing   = "02 · Swing Engine (reserved)"
string cfg_groupDev     = "03 · Development"

bool cfg_showDashboard = input.bool(
     defval = true,
     title = "Show dashboard",
     group = cfg_groupGeneral,
     tooltip = "Shows the Paraná Project status panel in the top-right corner."
)

bool cfg_showEngineStatus = input.bool(
     defval = true,
     title = "Show engine status",
     group = cfg_groupGeneral,
     tooltip = "Displays the current foundation status in the dashboard."
)

int cfg_pivotLength = input.int(
     defval = 5,
     title = "Pivot sensitivity",
     minval = 2,
     maxval = 50,
     group = cfg_groupSwing,
     tooltip = "Reserved for Swing Engine v0.2.0. No pivot is evaluated in this release."
)

float cfg_minSwingDistancePct = input.float(
     defval = 0.40,
     title = "Minimum swing distance (%)",
     minval = 0.01,
     step = 0.05,
     group = cfg_groupSwing,
     tooltip = "Reserved for the Paraná swing-quality filter."
)

bool cfg_developerMode = input.bool(
     defval = false,
     title = "Developer mode",
     group = cfg_groupDev,
     tooltip = "Shows internal framework information; it does not change analysis."
)

//==============================================================================
// 03. CORE DATA TYPES
//==============================================================================

// A Swing is the durable data contract between the Swing Engine and every
// future engine. Fields may be populated progressively in later releases.
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
var int g_nextSwingId = 1
var string g_lastLog = "Core initialized"
var string g_engineStatus = c_STATUS_READY

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

f_engineStatusText() =>
    // Integration point for later engines. Keep this function deterministic.
    g_engineStatus

f_foundationLogText() =>
    cfg_developerMode ? g_lastLog : "Developer mode disabled"

f_cellColor(string _status) =>
    _status == c_STATUS_READY ? color.new(color.lime, 78) : color.new(color.orange, 78)

//==============================================================================
// 06. LOGGER SCAFFOLD
//==============================================================================

// Pine Script has no external console. The project logger is deliberately
// lightweight and feeds the dashboard while development is in progress.
if barstate.isfirst
    g_lastLog := "Foundation loaded on " + syminfo.ticker + " · " + timeframe.period

//==============================================================================
// 07. SWING ENGINE SCAFFOLD
//==============================================================================

// This function centralizes storage so future Swing Engine versions never
// write directly to the global array from multiple locations.
f_storeSwing(Swing _swing) =>
    if array.size(g_swings) >= c_MAX_SWINGS
        array.shift(g_swings)
    array.push(g_swings, _swing)

// Factory kept here so every Swing begins with the same explicit state.
f_newSwing(int _kind, float _price, int _barIndex, int _timestamp) =>
    Swing.new(
         g_nextSwingId,
         _kind,
         _price,
         _barIndex,
         _timestamp,
         false,
         false,
         na,
         na,
         na,
         na,
         "candidate"
    )

// Reserved integration point.
// v0.1.0 does not yet call ta.pivothigh()/ta.pivotlow(). This ensures the
// first release is a non-repainting, compile-safe framework before market
// structure logic is introduced and visually validated.
bool sw_hasCandidate = false
bool sw_candidateIsHigh = false
float sw_candidatePrice = na
int sw_candidateBarIndex = na
int sw_candidateTime = na

// Placeholders retained as explicit variables for the next Swing Engine sprint.
// They make configuration and Core-to-engine data flow visible without claiming
// that a market-structure event has occurred.
int sw_configuredPivotLength = cfg_pivotLength
float sw_configuredMinDistance = cfg_minSwingDistancePct

//==============================================================================
// 08. DASHBOARD
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
            f_setDashboardCell(1, 2, f_engineStatusText(), f_cellColor(f_engineStatusText()), color.white)

            f_setDashboardCell(0, 3, "Symbol", labelBackground, color.silver)
            f_setDashboardCell(1, 3, syminfo.ticker, valueBackground, color.white)

            f_setDashboardCell(0, 4, "Timeframe", labelBackground, color.silver)
            f_setDashboardCell(1, 4, timeframe.period, valueBackground, color.white)

            f_setDashboardCell(0, 5, "Stored swings", labelBackground, color.silver)
            f_setDashboardCell(1, 5, str.tostring(array.size(g_swings)), valueBackground, color.white)

            f_setDashboardCell(0, 6, "Swing engine", labelBackground, color.silver)
            f_setDashboardCell(1, 6, "Reserved for v0.2.0", valueBackground, color.white)

            f_setDashboardCell(0, 7, "Developer mode", labelBackground, color.silver)
            f_setDashboardCell(1, 7, f_boolText(cfg_developerMode), valueBackground, color.white)

            string logText = cfg_showEngineStatus ? f_foundationLogText() : "Status display disabled"
            f_setDashboardCell(0, 8, "Log", labelBackground, color.silver)
            f_setDashboardCell(1, 8, logText, valueBackground, color.white)

//==============================================================================
// 09. MAIN LOOP
//==============================================================================

// Foundation release: Core state is live, while the actual non-repainting
// candidate detector will be added only after this scaffold is compiled and
// checked in TradingView across multiple assets/timeframes.
if sw_hasCandidate
    Swing sw_candidate = f_newSwing(
         sw_candidateIsHigh ? c_SWING_HIGH : c_SWING_LOW,
         sw_candidatePrice,
         sw_candidateBarIndex,
         sw_candidateTime
    )
    f_storeSwing(sw_candidate)
    g_nextSwingId += 1
    g_lastLog := "Swing candidate stored: " + f_swingKindText(sw_candidate.kind)

f_renderDashboard()

//==============================================================================
// END OF v0.1.0 FOUNDATION
// Next planned increment: v0.2.0 Paraná Swing Engine candidate detection,
// alternating swing validation, and visual labels for confirmed swings.
//==============================================================================
