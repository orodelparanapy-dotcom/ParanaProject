//@version=6
//==============================================================================
// PARANA PROJECT STRATEGY
//------------------------------------------------------------------------------
// Version : v0.2.0 Quality Filters
// Purpose : Compare the baseline structure-break strategy with a more selective
//           version. This is research code, not investment advice.
//
// Entry thesis
// 1. Confirmed close breaks the latest confirmed swing high/low.
// 2. Trend regime, directional strength, breakout candle and participation
//    filters agree with that direction.
//
// Testing discipline
// - Change one filter at a time and record its effect.
// - Keep a part of history untouched for out-of-sample validation.
// - Do not select settings solely because they improve one BTC chart.
//==============================================================================

strategy(
     title = "Parana Project Strategy v0.2.0 - Quality Filters",
     shorttitle = "PARANA STRAT v0.2",
     overlay = true,
     pyramiding = 0,
     initial_capital = 10000,
     default_qty_type = strategy.percent_of_equity,
     default_qty_value = 10,
     commission_type = strategy.commission.percent,
     commission_value = 0.10,
     process_orders_on_close = false,
     calc_on_order_fills = false
)

//==============================================================================
// 01. INPUTS
//==============================================================================

string cfg_groupEntry = "01 - Structure Entry"
string cfg_groupQuality = "02 - Quality Filters"
string cfg_groupRisk = "03 - Risk Model"
string cfg_groupTest = "04 - Test Controls"

int cfg_pivotLength = input.int(5, "Pivot sensitivity", minval = 2, maxval = 50, group = cfg_groupEntry)
int cfg_fastEmaLength = input.int(50, "Fast trend EMA", minval = 5, maxval = 500, group = cfg_groupEntry)
int cfg_slowEmaLength = input.int(200, "Slow regime EMA", minval = 20, maxval = 500, group = cfg_groupEntry)
string cfg_direction = input.string("Both", "Trade direction", options = ["Long", "Short", "Both"], group = cfg_groupEntry)

bool cfg_useTrendFilter = input.bool(true, "Require trend regime", group = cfg_groupQuality, tooltip = "Long: price and EMA 50 above EMA 200. Short: inverse.")
int cfg_slopeLookback = input.int(10, "EMA slope lookback", minval = 1, maxval = 100, group = cfg_groupQuality)
bool cfg_useAdxFilter = input.bool(true, "Require directional strength (ADX)", group = cfg_groupQuality)
int cfg_diLength = input.int(14, "DI length", minval = 2, maxval = 100, group = cfg_groupQuality)
int cfg_adxSmoothing = input.int(14, "ADX smoothing", minval = 2, maxval = 100, group = cfg_groupQuality)
float cfg_minAdx = input.float(20.0, "Minimum ADX", minval = 1.0, step = 0.5, group = cfg_groupQuality)
bool cfg_useVolumeFilter = input.bool(true, "Require relative volume", group = cfg_groupQuality)
int cfg_volumeLength = input.int(20, "Relative volume lookback", minval = 5, maxval = 200, group = cfg_groupQuality)
float cfg_minRelativeVolume = input.float(1.20, "Minimum relative volume", minval = 0.1, step = 0.05, group = cfg_groupQuality)
bool cfg_useBreakoutCandleFilter = input.bool(true, "Require quality breakout candle", group = cfg_groupQuality)
float cfg_minBodyAtr = input.float(0.40, "Minimum candle body (ATR)", minval = 0.05, step = 0.05, group = cfg_groupQuality)
float cfg_closeStrength = input.float(0.65, "Close strength (0-1)", minval = 0.50, maxval = 0.95, step = 0.05, group = cfg_groupQuality, tooltip = "Longs must close in the upper portion of the range; shorts in the lower portion.")
int cfg_cooldownBars = input.int(6, "Cooldown after an exit (bars)", minval = 0, maxval = 500, group = cfg_groupQuality)

int cfg_atrLength = input.int(14, "ATR length", minval = 2, maxval = 100, group = cfg_groupRisk)
float cfg_stopAtr = input.float(1.5, "Stop loss (ATR)", minval = 0.1, step = 0.1, group = cfg_groupRisk)
float cfg_rewardRisk = input.float(2.0, "Reward / risk", minval = 0.25, step = 0.25, group = cfg_groupRisk)
int cfg_maxHoldingBars = input.int(42, "Maximum holding bars", minval = 1, maxval = 1000, group = cfg_groupRisk, tooltip = "On a 4H chart, 42 bars are approximately seven days.")

bool cfg_showLevels = input.bool(true, "Show active stop and target", group = cfg_groupTest)
bool cfg_showSetups = input.bool(true, "Show accepted setups", group = cfg_groupTest)

//==============================================================================
// 02. MARKET DATA AND CONFIRMED SWINGS
//==============================================================================

float fastEma = ta.ema(close, cfg_fastEmaLength)
float slowEma = ta.ema(close, cfg_slowEmaLength)
float atr = ta.atr(cfg_atrLength)
float volumeAverage = ta.sma(volume, cfg_volumeLength)
float relativeVolume = not na(volumeAverage) and volumeAverage > 0.0 ? volume / volumeAverage : na
[plusDi, minusDi, adx] = ta.dmi(cfg_diLength, cfg_adxSmoothing)

float pivotHigh = ta.pivothigh(high, cfg_pivotLength, cfg_pivotLength)
float pivotLow = ta.pivotlow(low, cfg_pivotLength, cfg_pivotLength)

var float lastSwingHigh = na
var float lastSwingLow = na

if not na(pivotHigh)
    lastSwingHigh := pivotHigh
if not na(pivotLow)
    lastSwingLow := pivotLow

//==============================================================================
// 03. QUALITY FILTERS
//==============================================================================

bool crossedAboveSwing = ta.crossover(close, lastSwingHigh)
bool crossedBelowSwing = ta.crossunder(close, lastSwingLow)
bool longAllowed = cfg_direction == "Long" or cfg_direction == "Both"
bool shortAllowed = cfg_direction == "Short" or cfg_direction == "Both"

bool longTrendPass = not cfg_useTrendFilter or (close > fastEma and fastEma > slowEma and slowEma > slowEma[cfg_slopeLookback])
bool shortTrendPass = not cfg_useTrendFilter or (close < fastEma and fastEma < slowEma and slowEma < slowEma[cfg_slopeLookback])

bool longAdxPass = not cfg_useAdxFilter or (adx >= cfg_minAdx and plusDi > minusDi)
bool shortAdxPass = not cfg_useAdxFilter or (adx >= cfg_minAdx and minusDi > plusDi)

bool volumePass = not cfg_useVolumeFilter or (not na(relativeVolume) and relativeVolume >= cfg_minRelativeVolume)

float candleRange = high - low
float candleBody = math.abs(close - open)
float closeLocation = candleRange > 0.0 ? (close - low) / candleRange : 0.5
bool longCandlePass = not cfg_useBreakoutCandleFilter or (candleBody >= atr * cfg_minBodyAtr and close > open and closeLocation >= cfg_closeStrength)
bool shortCandlePass = not cfg_useBreakoutCandleFilter or (candleBody >= atr * cfg_minBodyAtr and close < open and closeLocation <= 1.0 - cfg_closeStrength)

var int g_lastExitBar = na
bool exitDetected = strategy.position_size == 0 and strategy.position_size[1] != 0
if exitDetected
    g_lastExitBar := bar_index

bool cooldownPass = na(g_lastExitBar) or bar_index - g_lastExitBar >= cfg_cooldownBars

bool longSetup = barstate.isconfirmed and longAllowed and cooldownPass and crossedAboveSwing and longTrendPass and longAdxPass and volumePass and longCandlePass
bool shortSetup = barstate.isconfirmed and shortAllowed and cooldownPass and crossedBelowSwing and shortTrendPass and shortAdxPass and volumePass and shortCandlePass

//==============================================================================
// 04. ORDERS AND RISK MANAGEMENT
//==============================================================================

var float g_pendingAtr = na
var float g_entryAtr = na
var int g_entryBar = na

if strategy.position_size == 0
    if longSetup
        g_pendingAtr := atr
        strategy.entry("Long", strategy.long)
    else if shortSetup
        g_pendingAtr := atr
        strategy.entry("Short", strategy.short)

bool positionOpened = strategy.position_size != 0 and strategy.position_size[1] == 0
if positionOpened
    g_entryAtr := g_pendingAtr
    g_entryBar := bar_index

float activeStop = na
float activeTarget = na

if strategy.position_size > 0 and not na(g_entryAtr)
    activeStop := strategy.position_avg_price - g_entryAtr * cfg_stopAtr
    activeTarget := strategy.position_avg_price + g_entryAtr * cfg_stopAtr * cfg_rewardRisk
    strategy.exit("Long bracket", "Long", stop = activeStop, limit = activeTarget)

if strategy.position_size < 0 and not na(g_entryAtr)
    activeStop := strategy.position_avg_price + g_entryAtr * cfg_stopAtr
    activeTarget := strategy.position_avg_price - g_entryAtr * cfg_stopAtr * cfg_rewardRisk
    strategy.exit("Short bracket", "Short", stop = activeStop, limit = activeTarget)

bool holdingLimitReached = strategy.position_size != 0 and not na(g_entryBar) and bar_index - g_entryBar >= cfg_maxHoldingBars
if holdingLimitReached and strategy.position_size > 0
    strategy.close("Long", comment = "Time exit")
if holdingLimitReached and strategy.position_size < 0
    strategy.close("Short", comment = "Time exit")

if strategy.position_size == 0 and not longSetup and not shortSetup
    g_entryAtr := na
    g_entryBar := na

//==============================================================================
// 05. VISUAL VALIDATION
//==============================================================================

plot(fastEma, "Fast trend EMA", color = color.orange, linewidth = 2)
plot(slowEma, "Slow regime EMA", color = color.blue, linewidth = 2)
plot(cfg_showLevels ? activeStop : na, "Active stop", color = color.red, style = plot.style_linebr)
plot(cfg_showLevels ? activeTarget : na, "Active target", color = color.lime, style = plot.style_linebr)

plotshape(cfg_showSetups and longSetup, title = "Accepted long setup", style = shape.triangleup, location = location.belowbar, color = color.lime, size = size.tiny, text = "L+")
plotshape(cfg_showSetups and shortSetup, title = "Accepted short setup", style = shape.triangledown, location = location.abovebar, color = color.red, size = size.tiny, text = "S+")

//==============================================================================
// END OF v0.2.0
// Next validation: compare v0.1 and v0.2 over the same BTC 4H period, then
// repeat the winning configuration on an untouched period and other symbols.
//==============================================================================
