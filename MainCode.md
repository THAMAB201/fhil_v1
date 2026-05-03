//@version=6
indicator("FHIL Bias System - Standalone + MM Structure", overlay=true, max_boxes_count=500, max_lines_count=500, max_labels_count=500, max_bars_back=500)

// ============================================================================
// Standalone Master Controls
// ============================================================================

groupStandalone = "- - - - - - - - - Hidden Master Controls - - - - - - - - -"
groupTradingWindows = "- - - - - - - - - Trading Windows - - - - - - - - -"
groupSessionLevels = "- - - - - - - - - Session High / Low Levels - - - - - - - - -"

timezone = input.string("America/New_York", "Timezone", group=groupStandalone, display=display.none)
startTimeHour = input.int(18, "Daily Reset Hour", minval=0, maxval=23, group=groupStandalone, display=display.none)
globalLookbackPeriod = input.int(10, "Object Lookback Count", minval=1, maxval=100, group=groupStandalone, display=display.none)

showAMNY = input.bool(true, "Enable AM Bias System", group=groupStandalone, display=display.none)
showPMNYMarketMaker = input.bool(true, "Enable PM Bias System", group=groupStandalone, display=display.none)
showFpiBiasTable = input.bool(true, "Show Bias Table", group=groupStandalone, display=display.none)
usePMEarlyStart = input.bool(false, "PM Session Early Start?", group=groupStandalone, display=display.none)

tradeStartHour = input.int(9, "Trading Start Hour", minval=0, maxval=23, group=groupTradingWindows)
tradeStartMinute = input.int(30, "Trading Start Minute", minval=0, maxval=59, group=groupTradingWindows)
amTradeEndHour = input.int(11, "AM Trade End Hour", minval=0, maxval=23, group=groupTradingWindows, display=display.none)
amTradeEndMinute = input.int(59, "AM Trade End Minute", minval=0, maxval=59, group=groupTradingWindows, display=display.none)
pmSessionStartHour = input.int(11, "PM Session Start Hour", minval=0, maxval=23, group=groupTradingWindows, display=display.none)
pmSessionStartMinute = input.int(30, "PM Session Start Minute", minval=0, maxval=59, group=groupTradingWindows, display=display.none)
pmTradeEndHour = input.int(16, "Trading End Hour", minval=0, maxval=23, group=groupTradingWindows)
pmTradeEndMinute = input.int(30, "Trading End Minute", minval=0, maxval=59, group=groupTradingWindows)

showLondonSessionLevels = input.bool(true, "Draw London Session 2AM-5AM High/Low", group=groupSessionLevels, display=display.none)
showAsianSessionLevels = input.bool(true, "Draw Asian Session 7PM-12AM High/Low", group=groupSessionLevels, display=display.none)
showNYPreSessionLevels = input.bool(true, "Draw NY Session 7AM-9AM High/Low", group=groupSessionLevels, display=display.none)
sessionLevelLineColor = input.color(color.new(color.white, 35), "Session Level Line Color", group=groupSessionLevels, display=display.none)
sessionLevelTextColor = input.color(color.new(color.white, 0), "Session Level Text Color", group=groupSessionLevels, display=display.none)

groupAMTime = "- - - - - - - - - AM Time Windows - - - - - - - - -"

amFpiStartHour = tradeStartHour
amFpiStartMinute = tradeStartMinute
amFpiEndHour = amTradeEndHour
amFpiEndMinute = amTradeEndMinute

amMacroStartHour = input.int(9, "AM Macro Start Hour", minval=0, maxval=23, group=groupAMTime, display=display.none)
amMacroStartMinute = input.int(50, "AM Macro Start Minute", minval=0, maxval=59, group=groupAMTime, display=display.none)
amMacroEndHour = input.int(10, "AM Macro End Hour", minval=0, maxval=23, group=groupAMTime, display=display.none)
amMacroEndMinute = input.int(10, "AM Macro End Minute", minval=0, maxval=59, group=groupAMTime, display=display.none)

groupStandaloneVisuals = "- - - - - - - - - Standalone Macro Visuals - - - - - - - - -"

CM_macroLineColor = input.color(color.new(color.black, 35), "Macro Line Color", group=groupStandaloneVisuals, display=display.none)
CM_macroBoxColor = input.color(color.new(color.gray, 90), "Macro Box Color", group=groupStandaloneVisuals, display=display.none)
CM_macroTextColor = input.color(color.new(color.black, 0), "Macro Text Color", group=groupStandaloneVisuals, display=display.none)
CM_macroLabelBgColor = input.color(color.new(color.white, 100), "Macro Label Background", group=groupStandaloneVisuals, display=display.none)

CM_imbLineColor = input.color(color.new(color.blue, 30), "Macro FPI Line Color", group=groupStandaloneVisuals, display=display.none)
CM_imbBoxColor = input.color(color.new(color.blue, 85), "Macro FPI Box Color", group=groupStandaloneVisuals, display=display.none)
CM_imbTextColor = input.color(color.new(color.black, 0), "Macro FPI Text Color", group=groupStandaloneVisuals, display=display.none)
CM_imbLabelBgColor = input.color(color.new(color.white, 100), "Macro FPI Label Background", group=groupStandaloneVisuals, display=display.none)

// ============================================================================
// Standalone Time / Style Helpers
// ============================================================================

CM_getLineStyle(_style) =>
    _style == "dashed" ? line.style_dashed : _style == "dotted" ? line.style_dotted : line.style_solid

isTimeBetween(t, _startH, _startM, _endH, _endM) =>
    _h = hour(t, timezone)
    _m = minute(t, timezone)
    inSession = if _startH < _endH
        (_h > _startH or (_h == _startH and _m >= _startM)) and (_h < _endH or (_h == _endH and _m <= _endM))
    else if _startH > _endH
        (_h > _startH or (_h == _startH and _m >= _startM)) or (_h < _endH or (_h == _endH and _m <= _endM))
    else
        _h == _startH and _m >= _startM and _m <= _endM
    inSession

isTimeInFpiSession(t) =>
    isTimeBetween(t, amFpiStartHour, amFpiStartMinute, amFpiEndHour, amFpiEndMinute)

isTimeInMacroWindow(t) =>
    isTimeBetween(t, amMacroStartHour, amMacroStartMinute, amMacroEndHour, amMacroEndMinute)

// ============================================================================
// Bias System Inputs
// ============================================================================

groupFpiBiasTable = "- - - - - - - - - FHIL System - - - - - - - - -"

showLowProbWarning = input.bool(true, "Show AM Low Probability Warning", group=groupFpiBiasTable, display=display.none)
showPMLowProbWarning = input.bool(true, "Show PM Warning Label", group=groupFpiBiasTable, display=display.none)
biasTableTextSizeOpt = input.string("small", "Table Text Size", options=["tiny", "small", "normal", "large"], group=groupFpiBiasTable, display=display.none)
warningLabelXOffset = input.int(8, "AM Warning Label Right Offset", minval=-100, maxval=300, group=groupFpiBiasTable, display=display.none)
warningLabelYOffsetATR = input.float(0.50, "AM Warning Label Y Offset ATR", minval=-10.0, maxval=10.0, step=0.25, group=groupFpiBiasTable, display=display.none)
pmWarningLabelXOffset = input.int(8, "PM Warning Label Right Offset", minval=-100, maxval=300, group=groupFpiBiasTable, display=display.none)
pmWarningLabelYOffsetATR = input.float(1.00, "PM Warning Label Y Offset ATR", minval=-10.0, maxval=10.0, step=0.25, group=groupFpiBiasTable, display=display.none)
warningLabelTextColor = input.color(color.yellow, "AM Warning Label Text Color", group=groupFpiBiasTable, display=display.none)
pmWarningLabelTextColor = input.color(color.yellow, "PM Warning Label Text Color", group=groupFpiBiasTable, display=display.none)

showBiasSignalArrows = input.bool(true, "Enable Entry / Exit Arrows", group=groupFpiBiasTable)
enableEntryAlerts = input.bool(true, "Enable Entry Alerts", group=groupFpiBiasTable, display=display.none)
maxBiasEntriesPerDay = input.int(5, "Max Entries Per Day", minval=1, maxval=50, group=groupFpiBiasTable, display=display.none)
maxPyramidEntriesPerTrade = input.int(1, "Max Pyramid Entries Per Trade", minval=1, maxval=20, group=groupFpiBiasTable, display=display.none)
allowOppositeSignalFlip = input.bool(false, "Allow Opposite Signal Flip", group=groupFpiBiasTable, display=display.none)
showBiasEntryPriceLabel = input.bool(true, "Show Entry Arrow Price", group=groupFpiBiasTable, display=display.none)
showBiasExitPriceLabel = input.bool(false, "Show Exit Arrow Price", group=groupFpiBiasTable, display=display.none)
biasArrowSizeOpt = input.string("small", "Entry / Exit Arrow Size", options=["tiny", "small", "normal", "large", "huge"], group=groupFpiBiasTable, display=display.none)
biasArrowYOffsetATR = input.float(0.15, "Entry / Exit Arrow Y Offset ATR", minval=0.0, maxval=10.0, step=0.05, group=groupFpiBiasTable, display=display.none)
bullEntryArrowColor = input.color(color.new(color.green, 0), "Bullish Entry Arrow Color", group=groupFpiBiasTable, display=display.none)
bullExitArrowColor = input.color(color.new(color.red, 0), "Bullish Exit Arrow Color", group=groupFpiBiasTable, display=display.none)
bearEntryArrowColor = input.color(color.new(color.red, 0), "Bearish Entry Arrow Color", group=groupFpiBiasTable, display=display.none)
bearExitArrowColor = input.color(color.new(color.green, 0), "Bearish Exit Arrow Color", group=groupFpiBiasTable, display=display.none)
biasArrowTextColor = input.color(color.white, "Entry / Exit Arrow Text Color", group=groupFpiBiasTable, display=display.none)

enableBiasLabelStacking = input.bool(true, "Stack Entry / Exit Text Labels", group=groupFpiBiasTable, display=display.none)
biasStackWindowBars = input.int(20, "Stack Labels Within Bars", minval=1, maxval=200, group=groupFpiBiasTable, display=display.none)
biasStackSpacingATR = input.float(0.35, "Stack Label Spacing ATR", minval=0.05, maxval=5.0, step=0.05, group=groupFpiBiasTable, display=display.none)

groupRisk = "- - - - - - - - - Risk / Stop Loss - - - - - - - - -"

defaultStopLossDistancePoints = input.float(30.0, "Default Stop Loss Distance From Entry", minval=0.25, step=0.25, group=groupRisk)
stopLossMode = input.string("Structure", "Stop Loss Mode", options=["Structure", "Default Distance", "Wider Of Both"], group=groupRisk)
enableTakeProfit = input.bool(true, "Enable Take Profit", group=groupRisk)
takeProfitDistancePoints = input.float(60.0, "Take Profit Distance From Entry", minval=0.25, step=0.25, group=groupRisk)
lockTakeProfitToFirstEntry = input.bool(true, "Lock TP To First Entry", group=groupRisk)
blockTradesAfterTakeProfit = input.bool(false, "Block New Trades After Take Profit", group=groupRisk)

// Macro extreme engine: focuses entries around macro highs/lows and their FPI boundaries.
groupMacroExtreme = "- - - - - - - - - Macro Extreme Engine - - - - - - - - -"
enableMacroExtremeEngine = input.bool(true, "Enable Macro High / Low Entries", group=groupMacroExtreme, display=display.none)
requireStructureNearMacroExtreme = input.bool(true, "Require OB / Imb Near Macro Extreme", group=groupMacroExtreme, display=display.none)
macroExtremeProximityPoints = input.float(12.0, "Macro Extreme Proximity", minval=0.25, step=0.25, group=groupMacroExtreme, display=display.none)
macroSweepBufferPoints = input.float(0.0, "Required Sweep Beyond Level", minval=0.0, step=0.25, group=groupMacroExtreme, display=display.none)
macroStopBufferPoints = input.float(6.0, "Macro SL Buffer", minval=0.0, step=0.25, group=groupMacroExtreme, display=display.none)
requireMacroReclaimClose = input.bool(true, "Require Close Back Inside Macro Level", group=groupMacroExtreme, display=display.none)
applyMaturityFilterToMacroExtreme = input.bool(false, "Apply Reversal Filter To Macro Extreme Entries", group=groupMacroExtreme, display=display.none)

// Hidden range engine: keeps the system anchored to the current dealing range and trades only at its extremes.
enableRangeExtremeEngine = input.bool(true, "Enable Range Extreme Ping-Pong Engine", group=groupMacroExtreme, display=display.none)
rangeExtremeProximityPoints = input.float(10.0, "Range Extreme Proximity", minval=0.25, step=0.25, group=groupMacroExtreme, display=display.none)
rangeExtremeStopBufferPoints = input.float(6.0, "Range Extreme SL Buffer", minval=0.0, step=0.25, group=groupMacroExtreme, display=display.none)
requireRangeReclaimClose = input.bool(true, "Require Range Reclaim Close", group=groupMacroExtreme, display=display.none)


// Reversal quality filter: prevents the system from buying/selling every OB/FVG in the middle of delivery.
groupReversalFilter = "- - - - - - - - - Reversal Quality Filter - - - - - - - - -"
enableReversalMaturityFilter = input.bool(true, "Enable Reversal Delivery Filter", group=groupReversalFilter, display=display.none)
applyFilterToMacroReversals = input.bool(true, "Apply Filter To Macro Reversal Entries", group=groupReversalFilter, display=display.none)
applyFilterToCounterBiasStructures = input.bool(true, "Apply Filter To Counter-Bias OB / Imb Entries", group=groupReversalFilter, display=display.none)
minReversalDeliveryPoints = input.float(80.0, "Minimum Session Delivery Before Reversal", minval=0.25, step=0.25, group=groupReversalFilter, display=display.none)
reversalExtremeProximityPoints = input.float(25.0, "Max Distance From Session Extreme", minval=0.25, step=0.25, group=groupReversalFilter, display=display.none)
minBarsBeforeReversal = input.int(20, "Minimum 1m Bars Before Reversal", minval=0, maxval=240, group=groupReversalFilter, display=display.none)
requireReversalTimingWindow = input.bool(false, "Require 00 / 30 Minute Timing Window", group=groupReversalFilter, display=display.none)
reversalTimingToleranceMinutes = input.int(3, "Timing Window Tolerance Minutes", minval=0, maxval=15, group=groupReversalFilter, display=display.none)
showReversalFilterWarnings = input.bool(true, "Show Reversal Filter Warning", group=groupReversalFilter, display=display.none)

// ============================================================================
// Market Maker Structure Inputs
// ============================================================================

groupMMStructure = "- - - - - - - - - Market Maker Structure Engine - - - - - - - - -"

enableMMStructureEngine = input.bool(true, "Enable OB / Imbalance Engine", group=groupMMStructure, display=display.none)
enableMMOBEntries = input.bool(true, "Enable OB Entries", group=groupMMStructure, display=display.none)
enableMMImbalanceEntries = input.bool(true, "Enable Imbalance Entries", group=groupMMStructure, display=display.none)
enableCounterBiasStructureEntries = input.bool(true, "Enable Counter-Bias Structure Entries After FPI Break", group=groupMMStructure, display=display.none)
allowExtremeCounterBiasBeforeFpiBreak = input.bool(true, "Allow Extreme Counter-Bias Entries Before FPI Break", group=groupMMStructure, display=display.none)
reversalBreakMode = input.string("Close", "FPI Break Confirmation Mode", options=["Close", "Wick"], group=groupMMStructure, display=display.none)
structureEntryTriggerMode = input.string("Body Respect", "Structure Entry Trigger Mode", options=["Body Respect", "Full Exit"], group=groupMMStructure, display=display.none)
showMMStructureWarnings = input.bool(true, "Show Active OB / Imbalance Warning Label", group=groupMMStructure, display=display.none)
showMMSelectedZones = input.bool(true, "Show Selected OB / Imbalance Zones", group=groupMMStructure, display=display.none)
mmOBRunLookback = input.int(8, "OB Consecutive Candle Run Lookback", minval=1, maxval=25, group=groupMMStructure, display=display.none)
mmMaxStoredStructures = input.int(80, "Max Stored Structures Per Session", minval=10, maxval=300, group=groupMMStructure, display=display.none)

mmWarningXOffset = input.int(10, "Structure Warning X Offset", minval=-100, maxval=300, group=groupMMStructure, display=display.none)
mmWarningYOffsetATR = input.float(1.50, "Structure Warning Y Offset ATR", minval=-10.0, maxval=10.0, step=0.25, group=groupMMStructure, display=display.none)
mmWarningTextColor = input.color(color.yellow, "Structure Warning Text Color", group=groupMMStructure, display=display.none)

bullOBBoxColor = input.color(color.new(color.green, 82), "Bullish OB Box Color", group=groupMMStructure, display=display.none)
bearOBBoxColor = input.color(color.new(color.red, 82), "Bearish OB Box Color", group=groupMMStructure, display=display.none)
bullImbBoxColor = input.color(color.new(color.teal, 84), "Bullish Imbalance Box Color", group=groupMMStructure, display=display.none)
bearImbBoxColor = input.color(color.new(color.maroon, 84), "Bearish Imbalance Box Color", group=groupMMStructure, display=display.none)
mmStructureBorderColor = input.color(color.new(color.white, 35), "Structure Border Color", group=groupMMStructure, display=display.none)

// ============================================================================
// Helper Functions
// ============================================================================

biasTableTextSize() =>
    sz = size.small
    if biasTableTextSizeOpt == "tiny"
        sz := size.tiny
    else if biasTableTextSizeOpt == "normal"
        sz := size.normal
    else if biasTableTextSizeOpt == "large"
        sz := size.large
    sz

biasArrowSize() =>
    sz = size.small
    if biasArrowSizeOpt == "tiny"
        sz := size.tiny
    else if biasArrowSizeOpt == "normal"
        sz := size.normal
    else if biasArrowSizeOpt == "large"
        sz := size.large
    else if biasArrowSizeOpt == "huge"
        sz := size.huge
    sz

biasDirText(_dir) =>
    txt = "Waiting"
    if _dir == 1
        txt := "Bullish"
    else if _dir == -1
        txt := "Bearish"
    txt

biasDirBg(_dir) =>
    bg = color.new(color.gray, 80)
    if _dir == 1
        bg := color.new(color.green, 70)
    else if _dir == -1
        bg := color.new(color.red, 70)
    bg

biasConfirmText(_biasDir, _macroFound, _bullCont, _bearCont, _bullRev, _bearRev, _sessionName) =>
    txt = "Waiting For " + _sessionName + " FPI"
    if _biasDir != 0 and not _macroFound
        txt := biasDirText(_biasDir) + " / Waiting For Macro FPI"
    else if _bullCont
        txt := "Bullish Confirmed"
    else if _bearCont
        txt := "Bearish Confirmed"
    else if _bullRev
        txt := "Bull Reversal Confirmed"
    else if _bearRev
        txt := "Bear Reversal Confirmed"
    else if _biasDir != 0
        txt := biasDirText(_biasDir) + " / Not Confirmed"
    txt

biasConfirmBg(_bullCont, _bearCont, _bullRev, _bearRev) =>
    bg = color.new(color.orange, 70)
    if _bullCont or _bullRev
        bg := color.new(color.green, 55)
    else if _bearCont or _bearRev
        bg := color.new(color.red, 55)
    bg

calcBiasSL(_dir, _entryPrice, _structureSL) =>
    defaultSL = _dir == 1 ? _entryPrice - defaultStopLossDistancePoints : _entryPrice + defaultStopLossDistancePoints
    sl = _structureSL
    if stopLossMode == "Default Distance"
        sl := defaultSL
    else if stopLossMode == "Wider Of Both"
        if na(_structureSL)
            sl := defaultSL
        else
            sl := _dir == 1 ? math.min(_structureSL, defaultSL) : math.max(_structureSL, defaultSL)
    else
        if na(_structureSL)
            sl := defaultSL
    sl

calcBiasTP(_dir, _entryPrice) =>
    enableTakeProfit ? (_dir == 1 ? _entryPrice + takeProfitDistancePoints : _entryPrice - takeProfitDistancePoints) : na

combineBiasSL(_dir, _oldSL, _newSL) =>
    combined = _newSL
    if not na(_oldSL)
        combined := _dir == 1 ? math.min(_oldSL, _newSL) : math.max(_oldSL, _newSL)
    combined

entryLabelText(_tradeId, _entryNum, _sessionName, _sourceName, _price, _dir, _sl, _tp) =>
    tradeType = _dir == 1 ? "Bull Trade" : "Bear Trade"
    tpText = enableTakeProfit and not na(_tp) ? "\nTP: " + str.tostring(_tp, format.mintick) : ""
    slText = not na(_sl) ? "\nSL: " + str.tostring(_sl, format.mintick) : "\nSL: -"
    showBiasEntryPriceLabel ? "Trade " + str.tostring(_tradeId) + "-" + str.tostring(_entryNum) + "\n" + _sessionName + " " + _sourceName + "\n" + tradeType + "\nEntry: " + str.tostring(_price, format.mintick) + tpText + slText : ""

entryAlertText(_tradeId, _entryNum, _sessionName, _sourceName, _price, _dir, _sl, _tp) =>
    tradeType = _dir == 1 ? "Bull Trade" : "Bear Trade"
    tpText = enableTakeProfit and not na(_tp) ? "\nTP: " + str.tostring(_tp, format.mintick) : ""
    slText = not na(_sl) ? "\nSL: " + str.tostring(_sl, format.mintick) : "\nSL: -"
    "FHIL Entry Alert" + "\n" + syminfo.ticker + " " + timeframe.period + "m" + "\nTrade " + str.tostring(_tradeId) + "-" + str.tostring(_entryNum) + "\n" + _sessionName + " " + _sourceName + "\n" + tradeType + "\nEntry: " + str.tostring(_price, format.mintick) + tpText + slText

breaksAboveLevel(_level) =>
    reversalBreakMode == "Wick" ? high > _level : close > _level

breaksBelowLevel(_level) =>
    reversalBreakMode == "Wick" ? low < _level : close < _level

bullZoneRetestExit(_zoneHigh, _zoneLow) =>
    validZone = not na(_zoneHigh) and not na(_zoneLow)
    touchedZone = validZone and low <= _zoneHigh and high >= _zoneLow
    bodyRespectsLow = touchedZone and close > _zoneLow
    fullExitAboveZone = validZone and close > _zoneHigh and (low <= _zoneHigh or close[1] <= _zoneHigh)
    structureEntryTriggerMode == "Full Exit" ? fullExitAboveZone : bodyRespectsLow

bearZoneRetestExit(_zoneHigh, _zoneLow) =>
    validZone = not na(_zoneHigh) and not na(_zoneLow)
    touchedZone = validZone and high >= _zoneLow and low <= _zoneHigh
    bodyRespectsHigh = touchedZone and close < _zoneHigh
    fullExitBelowZone = validZone and close < _zoneLow and (high >= _zoneLow or close[1] >= _zoneLow)
    structureEntryTriggerMode == "Full Exit" ? fullExitBelowZone : bodyRespectsHigh

bullLevelReject(float _level) =>
    not na(_level) and low <= _level + macroExtremeProximityPoints and (not requireMacroReclaimClose or close > _level) and (macroSweepBufferPoints <= 0 or low <= _level - macroSweepBufferPoints)

bearLevelReject(float _level) =>
    not na(_level) and high >= _level - macroExtremeProximityPoints and (not requireMacroReclaimClose or close < _level) and (macroSweepBufferPoints <= 0 or high >= _level + macroSweepBufferPoints)

zoneTouchesLevel(float _zoneHigh, float _zoneLow, float _level, float _prox) =>
    not na(_zoneHigh) and not na(_zoneLow) and not na(_level) and _zoneLow <= _level + _prox and _zoneHigh >= _level - _prox

zoneTouchesAny6(float _zoneHigh, float _zoneLow, float _prox, float _l1, float _l2, float _l3, float _l4, float _l5, float _l6) =>
    zoneTouchesLevel(_zoneHigh, _zoneLow, _l1, _prox) or zoneTouchesLevel(_zoneHigh, _zoneLow, _l2, _prox) or zoneTouchesLevel(_zoneHigh, _zoneLow, _l3, _prox) or zoneTouchesLevel(_zoneHigh, _zoneLow, _l4, _prox) or zoneTouchesLevel(_zoneHigh, _zoneLow, _l5, _prox) or zoneTouchesLevel(_zoneHigh, _zoneLow, _l6, _prox)

mergedHigh(_aFound, _aHigh, _bFound, _bHigh) =>
    _aFound and _bFound ? math.max(_aHigh, _bHigh) : _aFound ? _aHigh : _bFound ? _bHigh : na

mergedLow(_aFound, _aLow, _bFound, _bLow) =>
    _aFound and _bFound ? math.min(_aLow, _bLow) : _aFound ? _aLow : _bFound ? _bLow : na

// ============================================================================
// Market Maker Structure Type / Functions
// ============================================================================

type MMStruct
    int session
    int kind
    int dir
    float high
    float low
    int startBar
    int confirmedBar
    bool tested
    bool respected
    bool disrespected
    int respectBar
    bool used

updateMMStructStates(array<MMStruct> _arr) =>
    if array.size(_arr) > 0
        for i = 0 to array.size(_arr) - 1
            s = array.get(_arr, i)
            if not s.disrespected and bar_index > s.confirmedBar
                touched = low <= s.high and high >= s.low
                if s.dir == 1
                    if close < s.low
                        s.disrespected := true
                    else if touched
                        s.tested := true
                        if not s.respected
                            s.respected := true
                            s.respectBar := bar_index
                else if s.dir == -1
                    if close > s.high
                        s.disrespected := true
                    else if touched
                        s.tested := true
                        if not s.respected
                            s.respected := true
                            s.respectBar := bar_index
            array.set(_arr, i, s)

trimMMStructArray(array<MMStruct> _arr, int _maxSize) =>
    while array.size(_arr) > _maxSize
        array.shift(_arr)

markMMStructureUsed(array<MMStruct> _arr, int _idx) =>
    if _idx >= 0 and _idx < array.size(_arr)
        s = array.get(_arr, _idx)
        s.used := true
        array.set(_arr, _idx, s)

selectMMStructure(array<MMStruct> _arr, int _dir, int _biasDir, bool _requireRespected) =>
    bool found = false
    float selHigh = na
    float selLow = na
    int selStart = na
    int selRespectBar = na
    bool selUsed = false
    int selIdx = -1
    bool useHighest = _biasDir == -1
    if array.size(_arr) > 0
        for i = 0 to array.size(_arr) - 1
            s = array.get(_arr, i)
            okRespect = _requireRespected ? s.respected : true
            if s.dir == _dir and not s.disrespected and okRespect
                score = useHighest ? s.high : s.low
                oldScore = useHighest ? selHigh : selLow
                if not found or (useHighest and score > oldScore) or (not useHighest and score < oldScore)
                    found := true
                    selHigh := s.high
                    selLow := s.low
                    selStart := s.startBar
                    selRespectBar := s.respectBar
                    selUsed := s.used
                    selIdx := i
    [found, selHigh, selLow, selStart, selRespectBar, selUsed, selIdx]

// Scans every stored OB/FVG on the current candle and chooses the true live extreme.
// This fixes cases where the highlighted highest/lowest structure exists, but the old selected-structure logic is still focused on a different zone.
scanLiveExtremeStructure(array<MMStruct> _arr, int _dir, bool _requireMacro, float _l1, float _l2, float _l3, float _l4, float _l5, float _l6) =>
    bool found = false
    float selHigh = na
    float selLow = na
    int selStart = na
    int selIdx = -1
    if array.size(_arr) > 0
        for i = 0 to array.size(_arr) - 1
            s = array.get(_arr, i)
            liveZone = s.dir == _dir and not s.disrespected and not s.used and bar_index > s.confirmedBar
            trigger = _dir == 1 ? bullZoneRetestExit(s.high, s.low) : bearZoneRetestExit(s.high, s.low)
            macroOk = not _requireMacro or zoneTouchesAny6(s.high, s.low, macroExtremeProximityPoints, _l1, _l2, _l3, _l4, _l5, _l6)
            if liveZone and trigger and macroOk
                if _dir == 1
                    if not found or s.low < selLow
                        found := true
                        selHigh := s.high
                        selLow := s.low
                        selStart := s.startBar
                        selIdx := i
                else
                    if not found or s.high > selHigh
                        found := true
                        selHigh := s.high
                        selLow := s.low
                        selStart := s.startBar
                        selIdx := i
    [found, selHigh, selLow, selStart, selIdx]

// Finds the highest bearish or lowest bullish structure in the active session, even before it is touched.
// This gives the algo an actual range boundary to ping-pong from instead of waiting for one selected structure.
scanSessionExtremeStructure(array<MMStruct> _arr, int _dir, bool _requireMacro, float _l1, float _l2, float _l3, float _l4, float _l5, float _l6) =>
    bool found = false
    float selHigh = na
    float selLow = na
    int selStart = na
    int selIdx = -1
    if array.size(_arr) > 0
        for i = 0 to array.size(_arr) - 1
            s = array.get(_arr, i)
            liveZone = s.dir == _dir and not s.disrespected and not s.used and bar_index > s.confirmedBar
            macroOk = not _requireMacro or zoneTouchesAny6(s.high, s.low, macroExtremeProximityPoints, _l1, _l2, _l3, _l4, _l5, _l6)
            if liveZone and macroOk
                if _dir == 1
                    if not found or s.low < selLow
                        found := true
                        selHigh := s.high
                        selLow := s.low
                        selStart := s.startBar
                        selIdx := i
                else
                    if not found or s.high > selHigh
                        found := true
                        selHigh := s.high
                        selLow := s.low
                        selStart := s.startBar
                        selIdx := i
    [found, selHigh, selLow, selStart, selIdx]

max2(float _a, float _b) =>
    na(_a) ? _b : na(_b) ? _a : math.max(_a, _b)

min2(float _a, float _b) =>
    na(_a) ? _b : na(_b) ? _a : math.min(_a, _b)

levelRejectsLow(float _level) =>
    not na(_level) and low <= _level + rangeExtremeProximityPoints and (not requireRangeReclaimClose or close > _level)

levelRejectsHigh(float _level) =>
    not na(_level) and high >= _level - rangeExtremeProximityPoints and (not requireRangeReclaimClose or close < _level)

findBullOB(_maxBack) =>
    bool found = false
    float lowestClose = na
    float farthestOpen = na
    int farthestOffset = na
    int off = 2
    while off <= _maxBack and close[off] < open[off]
        lowestClose := na(lowestClose) ? close[off] : math.min(lowestClose, close[off])
        farthestOpen := open[off]
        farthestOffset := off
        off += 1
    float obHigh = na
    float obLow = na
    int obStart = na
    if not na(farthestOffset)
        found := true
        obHigh := math.max(farthestOpen, lowestClose)
        obLow := math.min(farthestOpen, lowestClose)
        obStart := bar_index - farthestOffset
    [found, obHigh, obLow, obStart]

findBearOB(_maxBack) =>
    bool found = false
    float highestClose = na
    float farthestOpen = na
    int farthestOffset = na
    int off = 2
    while off <= _maxBack and close[off] > open[off]
        highestClose := na(highestClose) ? close[off] : math.max(highestClose, close[off])
        farthestOpen := open[off]
        farthestOffset := off
        off += 1
    float obHigh = na
    float obLow = na
    int obStart = na
    if not na(farthestOffset)
        found := true
        obHigh := math.max(farthestOpen, highestClose)
        obLow := math.min(farthestOpen, highestClose)
        obStart := bar_index - farthestOffset
    [found, obHigh, obLow, obStart]

// ============================================================================
// State Variables
// ============================================================================

var bool bias930Found = false
var int bias930Dir = 0
var float bias930High = na
var float bias930Low = na
var float bias930Mid = na

var bool bias950Found = false
var int bias950Dir = 0
var float bias950High = na
var float bias950Low = na
var float bias950Mid = na

var bool pm_bias1200Found = false
var int pm_bias1200Dir = 0
var float pm_bias1200High = na
var float pm_bias1200Low = na
var float pm_bias1200Mid = na

var bool PM1250_biasFound = false
var int PM1250_biasDir = 0
var float PM1250_biasHigh = na
var float PM1250_biasLow = na
var float PM1250_biasMid = na

var float amMacroRangeHigh = na
var float amMacroRangeLow = na
var int amMacroRangeStartBar = na
var int amMacroRangeEndBar = na
var bool amMacroHighUsed = false
var bool amMacroLowUsed = false

var float PM1250_winHigh = na
var float PM1250_winLow = na
var int PM1250_winStartBar = na
var int PM1250_winEndBar = na
var bool pmMacroHighUsed = false
var bool pmMacroLowUsed = false

var float close1159 = na

var float amSessionHigh = na
var float amSessionLow = na
var int amSessionStartBar = na
var float pmSessionHigh = na
var float pmSessionLow = na
var int pmSessionStartBar = na
var label reversalFilterWarningLabel = na

var int biasEntriesToday = 0
var int biasTradeId = 0
var int biasPyramidEntryCount = 0
var bool biasInPosition = false
var int biasActiveDir = 0
var int biasActiveSession = 0
var float biasEntryPrice = na
var float biasEntrySL = na
var float biasEntryTP = na
var bool biasTakeProfitHitToday = false
var int biasLastEntryBar = na
var label[] biasSignalLabels = array.new_label()

var int biasUpperLabelStack = 0
var int biasLowerLabelStack = 0
var int biasLastUpperLabelBar = na
var int biasLastLowerLabelBar = na

var bool amBullJudasArmed = false
var bool amBearJudasArmed = false
var bool pmBullJudasArmed = false
var bool pmBearJudasArmed = false

var array<MMStruct> amMMFVGs = array.new<MMStruct>()
var array<MMStruct> amMMOBs = array.new<MMStruct>()
var array<MMStruct> pmMMFVGs = array.new<MMStruct>()
var array<MMStruct> pmMMOBs = array.new<MMStruct>()

var table fpiBiasTable = table.new(position.bottom_left, 2, 14, border_width=1)
var label fpiLowProbabilityLabel = na
var label fpiPMLowProbabilityLabel = na
var label mmStructureWarningLabel = na

var box amBullOBBox = na
var box amBearOBBox = na
var box amBullFVGBox = na
var box amBearFVGBox = na
var box pmBullOBBox = na
var box pmBearOBBox = na
var box pmBullFVGBox = na
var box pmBearFVGBox = na
var float londonSessionHigh = na
var float londonSessionLow = na
var int londonSessionStartBar = na
var int londonSessionEndBar = na
var bool londonSessionDrawn = false
var float asianSessionHigh = na
var float asianSessionLow = na
var int asianSessionStartBar = na
var int asianSessionEndBar = na
var bool asianSessionDrawn = false
var float nyPreSessionHigh = na
var float nyPreSessionLow = na
var int nyPreSessionStartBar = na
var int nyPreSessionEndBar = na
var bool nyPreSessionDrawn = false
var line[] sessionLevelLines = array.new_line()
var label[] sessionLevelLabels = array.new_label()

// ============================================================================
// Reversal Quality / Delivery Completion Helpers
// ============================================================================

isThirtyMinuteTimingOk() =>
    _m = minute(time, timezone)
    _distTo00 = math.min(_m, 60 - _m)
    _distTo30 = math.abs(_m - 30)
    math.min(_distTo00, _distTo30) <= reversalTimingToleranceMinutes

reversalMaturityAllows(_dir, _sessionId) =>
    if not enableReversalMaturityFilter
        true
    else
        _hi = _sessionId == 1 ? amSessionHigh : pmSessionHigh
        _lo = _sessionId == 1 ? amSessionLow : pmSessionLow
        _startBar = _sessionId == 1 ? amSessionStartBar : pmSessionStartBar
        _delivery = not na(_hi) and not na(_lo) ? _hi - _lo : na
        _nearExtreme = _dir == 1 ? (not na(_lo) and low <= _lo + reversalExtremeProximityPoints) : (not na(_hi) and high >= _hi - reversalExtremeProximityPoints)
        _enoughDelivery = not na(_delivery) and _delivery >= minReversalDeliveryPoints
        _barsOk = not na(_startBar) and bar_index - _startBar >= minBarsBeforeReversal
        _timingOk = not requireReversalTimingWindow or isThirtyMinuteTimingOk()
        _enoughDelivery and _nearExtreme and _barsOk and _timingOk

reversalFilterStatusText(_sessionId) =>
    _hi = _sessionId == 1 ? amSessionHigh : pmSessionHigh
    _lo = _sessionId == 1 ? amSessionLow : pmSessionLow
    _delivery = not na(_hi) and not na(_lo) ? _hi - _lo : na
    not enableReversalMaturityFilter ? "OFF" : na(_delivery) ? "Waiting" : str.tostring(_delivery, format.mintick) + " / " + str.tostring(minReversalDeliveryPoints, format.mintick)


// ============================================================================
// Label Stacking Helpers
// ============================================================================

biasUpperStackIndex() =>
    enableBiasLabelStacking and not na(biasLastUpperLabelBar) and bar_index - biasLastUpperLabelBar <= biasStackWindowBars ? biasUpperLabelStack : 0

biasLowerStackIndex() =>
    enableBiasLabelStacking and not na(biasLastLowerLabelBar) and bar_index - biasLastLowerLabelBar <= biasStackWindowBars ? biasLowerLabelStack : 0

biasUpperLabelY(_baseHigh, _atr) =>
    _stackIndex = biasUpperStackIndex()
    _baseHigh + _atr * (biasArrowYOffsetATR + _stackIndex * biasStackSpacingATR)

biasLowerLabelY(_baseLow, _atr) =>
    _stackIndex = biasLowerStackIndex()
    _baseLow - _atr * (biasArrowYOffsetATR + _stackIndex * biasStackSpacingATR)

canEnterBiasDir(_dir) =>
    sameDirCanPyramid = biasInPosition and biasActiveDir == _dir and biasPyramidEntryCount < maxPyramidEntriesPerTrade
    oppositeCanFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != _dir
    tradeLockOk = not (blockTradesAfterTakeProfit and biasTakeProfitHitToday)
    notSameBar = na(biasLastEntryBar) or biasLastEntryBar != bar_index
    tradeLockOk and notSameBar and (not biasInPosition or sameDirCanPyramid or oppositeCanFlip)

// ============================================================================
// Daily Reset
// ============================================================================

biasResetNow = hour(time, timezone) == startTimeHour and minute(time, timezone) == 0 and (hour(time[1], timezone) != startTimeHour or minute(time[1], timezone) != 0)

if biasResetNow
    bias930Found := false
    bias930Dir := 0
    bias930High := na
    bias930Low := na
    bias930Mid := na

    bias950Found := false
    bias950Dir := 0
    bias950High := na
    bias950Low := na
    bias950Mid := na

    pm_bias1200Found := false
    pm_bias1200Dir := 0
    pm_bias1200High := na
    pm_bias1200Low := na
    pm_bias1200Mid := na

    PM1250_biasFound := false
    PM1250_biasDir := 0
    PM1250_biasHigh := na
    PM1250_biasLow := na
    PM1250_biasMid := na

    amMacroRangeHigh := na
    amMacroRangeLow := na
    amMacroRangeStartBar := na
    amMacroRangeEndBar := na
    amMacroHighUsed := false
    amMacroLowUsed := false

    PM1250_winHigh := na
    PM1250_winLow := na
    PM1250_winStartBar := na
    PM1250_winEndBar := na
    pmMacroHighUsed := false
    pmMacroLowUsed := false

    close1159 := na
    amSessionHigh := na
    amSessionLow := na
    amSessionStartBar := na
    pmSessionHigh := na
    pmSessionLow := na
    pmSessionStartBar := na
    londonSessionHigh := na
    londonSessionLow := na
    londonSessionStartBar := na
    londonSessionEndBar := na
    londonSessionDrawn := false
    asianSessionHigh := na
    asianSessionLow := na
    asianSessionStartBar := na
    asianSessionEndBar := na
    asianSessionDrawn := false
    nyPreSessionHigh := na
    nyPreSessionLow := na
    nyPreSessionStartBar := na
    nyPreSessionEndBar := na
    nyPreSessionDrawn := false

    biasEntriesToday := 0
    biasTradeId := 0
    biasPyramidEntryCount := 0
    biasInPosition := false
    biasActiveDir := 0
    biasActiveSession := 0
    biasEntryPrice := na
    biasEntrySL := na
    biasEntryTP := na
    biasTakeProfitHitToday := false
    biasLastEntryBar := na

    biasUpperLabelStack := 0
    biasLowerLabelStack := 0
    biasLastUpperLabelBar := na
    biasLastLowerLabelBar := na

    amBullJudasArmed := false
    amBearJudasArmed := false
    pmBullJudasArmed := false
    pmBearJudasArmed := false

    array.clear(amMMFVGs)
    array.clear(amMMOBs)
    array.clear(pmMMFVGs)
    array.clear(pmMMOBs)

    if not na(fpiLowProbabilityLabel)
        label.delete(fpiLowProbabilityLabel)
        fpiLowProbabilityLabel := na

    if not na(fpiPMLowProbabilityLabel)
        label.delete(fpiPMLowProbabilityLabel)
        fpiPMLowProbabilityLabel := na

    if not na(mmStructureWarningLabel)
        label.delete(mmStructureWarningLabel)
        mmStructureWarningLabel := na

    if not na(reversalFilterWarningLabel)
        label.delete(reversalFilterWarningLabel)
        reversalFilterWarningLabel := na

// ============================================================================
// Session High / Low Drawing Helpers
// ============================================================================

drawSessionHL(int _startBar, int _endBar, float _high, float _low, string _name) =>
    if not na(_startBar) and not na(_endBar) and not na(_high) and not na(_low)
        _extendTo = bar_index + 500
        _hiLine = line.new(x1=_startBar, y1=_high, x2=_extendTo, y2=_high, xloc=xloc.bar_index, color=sessionLevelLineColor, style=line.style_dashed, width=1)
        _loLine = line.new(x1=_startBar, y1=_low, x2=_extendTo, y2=_low, xloc=xloc.bar_index, color=sessionLevelLineColor, style=line.style_dashed, width=1)
        _hiLabel = label.new(x=_extendTo, y=_high, text=_name + " High | " + str.tostring(_high, format.mintick), xloc=xloc.bar_index, yloc=yloc.price, style=label.style_label_left, textcolor=sessionLevelTextColor, size=size.small, color=color.new(color.black, 100))
        _loLabel = label.new(x=_extendTo, y=_low, text=_name + " Low | " + str.tostring(_low, format.mintick), xloc=xloc.bar_index, yloc=yloc.price, style=label.style_label_left, textcolor=sessionLevelTextColor, size=size.small, color=color.new(color.black, 100))
        array.push(sessionLevelLines, _hiLine)
        array.push(sessionLevelLines, _loLine)
        array.push(sessionLevelLabels, _hiLabel)
        array.push(sessionLevelLabels, _loLabel)
        while array.size(sessionLevelLines) > globalLookbackPeriod * 6
            line.delete(array.shift(sessionLevelLines))
        while array.size(sessionLevelLabels) > globalLookbackPeriod * 6
            label.delete(array.shift(sessionLevelLabels))

// ============================================================================
// Session Time Helpers
// ============================================================================

amTradeWindow = timeframe.period == "1" and isTimeBetween(time, tradeStartHour, tradeStartMinute, amTradeEndHour, amTradeEndMinute)
amForceCloseNow = timeframe.period == "1" and barstate.isconfirmed and hour(time, timezone) == amTradeEndHour and minute(time, timezone) == amTradeEndMinute

pmUse1150Macro = pmSessionStartHour < 12
pmBiasFpiStartHour = pmSessionStartHour
pmBiasFpiStartMinute = pmSessionStartMinute
pmBiasFpiEndHour = pmTradeEndHour
pmBiasFpiEndMinute = pmTradeEndMinute

isTimeInPMBiasFpiSession(t) =>
    isTimeBetween(t, pmBiasFpiStartHour, pmBiasFpiStartMinute, pmBiasFpiEndHour, pmBiasFpiEndMinute)

isTimeInPM1250MacroWindow(t) =>
    _h = hour(t, timezone)
    _m = minute(t, timezone)
    (_h == 12 and _m >= 50) or (_h == 13 and _m <= 10)

pmTradeWindow = timeframe.period == "1" and isTimeBetween(time, pmBiasFpiStartHour, pmBiasFpiStartMinute, pmTradeEndHour, pmTradeEndMinute)

amTradeStartNow = timeframe.period == "1" and hour(time, timezone) == tradeStartHour and minute(time, timezone) == tradeStartMinute and (hour(time[1], timezone) != tradeStartHour or minute(time[1], timezone) != tradeStartMinute)
pmTradeStartNow = timeframe.period == "1" and hour(time, timezone) == pmBiasFpiStartHour and minute(time, timezone) == pmBiasFpiStartMinute and (hour(time[1], timezone) != pmBiasFpiStartHour or minute(time[1], timezone) != pmBiasFpiStartMinute)

if amTradeStartNow or pmTradeStartNow
    biasTakeProfitHitToday := false

if timeframe.period == "1" and showAMNY and isTimeInMacroWindow(time)
    amMacroRangeHigh := na(amMacroRangeHigh) ? high : math.max(amMacroRangeHigh, high)
    amMacroRangeLow := na(amMacroRangeLow) ? low : math.min(amMacroRangeLow, low)
    amMacroRangeStartBar := na(amMacroRangeStartBar) ? bar_index : amMacroRangeStartBar
    amMacroRangeEndBar := bar_index

if timeframe.period == "1" and showPMNYMarketMaker and not pmUse1150Macro and isTimeInPM1250MacroWindow(time)
    PM1250_winHigh := na(PM1250_winHigh) ? high : math.max(PM1250_winHigh, high)
    PM1250_winLow := na(PM1250_winLow) ? low : math.min(PM1250_winLow, low)
    PM1250_winStartBar := na(PM1250_winStartBar) ? bar_index : PM1250_winStartBar
    PM1250_winEndBar := bar_index

if timeframe.period == "1" and showAMNY and amTradeWindow
    amSessionHigh := na(amSessionHigh) ? high : math.max(amSessionHigh, high)
    amSessionLow := na(amSessionLow) ? low : math.min(amSessionLow, low)
    amSessionStartBar := na(amSessionStartBar) ? bar_index : amSessionStartBar

if timeframe.period == "1" and showPMNYMarketMaker and pmTradeWindow
    pmSessionHigh := na(pmSessionHigh) ? high : math.max(pmSessionHigh, high)
    pmSessionLow := na(pmSessionLow) ? low : math.min(pmSessionLow, low)
    pmSessionStartBar := na(pmSessionStartBar) ? bar_index : pmSessionStartBar

inLondonSession = timeframe.period == "1" and isTimeBetween(time, 2, 0, 5, 0)
prevInLondonSession = timeframe.period == "1" and isTimeBetween(time[1], 2, 0, 5, 0)
inAsianSession = timeframe.period == "1" and isTimeBetween(time, 19, 0, 0, 0)
prevInAsianSession = timeframe.period == "1" and isTimeBetween(time[1], 19, 0, 0, 0)
inNYPreSession = timeframe.period == "1" and isTimeBetween(time, 7, 0, 9, 0)
prevInNYPreSession = timeframe.period == "1" and isTimeBetween(time[1], 7, 0, 9, 0)

if showLondonSessionLevels and inLondonSession
    londonSessionHigh := na(londonSessionHigh) ? high : math.max(londonSessionHigh, high)
    londonSessionLow := na(londonSessionLow) ? low : math.min(londonSessionLow, low)
    londonSessionStartBar := na(londonSessionStartBar) ? bar_index : londonSessionStartBar
    londonSessionEndBar := bar_index
    londonSessionDrawn := false
if showLondonSessionLevels and not inLondonSession and prevInLondonSession and not londonSessionDrawn
    drawSessionHL(londonSessionStartBar, londonSessionEndBar, londonSessionHigh, londonSessionLow, "London 2-5")
    londonSessionDrawn := true

if showAsianSessionLevels and inAsianSession
    asianSessionHigh := na(asianSessionHigh) ? high : math.max(asianSessionHigh, high)
    asianSessionLow := na(asianSessionLow) ? low : math.min(asianSessionLow, low)
    asianSessionStartBar := na(asianSessionStartBar) ? bar_index : asianSessionStartBar
    asianSessionEndBar := bar_index
    asianSessionDrawn := false
if showAsianSessionLevels and not inAsianSession and prevInAsianSession and not asianSessionDrawn
    drawSessionHL(asianSessionStartBar, asianSessionEndBar, asianSessionHigh, asianSessionLow, "Asian 7-12")
    asianSessionDrawn := true

if showNYPreSessionLevels and inNYPreSession
    nyPreSessionHigh := na(nyPreSessionHigh) ? high : math.max(nyPreSessionHigh, high)
    nyPreSessionLow := na(nyPreSessionLow) ? low : math.min(nyPreSessionLow, low)
    nyPreSessionStartBar := na(nyPreSessionStartBar) ? bar_index : nyPreSessionStartBar
    nyPreSessionEndBar := bar_index
    nyPreSessionDrawn := false
if showNYPreSessionLevels and not inNYPreSession and prevInNYPreSession and not nyPreSessionDrawn
    drawSessionHL(nyPreSessionStartBar, nyPreSessionEndBar, nyPreSessionHigh, nyPreSessionLow, "NY 7-9")
    nyPreSessionDrawn := true

// ============================================================================
// 11:59 Close Capture
// ============================================================================

if timeframe.period == "1" and hour(time, timezone) == 11 and minute(time, timezone) == 59
    close1159 := close

// ============================================================================
// 9:30 AM FPI Direction Detection
// ============================================================================

bias930BullFpi = close[1] > open[1] and high[2] < low[0]
bias930BearFpi = close[1] < open[1] and low[2] > high[0]

if not bias930Found and showAMNY and isTimeInFpiSession(time[1])
    if bias930BullFpi or bias930BearFpi
        bias930Found := true
        bias930Dir := bias930BullFpi ? 1 : -1
        bias930High := bias930BullFpi ? low[0] : low[2]
        bias930Low := bias930BullFpi ? high[2] : high[0]
        bias930Mid := (bias930High + bias930Low) / 2

// ============================================================================
// 9:50 AM Macro FPI Detection
// ============================================================================

bias950BullFpi = high[2] < low[0]
bias950BearFpi = low[2] > high[0]

if not bias950Found and showAMNY and timeframe.period == "1" and isTimeInMacroWindow(time[1])
    if bias950BullFpi or bias950BearFpi
        bias950Found := true
        bias950Dir := bias950BullFpi ? 1 : -1
        bias950High := bias950BullFpi ? low[0] : low[2]
        bias950Low := bias950BullFpi ? high[2] : high[0]
        bias950Mid := (bias950High + bias950Low) / 2

// ============================================================================
// AM Confirmation + Reversal Logic
// ============================================================================

amBullFullConfirm = bias930Found and bias950Found and close > bias930High and close > bias950High
amBearFullConfirm = bias930Found and bias950Found and close < bias930Low and close < bias950Low

amBullContinuationConfirmed = amBullFullConfirm and bias930Dir == 1
amBearContinuationConfirmed = amBearFullConfirm and bias930Dir == -1
amBullReversalConfirmed = amBullFullConfirm and bias930Dir == -1
amBearReversalConfirmed = amBearFullConfirm and bias930Dir == 1

biasBullConfirmed = amBullFullConfirm
biasBearConfirmed = amBearFullConfirm
biasLowProbability = bias930Found and bias950Found and not biasBullConfirmed and not biasBearConfirmed

biasPriceVs950 = "Waiting"
if bias950Found
    if close > bias950High
        biasPriceVs950 := "Above 9:50 FPI"
    else if close < bias950Low
        biasPriceVs950 := "Below 9:50 FPI"
    else
        biasPriceVs950 := "Inside 9:50 FPI"

biasFinalText = biasConfirmText(bias930Dir, bias950Found, amBullContinuationConfirmed, amBearContinuationConfirmed, amBullReversalConfirmed, amBearReversalConfirmed, "9:30")
biasFinalBg = biasConfirmBg(amBullContinuationConfirmed, amBearContinuationConfirmed, amBullReversalConfirmed, amBearReversalConfirmed)

// ============================================================================
// Market Maker Structure Detection: AM / PM FVGs and OBs
// ============================================================================

mmBullFVG = close[1] > open[1] and high[2] < low[0]
mmBearFVG = close[1] < open[1] and low[2] > high[0]

if enableMMStructureEngine and timeframe.period == "1"
    if showAMNY and isTimeInFpiSession(time[1])
        if mmBullFVG
            array.push(amMMFVGs, MMStruct.new(1, 1, 1, low[0], high[2], bar_index - 2, bar_index, false, false, false, int(na), false))
            [bullOBFound, bullOBHigh, bullOBLow, bullOBStart] = findBullOB(mmOBRunLookback)
            if bullOBFound
                array.push(amMMOBs, MMStruct.new(1, 2, 1, bullOBHigh, bullOBLow, bullOBStart, bar_index, false, false, false, int(na), false))
        if mmBearFVG
            array.push(amMMFVGs, MMStruct.new(1, 1, -1, low[2], high[0], bar_index - 2, bar_index, false, false, false, int(na), false))
            [bearOBFound, bearOBHigh, bearOBLow, bearOBStart] = findBearOB(mmOBRunLookback)
            if bearOBFound
                array.push(amMMOBs, MMStruct.new(1, 2, -1, bearOBHigh, bearOBLow, bearOBStart, bar_index, false, false, false, int(na), false))

    if showPMNYMarketMaker and isTimeInPMBiasFpiSession(time[1])
        if mmBullFVG
            array.push(pmMMFVGs, MMStruct.new(2, 1, 1, low[0], high[2], bar_index - 2, bar_index, false, false, false, int(na), false))
            [pmBullOBFound, pmBullOBHigh, pmBullOBLow, pmBullOBStart] = findBullOB(mmOBRunLookback)
            if pmBullOBFound
                array.push(pmMMOBs, MMStruct.new(2, 2, 1, pmBullOBHigh, pmBullOBLow, pmBullOBStart, bar_index, false, false, false, int(na), false))
        if mmBearFVG
            array.push(pmMMFVGs, MMStruct.new(2, 1, -1, low[2], high[0], bar_index - 2, bar_index, false, false, false, int(na), false))
            [pmBearOBFound, pmBearOBHigh, pmBearOBLow, pmBearOBStart] = findBearOB(mmOBRunLookback)
            if pmBearOBFound
                array.push(pmMMOBs, MMStruct.new(2, 2, -1, pmBearOBHigh, pmBearOBLow, pmBearOBStart, bar_index, false, false, false, int(na), false))

updateMMStructStates(amMMFVGs)
updateMMStructStates(amMMOBs)
updateMMStructStates(pmMMFVGs)
updateMMStructStates(pmMMOBs)

trimMMStructArray(amMMFVGs, mmMaxStoredStructures)
trimMMStructArray(amMMOBs, mmMaxStoredStructures)
trimMMStructArray(pmMMFVGs, mmMaxStoredStructures)
trimMMStructArray(pmMMOBs, mmMaxStoredStructures)

// ============================================================================
// 11:50 AM - 12:10 PM Macro FPI Block
// ============================================================================

groupPM1150Macro = "- - - - - - - - - 11:50 AM Macro For PM Bias - - - - - - - - -"
PM1150_showMacroBox = input.bool(true, "Show 11:50 Macro Box", group=groupPM1150Macro, display=display.none)
PM1150_showImbalance = input.bool(true, "Show 11:50 Macro FPI", group=groupPM1150Macro, display=display.none)
PM1150_extendMacroLines = input.bool(true, "Extend 11:50 Macro Lines to 3PM", group=groupPM1150Macro, display=display.none)
PM1150_extendImbalanceLines = input.bool(true, "Extend 11:50 FPI Lines to 3PM", group=groupPM1150Macro, display=display.none)
PM1150_macroLineStyle = input.string("solid", "11:50 Macro Line Style", options=["solid", "dashed", "dotted"], group=groupPM1150Macro, display=display.none)
PM1150_imbLineStyle = input.string("solid", "11:50 FPI Line Style", options=["solid", "dashed", "dotted"], group=groupPM1150Macro, display=display.none)

PM1150_macroHour = 11
PM1150_macroStartMinute = 50
PM1150_macroEndMinute = 10

var box[] PM1150_macroBoxes = array.new_box()
var line[] PM1150_macroHighLines = array.new_line()
var line[] PM1150_macroLowLines = array.new_line()
var line[] PM1150_macroMidLines = array.new_line()
var label[] PM1150_macroLabels = array.new_label()
var label[] PM1150_macroPriceLabels = array.new_label()

var box[] PM1150_imbBoxes = array.new_box()
var line[] PM1150_imbHighLines = array.new_line()
var line[] PM1150_imbLowLines = array.new_line()
var line[] PM1150_imbMidLines = array.new_line()
var label[] PM1150_imbLabels = array.new_label()
var label[] PM1150_imbPriceLabels = array.new_label()

var float PM1150_winHigh = na
var float PM1150_winLow = na
var int PM1150_winStart = na
var int PM1150_winEnd = na
var bool PM1150_winDrawn = false

var bool PM1150_imbFound = false
var int PM1150_imbDir = 0
var float PM1150_imbHigh = na
var float PM1150_imbLow = na
var float PM1150_imbMid = na
var int PM1150_imbStart = na

getCustomPM1150MacroTimes() =>
    currentYear = year(time)
    currentMonth = month(time)
    currentDay = dayofmonth(time)
    endHour = PM1150_macroEndMinute < PM1150_macroStartMinute ? PM1150_macroHour + 1 : PM1150_macroHour
    startTime = timestamp(timezone, currentYear, currentMonth, currentDay, PM1150_macroHour, PM1150_macroStartMinute)
    endTime = timestamp(timezone, currentYear, currentMonth, currentDay, endHour, PM1150_macroEndMinute)
    [startTime, endTime]

isTimeInPM1150MacroWindow(t) =>
    _year = year(t, timezone)
    _month = month(t, timezone)
    _day = dayofmonth(t, timezone)
    _endHour = PM1150_macroEndMinute < PM1150_macroStartMinute ? PM1150_macroHour + 1 : PM1150_macroHour
    _startTime = timestamp(timezone, _year, _month, _day, PM1150_macroHour, PM1150_macroStartMinute)
    _endTime = timestamp(timezone, _year, _month, _day, _endHour, PM1150_macroEndMinute)
    t >= _startTime and t <= _endTime

getPM1150_ExtensionTime() =>
    currentYear = year(time)
    currentMonth = month(time)
    currentDay = dayofmonth(time)
    extTime = timestamp(timezone, currentYear, currentMonth, currentDay, 15, 0)
    if extTime <= time
        nextDay = time + 86400000
        extTime := timestamp(timezone, year(nextDay), month(nextDay), dayofmonth(nextDay), 15, 0)
    extTime

[PM1150_t_start, PM1150_t_end] = getCustomPM1150MacroTimes()
PM1150_is_1m = timeframe.period == "1"
inPM1150MacroWindow = time >= PM1150_t_start and time <= PM1150_t_end

if pmUse1150Macro and inPM1150MacroWindow and showPMNYMarketMaker and PM1150_showMacroBox and PM1150_is_1m
    PM1150_winHigh := na(PM1150_winHigh) ? high : math.max(PM1150_winHigh, high)
    PM1150_winLow := na(PM1150_winLow) ? low : math.min(PM1150_winLow, low)
    PM1150_winStart := na(PM1150_winStart) ? bar_index : PM1150_winStart
    PM1150_winEnd := bar_index
    PM1150_winDrawn := false

PM1150_bullFpi = high[2] < low[0]
PM1150_bearFpi = low[2] > high[0]

if pmUse1150Macro and not PM1150_imbFound and showPMNYMarketMaker and PM1150_showImbalance and PM1150_is_1m and isTimeInPM1150MacroWindow(time[1])
    if PM1150_bullFpi or PM1150_bearFpi
        PM1150_imbFound := true
        PM1150_imbDir := PM1150_bullFpi ? 1 : -1
        PM1150_imbHigh := PM1150_bullFpi ? low[0] : low[2]
        PM1150_imbLow := PM1150_bullFpi ? high[2] : high[0]
        PM1150_imbMid := (PM1150_imbHigh + PM1150_imbLow) / 2
        PM1150_imbStart := bar_index - 2

if pmUse1150Macro and time > PM1150_t_end and not PM1150_winDrawn and PM1150_is_1m and showPMNYMarketMaker and PM1150_showMacroBox
    if not na(PM1150_winStart) and not na(PM1150_winHigh) and not na(PM1150_winLow)
        macroBox = box.new(left=PM1150_winStart, top=PM1150_winHigh, right=PM1150_winEnd, bottom=PM1150_winLow, border_color=CM_macroLineColor, bgcolor=CM_macroBoxColor)
        array.push(PM1150_macroBoxes, macroBox)

        windowLabel = "11:50 AM - 12:10 PM Macro"
        macroLabel = label.new(x=PM1150_winStart, y=PM1150_winHigh, text=windowLabel, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_label_down, textcolor=CM_macroTextColor, size=biasTableTextSize(), color=CM_macroLabelBgColor)
        array.push(PM1150_macroLabels, macroLabel)

        if PM1150_extendMacroLines
            extensionTargetTime = getPM1150_ExtensionTime()
            extensionBars = bar_index + int(math.round((extensionTargetTime - time) / (timeframe.in_seconds() * 1000)))
            extensionBars := math.min(extensionBars, bar_index + 500)
            ls = CM_getLineStyle(PM1150_macroLineStyle)
            macroName = "11:50 Macro"
            array.push(PM1150_macroHighLines, line.new(x1=PM1150_winEnd, y1=PM1150_winHigh, x2=extensionBars, y2=PM1150_winHigh, color=CM_macroLineColor, style=ls))
            array.push(PM1150_macroLowLines, line.new(x1=PM1150_winEnd, y1=PM1150_winLow, x2=extensionBars, y2=PM1150_winLow, color=CM_macroLineColor, style=ls))
            array.push(PM1150_macroMidLines, line.new(x1=PM1150_winEnd, y1=(PM1150_winHigh + PM1150_winLow) / 2, x2=extensionBars, y2=(PM1150_winHigh + PM1150_winLow) / 2, color=CM_macroLineColor, style=line.style_dashed))
            array.push(PM1150_macroPriceLabels, label.new(x=extensionBars, y=PM1150_winHigh, text=macroName + " | High | " + str.tostring(PM1150_winHigh, format.mintick), xloc=xloc.bar_index, style=label.style_label_left, textcolor=CM_macroTextColor, size=biasTableTextSize(), color=CM_macroLabelBgColor))
            array.push(PM1150_macroPriceLabels, label.new(x=extensionBars, y=(PM1150_winHigh + PM1150_winLow) / 2, text=macroName + " | Mid | " + str.tostring((PM1150_winHigh + PM1150_winLow) / 2, format.mintick), xloc=xloc.bar_index, style=label.style_label_left, textcolor=CM_macroTextColor, size=biasTableTextSize(), color=CM_macroLabelBgColor))
            array.push(PM1150_macroPriceLabels, label.new(x=extensionBars, y=PM1150_winLow, text=macroName + " | Low | " + str.tostring(PM1150_winLow, format.mintick), xloc=xloc.bar_index, style=label.style_label_left, textcolor=CM_macroTextColor, size=biasTableTextSize(), color=CM_macroLabelBgColor))

    if PM1150_imbFound and not na(PM1150_imbStart)
        imbBox = box.new(left=PM1150_imbStart, top=PM1150_imbHigh, right=PM1150_imbStart + 2, bottom=PM1150_imbLow, border_color=CM_imbLineColor, bgcolor=CM_imbBoxColor)
        array.push(PM1150_imbBoxes, imbBox)

        imbLabel = label.new(x=PM1150_imbStart, y=PM1150_imbHigh, text="11:50 Macro FPI", xloc=xloc.bar_index, style=label.style_label_down, textcolor=CM_imbTextColor, size=biasTableTextSize(), color=CM_imbLabelBgColor)
        array.push(PM1150_imbLabels, imbLabel)

        if PM1150_extendImbalanceLines
            extensionTargetTime = getPM1150_ExtensionTime()
            extensionBars = bar_index + int(math.round((extensionTargetTime - time) / (timeframe.in_seconds() * 1000)))
            extensionBars := math.min(extensionBars, bar_index + 500)
            ls = CM_getLineStyle(PM1150_imbLineStyle)
            imbName = "11:50 Macro FPI"
            array.push(PM1150_imbHighLines, line.new(x1=PM1150_imbStart + 2, y1=PM1150_imbHigh, x2=extensionBars, y2=PM1150_imbHigh, color=CM_imbLineColor, style=ls))
            array.push(PM1150_imbMidLines, line.new(x1=PM1150_imbStart + 2, y1=PM1150_imbMid, x2=extensionBars, y2=PM1150_imbMid, color=CM_imbLineColor, style=line.style_dashed))
            array.push(PM1150_imbLowLines, line.new(x1=PM1150_imbStart + 2, y1=PM1150_imbLow, x2=extensionBars, y2=PM1150_imbLow, color=CM_imbLineColor, style=ls))
            array.push(PM1150_imbPriceLabels, label.new(x=extensionBars, y=PM1150_imbHigh, text=imbName + " | High | " + str.tostring(PM1150_imbHigh, format.mintick), xloc=xloc.bar_index, style=label.style_label_left, textcolor=CM_imbTextColor, size=biasTableTextSize(), color=CM_imbLabelBgColor))
            array.push(PM1150_imbPriceLabels, label.new(x=extensionBars, y=PM1150_imbMid, text=imbName + " | Mid | " + str.tostring(PM1150_imbMid, format.mintick), xloc=xloc.bar_index, style=label.style_label_left, textcolor=CM_imbTextColor, size=biasTableTextSize(), color=CM_imbLabelBgColor))
            array.push(PM1150_imbPriceLabels, label.new(x=extensionBars, y=PM1150_imbLow, text=imbName + " | Low | " + str.tostring(PM1150_imbLow, format.mintick), xloc=xloc.bar_index, style=label.style_label_left, textcolor=CM_imbTextColor, size=biasTableTextSize(), color=CM_imbLabelBgColor))

    PM1150_winDrawn := true

if biasResetNow
    PM1150_winHigh := na
    PM1150_winLow := na
    PM1150_winStart := na
    PM1150_winEnd := na
    PM1150_winDrawn := false
    PM1150_imbFound := false
    PM1150_imbDir := 0
    PM1150_imbHigh := na
    PM1150_imbLow := na
    PM1150_imbMid := na
    PM1150_imbStart := na

manage_PM1150_objects() =>
    while array.size(PM1150_macroBoxes) > globalLookbackPeriod
        box.delete(array.shift(PM1150_macroBoxes))
    while array.size(PM1150_macroHighLines) > globalLookbackPeriod
        line.delete(array.shift(PM1150_macroHighLines))
    while array.size(PM1150_macroLowLines) > globalLookbackPeriod
        line.delete(array.shift(PM1150_macroLowLines))
    while array.size(PM1150_macroMidLines) > globalLookbackPeriod
        line.delete(array.shift(PM1150_macroMidLines))
    while array.size(PM1150_macroLabels) > globalLookbackPeriod
        label.delete(array.shift(PM1150_macroLabels))
    while array.size(PM1150_macroPriceLabels) > globalLookbackPeriod * 3
        label.delete(array.shift(PM1150_macroPriceLabels))
    while array.size(PM1150_imbBoxes) > globalLookbackPeriod
        box.delete(array.shift(PM1150_imbBoxes))
    while array.size(PM1150_imbHighLines) > globalLookbackPeriod
        line.delete(array.shift(PM1150_imbHighLines))
    while array.size(PM1150_imbLowLines) > globalLookbackPeriod
        line.delete(array.shift(PM1150_imbLowLines))
    while array.size(PM1150_imbMidLines) > globalLookbackPeriod
        line.delete(array.shift(PM1150_imbMidLines))
    while array.size(PM1150_imbLabels) > globalLookbackPeriod
        label.delete(array.shift(PM1150_imbLabels))
    while array.size(PM1150_imbPriceLabels) > globalLookbackPeriod * 3
        label.delete(array.shift(PM1150_imbPriceLabels))

manage_PM1150_objects()

// ============================================================================
// 12:50 PM Macro FPI Detection
// ============================================================================

PM1250_bullFpi = high[2] < low[0]
PM1250_bearFpi = low[2] > high[0]

if not pmUse1150Macro and not PM1250_biasFound and showPMNYMarketMaker and timeframe.period == "1" and isTimeInPM1250MacroWindow(time[1])
    if PM1250_bullFpi or PM1250_bearFpi
        PM1250_biasFound := true
        PM1250_biasDir := PM1250_bullFpi ? 1 : -1
        PM1250_biasHigh := PM1250_bullFpi ? low[0] : low[2]
        PM1250_biasLow := PM1250_bullFpi ? high[2] : high[0]
        PM1250_biasMid := (PM1250_biasHigh + PM1250_biasLow) / 2

// ============================================================================
// PM FPI Session Detection
// ============================================================================

pmBiasBullFpi = close[1] > open[1] and high[2] < low[0]
pmBiasBearFpi = close[1] < open[1] and low[2] > high[0]

if not pm_bias1200Found and showPMNYMarketMaker and timeframe.period == "1" and isTimeInPMBiasFpiSession(time[1])
    if pmBiasBullFpi or pmBiasBearFpi
        pm_bias1200Found := true
        pm_bias1200Dir := pmBiasBullFpi ? 1 : -1
        pm_bias1200High := pmBiasBullFpi ? low[0] : low[2]
        pm_bias1200Low := pmBiasBullFpi ? high[2] : high[0]
        pm_bias1200Mid := (pm_bias1200High + pm_bias1200Low) / 2

// ============================================================================
// PM Macro Selection
// ============================================================================

pmSelectedMacroName = pmUse1150Macro ? "11:50 Macro" : "12:50 Macro"

pm_bias1250Found = pmUse1150Macro ? PM1150_imbFound : PM1250_biasFound
pm_bias1250Dir = pmUse1150Macro ? PM1150_imbDir : PM1250_biasDir
pm_bias1250High = pmUse1150Macro ? PM1150_imbHigh : PM1250_biasHigh
pm_bias1250Low = pmUse1150Macro ? PM1150_imbLow : PM1250_biasLow
pm_bias1250Mid = pmUse1150Macro ? PM1150_imbMid : PM1250_biasMid

pmSelectedMacroRangeHigh = pmUse1150Macro ? PM1150_winHigh : PM1250_winHigh
pmSelectedMacroRangeLow = pmUse1150Macro ? PM1150_winLow : PM1250_winLow

// ============================================================================
// PM Bias Confirmation + Complete Condition Mapping
// ============================================================================

pmBullFullConfirm = pm_bias1200Found and pm_bias1250Found and close > pm_bias1200High and close > pm_bias1250High
pmBearFullConfirm = pm_bias1200Found and pm_bias1250Found and close < pm_bias1200Low and close < pm_bias1250Low

pmBullContinuationConfirmed = pmBullFullConfirm and pm_bias1200Dir == 1
pmBearContinuationConfirmed = pmBearFullConfirm and pm_bias1200Dir == -1
pmBullReversalConfirmed = pmBullFullConfirm and pm_bias1200Dir == -1
pmBearReversalConfirmed = pmBearFullConfirm and pm_bias1200Dir == 1

pm_biasBullConfirmed = pmBullFullConfirm
pm_biasBearConfirmed = pmBearFullConfirm
pm_biasLowProbability = pm_bias1200Found and pm_bias1250Found and not pm_biasBullConfirmed and not pm_biasBearConfirmed

// ============================================================================
// Judas Reversal Arming
// ============================================================================

if enableMMStructureEngine and bias930Found
    if bias930Dir == -1 and breaksAboveLevel(bias930High)
        amBullJudasArmed := true
    if bias930Dir == 1 and breaksBelowLevel(bias930Low)
        amBearJudasArmed := true

if enableMMStructureEngine and pm_bias1200Found
    if pm_bias1200Dir == -1 and breaksAboveLevel(pm_bias1200High)
        pmBullJudasArmed := true
    if pm_bias1200Dir == 1 and breaksBelowLevel(pm_bias1200Low)
        pmBearJudasArmed := true

pmPriceVs1159 = "Waiting"
if not na(close1159)
    if close > close1159
        pmPriceVs1159 := "Above 11:59"
    else if close < close1159
        pmPriceVs1159 := "Below 11:59"
    else
        pmPriceVs1159 := "At 11:59"

pmPriceVsMacro = "Waiting"
if pm_bias1250Found
    if close > pm_bias1250High
        pmPriceVsMacro := "Above PM Macro FPI"
    else if close < pm_bias1250Low
        pmPriceVsMacro := "Below PM Macro FPI"
    else
        pmPriceVsMacro := "Inside PM Macro FPI"

pmPriceInsideMacro = pm_bias1250Found and close <= pm_bias1250High and close >= pm_bias1250Low

// ============================================================================
// Selected Respected Structures
// ============================================================================

[amBullOBFound, amBullOBHigh, amBullOBLow, amBullOBStart, amBullOBRespectBar, amBullOBUsed, amBullOBIdx] = selectMMStructure(amMMOBs, 1, bias930Dir, true)
[amBearOBFound, amBearOBHigh, amBearOBLow, amBearOBStart, amBearOBRespectBar, amBearOBUsed, amBearOBIdx] = selectMMStructure(amMMOBs, -1, bias930Dir, true)
[amBullFVGFound, amBullFVGHigh, amBullFVGLow, amBullFVGStart, amBullFVGRespectBar, amBullFVGUsed, amBullFVGIdx] = selectMMStructure(amMMFVGs, 1, bias930Dir, true)
[amBearFVGFound, amBearFVGHigh, amBearFVGLow, amBearFVGStart, amBearFVGRespectBar, amBearFVGUsed, amBearFVGIdx] = selectMMStructure(amMMFVGs, -1, bias930Dir, true)

[amBullRevOBFound, amBullRevOBHigh, amBullRevOBLow, amBullRevOBStart, amBullRevOBRespectBar, amBullRevOBUsed, amBullRevOBIdx] = selectMMStructure(amMMOBs, 1, 1, true)
[amBearRevOBFound, amBearRevOBHigh, amBearRevOBLow, amBearRevOBStart, amBearRevOBRespectBar, amBearRevOBUsed, amBearRevOBIdx] = selectMMStructure(amMMOBs, -1, -1, true)
[amBullRevFVGFound, amBullRevFVGHigh, amBullRevFVGLow, amBullRevFVGStart, amBullRevFVGRespectBar, amBullRevFVGUsed, amBullRevFVGIdx] = selectMMStructure(amMMFVGs, 1, 1, true)
[amBearRevFVGFound, amBearRevFVGHigh, amBearRevFVGLow, amBearRevFVGStart, amBearRevFVGRespectBar, amBearRevFVGUsed, amBearRevFVGIdx] = selectMMStructure(amMMFVGs, -1, -1, true)

[pmBullOBFound, pmBullOBHigh, pmBullOBLow, pmBullOBStart, pmBullOBRespectBar, pmBullOBUsed, pmBullOBIdx] = selectMMStructure(pmMMOBs, 1, pm_bias1200Dir, true)
[pmBearOBFound, pmBearOBHigh, pmBearOBLow, pmBearOBStart, pmBearOBRespectBar, pmBearOBUsed, pmBearOBIdx] = selectMMStructure(pmMMOBs, -1, pm_bias1200Dir, true)
[pmBullFVGFound, pmBullFVGHigh, pmBullFVGLow, pmBullFVGStart, pmBullFVGRespectBar, pmBullFVGUsed, pmBullFVGIdx] = selectMMStructure(pmMMFVGs, 1, pm_bias1200Dir, true)
[pmBearFVGFound, pmBearFVGHigh, pmBearFVGLow, pmBearFVGStart, pmBearFVGRespectBar, pmBearFVGUsed, pmBearFVGIdx] = selectMMStructure(pmMMFVGs, -1, pm_bias1200Dir, true)

[pmBullRevOBFound, pmBullRevOBHigh, pmBullRevOBLow, pmBullRevOBStart, pmBullRevOBRespectBar, pmBullRevOBUsed, pmBullRevOBIdx] = selectMMStructure(pmMMOBs, 1, 1, true)
[pmBearRevOBFound, pmBearRevOBHigh, pmBearRevOBLow, pmBearRevOBStart, pmBearRevOBRespectBar, pmBearRevOBUsed, pmBearRevOBIdx] = selectMMStructure(pmMMOBs, -1, -1, true)
[pmBullRevFVGFound, pmBullRevFVGHigh, pmBullRevFVGLow, pmBullRevFVGStart, pmBullRevFVGRespectBar, pmBullRevFVGUsed, pmBullRevFVGIdx] = selectMMStructure(pmMMFVGs, 1, 1, true)
[pmBearRevFVGFound, pmBearRevFVGHigh, pmBearRevFVGLow, pmBearRevFVGStart, pmBearRevFVGRespectBar, pmBearRevFVGUsed, pmBearRevFVGIdx] = selectMMStructure(pmMMFVGs, -1, -1, true)

// ============================================================================
// Entry Structure Selection
// ============================================================================

amUseBullExtreme = bias930Found and bias930Dir == -1 and enableCounterBiasStructureEntries and (amBullJudasArmed or allowExtremeCounterBiasBeforeFpiBreak)
amUseBearExtreme = bias930Found and bias930Dir == 1 and enableCounterBiasStructureEntries and (amBearJudasArmed or allowExtremeCounterBiasBeforeFpiBreak)

pmUseBullExtreme = pm_bias1200Found and pm_bias1200Dir == -1 and enableCounterBiasStructureEntries and (pmBullJudasArmed or allowExtremeCounterBiasBeforeFpiBreak)
pmUseBearExtreme = pm_bias1200Found and pm_bias1200Dir == 1 and enableCounterBiasStructureEntries and (pmBearJudasArmed or allowExtremeCounterBiasBeforeFpiBreak)

amBullOBEntryFound = amUseBullExtreme ? amBullRevOBFound : amBullOBFound
amBullOBEntryHigh = amUseBullExtreme ? amBullRevOBHigh : amBullOBHigh
amBullOBEntryLow = amUseBullExtreme ? amBullRevOBLow : amBullOBLow
amBullOBEntryUsed = amUseBullExtreme ? amBullRevOBUsed : amBullOBUsed
amBullOBEntryIdx = amUseBullExtreme ? amBullRevOBIdx : amBullOBIdx

amBearOBEntryFound = amUseBearExtreme ? amBearRevOBFound : amBearOBFound
amBearOBEntryHigh = amUseBearExtreme ? amBearRevOBHigh : amBearOBHigh
amBearOBEntryLow = amUseBearExtreme ? amBearRevOBLow : amBearOBLow
amBearOBEntryUsed = amUseBearExtreme ? amBearRevOBUsed : amBearOBUsed
amBearOBEntryIdx = amUseBearExtreme ? amBearRevOBIdx : amBearOBIdx

amBullFVGEntryFound = amUseBullExtreme ? amBullRevFVGFound : amBullFVGFound
amBullFVGEntryHigh = amUseBullExtreme ? amBullRevFVGHigh : amBullFVGHigh
amBullFVGEntryLow = amUseBullExtreme ? amBullRevFVGLow : amBullFVGLow
amBullFVGEntryUsed = amUseBullExtreme ? amBullRevFVGUsed : amBullFVGUsed
amBullFVGEntryIdx = amUseBullExtreme ? amBullRevFVGIdx : amBullFVGIdx

amBearFVGEntryFound = amUseBearExtreme ? amBearRevFVGFound : amBearFVGFound
amBearFVGEntryHigh = amUseBearExtreme ? amBearRevFVGHigh : amBearFVGHigh
amBearFVGEntryLow = amUseBearExtreme ? amBearRevFVGLow : amBearFVGLow
amBearFVGEntryUsed = amUseBearExtreme ? amBearRevFVGUsed : amBearFVGUsed
amBearFVGEntryIdx = amUseBearExtreme ? amBearRevFVGIdx : amBearFVGIdx

pmBullOBEntryFound = pmUseBullExtreme ? pmBullRevOBFound : pmBullOBFound
pmBullOBEntryHigh = pmUseBullExtreme ? pmBullRevOBHigh : pmBullOBHigh
pmBullOBEntryLow = pmUseBullExtreme ? pmBullRevOBLow : pmBullOBLow
pmBullOBEntryUsed = pmUseBullExtreme ? pmBullRevOBUsed : pmBullOBUsed
pmBullOBEntryIdx = pmUseBullExtreme ? pmBullRevOBIdx : pmBullOBIdx

pmBearOBEntryFound = pmUseBearExtreme ? pmBearRevOBFound : pmBearOBFound
pmBearOBEntryHigh = pmUseBearExtreme ? pmBearRevOBHigh : pmBearOBHigh
pmBearOBEntryLow = pmUseBearExtreme ? pmBearRevOBLow : pmBearOBLow
pmBearOBEntryUsed = pmUseBearExtreme ? pmBearRevOBUsed : pmBearOBUsed
pmBearOBEntryIdx = pmUseBearExtreme ? pmBearRevOBIdx : pmBearOBIdx

pmBullFVGEntryFound = pmUseBullExtreme ? pmBullRevFVGFound : pmBullFVGFound
pmBullFVGEntryHigh = pmUseBullExtreme ? pmBullRevFVGHigh : pmBullFVGHigh
pmBullFVGEntryLow = pmUseBullExtreme ? pmBullRevFVGLow : pmBullFVGLow
pmBullFVGEntryUsed = pmUseBullExtreme ? pmBullRevFVGUsed : pmBullFVGUsed
pmBullFVGEntryIdx = pmUseBullExtreme ? pmBullRevFVGIdx : pmBullFVGIdx

pmBearFVGEntryFound = pmUseBearExtreme ? pmBearRevFVGFound : pmBearFVGFound
pmBearFVGEntryHigh = pmUseBearExtreme ? pmBearRevFVGHigh : pmBearFVGHigh
pmBearFVGEntryLow = pmUseBearExtreme ? pmBearRevFVGLow : pmBearFVGLow
pmBearFVGEntryUsed = pmUseBearExtreme ? pmBearRevFVGUsed : pmBearFVGUsed
pmBearFVGEntryIdx = pmUseBearExtreme ? pmBearRevFVGIdx : pmBearFVGIdx

// ============================================================================
// Live Extreme Structure Override
// ============================================================================

[amBullOBLiveFound, amBullOBLiveHigh, amBullOBLiveLow, amBullOBLiveStart, amBullOBLiveIdx] = scanLiveExtremeStructure(amMMOBs, 1, requireStructureNearMacroExtreme, amMacroRangeLow, bias950Low, bias930Low, amSessionLow, londonSessionLow, nyPreSessionLow)
[amBearOBLiveFound, amBearOBLiveHigh, amBearOBLiveLow, amBearOBLiveStart, amBearOBLiveIdx] = scanLiveExtremeStructure(amMMOBs, -1, requireStructureNearMacroExtreme, amMacroRangeHigh, bias950High, bias930High, amSessionHigh, londonSessionHigh, nyPreSessionHigh)
[amBullFVGLiveFound, amBullFVGLiveHigh, amBullFVGLiveLow, amBullFVGLiveStart, amBullFVGLiveIdx] = scanLiveExtremeStructure(amMMFVGs, 1, requireStructureNearMacroExtreme, amMacroRangeLow, bias950Low, bias930Low, amSessionLow, londonSessionLow, nyPreSessionLow)
[amBearFVGLiveFound, amBearFVGLiveHigh, amBearFVGLiveLow, amBearFVGLiveStart, amBearFVGLiveIdx] = scanLiveExtremeStructure(amMMFVGs, -1, requireStructureNearMacroExtreme, amMacroRangeHigh, bias950High, bias930High, amSessionHigh, londonSessionHigh, nyPreSessionHigh)

[pmBullOBLiveFound, pmBullOBLiveHigh, pmBullOBLiveLow, pmBullOBLiveStart, pmBullOBLiveIdx] = scanLiveExtremeStructure(pmMMOBs, 1, requireStructureNearMacroExtreme, pmSelectedMacroRangeLow, pm_bias1250Low, pm_bias1200Low, pmSessionLow, close1159, asianSessionLow)
[pmBearOBLiveFound, pmBearOBLiveHigh, pmBearOBLiveLow, pmBearOBLiveStart, pmBearOBLiveIdx] = scanLiveExtremeStructure(pmMMOBs, -1, requireStructureNearMacroExtreme, pmSelectedMacroRangeHigh, pm_bias1250High, pm_bias1200High, pmSessionHigh, close1159, asianSessionHigh)
[pmBullFVGLiveFound, pmBullFVGLiveHigh, pmBullFVGLiveLow, pmBullFVGLiveStart, pmBullFVGLiveIdx] = scanLiveExtremeStructure(pmMMFVGs, 1, requireStructureNearMacroExtreme, pmSelectedMacroRangeLow, pm_bias1250Low, pm_bias1200Low, pmSessionLow, close1159, asianSessionLow)
[pmBearFVGLiveFound, pmBearFVGLiveHigh, pmBearFVGLiveLow, pmBearFVGLiveStart, pmBearFVGLiveIdx] = scanLiveExtremeStructure(pmMMFVGs, -1, requireStructureNearMacroExtreme, pmSelectedMacroRangeHigh, pm_bias1250High, pm_bias1200High, pmSessionHigh, close1159, asianSessionHigh)

if amBullOBLiveFound
    amBullOBEntryFound := true
    amBullOBEntryHigh := amBullOBLiveHigh
    amBullOBEntryLow := amBullOBLiveLow
    amBullOBEntryUsed := false
    amBullOBEntryIdx := amBullOBLiveIdx
if amBearOBLiveFound
    amBearOBEntryFound := true
    amBearOBEntryHigh := amBearOBLiveHigh
    amBearOBEntryLow := amBearOBLiveLow
    amBearOBEntryUsed := false
    amBearOBEntryIdx := amBearOBLiveIdx
if amBullFVGLiveFound
    amBullFVGEntryFound := true
    amBullFVGEntryHigh := amBullFVGLiveHigh
    amBullFVGEntryLow := amBullFVGLiveLow
    amBullFVGEntryUsed := false
    amBullFVGEntryIdx := amBullFVGLiveIdx
if amBearFVGLiveFound
    amBearFVGEntryFound := true
    amBearFVGEntryHigh := amBearFVGLiveHigh
    amBearFVGEntryLow := amBearFVGLiveLow
    amBearFVGEntryUsed := false
    amBearFVGEntryIdx := amBearFVGLiveIdx

if pmBullOBLiveFound
    pmBullOBEntryFound := true
    pmBullOBEntryHigh := pmBullOBLiveHigh
    pmBullOBEntryLow := pmBullOBLiveLow
    pmBullOBEntryUsed := false
    pmBullOBEntryIdx := pmBullOBLiveIdx
if pmBearOBLiveFound
    pmBearOBEntryFound := true
    pmBearOBEntryHigh := pmBearOBLiveHigh
    pmBearOBEntryLow := pmBearOBLiveLow
    pmBearOBEntryUsed := false
    pmBearOBEntryIdx := pmBearOBLiveIdx
if pmBullFVGLiveFound
    pmBullFVGEntryFound := true
    pmBullFVGEntryHigh := pmBullFVGLiveHigh
    pmBullFVGEntryLow := pmBullFVGLiveLow
    pmBullFVGEntryUsed := false
    pmBullFVGEntryIdx := pmBullFVGLiveIdx
if pmBearFVGLiveFound
    pmBearFVGEntryFound := true
    pmBearFVGEntryHigh := pmBearFVGLiveHigh
    pmBearFVGEntryLow := pmBearFVGLiveLow
    pmBearFVGEntryUsed := false
    pmBearFVGEntryIdx := pmBearFVGLiveIdx

// ============================================================================
// Active Range Extreme Engine
// ============================================================================

[amBullOBRangeFound, amBullOBRangeHigh, amBullOBRangeLow, amBullOBRangeStart, amBullOBRangeIdx] = scanSessionExtremeStructure(amMMOBs, 1, requireStructureNearMacroExtreme, amMacroRangeLow, bias950Low, bias930Low, amSessionLow, londonSessionLow, nyPreSessionLow)
[amBearOBRangeFound, amBearOBRangeHigh, amBearOBRangeLow, amBearOBRangeStart, amBearOBRangeIdx] = scanSessionExtremeStructure(amMMOBs, -1, requireStructureNearMacroExtreme, amMacroRangeHigh, bias950High, bias930High, amSessionHigh, londonSessionHigh, nyPreSessionHigh)
[amBullFVGRangeFound, amBullFVGRangeHigh, amBullFVGRangeLow, amBullFVGRangeStart, amBullFVGRangeIdx] = scanSessionExtremeStructure(amMMFVGs, 1, requireStructureNearMacroExtreme, amMacroRangeLow, bias950Low, bias930Low, amSessionLow, londonSessionLow, nyPreSessionLow)
[amBearFVGRangeFound, amBearFVGRangeHigh, amBearFVGRangeLow, amBearFVGRangeStart, amBearFVGRangeIdx] = scanSessionExtremeStructure(amMMFVGs, -1, requireStructureNearMacroExtreme, amMacroRangeHigh, bias950High, bias930High, amSessionHigh, londonSessionHigh, nyPreSessionHigh)

[pmBullOBRangeFound, pmBullOBRangeHigh, pmBullOBRangeLow, pmBullOBRangeStart, pmBullOBRangeIdx] = scanSessionExtremeStructure(pmMMOBs, 1, requireStructureNearMacroExtreme, pmSelectedMacroRangeLow, pm_bias1250Low, pm_bias1200Low, pmSessionLow, close1159, asianSessionLow)
[pmBearOBRangeFound, pmBearOBRangeHigh, pmBearOBRangeLow, pmBearOBRangeStart, pmBearOBRangeIdx] = scanSessionExtremeStructure(pmMMOBs, -1, requireStructureNearMacroExtreme, pmSelectedMacroRangeHigh, pm_bias1250High, pm_bias1200High, pmSessionHigh, close1159, asianSessionHigh)
[pmBullFVGRangeFound, pmBullFVGRangeHigh, pmBullFVGRangeLow, pmBullFVGRangeStart, pmBullFVGRangeIdx] = scanSessionExtremeStructure(pmMMFVGs, 1, requireStructureNearMacroExtreme, pmSelectedMacroRangeLow, pm_bias1250Low, pm_bias1200Low, pmSessionLow, close1159, asianSessionLow)
[pmBearFVGRangeFound, pmBearFVGRangeHigh, pmBearFVGRangeLow, pmBearFVGRangeStart, pmBearFVGRangeIdx] = scanSessionExtremeStructure(pmMMFVGs, -1, requireStructureNearMacroExtreme, pmSelectedMacroRangeHigh, pm_bias1250High, pm_bias1200High, pmSessionHigh, close1159, asianSessionHigh)

amStructRangeLow = min2(amBullOBRangeLow, amBullFVGRangeLow)
amStructRangeHigh = max2(amBearOBRangeHigh, amBearFVGRangeHigh)
pmStructRangeLow = min2(pmBullOBRangeLow, pmBullFVGRangeLow)
pmStructRangeHigh = max2(pmBearOBRangeHigh, pmBearFVGRangeHigh)

amActiveRangeLow = min2(min2(amMacroRangeLow, bias950Low), min2(min2(bias930Low, amStructRangeLow), min2(londonSessionLow, nyPreSessionLow)))
amActiveRangeHigh = max2(max2(amMacroRangeHigh, bias950High), max2(max2(bias930High, amStructRangeHigh), max2(londonSessionHigh, nyPreSessionHigh)))
pmActiveRangeLow = min2(min2(pmSelectedMacroRangeLow, pm_bias1250Low), min2(min2(pm_bias1200Low, pmStructRangeLow), min2(asianSessionLow, nyPreSessionLow)))
pmActiveRangeHigh = max2(max2(pmSelectedMacroRangeHigh, pm_bias1250High), max2(max2(pm_bias1200High, pmStructRangeHigh), max2(asianSessionHigh, nyPreSessionHigh)))

amActiveRangeReady = not na(amActiveRangeHigh) and not na(amActiveRangeLow) and amActiveRangeHigh > amActiveRangeLow
pmActiveRangeReady = not na(pmActiveRangeHigh) and not na(pmActiveRangeLow) and pmActiveRangeHigh > pmActiveRangeLow

amBullRangeZoneTouch = (amBullOBRangeFound and bullZoneRetestExit(amBullOBRangeHigh, amBullOBRangeLow)) or (amBullFVGRangeFound and bullZoneRetestExit(amBullFVGRangeHigh, amBullFVGRangeLow))
amBearRangeZoneTouch = (amBearOBRangeFound and bearZoneRetestExit(amBearOBRangeHigh, amBearOBRangeLow)) or (amBearFVGRangeFound and bearZoneRetestExit(amBearFVGRangeHigh, amBearFVGRangeLow))
pmBullRangeZoneTouch = (pmBullOBRangeFound and bullZoneRetestExit(pmBullOBRangeHigh, pmBullOBRangeLow)) or (pmBullFVGRangeFound and bullZoneRetestExit(pmBullFVGRangeHigh, pmBullFVGRangeLow))
pmBearRangeZoneTouch = (pmBearOBRangeFound and bearZoneRetestExit(pmBearOBRangeHigh, pmBearOBRangeLow)) or (pmBearFVGRangeFound and bearZoneRetestExit(pmBearFVGRangeHigh, pmBearFVGRangeLow))

amBullRangeTrigger = enableRangeExtremeEngine and amActiveRangeReady and (levelRejectsLow(amActiveRangeLow) or amBullRangeZoneTouch)
amBearRangeTrigger = enableRangeExtremeEngine and amActiveRangeReady and (levelRejectsHigh(amActiveRangeHigh) or amBearRangeZoneTouch)
pmBullRangeTrigger = enableRangeExtremeEngine and pmActiveRangeReady and (levelRejectsLow(pmActiveRangeLow) or pmBullRangeZoneTouch)
pmBearRangeTrigger = enableRangeExtremeEngine and pmActiveRangeReady and (levelRejectsHigh(pmActiveRangeHigh) or pmBearRangeZoneTouch)

amBullRangeSL = not na(amStructRangeLow) and amBullRangeZoneTouch ? amStructRangeLow - rangeExtremeStopBufferPoints : amActiveRangeLow - rangeExtremeStopBufferPoints
amBearRangeSL = not na(amStructRangeHigh) and amBearRangeZoneTouch ? amStructRangeHigh + rangeExtremeStopBufferPoints : amActiveRangeHigh + rangeExtremeStopBufferPoints
pmBullRangeSL = not na(pmStructRangeLow) and pmBullRangeZoneTouch ? pmStructRangeLow - rangeExtremeStopBufferPoints : pmActiveRangeLow - rangeExtremeStopBufferPoints
pmBearRangeSL = not na(pmStructRangeHigh) and pmBearRangeZoneTouch ? pmStructRangeHigh + rangeExtremeStopBufferPoints : pmActiveRangeHigh + rangeExtremeStopBufferPoints

amBullRangeName = amBullRangeZoneTouch ? "Lowest OB/Imb Range Low" : "Active Range Low"
amBearRangeName = amBearRangeZoneTouch ? "Highest OB/Imb Range High" : "Active Range High"
pmBullRangeName = pmBullRangeZoneTouch ? "Lowest OB/Imb Range Low" : "Active Range Low"
pmBearRangeName = pmBearRangeZoneTouch ? "Highest OB/Imb Range High" : "Active Range High"

// ============================================================================
// Selected Structure Zone Drawing
// ============================================================================

if barstate.islast
    if not na(amBullOBBox)
        box.delete(amBullOBBox)
        amBullOBBox := na
    if not na(amBearOBBox)
        box.delete(amBearOBBox)
        amBearOBBox := na
    if not na(amBullFVGBox)
        box.delete(amBullFVGBox)
        amBullFVGBox := na
    if not na(amBearFVGBox)
        box.delete(amBearFVGBox)
        amBearFVGBox := na
    if not na(pmBullOBBox)
        box.delete(pmBullOBBox)
        pmBullOBBox := na
    if not na(pmBearOBBox)
        box.delete(pmBearOBBox)
        pmBearOBBox := na
    if not na(pmBullFVGBox)
        box.delete(pmBullFVGBox)
        pmBullFVGBox := na
    if not na(pmBearFVGBox)
        box.delete(pmBearFVGBox)
        pmBearFVGBox := na

    if showMMSelectedZones
        if amBullOBEntryFound
            amBullOBBox := box.new(left=amUseBullExtreme ? amBullRevOBStart : amBullOBStart, top=amBullOBEntryHigh, right=bar_index, bottom=amBullOBEntryLow, bgcolor=bullOBBoxColor, border_color=mmStructureBorderColor)
        if amBearOBEntryFound
            amBearOBBox := box.new(left=amUseBearExtreme ? amBearRevOBStart : amBearOBStart, top=amBearOBEntryHigh, right=bar_index, bottom=amBearOBEntryLow, bgcolor=bearOBBoxColor, border_color=mmStructureBorderColor)
        if amBullFVGEntryFound
            amBullFVGBox := box.new(left=amUseBullExtreme ? amBullRevFVGStart : amBullFVGStart, top=amBullFVGEntryHigh, right=bar_index, bottom=amBullFVGEntryLow, bgcolor=bullImbBoxColor, border_color=mmStructureBorderColor)
        if amBearFVGEntryFound
            amBearFVGBox := box.new(left=amUseBearExtreme ? amBearRevFVGStart : amBearFVGStart, top=amBearFVGEntryHigh, right=bar_index, bottom=amBearFVGEntryLow, bgcolor=bearImbBoxColor, border_color=mmStructureBorderColor)

        if pmBullOBEntryFound
            pmBullOBBox := box.new(left=pmUseBullExtreme ? pmBullRevOBStart : pmBullOBStart, top=pmBullOBEntryHigh, right=bar_index, bottom=pmBullOBEntryLow, bgcolor=bullOBBoxColor, border_color=mmStructureBorderColor)
        if pmBearOBEntryFound
            pmBearOBBox := box.new(left=pmUseBearExtreme ? pmBearRevOBStart : pmBearOBStart, top=pmBearOBEntryHigh, right=bar_index, bottom=pmBearOBEntryLow, bgcolor=bearOBBoxColor, border_color=mmStructureBorderColor)
        if pmBullFVGEntryFound
            pmBullFVGBox := box.new(left=pmUseBullExtreme ? pmBullRevFVGStart : pmBullFVGStart, top=pmBullFVGEntryHigh, right=bar_index, bottom=pmBullFVGEntryLow, bgcolor=bullImbBoxColor, border_color=mmStructureBorderColor)
        if pmBearFVGEntryFound
            pmBearFVGBox := box.new(left=pmUseBearExtreme ? pmBearRevFVGStart : pmBearFVGStart, top=pmBearFVGEntryHigh, right=bar_index, bottom=pmBearFVGEntryLow, bgcolor=bearImbBoxColor, border_color=mmStructureBorderColor)

// ============================================================================
// Trade Engine
// ============================================================================

biasArrowATR = ta.atr(14)

amBullReversalMature = reversalMaturityAllows(1, 1)
amBearReversalMature = reversalMaturityAllows(-1, 1)
pmBullReversalMature = reversalMaturityAllows(1, 2)
pmBearReversalMature = reversalMaturityAllows(-1, 2)

amBullContinuationSetup = showBiasSignalArrows and amTradeWindow and barstate.isconfirmed and amBullContinuationConfirmed
amBearContinuationSetup = showBiasSignalArrows and amTradeWindow and barstate.isconfirmed and amBearContinuationConfirmed
amBullReversalSetup = showBiasSignalArrows and amTradeWindow and barstate.isconfirmed and amBullReversalConfirmed and (not applyFilterToMacroReversals or amBullReversalMature)
amBearReversalSetup = showBiasSignalArrows and amTradeWindow and barstate.isconfirmed and amBearReversalConfirmed and (not applyFilterToMacroReversals or amBearReversalMature)

pmBullContinuationSetup = showBiasSignalArrows and pmTradeWindow and barstate.isconfirmed and pmBullContinuationConfirmed
pmBearContinuationSetup = showBiasSignalArrows and pmTradeWindow and barstate.isconfirmed and pmBearContinuationConfirmed
pmBullReversalSetup = showBiasSignalArrows and pmTradeWindow and barstate.isconfirmed and pmBullReversalConfirmed and (not applyFilterToMacroReversals or pmBullReversalMature)
pmBearReversalSetup = showBiasSignalArrows and pmTradeWindow and barstate.isconfirmed and pmBearReversalConfirmed and (not applyFilterToMacroReversals or pmBearReversalMature)

// ============================================================================
// Structure Entry Conditions
// ============================================================================

amBullStructureAllowed = bias930Found and (bias930Dir == 1 or amUseBullExtreme)
amBearStructureAllowed = bias930Found and (bias930Dir == -1 or amUseBearExtreme)

pmBullStructureAllowed = pmTradeWindow and (not pm_bias1200Found or pm_bias1200Dir == 1 or pmUseBullExtreme)
pmBearStructureAllowed = pmTradeWindow and (not pm_bias1200Found or pm_bias1200Dir == -1 or pmUseBearExtreme)

amBullOBRetestExit = amBullOBEntryFound and not amBullOBEntryUsed and bullZoneRetestExit(amBullOBEntryHigh, amBullOBEntryLow)
amBearOBRetestExit = amBearOBEntryFound and not amBearOBEntryUsed and bearZoneRetestExit(amBearOBEntryHigh, amBearOBEntryLow)
amBullFVGRetestExit = amBullFVGEntryFound and not amBullFVGEntryUsed and bullZoneRetestExit(amBullFVGEntryHigh, amBullFVGEntryLow)
amBearFVGRetestExit = amBearFVGEntryFound and not amBearFVGEntryUsed and bearZoneRetestExit(amBearFVGEntryHigh, amBearFVGEntryLow)

pmBullOBRetestExit = pmBullOBEntryFound and not pmBullOBEntryUsed and bullZoneRetestExit(pmBullOBEntryHigh, pmBullOBEntryLow)
pmBearOBRetestExit = pmBearOBEntryFound and not pmBearOBEntryUsed and bearZoneRetestExit(pmBearOBEntryHigh, pmBearOBEntryLow)
pmBullFVGRetestExit = pmBullFVGEntryFound and not pmBullFVGEntryUsed and bullZoneRetestExit(pmBullFVGEntryHigh, pmBullFVGEntryLow)
pmBearFVGRetestExit = pmBearFVGEntryFound and not pmBearFVGEntryUsed and bearZoneRetestExit(pmBearFVGEntryHigh, pmBearFVGEntryLow)

amBullStructureMature = not applyFilterToCounterBiasStructures or not amUseBullExtreme or amBullReversalMature
amBearStructureMature = not applyFilterToCounterBiasStructures or not amUseBearExtreme or amBearReversalMature
pmBullStructureMature = not applyFilterToCounterBiasStructures or not pmUseBullExtreme or pmBullReversalMature
pmBearStructureMature = not applyFilterToCounterBiasStructures or not pmUseBearExtreme or pmBearReversalMature

amBullOBMacroOk = not requireStructureNearMacroExtreme or zoneTouchesAny6(amBullOBEntryHigh, amBullOBEntryLow, macroExtremeProximityPoints, amMacroRangeLow, bias950Low, bias930Low, amSessionLow, londonSessionLow, nyPreSessionLow)
amBullFVGMacroOk = not requireStructureNearMacroExtreme or zoneTouchesAny6(amBullFVGEntryHigh, amBullFVGEntryLow, macroExtremeProximityPoints, amMacroRangeLow, bias950Low, bias930Low, amSessionLow, londonSessionLow, nyPreSessionLow)
amBearOBMacroOk = not requireStructureNearMacroExtreme or zoneTouchesAny6(amBearOBEntryHigh, amBearOBEntryLow, macroExtremeProximityPoints, amMacroRangeHigh, bias950High, bias930High, amSessionHigh, londonSessionHigh, nyPreSessionHigh)
amBearFVGMacroOk = not requireStructureNearMacroExtreme or zoneTouchesAny6(amBearFVGEntryHigh, amBearFVGEntryLow, macroExtremeProximityPoints, amMacroRangeHigh, bias950High, bias930High, amSessionHigh, londonSessionHigh, nyPreSessionHigh)

pmBullOBMacroOk = not requireStructureNearMacroExtreme or zoneTouchesAny6(pmBullOBEntryHigh, pmBullOBEntryLow, macroExtremeProximityPoints, pmSelectedMacroRangeLow, pm_bias1250Low, pm_bias1200Low, pmSessionLow, close1159, asianSessionLow)
pmBullFVGMacroOk = not requireStructureNearMacroExtreme or zoneTouchesAny6(pmBullFVGEntryHigh, pmBullFVGEntryLow, macroExtremeProximityPoints, pmSelectedMacroRangeLow, pm_bias1250Low, pm_bias1200Low, pmSessionLow, close1159, asianSessionLow)
pmBearOBMacroOk = not requireStructureNearMacroExtreme or zoneTouchesAny6(pmBearOBEntryHigh, pmBearOBEntryLow, macroExtremeProximityPoints, pmSelectedMacroRangeHigh, pm_bias1250High, pm_bias1200High, pmSessionHigh, close1159, asianSessionHigh)
pmBearFVGMacroOk = not requireStructureNearMacroExtreme or zoneTouchesAny6(pmBearFVGEntryHigh, pmBearFVGEntryLow, macroExtremeProximityPoints, pmSelectedMacroRangeHigh, pm_bias1250High, pm_bias1200High, pmSessionHigh, close1159, asianSessionHigh)

amBullRangeExtremeSetup = showBiasSignalArrows and enableRangeExtremeEngine and amTradeWindow and barstate.isconfirmed and amBullRangeTrigger
amBearRangeExtremeSetup = showBiasSignalArrows and enableRangeExtremeEngine and amTradeWindow and barstate.isconfirmed and amBearRangeTrigger
pmBullRangeExtremeSetup = showBiasSignalArrows and enableRangeExtremeEngine and pmTradeWindow and barstate.isconfirmed and pmBullRangeTrigger
pmBearRangeExtremeSetup = showBiasSignalArrows and enableRangeExtremeEngine and pmTradeWindow and barstate.isconfirmed and pmBearRangeTrigger

amBullMacroTriggerLevel = bullLevelReject(amMacroRangeLow) ? amMacroRangeLow : bullLevelReject(bias950Low) ? bias950Low : bullLevelReject(bias930Low) ? bias930Low : bullLevelReject(amSessionLow) ? amSessionLow : bullLevelReject(londonSessionLow) ? londonSessionLow : bullLevelReject(nyPreSessionLow) ? nyPreSessionLow : na
amBearMacroTriggerLevel = bearLevelReject(amMacroRangeHigh) ? amMacroRangeHigh : bearLevelReject(bias950High) ? bias950High : bearLevelReject(bias930High) ? bias930High : bearLevelReject(amSessionHigh) ? amSessionHigh : bearLevelReject(londonSessionHigh) ? londonSessionHigh : bearLevelReject(nyPreSessionHigh) ? nyPreSessionHigh : na
pmBullMacroTriggerLevel = bullLevelReject(pmSelectedMacroRangeLow) ? pmSelectedMacroRangeLow : bullLevelReject(pm_bias1250Low) ? pm_bias1250Low : bullLevelReject(pm_bias1200Low) ? pm_bias1200Low : bullLevelReject(pmSessionLow) ? pmSessionLow : bullLevelReject(close1159) ? close1159 : bullLevelReject(asianSessionLow) ? asianSessionLow : bullLevelReject(nyPreSessionLow) ? nyPreSessionLow : na
pmBearMacroTriggerLevel = bearLevelReject(pmSelectedMacroRangeHigh) ? pmSelectedMacroRangeHigh : bearLevelReject(pm_bias1250High) ? pm_bias1250High : bearLevelReject(pm_bias1200High) ? pm_bias1200High : bearLevelReject(pmSessionHigh) ? pmSessionHigh : bearLevelReject(close1159) ? close1159 : bearLevelReject(asianSessionHigh) ? asianSessionHigh : bearLevelReject(nyPreSessionHigh) ? nyPreSessionHigh : na

amBullMacroTriggerName = bullLevelReject(amMacroRangeLow) ? "9:50 Macro Low" : bullLevelReject(bias950Low) ? "9:50 FPI Low" : bullLevelReject(bias930Low) ? "AM FPI Low" : bullLevelReject(amSessionLow) ? "AM Session Low" : bullLevelReject(londonSessionLow) ? "London Low" : bullLevelReject(nyPreSessionLow) ? "NY 7-9 Low" : "Macro Low"
amBearMacroTriggerName = bearLevelReject(amMacroRangeHigh) ? "9:50 Macro High" : bearLevelReject(bias950High) ? "9:50 FPI High" : bearLevelReject(bias930High) ? "AM FPI High" : bearLevelReject(amSessionHigh) ? "AM Session High" : bearLevelReject(londonSessionHigh) ? "London High" : bearLevelReject(nyPreSessionHigh) ? "NY 7-9 High" : "Macro High"
pmBullMacroTriggerName = bullLevelReject(pmSelectedMacroRangeLow) ? pmSelectedMacroName + " Low" : bullLevelReject(pm_bias1250Low) ? "PM Macro FPI Low" : bullLevelReject(pm_bias1200Low) ? "PM FPI Low" : bullLevelReject(pmSessionLow) ? "PM Session Low" : bullLevelReject(close1159) ? "11:59 Close" : bullLevelReject(asianSessionLow) ? "Asian Low" : bullLevelReject(nyPreSessionLow) ? "NY 7-9 Low" : "Macro Low"
pmBearMacroTriggerName = bearLevelReject(pmSelectedMacroRangeHigh) ? pmSelectedMacroName + " High" : bearLevelReject(pm_bias1250High) ? "PM Macro FPI High" : bearLevelReject(pm_bias1200High) ? "PM FPI High" : bearLevelReject(pmSessionHigh) ? "PM Session High" : bearLevelReject(close1159) ? "11:59 Close" : bearLevelReject(asianSessionHigh) ? "Asian High" : bearLevelReject(nyPreSessionHigh) ? "NY 7-9 High" : "Macro High"

amMacroWindowCompleted = not isTimeInMacroWindow(time) and not na(amMacroRangeHigh) and not na(amMacroRangeLow)
pmSelectedMacroWindowCompleted = pmUse1150Macro ? (not isTimeInPM1150MacroWindow(time) and not na(PM1150_winHigh) and not na(PM1150_winLow)) : (not isTimeInPM1250MacroWindow(time) and not na(PM1250_winHigh) and not na(PM1250_winLow))

amBullMacroExtremeSetup = showBiasSignalArrows and enableMacroExtremeEngine and amMacroWindowCompleted and amTradeWindow and barstate.isconfirmed and not amMacroLowUsed and not na(amBullMacroTriggerLevel) and (not applyMaturityFilterToMacroExtreme or amBullReversalMature)
amBearMacroExtremeSetup = showBiasSignalArrows and enableMacroExtremeEngine and amMacroWindowCompleted and amTradeWindow and barstate.isconfirmed and not amMacroHighUsed and not na(amBearMacroTriggerLevel) and (not applyMaturityFilterToMacroExtreme or amBearReversalMature)
pmBullMacroExtremeSetup = showBiasSignalArrows and enableMacroExtremeEngine and pmSelectedMacroWindowCompleted and pmTradeWindow and barstate.isconfirmed and not pmMacroLowUsed and not na(pmBullMacroTriggerLevel) and (not applyMaturityFilterToMacroExtreme or pmBullReversalMature)
pmBearMacroExtremeSetup = showBiasSignalArrows and enableMacroExtremeEngine and pmSelectedMacroWindowCompleted and pmTradeWindow and barstate.isconfirmed and not pmMacroHighUsed and not na(pmBearMacroTriggerLevel) and (not applyMaturityFilterToMacroExtreme or pmBearReversalMature)

amBullOBEntrySetup = showBiasSignalArrows and enableMMStructureEngine and enableMMOBEntries and amTradeWindow and barstate.isconfirmed and amBullStructureAllowed and amBullOBRetestExit and amBullStructureMature and amBullOBMacroOk
amBearOBEntrySetup = showBiasSignalArrows and enableMMStructureEngine and enableMMOBEntries and amTradeWindow and barstate.isconfirmed and amBearStructureAllowed and amBearOBRetestExit and amBearStructureMature and amBearOBMacroOk
amBullFVGEntrySetup = showBiasSignalArrows and enableMMStructureEngine and enableMMImbalanceEntries and amTradeWindow and barstate.isconfirmed and amBullStructureAllowed and amBullFVGRetestExit and amBullStructureMature and amBullFVGMacroOk
amBearFVGEntrySetup = showBiasSignalArrows and enableMMStructureEngine and enableMMImbalanceEntries and amTradeWindow and barstate.isconfirmed and amBearStructureAllowed and amBearFVGRetestExit and amBearStructureMature and amBearFVGMacroOk

pmBullOBEntrySetup = showBiasSignalArrows and enableMMStructureEngine and enableMMOBEntries and pmTradeWindow and barstate.isconfirmed and pmBullStructureAllowed and pmBullOBRetestExit and pmBullStructureMature and pmBullOBMacroOk
pmBearOBEntrySetup = showBiasSignalArrows and enableMMStructureEngine and enableMMOBEntries and pmTradeWindow and barstate.isconfirmed and pmBearStructureAllowed and pmBearOBRetestExit and pmBearStructureMature and pmBearOBMacroOk
pmBullFVGEntrySetup = showBiasSignalArrows and enableMMStructureEngine and enableMMImbalanceEntries and pmTradeWindow and barstate.isconfirmed and pmBullStructureAllowed and pmBullFVGRetestExit and pmBullStructureMature and pmBullFVGMacroOk
pmBearFVGEntrySetup = showBiasSignalArrows and enableMMStructureEngine and enableMMImbalanceEntries and pmTradeWindow and barstate.isconfirmed and pmBearStructureAllowed and pmBearFVGRetestExit and pmBearStructureMature and pmBearFVGMacroOk

reversalBlockedNow = showReversalFilterWarnings and enableReversalMaturityFilter and barstate.islast and ((amTradeWindow and ((amBullReversalConfirmed and not amBullReversalMature) or (amBearReversalConfirmed and not amBearReversalMature) or (amUseBullExtreme and not amBullReversalMature) or (amUseBearExtreme and not amBearReversalMature))) or (pmTradeWindow and ((pmBullReversalConfirmed and not pmBullReversalMature) or (pmBearReversalConfirmed and not pmBearReversalMature) or (pmUseBullExtreme and not pmBullReversalMature) or (pmUseBearExtreme and not pmBearReversalMature))))

if reversalBlockedNow
    if not na(reversalFilterWarningLabel)
        label.delete(reversalFilterWarningLabel)
    _sessionName = pmTradeWindow ? "PM" : "AM"
    _status = _sessionName + " reversal held: delivery not complete" + "\nAM: " + reversalFilterStatusText(1) + " | PM: " + reversalFilterStatusText(2)
    reversalFilterWarningLabel := label.new(x=bar_index + 8, y=high + biasArrowATR * 2.0, text=_status, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_label_down, textcolor=color.yellow, size=size.small, color=color.new(color.black, 20))
else if barstate.islast and not na(reversalFilterWarningLabel)
    label.delete(reversalFilterWarningLabel)
    reversalFilterWarningLabel := na

amBullExitSignal = showBiasSignalArrows and timeframe.period == "1" and barstate.isconfirmed and biasInPosition and biasActiveSession == 1 and biasActiveDir == 1 and not na(biasEntrySL) and close < biasEntrySL
amBearExitSignal = showBiasSignalArrows and timeframe.period == "1" and barstate.isconfirmed and biasInPosition and biasActiveSession == 1 and biasActiveDir == -1 and not na(biasEntrySL) and close > biasEntrySL

pmBullExitSignal = showBiasSignalArrows and timeframe.period == "1" and barstate.isconfirmed and biasInPosition and biasActiveSession == 2 and biasActiveDir == 1 and not na(biasEntrySL) and close < biasEntrySL
pmBearExitSignal = showBiasSignalArrows and timeframe.period == "1" and barstate.isconfirmed and biasInPosition and biasActiveSession == 2 and biasActiveDir == -1 and not na(biasEntrySL) and close > biasEntrySL

amBullTakeProfitSignal = showBiasSignalArrows and enableTakeProfit and timeframe.period == "1" and barstate.isconfirmed and biasInPosition and biasActiveSession == 1 and biasActiveDir == 1 and not na(biasEntryTP) and high >= biasEntryTP
amBearTakeProfitSignal = showBiasSignalArrows and enableTakeProfit and timeframe.period == "1" and barstate.isconfirmed and biasInPosition and biasActiveSession == 1 and biasActiveDir == -1 and not na(biasEntryTP) and low <= biasEntryTP
pmBullTakeProfitSignal = showBiasSignalArrows and enableTakeProfit and timeframe.period == "1" and barstate.isconfirmed and biasInPosition and biasActiveSession == 2 and biasActiveDir == 1 and not na(biasEntryTP) and high >= biasEntryTP
pmBearTakeProfitSignal = showBiasSignalArrows and enableTakeProfit and timeframe.period == "1" and barstate.isconfirmed and biasInPosition and biasActiveSession == 2 and biasActiveDir == -1 and not na(biasEntryTP) and low <= biasEntryTP

amForcedCloseSignal = showBiasSignalArrows and amForceCloseNow and biasInPosition and biasActiveSession == 1

// ============================================================================
// Exit Logic
// ============================================================================

if (amBullExitSignal or amBullTakeProfitSignal or (amForcedCloseSignal and biasActiveDir == 1))
    exitReason = amForcedCloseSignal ? "11:59 AM Close" : amBullTakeProfitSignal ? "AM TP Hit" : "AM SL Hit"
    exitFillPrice = amBullTakeProfitSignal ? biasEntryTP : close
    exitText = showBiasExitPriceLabel ? "Bull Exit\n" + exitReason + "\n" + str.tostring(exitFillPrice, format.mintick) : ""
    exitLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=exitText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullExitArrowColor)
    array.push(biasSignalLabels, exitLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index
    if amBullTakeProfitSignal
        biasTakeProfitHitToday := true
    biasInPosition := false
    biasActiveDir := 0
    biasActiveSession := 0
    biasEntryPrice := na
    biasEntrySL := na
    biasEntryTP := na
    biasPyramidEntryCount := 0

if (amBearExitSignal or amBearTakeProfitSignal or (amForcedCloseSignal and biasActiveDir == -1))
    exitReason = amForcedCloseSignal ? "11:59 AM Close" : amBearTakeProfitSignal ? "AM TP Hit" : "AM SL Hit"
    exitFillPrice = amBearTakeProfitSignal ? biasEntryTP : close
    exitText = showBiasExitPriceLabel ? "Bear Exit\n" + exitReason + "\n" + str.tostring(exitFillPrice, format.mintick) : ""
    exitLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=exitText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearExitArrowColor)
    array.push(biasSignalLabels, exitLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index
    if amBearTakeProfitSignal
        biasTakeProfitHitToday := true
    biasInPosition := false
    biasActiveDir := 0
    biasActiveSession := 0
    biasEntryPrice := na
    biasEntrySL := na
    biasEntryTP := na
    biasPyramidEntryCount := 0

if pmBullExitSignal or pmBullTakeProfitSignal
    exitReason = pmBullTakeProfitSignal ? "PM TP Hit" : "PM SL Hit"
    exitFillPrice = pmBullTakeProfitSignal ? biasEntryTP : close
    exitText = showBiasExitPriceLabel ? "Bull Exit\n" + exitReason + "\n" + str.tostring(exitFillPrice, format.mintick) : ""
    exitLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=exitText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullExitArrowColor)
    array.push(biasSignalLabels, exitLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index
    if pmBullTakeProfitSignal
        biasTakeProfitHitToday := true
    biasInPosition := false
    biasActiveDir := 0
    biasActiveSession := 0
    biasEntryPrice := na
    biasEntrySL := na
    biasEntryTP := na
    biasPyramidEntryCount := 0

if pmBearExitSignal or pmBearTakeProfitSignal
    exitReason = pmBearTakeProfitSignal ? "PM TP Hit" : "PM SL Hit"
    exitFillPrice = pmBearTakeProfitSignal ? biasEntryTP : close
    exitText = showBiasExitPriceLabel ? "Bear Exit\n" + exitReason + "\n" + str.tostring(exitFillPrice, format.mintick) : ""
    exitLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=exitText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearExitArrowColor)
    array.push(biasSignalLabels, exitLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index
    if pmBearTakeProfitSignal
        biasTakeProfitHitToday := true
    biasInPosition := false
    biasActiveDir := 0
    biasActiveSession := 0
    biasEntryPrice := na
    biasEntrySL := na
    biasEntryTP := na
    biasPyramidEntryCount := 0

// ============================================================================
// Entry Logic: Active Range Extreme Ping-Pong Entries
// ============================================================================

if amBullRangeExtremeSetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, amBullRangeSL)
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", amBullRangeName, close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", amBullRangeName, close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if amBearRangeExtremeSetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, amBearRangeSL)
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", amBearRangeName, close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", amBearRangeName, close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

if pmBullRangeExtremeSetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, pmBullRangeSL)
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", pmBullRangeName, close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", pmBullRangeName, close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if pmBearRangeExtremeSetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, pmBearRangeSL)
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", pmBearRangeName, close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", pmBearRangeName, close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

// ============================================================================
// Entry Logic: Macro Extreme High / Low Entries
// ============================================================================

if amBullMacroExtremeSetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, amBullMacroTriggerLevel - macroStopBufferPoints)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != 1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    amMacroLowUsed := true
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", amBullMacroTriggerName, close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", amBullMacroTriggerName, close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if amBearMacroExtremeSetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, amBearMacroTriggerLevel + macroStopBufferPoints)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != -1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    amMacroHighUsed := true
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", amBearMacroTriggerName, close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", amBearMacroTriggerName, close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

if pmBullMacroExtremeSetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, pmBullMacroTriggerLevel - macroStopBufferPoints)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != 1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    pmMacroLowUsed := true
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", pmBullMacroTriggerName, close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", pmBullMacroTriggerName, close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if pmBearMacroExtremeSetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, pmBearMacroTriggerLevel + macroStopBufferPoints)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != -1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    pmMacroHighUsed := true
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", pmBearMacroTriggerName, close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", pmBearMacroTriggerName, close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

// ============================================================================
// Entry Logic: Macro Entries
// ============================================================================

if amBullContinuationSetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, bias950Low)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != 1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", "Bull Macro", close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", "Bull Macro", close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if amBearContinuationSetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, bias950High)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != -1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", "Bear Macro", close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", "Bear Macro", close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

if amBullReversalSetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, bias950Low)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != 1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", "Bull Reversal", close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", "Bull Reversal", close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if amBearReversalSetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, bias950High)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != -1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", "Bear Reversal", close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", "Bear Reversal", close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

// ============================================================================
// Entry Logic: AM Structure Entries
// ============================================================================

if amBullOBEntrySetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, amBullOBEntryLow)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != 1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    markMMStructureUsed(amMMOBs, amBullOBEntryIdx)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", amUseBullExtreme ? "Bull Extreme OB" : "Bull OB", close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", amUseBullExtreme ? "Bull Extreme OB" : "Bull OB", close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if amBearOBEntrySetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, amBearOBEntryHigh)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != -1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    markMMStructureUsed(amMMOBs, amBearOBEntryIdx)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", amUseBearExtreme ? "Bear Extreme OB" : "Bear OB", close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", amUseBearExtreme ? "Bear Extreme OB" : "Bear OB", close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

if amBullFVGEntrySetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, amBullFVGEntryLow)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != 1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    markMMStructureUsed(amMMFVGs, amBullFVGEntryIdx)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", amUseBullExtreme ? "Bull Extreme Imb" : "Bull Imb", close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", amUseBullExtreme ? "Bull Extreme Imb" : "Bull Imb", close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if amBearFVGEntrySetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, amBearFVGEntryHigh)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != -1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 1
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    markMMStructureUsed(amMMFVGs, amBearFVGEntryIdx)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "AM", amUseBearExtreme ? "Bear Extreme Imb" : "Bear Imb", close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "AM", amUseBearExtreme ? "Bear Extreme Imb" : "Bear Imb", close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

// ============================================================================
// Entry Logic: PM Macro / Structure Entries
// ============================================================================

if pmBullContinuationSetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, pm_bias1250Low)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != 1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", "Bull Macro", close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", "Bull Macro", close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if pmBearContinuationSetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, pm_bias1250High)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != -1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", "Bear Macro", close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", "Bear Macro", close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

if pmBullReversalSetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, pm_bias1250Low)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != 1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", "Bull Reversal", close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", "Bull Reversal", close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if pmBearReversalSetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, pm_bias1250High)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != -1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", "Bear Reversal", close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", "Bear Reversal", close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

if pmBullOBEntrySetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, pmBullOBEntryLow)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != 1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    markMMStructureUsed(pmMMOBs, pmBullOBEntryIdx)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", pmUseBullExtreme ? "Bull Extreme OB" : "Bull OB", close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", pmUseBullExtreme ? "Bull Extreme OB" : "Bull OB", close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if pmBearOBEntrySetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, pmBearOBEntryHigh)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != -1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    markMMStructureUsed(pmMMOBs, pmBearOBEntryIdx)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", pmUseBearExtreme ? "Bear Extreme OB" : "Bear OB", close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", pmUseBearExtreme ? "Bear Extreme OB" : "Bear OB", close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

if pmBullFVGEntrySetup and canEnterBiasDir(1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(1, close, pmBullFVGEntryLow)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != 1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := 1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    markMMStructureUsed(pmMMFVGs, pmBullFVGEntryIdx)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", pmUseBullExtreme ? "Bull Extreme Imb" : "Bull Imb", close, 1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", pmUseBullExtreme ? "Bull Extreme Imb" : "Bull Imb", close, 1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasLowerLabelY(low, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowup, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bullEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasLowerLabelStack := biasLowerStackIndex() + 1
        biasLastLowerLabelBar := bar_index

if pmBearFVGEntrySetup and canEnterBiasDir(-1) and biasEntriesToday < maxBiasEntriesPerDay
    newSL = calcBiasSL(-1, close, pmBearFVGEntryHigh)
    entryIsFlip = allowOppositeSignalFlip and biasInPosition and biasActiveDir != 0 and biasActiveDir != -1
    if entryIsFlip
        biasInPosition := false
        biasActiveDir := 0
        biasActiveSession := 0
        biasEntryPrice := na
        biasEntrySL := na
        biasEntryTP := na
        biasPyramidEntryCount := 0
    if not biasInPosition
        biasTradeId += 1
        biasPyramidEntryCount := 1
        biasEntrySL := newSL
    else
        biasPyramidEntryCount += 1
        biasEntrySL := combineBiasSL(-1, biasEntrySL, newSL)
    biasEntriesToday += 1
    biasLastEntryBar := bar_index
    biasInPosition := true
    biasActiveDir := -1
    biasActiveSession := 2
    biasEntryPrice := na(biasEntryPrice) or biasPyramidEntryCount <= 1 ? close : ((biasEntryPrice * (biasPyramidEntryCount - 1)) + close) / biasPyramidEntryCount
    if na(biasEntryTP) or biasPyramidEntryCount <= 1 or not lockTakeProfitToFirstEntry
        biasEntryTP := calcBiasTP(-1, lockTakeProfitToFirstEntry ? close : biasEntryPrice)
    markMMStructureUsed(pmMMFVGs, pmBearFVGEntryIdx)
    entryText = entryLabelText(biasTradeId, biasPyramidEntryCount, "PM", pmUseBearExtreme ? "Bear Extreme Imb" : "Bear Imb", close, -1, newSL, biasEntryTP)
    if enableEntryAlerts
        alert(entryAlertText(biasTradeId, biasPyramidEntryCount, "PM", pmUseBearExtreme ? "Bear Extreme Imb" : "Bear Imb", close, -1, newSL, biasEntryTP), alert.freq_once_per_bar_close)
    entryLabel = label.new(x=bar_index, y=biasUpperLabelY(high, biasArrowATR), text=entryText, xloc=xloc.bar_index, yloc=yloc.price, style=label.style_arrowdown, textcolor=biasArrowTextColor, size=biasArrowSize(), color=bearEntryArrowColor)
    array.push(biasSignalLabels, entryLabel)
    if enableBiasLabelStacking
        biasUpperLabelStack := biasUpperStackIndex() + 1
        biasLastUpperLabelBar := bar_index

// ============================================================================
// Label Object Management
// ============================================================================

manage_biasSignalObjects() =>
    while array.size(biasSignalLabels) > globalLookbackPeriod * maxBiasEntriesPerDay * 2
        label.delete(array.shift(biasSignalLabels))

manage_biasSignalObjects()

// ============================================================================
// Current Trade State
// ============================================================================

float biasCurrentSL = na
if biasInPosition and not na(biasEntrySL)
    biasCurrentSL := biasEntrySL

biasCallText = biasInPosition ? biasDirText(biasActiveDir) : biasTakeProfitHitToday ? "TP Locked" : "Flat"
biasEntryPriceText = biasInPosition and not na(biasEntryPrice) ? str.tostring(biasEntryPrice, format.mintick) : "-"
biasSLText = biasInPosition and not na(biasCurrentSL) ? str.tostring(biasCurrentSL, format.mintick) : "-"
biasTPText = biasInPosition and enableTakeProfit and not na(biasEntryTP) ? str.tostring(biasEntryTP, format.mintick) : biasTakeProfitHitToday ? "TP Hit / Locked" : "-"
biasCallBg = biasInPosition ? biasDirBg(biasActiveDir) : biasTakeProfitHitToday ? color.new(color.green, 65) : color.new(color.gray, 80)

// ============================================================================
// PM Outcome Buckets
// ============================================================================

pmOutcomeText = "Waiting for PM Setup to Complete"
pmFinalTableText = "Waiting"
pmFinalBg = color.new(color.gray, 80)

if pmBullContinuationConfirmed
    pmOutcomeText := "PM Bull Bias Confirmed\n- PM algo is long-sided"
    pmFinalTableText := "PM Bull Confirmed"
    pmFinalBg := color.new(color.green, 55)
else if pmBearContinuationConfirmed
    pmOutcomeText := "PM Bear Bias Confirmed\n- PM algo is short-sided"
    pmFinalTableText := "PM Bear Confirmed"
    pmFinalBg := color.new(color.red, 55)
else if pmBullReversalConfirmed
    pmOutcomeText := "PM Bull Reversal Confirmed\n- PM FPI invalidated upward"
    pmFinalTableText := "PM Bull Reversal"
    pmFinalBg := color.new(color.green, 55)
else if pmBearReversalConfirmed
    pmOutcomeText := "PM Bear Reversal Confirmed\n- PM FPI invalidated downward"
    pmFinalTableText := "PM Bear Reversal"
    pmFinalBg := color.new(color.red, 55)
else if pm_bias1200Found and not pm_bias1250Found
    pmOutcomeText := "PM FPI Found\n- Waiting for " + pmSelectedMacroName
    pmFinalTableText := "Waiting Macro"
    pmFinalBg := color.new(color.orange, 70)
else if pm_biasLowProbability
    pmOutcomeText := "PM FPI / Macro Conflict\n- Low Probability"
    pmFinalTableText := "PM Conflict"
    pmFinalBg := color.new(color.orange, 70)
else if pmPriceInsideMacro
    pmOutcomeText := "Waiting for PM Setup to Complete"
    pmFinalTableText := "Inside PM FPI"
    pmFinalBg := color.new(color.gray, 80)

// ============================================================================
// Market Maker Structure Warning Label
// ============================================================================

if barstate.islast
    if not na(mmStructureWarningLabel)
        label.delete(mmStructureWarningLabel)
        mmStructureWarningLabel := na

    string mmWarningText = ""

    if showMMStructureWarnings and enableMMStructureEngine
        if bias930Found and bias930Dir == -1 and (amBullOBEntryFound or amBullFVGEntryFound)
            structHigh = mergedHigh(amBullOBEntryFound, amBullOBEntryHigh, amBullFVGEntryFound, amBullFVGEntryHigh)
            structLow = mergedLow(amBullOBEntryFound, amBullOBEntryLow, amBullFVGEntryFound, amBullFVGEntryLow)
            structName = amBullOBEntryFound and amBullFVGEntryFound ? "Bullish OB + Imbalance Active" : amBullOBEntryFound ? "Bullish OB Active" : "Bullish Imbalance Active"
            mmWarningText := mmWarningText + structName + "\nPossible 9:30 FPI Judas Reversal\nHigh - " + str.tostring(structHigh, format.mintick) + "\nLow - " + str.tostring(structLow, format.mintick)

        if bias930Found and bias930Dir == 1 and (amBearOBEntryFound or amBearFVGEntryFound)
            structHigh = mergedHigh(amBearOBEntryFound, amBearOBEntryHigh, amBearFVGEntryFound, amBearFVGEntryHigh)
            structLow = mergedLow(amBearOBEntryFound, amBearOBEntryLow, amBearFVGEntryFound, amBearFVGEntryLow)
            structName = amBearOBEntryFound and amBearFVGEntryFound ? "Bearish OB + Imbalance Active" : amBearOBEntryFound ? "Bearish OB Active" : "Bearish Imbalance Active"
            mmWarningText := mmWarningText + (mmWarningText == "" ? "" : "\n\n") + structName + "\nPossible 9:30 FPI Judas Reversal\nHigh - " + str.tostring(structHigh, format.mintick) + "\nLow - " + str.tostring(structLow, format.mintick)

        if pm_bias1200Found and pm_bias1200Dir == -1 and (pmBullOBEntryFound or pmBullFVGEntryFound)
            structHigh = mergedHigh(pmBullOBEntryFound, pmBullOBEntryHigh, pmBullFVGEntryFound, pmBullFVGEntryHigh)
            structLow = mergedLow(pmBullOBEntryFound, pmBullOBEntryLow, pmBullFVGEntryFound, pmBullFVGEntryLow)
            structName = pmBullOBEntryFound and pmBullFVGEntryFound ? "PM Bullish OB + Imbalance Active" : pmBullOBEntryFound ? "PM Bullish OB Active" : "PM Bullish Imbalance Active"
            mmWarningText := mmWarningText + (mmWarningText == "" ? "" : "\n\n") + structName + "\nPM FPI Reversal Pressure\nHigh - " + str.tostring(structHigh, format.mintick) + "\nLow - " + str.tostring(structLow, format.mintick)

        if pm_bias1200Found and pm_bias1200Dir == 1 and (pmBearOBEntryFound or pmBearFVGEntryFound)
            structHigh = mergedHigh(pmBearOBEntryFound, pmBearOBEntryHigh, pmBearFVGEntryFound, pmBearFVGEntryHigh)
            structLow = mergedLow(pmBearOBEntryFound, pmBearOBEntryLow, pmBearFVGEntryFound, pmBearFVGEntryLow)
            structName = pmBearOBEntryFound and pmBearFVGEntryFound ? "PM Bearish OB + Imbalance Active" : pmBearOBEntryFound ? "PM Bearish OB Active" : "PM Bearish Imbalance Active"
            mmWarningText := mmWarningText + (mmWarningText == "" ? "" : "\n\n") + structName + "\nPM FPI Reversal Pressure\nHigh - " + str.tostring(structHigh, format.mintick) + "\nLow - " + str.tostring(structLow, format.mintick)

    if mmWarningText != ""
        mmWarningY = high + ta.atr(14) * mmWarningYOffsetATR
        mmStructureWarningLabel := label.new(bar_index + mmWarningXOffset, mmWarningY, mmWarningText, style=label.style_none, textcolor=mmWarningTextColor, size=biasTableTextSize(), textalign=text.align_left)

// ============================================================================
// Table Display
// ============================================================================

if showFpiBiasTable and barstate.islast
    table.cell(fpiBiasTable, 0, 0, "Bias System", text_color=color.white, bgcolor=color.new(color.black, 0), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 0, "Status", text_color=color.white, bgcolor=color.new(color.black, 0), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 1, "Call", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 1, biasCallText, text_color=color.white, bgcolor=biasCallBg, text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 2, "Price", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 2, biasEntryPriceText, text_color=color.white, bgcolor=color.new(color.gray, 75), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 3, "SL", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 3, biasSLText, text_color=color.white, bgcolor=color.new(color.gray, 75), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 4, "TP", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 4, biasTPText, text_color=color.white, bgcolor=color.new(color.gray, 75), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 5, "NY AM Bias", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 5, biasDirText(bias930Dir), text_color=color.white, bgcolor=biasDirBg(bias930Dir), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 6, "9:50 Macro FPI", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 6, bias950Found ? "Found" : "Waiting", text_color=color.white, bgcolor=bias950Found ? color.new(color.blue, 70) : color.new(color.gray, 80), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 7, "Price vs 9:50", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 7, biasPriceVs950, text_color=color.white, bgcolor=color.new(color.gray, 75), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 8, "Final AM Bias", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 8, biasFinalText, text_color=color.white, bgcolor=biasFinalBg, text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 9, "PM FPI Bias", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 9, biasDirText(pm_bias1200Dir), text_color=color.white, bgcolor=biasDirBg(pm_bias1200Dir), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 10, "PM Macro FPI", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 10, pm_bias1250Found ? pmSelectedMacroName + " Found" : "Waiting", text_color=color.white, bgcolor=pm_bias1250Found ? color.new(color.blue, 70) : color.new(color.gray, 80), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 11, "Price vs PM Macro", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 11, pmPriceVsMacro, text_color=color.white, bgcolor=color.new(color.gray, 75), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 12, "Price vs 11:59", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 12, pmPriceVs1159, text_color=color.white, bgcolor=color.new(color.gray, 75), text_size=biasTableTextSize())

    table.cell(fpiBiasTable, 0, 13, "Final PM Bias", text_color=color.white, bgcolor=color.new(color.black, 20), text_size=biasTableTextSize())
    table.cell(fpiBiasTable, 1, 13, pmFinalTableText, text_color=color.white, bgcolor=pmFinalBg, text_size=biasTableTextSize())

if not showFpiBiasTable and barstate.islast
    table.clear(fpiBiasTable, 0, 0, 1, 13)

// ============================================================================
// Outside AM Warning Label
// ============================================================================

if barstate.islast
    if not na(fpiLowProbabilityLabel)
        label.delete(fpiLowProbabilityLabel)
        fpiLowProbabilityLabel := na

    if showFpiBiasTable and showLowProbWarning and biasLowProbability
        warningY = low - ta.atr(14) * warningLabelYOffsetATR
        fpiLowProbabilityLabel := label.new(bar_index + warningLabelXOffset, warningY, "*Possible Consolidation / Seek and Destroy\n- Low Probability", style=label.style_none, textcolor=warningLabelTextColor, size=biasTableTextSize(), textalign=text.align_left)

// ============================================================================
// Outside PM Warning Label
// ============================================================================

if barstate.islast
    if not na(fpiPMLowProbabilityLabel)
        label.delete(fpiPMLowProbabilityLabel)
        fpiPMLowProbabilityLabel := na

    if showFpiBiasTable and showPMLowProbWarning
        pmWarningY = low - ta.atr(14) * pmWarningLabelYOffsetATR
        fpiPMLowProbabilityLabel := label.new(bar_index + pmWarningLabelXOffset, pmWarningY, pmOutcomeText, style=label.style_none, textcolor=pmWarningLabelTextColor, size=biasTableTextSize(), textalign=text.align_left)
 
