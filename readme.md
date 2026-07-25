//@version=6
//==============================================================================
// PARANA PROJECT STRATEGY
//------------------------------------------------------------------------------
// Version : v0.1.0 Structure Validation
// Purpose : Backtest a deliberately narrow, explainable hypothesis.
//
// Entry thesis
// 1. A confirmed close breaks the latest confirmed swing high/low.
// 2. Price is on the corresponding side of the trend EMA.
// 3. Relative volume meets the configured minimum.
//
// This is a research strategy, not investment advice or an automated-trading
// system. Validate it over out-of-sample data before drawing conclusions.
//==============================================================================

strategy(
     title = "Parana Project Strategy v0.1.0 - Structure Validation",
     shorttitle = "PARANA STRAT v0.1",
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
string cfg_groupRisk = "02 - Risk Model"
string cfg_groupTest = "03 - Test Controls"

int cfg_pivotLength = input.int(5, "Pivot sensitivity", minval = 2, maxval = 50, group = cfg_groupEntry)
int cfg_emaLength = input.int(50, "Trend EMA length", minval = 5, maxval = 500, group = cfg_groupEntry)
int cfg_volumeLength = input.int(20, "Relative volume lookback", minval = 5, maxval = 200, group = cfg_groupEntry)
float cfg_minRelativeVolume = input.float(1.0, "Minimum relative volume", minval = 0.1, step = 0.05, group = cfg_groupEntry)
string cfg_direction = input.string("Both", "Trade direction", options = ["Long", "Short", "Both"], group = cfg_groupEntry)

int cfg_atrLength = input.int(14, "ATR length", minval = 2, maxval = 100, group = cfg_groupRisk)
float cfg_stopAtr = input.float(1.5, "Stop loss (ATR)", minval = 0.1, step = 0.1, group = cfg_groupRisk)
float cfg_rewardRisk = input.float(2.0, "Reward / risk", minval = 0.25, step = 0.25, group = cfg_groupRisk)
int cfg_maxHoldingBars = input.int(42, "Maximum holding bars", minval = 1, maxval = 1000, group = cfg_groupRisk, tooltip = "For a 4H chart, 42 bars are approximately seven days.")

bool cfg_useVolumeFilter = input.bool(true, "Use volume filter", group = cfg_groupTest)
bool cfg_showLevels = input.bool(true, "Show active stop and target", group = cfg_groupTest)

//==============================================================================
// 02. MARKET DATA AND CONFIRMED SWINGS
//==============================================================================

float trendEma = ta.ema(close, cfg_emaLength)
float atr = ta.atr(cfg_atrLength)
float volumeAverage = ta.sma(volume, cfg_volumeLength)
float relativeVolume = not na(volumeAverage) and volumeAverage > 0.0 ? volume / volumeAverage : na

float pivotHigh = ta.pivothigh(high, cfg_pivotLength, cfg_pivotLength)
float pivotLow = ta.pivotlow(low, cfg_pivotLength, cfg_pivotLength)

var float lastSwingHigh = na
var float lastSwingLow = na

if not na(pivotHigh)
    lastSwingHigh := pivotHigh
if not na(pivotLow)
    lastSwingLow := pivotLow

// Cross functions execute on every bar. The barstate filter prevents entries
// from using a still-forming realtime candle.
bool crossedAboveSwing = ta.crossover(close, lastSwingHigh)
bool crossedBelowSwing = ta.crossunder(close, lastSwingLow)
bool volumePass = not cfg_useVolumeFilter or (not na(relativeVolume) and relativeVolume >= cfg_minRelativeVolume)

bool longAllowed = cfg_direction == "Long" or cfg_direction == "Both"
bool shortAllowed = cfg_direction == "Short" or cfg_direction == "Both"

bool longSetup = barstate.isconfirmed and longAllowed and crossedAboveSwing and close > trendEma and volumePass
bool shortSetup = barstate.isconfirmed and shortAllowed and crossedBelowSwing and close < trendEma and volumePass

//==============================================================================
// 03. ORDERS AND RISK MANAGEMENT
//==============================================================================

var float g_entryAtr = na
var int g_entryBar = na

if strategy.position_size == 0
    if longSetup
        strategy.entry("Long", strategy.long)
        g_entryAtr := atr
        g_entryBar := bar_index
    else if shortSetup
        strategy.entry("Short", strategy.short)
        g_entryAtr := atr
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

bool holdingLimitReached = not na(g_entryBar) and bar_index - g_entryBar >= cfg_maxHoldingBars
if holdingLimitReached and strategy.position_size > 0
    strategy.close("Long", comment = "Time exit")
if holdingLimitReached and strategy.position_size < 0
    strategy.close("Short", comment = "Time exit")

if strategy.position_size == 0 and not longSetup and not shortSetup
    g_entryAtr := na
    g_entryBar := na

//==============================================================================
// 04. VISUAL VALIDATION
//==============================================================================

plot(trendEma, "Trend EMA", color = color.orange, linewidth = 2)
plot(cfg_showLevels ? activeStop : na, "Active stop", color = color.red, style = plot.style_linebr)
plot(cfg_showLevels ? activeTarget : na, "Active target", color = color.lime, style = plot.style_linebr)

plotshape(longSetup, title = "Validated long setup", style = shape.triangleup, location = location.belowbar, color = color.lime, size = size.tiny, text = "L")
plotshape(shortSetup, title = "Validated short setup", style = shape.triangledown, location = location.abovebar, color = color.red, size = size.tiny, text = "S")

//==============================================================================
// END OF v0.1.0
// Next validation: compare outcomes across BTC, ETH, SOL, and multiple
// timeframes before adding the richer Parana Decision Profile filters.
//==============================================================================
