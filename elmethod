#property copyright "Elmethod EA v1.0"
#property version   "1.0"
#property strict

#include <Trade\Trade.mqh>
#include <Trade\PositionInfo.mqh>
#include <Trade\OrderInfo.mqh>

CTrade        trade;
CPositionInfo posInfo;
COrderInfo    orderInfo;

//+------------------------------------------------------------------+
// Inputs
//+------------------------------------------------------------------+
input group "=== RISK MANAGEMENT ==="
input double RiskPercent       = 2.0;
input double MinLot            = 0.01;
input double MaxLot            = 10.0;
bool   EAEnabled       = true;
double StartBalanceRef = 0.0;

input group "=== HTF TIMEFRAMES ==="
input bool   UseWeekly         = true;
input bool   UseDaily          = true;
input bool   UseH4             = true;
input bool   UseH1             = true;

input group "=== HTF POI SETTINGS ==="
input int    SwingLookback     = 20;
input double FVGMinDollar      = 3.0;
input double POIZoneBufDollar  = 5.0;
input int    RBS_SBR_Lookback  = 50;
input int    MinSRTouches      = 2;
input int    OCL_Lookback      = 50;
input int    FreshZoneBars     = 5;
input double MaxPOIDistDollar  = 200.0;

input group "=== M1 CONFIRMATION ==="
input int    M1ConfirmBarsMax  = 5;
input int    OB_Lookback       = 12;
input double M1_FVGMinDollar   = 0.5;
input int    OB_TimeoutMin     = 5;   // fail-safe

input group "=== M5 ENTRY ==="
input double SLBufferDollar    = 1.0;
input bool   UseM5FallbackMarketOn4th = true;
input double FallbackSL_Pips          = 150.0;
input double FallbackTP_Pips          = 300.0;

input group "=== TP LIQUIDITY ==="
input int    EQH_EQL_Lookback  = 50;
input double EQH_EQL_BufDollar = 2.0;
input double MinRR             = 3.0;
input double MinRRHard           = 3.0;
input double TargetRRIdeal       = 3.0;
input double PartialRR           = 2.0;
input double PartialClosePct     = 50.0;
input bool   MoveSLToBEAfterP1   = true;
input bool   UsePartialAt2R = true;

input group "=== FILTER ==="
input int    MagicNumber       = 215001;
input bool   UseSessionFilter  = false;
input int    SessionStartHour  = 7;
input int    SessionEndHour    = 20;
input int    MaxTradesPerDay   = 3;

input group "=== VISUAL / DEBUG ==="
input bool   DebugScan         = true;
input bool   EnableVisualBoxes = true;
input color  ColorWatchBull    = clrLime;
input color  ColorWatchBear    = clrTomato;
input color  ColorActiveBull   = clrDeepSkyBlue;
input color  ColorActiveBear   = clrOrangeRed;
input color  ColorMid          = clrSilver;
input color  ColorMid50        = clrGainsboro;
input color  ColorOB           = clrDodgerBlue;
input color  ColorEntry        = clrYellow;
input color  ColorSL           = clrRed;
input color  ColorTP           = clrAqua;
input color  ColorStatus       = clrWhite;
input color  ColorReject       = clrOrange;

input group "=== PANEL ==="
input bool   ShowStatsPanel        = true;
input int    PanelX                = 10;
input int    PanelY                = 55;
input int    PanelW                = 280;
input int    PanelH                = 220;
input double GainReferenceBalance  = 0.0;   // 0 = pakai balance saat EA start

string PANEL_BG        = "ELM_PANEL_BG";
string PANEL_TITLE     = "ELM_PANEL_TITLE";
string PANEL_BALANCE   = "ELM_PANEL_BALANCE";
string PANEL_FLOATING  = "ELM_PANEL_FLOATING";
string PANEL_TRADES    = "ELM_PANEL_TRADES";
string PANEL_GAIN      = "ELM_PANEL_GAIN";
string PANEL_EQUITY    = "ELM_PANEL_EQUITY";
string PANEL_PROFIT    = "ELM_PANEL_PROFIT";
string PANEL_LOSS      = "ELM_PANEL_LOSS";
string PANEL_WINRATE   = "ELM_PANEL_WINRATE";
string PANEL_STATUS    = "ELM_PANEL_STATUS";
string PANEL_BTN_EA    = "ELM_PANEL_BTN_EA";

//+------------------------------------------------------------------+
// Constants
//+------------------------------------------------------------------+
#define ELM_MAX_WATCH_POIS     5
#define ELM_MAX_TEMP_POIS      24
#define ELM_MAX_BLOCKED_ZONES  30

//+------------------------------------------------------------------+
// Globals
//+------------------------------------------------------------------+
double   Pip                 = 0.0;
double   PipVal              = 0.0;
int      TradesToday         = 0;
datetime LastTradeDate       = 0;
ulong    PendingTicket       = 0;

datetime LastScanM1Bar       = 0;
datetime TapTime             = 0;
datetime TapM1BarOpen        = 0;
datetime OB_Timeout          = 0;
datetime SR_Deadline         = 0;
datetime lastWatchRefreshM1  = 0;

bool     ForceHTFRescan      = true;
bool     WatchlistDirty      = false;

enum EA_STATE
{
   STATE_SCAN_HTF = 0,
   STATE_WAIT_ANY_TAP,
   STATE_WAIT_M1_OB,
   STATE_WAIT_M5_SR,
   STATE_ORDER_PLACED
};
EA_STATE State = STATE_SCAN_HTF;

//+------------------------------------------------------------------+
struct POI_DATA
{
   double zoneHigh;
   double zoneLow;
   double midLevel;
   bool   isBullish;
   bool   isValid;
   string poiType;
   bool   isSRType;
   double edgeLevel;
   double midLevel50;
   ENUM_TIMEFRAMES poiTF;

   datetime originTime;   // waktu zona pertama terbentuk
};

struct OB_DATA
{
   double   obHigh;
   double   obLow;
   bool     isBullish;
   bool     isValid;
   datetime obTime;
};

struct SR_DATA
{
   double   entryPrice;
   double   slPrice;
   double   tpPrice;
   bool     isBullish;
   bool     isValid;
   datetime setupTime;
};

struct WATCH_POI
{
   POI_DATA poi;
   double   dist;
   bool     valid;
};

struct BLOCKED_ZONE
{
   double   zoneHigh;
   double   zoneLow;
   double   midLevel;
   string   poiType;
   ENUM_TIMEFRAMES poiTF;
   datetime untilTime;
   bool     valid;
};

struct TP_PLAN
{
   double tp1;
   double tp2;
   bool   usePartial;
   bool   valid;
};


POI_DATA     activePOI;
OB_DATA      activeOB;
SR_DATA      activeSR;
WATCH_POI    watchPOIs[ELM_MAX_WATCH_POIS];
int          watchCount = 0;
BLOCKED_ZONE blockedZones[ELM_MAX_BLOCKED_ZONES];

datetime lastM5CheckBar = 0;
datetime lastM1CheckBar = 0;
int      M5SkipCount    = 0;

//+------------------------------------------------------------------+
// Init helpers
//+------------------------------------------------------------------+
void InitPOI(POI_DATA &p)
{
   p.zoneHigh   = 0.0;
   p.zoneLow    = 0.0;
   p.midLevel   = 0.0;
   p.isBullish  = false;
   p.isValid    = false;
   p.poiType    = "";
   p.isSRType   = false;
   p.edgeLevel  = 0.0;
   p.midLevel50 = 0.0;
   p.poiTF      = PERIOD_CURRENT;
   p.originTime = 0;
}

void InitOB(OB_DATA &o)
{
   o.obHigh    = 0.0;
   o.obLow     = 0.0;
   o.isBullish = false;
   o.isValid   = false;
   o.obTime    = 0;
}

void InitSR(SR_DATA &s)
{
   s.entryPrice = 0.0;
   s.slPrice    = 0.0;
   s.tpPrice    = 0.0;
   s.isBullish  = false;
   s.isValid    = false;
   s.setupTime  = 0;
}

void InitWatch(WATCH_POI &w)
{
   InitPOI(w.poi);
   w.dist  = 0.0;
   w.valid = false;
}

void InitBlocked(BLOCKED_ZONE &b)
{
   b.zoneHigh  = 0.0;
   b.zoneLow   = 0.0;
   b.midLevel  = 0.0;
   b.poiType   = "";
   b.poiTF     = PERIOD_CURRENT;
   b.untilTime = 0;
   b.valid     = false;
}

bool ResolvePOIDirectionByLocation(POI_DATA &poi, double currentPrice)
{
   // zone di atas harga = SELL
   if(currentPrice < poi.zoneLow)
      return false;

   // zone di bawah harga = BUY
   if(currentPrice > poi.zoneHigh)
      return true;

   // kalau harga sudah masuk zone, pakai mid
   if(currentPrice < poi.midLevel)
      return false;

   return true;
}


double PreferredTapLevel(POI_DATA &poi)
{
   // FVG wajib 50%
   if(poi.poiType == "FVG")
      return poi.midLevel;   // FVG mid = 50%

   // SR tap di open/close level
   if(poi.poiType == "SR")
      return poi.midLevel;

   // selain itu tetap pakai mid normal
   return poi.midLevel;
}


bool IsPOITappedNow(POI_DATA &poi,
                    double bid,
                    double ask,
                    double barHigh,
                    double barLow,
                    bool forBuy,
                    double &tapScore)
{
   tapScore = DBL_MAX;

   double buf = D2P(POIZoneBufDollar) * 0.5;
   double refLevel = PreferredTapLevel(poi);

   // cek overlap current bar dengan zone
   bool overlap = (barLow <= (poi.zoneHigh + buf) && barHigh >= (poi.zoneLow - buf));
   if(!overlap)
      return false;

   // KHUSUS FVG: wajib sentuh 50%
   if(poi.poiType == "FVG")
   {
      bool hit50 = (barLow <= refLevel && barHigh >= refLevel);
      if(!hit50)
         return false;
   }

   // selain FVG:
   // cukup overlap zone, score tetap pakai refLevel
   double refPrice = forBuy ? bid : ask;
   tapScore = MathAbs(refPrice - refLevel);
   return true;
}

double PreferredSRZoneEntry(POI_DATA &poi)
{
   if(!poi.isValid)
      return 0.0;

   if(poi.poiType == "SR")
      return poi.midLevel50;   // entry refinement di 50% zone

   return poi.midLevel;
}


void ApplyPOIVisibility(string objName)
{
   if(ObjectFind(0, objName) >= 0)
      ObjectSetInteger(0, objName, OBJPROP_TIMEFRAMES, OBJ_ALL_PERIODS);
}

void GetWideDrawRange(datetime &leftTime, datetime &rightTime)
{
   // sengaja lebar supaya tetap terlihat saat ganti TF
   leftTime  = TimeCurrent() - 86400 * 5;  // 5 hari ke kiri
   rightTime = TimeCurrent() + 86400 * 2;  // 2 hari ke kanan
}

void GetChartDrawRange(datetime &leftTime, datetime &rightTime)
{
   int sec = PeriodSeconds((ENUM_TIMEFRAMES)_Period);
   if(sec <= 0)
      sec = 60;

   leftTime  = TimeCurrent() - (sec * 200);
   rightTime = TimeCurrent() + (sec * 40);
}
//==================== PART 2/8 ====================
// lines 351-700

//+------------------------------------------------------------------+
// Utility
//+------------------------------------------------------------------+
double D2P(double dollar)
{
   return (PipVal > 0.0) ? ((dollar / PipVal) * Pip) : (dollar * _Point);
}

double P2D(double priceDiff)
{
   if(Pip <= 0.0) return 0.0;
   return (priceDiff / Pip) * PipVal;
}

double GetBid() { return SymbolInfoDouble(_Symbol, SYMBOL_BID); }
double GetAsk() { return SymbolInfoDouble(_Symbol, SYMBOL_ASK); }

bool IsWithinMaxDist(double level, double price)
{
   if(MaxPOIDistDollar <= 0.0) return true;
   return (MathAbs(price - level) <= D2P(MaxPOIDistDollar));
}

int TFPriority(ENUM_TIMEFRAMES tf)
{
   if(tf == PERIOD_W1) return 4;
   if(tf == PERIOD_D1) return 3;
   if(tf == PERIOD_H4) return 2;
   if(tf == PERIOD_H1) return 1;
   return 0;
}

int TypePriority(string type)
{
   if(type == "OCL") return 5;
   if(type == "FVG") return 4;
   if(type == "SR")  return 3;
   if(type == "RBS") return 2;
   if(type == "SBR") return 1;
   return 0;
}

void DebugPrint(string txt)
{
   if(DebugScan)
      Print(txt);
}

datetime CurrentM1BarOpen()
{
   return iTime(_Symbol, PERIOD_M1, 0);
}

bool NeedRefreshWatchlistByM1()
{
   datetime cur = CurrentM1BarOpen();
   if(cur <= 0) return false;
   return (cur != lastWatchRefreshM1);
}

int CountM1BarsSinceTap()
{
   if(TapM1BarOpen <= 0)
      return 0;

   datetime t[];
   ArraySetAsSeries(t, true);

   int copied = CopyTime(_Symbol, PERIOD_M1, 0, 300, t);
   if(copied <= 0)
      return 0;

   for(int i = 0; i < copied; i++)
   {
      if(t[i] == TapM1BarOpen)
         return i;
   }

   return 999;
}

//+------------------------------------------------------------------+
// Visual helpers
//+------------------------------------------------------------------+
void CreateStatsPanel()
{
   if(!ShowStatsPanel)
      return;

   if(ObjectFind(0, PANEL_BG) < 0)
      ObjectCreate(0, PANEL_BG, OBJ_RECTANGLE_LABEL, 0, 0, 0);

   ObjectSetInteger(0, PANEL_BG, OBJPROP_CORNER, CORNER_LEFT_UPPER);
   ObjectSetInteger(0, PANEL_BG, OBJPROP_XDISTANCE, PanelX);
   ObjectSetInteger(0, PANEL_BG, OBJPROP_YDISTANCE, PanelY);
   ObjectSetInteger(0, PANEL_BG, OBJPROP_XSIZE, PanelW);
   ObjectSetInteger(0, PANEL_BG, OBJPROP_YSIZE, PanelH);
   ObjectSetInteger(0, PANEL_BG, OBJPROP_BGCOLOR, clrBlack);
   ObjectSetInteger(0, PANEL_BG, OBJPROP_COLOR, clrDimGray);
   ObjectSetInteger(0, PANEL_BG, OBJPROP_BORDER_COLOR, clrDimGray);

   CreatePanelLabel(PANEL_TITLE,    PanelX + 12, PanelY + 10, "ELMETHOD STATS", clrWhite, 10, "Arial");
   CreatePanelLabel(PANEL_BALANCE,  PanelX + 12, PanelY + 38, "Balance     : -", clrWhite, 9);
   CreatePanelLabel(PANEL_EQUITY,   PanelX + 12, PanelY + 58, "Equity      : -", clrWhite, 9);
   CreatePanelLabel(PANEL_FLOATING, PanelX + 12, PanelY + 78, "Floating P/L: -", clrWhite, 9);
   CreatePanelLabel(PANEL_GAIN,     PanelX + 12, PanelY + 98, "Gain        : -", clrWhite, 9);
   CreatePanelLabel(PANEL_PROFIT,   PanelX + 12, PanelY + 118,"Profit      : -", clrWhite, 9);
   CreatePanelLabel(PANEL_LOSS,     PanelX + 12, PanelY + 138,"Loss        : -", clrWhite, 9);
   CreatePanelLabel(PANEL_WINRATE,  PanelX + 12, PanelY + 158,"Winrate     : -", clrWhite, 9);
   CreatePanelLabel(PANEL_TRADES,   PanelX + 12, PanelY + 178,"Trades Today: -", clrWhite, 9);
   CreatePanelLabel(PANEL_STATUS,   PanelX + 12, PanelY + 198,"EA Status   : ON", clrLime, 9);

   CreatePanelButton(PANEL_BTN_EA, PanelX + 175, PanelY + 190, 90, 22, "EA ON", clrLime, clrBlack);
}

void UpdateStatsPanel()
{
   if(!ShowStatsPanel)
      return;

   double balance  = AccountInfoDouble(ACCOUNT_BALANCE);
   double equity   = AccountInfoDouble(ACCOUNT_EQUITY);
   double floating = GetFloatingPLEA();

   double profitSum, lossSum, netProfit;
   int winCount, lossCount;
   GetEAClosedStats(profitSum, lossSum, winCount, lossCount, netProfit);

   int totalTrades = winCount + lossCount;
   double winrate  = (totalTrades > 0) ? (100.0 * winCount / totalTrades) : 0.0;

   double refBal  = (StartBalanceRef > 0.0 ? StartBalanceRef : balance);
   double gainPct = (refBal > 0.0) ? ((equity - refBal) / refBal) * 100.0 : 0.0;

   color floatingClr = (floating >= 0.0 ? clrLime : clrTomato);
   color gainClr     = (gainPct >= 0.0 ? clrLime : clrTomato);
   color profitClr   = clrAqua;
   color lossClr     = clrTomato;
   color statusClr   = (EAEnabled ? clrLime : clrTomato);

   ObjectSetString(0, PANEL_BALANCE,  OBJPROP_TEXT, "Balance     : $" + DoubleToString(balance, 2));
   ObjectSetString(0, PANEL_EQUITY,   OBJPROP_TEXT, "Equity      : $" + DoubleToString(equity, 2));
   ObjectSetString(0, PANEL_FLOATING, OBJPROP_TEXT, "Floating P/L: $" + DoubleToString(floating, 2));
   ObjectSetInteger(0, PANEL_FLOATING, OBJPROP_COLOR, floatingClr);

   ObjectSetString(0, PANEL_GAIN, OBJPROP_TEXT, "Gain        : " + DoubleToString(gainPct, 2) + "%");
   ObjectSetInteger(0, PANEL_GAIN, OBJPROP_COLOR, gainClr);

   ObjectSetString(0, PANEL_PROFIT, OBJPROP_TEXT, "Profit      : $" + DoubleToString(profitSum, 2));
   ObjectSetInteger(0, PANEL_PROFIT, OBJPROP_COLOR, profitClr);

   ObjectSetString(0, PANEL_LOSS, OBJPROP_TEXT, "Loss        : $" + DoubleToString(lossSum, 2));
   ObjectSetInteger(0, PANEL_LOSS, OBJPROP_COLOR, lossClr);

   ObjectSetString(0, PANEL_WINRATE, OBJPROP_TEXT,
                   "Winrate     : " + DoubleToString(winrate, 1) + "% (" +
                   IntegerToString(winCount) + "/" + IntegerToString(totalTrades) + ")");

   ObjectSetString(0, PANEL_TRADES, OBJPROP_TEXT, "Trades Today: " + IntegerToString(TradesToday));

   ObjectSetString(0, PANEL_STATUS, OBJPROP_TEXT, "EA Status   : " + (EAEnabled ? "ON" : "OFF"));
   ObjectSetInteger(0, PANEL_STATUS, OBJPROP_COLOR, statusClr);

   ObjectSetString(0, PANEL_BTN_EA, OBJPROP_TEXT, (EAEnabled ? "EA ON" : "EA OFF"));
   ObjectSetInteger(0, PANEL_BTN_EA, OBJPROP_BGCOLOR, (EAEnabled ? clrLime : clrTomato));
}

void DeleteStatsPanel()
{
   ObjectDelete(0, PANEL_BG);
   ObjectDelete(0, PANEL_TITLE);
   ObjectDelete(0, PANEL_BALANCE);
   ObjectDelete(0, PANEL_EQUITY);
   ObjectDelete(0, PANEL_FLOATING);
   ObjectDelete(0, PANEL_GAIN);
   ObjectDelete(0, PANEL_PROFIT);
   ObjectDelete(0, PANEL_LOSS);
   ObjectDelete(0, PANEL_WINRATE);
   ObjectDelete(0, PANEL_TRADES);
   ObjectDelete(0, PANEL_STATUS);
   ObjectDelete(0, PANEL_BTN_EA);
}

void SetStatusLabel(string txt)
{
   if(!EnableVisualBoxes) return;

   string name = "ELM_STATUS_LABEL";
   if(ObjectFind(0, name) < 0)
   {
      ObjectCreate(0, name, OBJ_LABEL, 0, 0, 0);
      ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
      ObjectSetInteger(0, name, OBJPROP_XDISTANCE, 10);
      ObjectSetInteger(0, name, OBJPROP_YDISTANCE, 15);
      ObjectSetInteger(0, name, OBJPROP_FONTSIZE, 10);
   }

   ObjectSetString(0, name, OBJPROP_TEXT, "STATUS: " + txt);
   ObjectSetInteger(0, name, OBJPROP_COLOR, ColorStatus);
}

void SetRejectLabel(string txt)
{
   Print("REJECT: ", txt);

   if(!EnableVisualBoxes) return;

   string name = "ELM_REJECT_LABEL";
   if(ObjectFind(0, name) < 0)
   {
      ObjectCreate(0, name, OBJ_LABEL, 0, 0, 0);
      ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
      ObjectSetInteger(0, name, OBJPROP_XDISTANCE, 10);
      ObjectSetInteger(0, name, OBJPROP_YDISTANCE, 35);
      ObjectSetInteger(0, name, OBJPROP_FONTSIZE, 9);
   }

   ObjectSetString(0, name, OBJPROP_TEXT, "REJECT: " + txt);
   ObjectSetInteger(0, name, OBJPROP_COLOR, ColorReject);
}

void CreatePanelLabel(string name, int x, int y, string text, color clr, int fontSize = 9, string fontName = "Arial")
{
   if(ObjectFind(0, name) < 0)
      ObjectCreate(0, name, OBJ_LABEL, 0, 0, 0);

   ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
   ObjectSetInteger(0, name, OBJPROP_XDISTANCE, x);
   ObjectSetInteger(0, name, OBJPROP_YDISTANCE, y);
   ObjectSetInteger(0, name, OBJPROP_COLOR, clr);
   ObjectSetInteger(0, name, OBJPROP_FONTSIZE, fontSize);
   ObjectSetString(0, name, OBJPROP_FONT, fontName);
   ObjectSetString(0, name, OBJPROP_TEXT, text);
}

void CreatePanelButton(string name, int x, int y, int w, int h, string text, color bg, color fg)
{
   if(ObjectFind(0, name) < 0)
      ObjectCreate(0, name, OBJ_BUTTON, 0, 0, 0);

   ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
   ObjectSetInteger(0, name, OBJPROP_XDISTANCE, x);
   ObjectSetInteger(0, name, OBJPROP_YDISTANCE, y);
   ObjectSetInteger(0, name, OBJPROP_XSIZE, w);
   ObjectSetInteger(0, name, OBJPROP_YSIZE, h);
   ObjectSetInteger(0, name, OBJPROP_BGCOLOR, bg);
   ObjectSetInteger(0, name, OBJPROP_COLOR, fg);
   ObjectSetInteger(0, name, OBJPROP_BORDER_COLOR, clrBlack);
   ObjectSetString(0, name, OBJPROP_TEXT, text);
}

void ClearWatchObjects()
{
   for(int i = 0; i < ELM_MAX_WATCH_POIS; i++)
   {
      ObjectDelete(0, "ELM_WBOX_" + IntegerToString(i));
      ObjectDelete(0, "ELM_WTXT_" + IntegerToString(i));
      ObjectDelete(0, "ELM_WMID_" + IntegerToString(i));
      ObjectDelete(0, "ELM_WZH_"  + IntegerToString(i));
      ObjectDelete(0, "ELM_WZL_"  + IntegerToString(i));
   }
}

void ClearSetupObjects()
{
   ObjectDelete(0, "ELM_ACTIVE_BOX");
   ObjectDelete(0, "ELM_ACTIVE_TXT");
   ObjectDelete(0, "ELM_ACTIVE_MID");
   ObjectDelete(0, "ELM_ACTIVE_M50");
   ObjectDelete(0, "ELM_ACTIVE_ZH");
   ObjectDelete(0, "ELM_ACTIVE_ZL");

   ObjectDelete(0, "ELM_OB_BOX");
   ObjectDelete(0, "ELM_OB_TXT");

   ObjectDelete(0, "ELM_ENTRY");
   ObjectDelete(0, "ELM_SL");
   ObjectDelete(0, "ELM_TP");
   ObjectDelete(0, "ELM_ENTRY_TXT");
   ObjectDelete(0, "ELM_SL_TXT");
   ObjectDelete(0, "ELM_TP_TXT");
}

void ClearAllVisualObjects()
{
   ClearWatchObjects();
   ClearSetupObjects();
   ObjectDelete(0, "ELM_STATUS_LABEL");
   ObjectDelete(0, "ELM_REJECT_LABEL");
}

void DrawWatchPOI(int idx, POI_DATA &poi)
{
   if(!EnableVisualBoxes || !poi.isValid)
      return;

   if(_Period != PERIOD_M5)
      return;

   string nBox = "ELM_WBOX_" + IntegerToString(idx);
   string nTxt = "ELM_WTXT_" + IntegerToString(idx);

   // hapus object watch lama
   ObjectDelete(0, nBox);
   ObjectDelete(0, nTxt);
   ObjectDelete(0, "ELM_WMID_" + IntegerToString(idx));
   ObjectDelete(0, "ELM_WZH_"  + IntegerToString(idx));
   ObjectDelete(0, "ELM_WZL_"  + IntegerToString(idx));

bool dirBuy = (poi.poiType == "FVG") ? poi.isBullish
                                     : ResolvePOIDirectionByLocation(poi, GetBid());
   color zoneColor = dirBuy ? clrLime : clrTomato;

   datetime leftTime  = iTime(_Symbol, PERIOD_M5, 8);
   datetime rightTime = iTime(_Symbol, PERIOD_M5, 0) + PeriodSeconds(PERIOD_M5);

   if(leftTime <= 0)
      leftTime = TimeCurrent() - PeriodSeconds(PERIOD_M5) * 8;

   if(rightTime <= 0)
      rightTime = TimeCurrent();

   ObjectCreate(0, nBox, OBJ_RECTANGLE, 0, leftTime, poi.zoneHigh, rightTime, poi.zoneLow);
   ObjectSetInteger(0, nBox, OBJPROP_COLOR, zoneColor);
   ObjectSetInteger(0, nBox, OBJPROP_WIDTH, 1);
   ObjectSetInteger(0, nBox, OBJPROP_FILL, false);
   ObjectSetInteger(0, nBox, OBJPROP_BACK, false);

   ObjectCreate(0, nTxt, OBJ_TEXT, 0, rightTime, poi.midLevel);
   ObjectSetString(0, nTxt, OBJPROP_TEXT,
                   EnumToString(poi.poiTF) + " " + GetPOIDisplayName(poi, dirBuy));
   ObjectSetInteger(0, nTxt, OBJPROP_COLOR, zoneColor);
   ObjectSetInteger(0, nTxt, OBJPROP_ANCHOR, ANCHOR_LEFT);
   ObjectSetInteger(0, nTxt, OBJPROP_FONTSIZE, 8);
}


void DrawAllWatchPOIs()
{
   if(!EnableVisualBoxes) return;

   ClearWatchObjects();

   for(int i = 0; i < watchCount; i++)
      if(watchPOIs[i].valid)
         DrawWatchPOI(i, watchPOIs[i].poi);
}
void DrawActivePOI()
{
   if(!EnableVisualBoxes || !activePOI.isValid)
      return;

   ObjectDelete(0, "ELM_ACTIVE_BOX");
   ObjectDelete(0, "ELM_ACTIVE_TXT");
   ObjectDelete(0, "ELM_ACTIVE_ZH");
   ObjectDelete(0, "ELM_ACTIVE_ZL");
   ObjectDelete(0, "ELM_ACTIVE_MID");
   ObjectDelete(0, "ELM_ACTIVE_M50");

   datetime leftTime, rightTime;
   GetPOIDrawRange(activePOI, leftTime, rightTime);

   color activeBlue = clrDodgerBlue;
   color midColor   = clrWhite;

   // kotak active full biru
   ObjectCreate(0, "ELM_ACTIVE_BOX", OBJ_RECTANGLE, 0, leftTime, activePOI.zoneHigh, rightTime, activePOI.zoneLow);
   ObjectSetInteger(0, "ELM_ACTIVE_BOX", OBJPROP_COLOR, activeBlue);
   ObjectSetInteger(0, "ELM_ACTIVE_BOX", OBJPROP_WIDTH, 2);
   ObjectSetInteger(0, "ELM_ACTIVE_BOX", OBJPROP_FILL, true);
   ObjectSetInteger(0, "ELM_ACTIVE_BOX", OBJPROP_BACK, false);   // kalau mau di belakang candle, ubah jadi true

   // garis pendek, bukan hline
   DrawShortLine("ELM_ACTIVE_ZH",  leftTime, rightTime, activePOI.zoneHigh, activeBlue, STYLE_SOLID, 1, false);
   DrawShortLine("ELM_ACTIVE_ZL",  leftTime, rightTime, activePOI.zoneLow,  activeBlue, STYLE_SOLID, 1, false);
   DrawShortLine("ELM_ACTIVE_MID", leftTime, rightTime, activePOI.midLevel, midColor,   STYLE_DASH,  1, false);

   if(MathAbs(activePOI.midLevel50 - activePOI.midLevel) > (_Point * 2.0))
      DrawShortLine("ELM_ACTIVE_M50", leftTime, rightTime, activePOI.midLevel50, clrAqua, STYLE_DOT, 1, false);

   ObjectCreate(0, "ELM_ACTIVE_TXT", OBJ_TEXT, 0, rightTime, activePOI.midLevel);
   ObjectSetString(0, "ELM_ACTIVE_TXT", OBJPROP_TEXT,
                   EnumToString(activePOI.poiTF) + " ACTIVE");
   ObjectSetInteger(0, "ELM_ACTIVE_TXT", OBJPROP_COLOR, activeBlue);
   ObjectSetInteger(0, "ELM_ACTIVE_TXT", OBJPROP_ANCHOR, ANCHOR_LEFT);
   ObjectSetInteger(0, "ELM_ACTIVE_TXT", OBJPROP_FONTSIZE, 9);

   ChartRedraw(0);
}


void DrawOBBox()
{
   if(!EnableVisualBoxes || !activeOB.isValid) return;

   ObjectDelete(0, "ELM_OB_BOX");
   ObjectDelete(0, "ELM_OB_TXT");

   int sec = PeriodSeconds(PERIOD_M1);
   if(sec <= 0) sec = 60;

   datetime leftTime  = activeOB.obTime;
   datetime rightTime = activeOB.obTime + sec;

   if(leftTime <= 0)  leftTime  = TimeCurrent() - sec;
   if(rightTime <= 0) rightTime = TimeCurrent();

   ObjectCreate(0, "ELM_OB_BOX", OBJ_RECTANGLE, 0, leftTime, activeOB.obHigh, rightTime, activeOB.obLow);
   ObjectSetInteger(0, "ELM_OB_BOX", OBJPROP_COLOR, ColorOB);
   ObjectSetInteger(0, "ELM_OB_BOX", OBJPROP_WIDTH, 1);
   ObjectSetInteger(0, "ELM_OB_BOX", OBJPROP_FILL, false);
   ObjectSetInteger(0, "ELM_OB_BOX", OBJPROP_BACK, false);

   ObjectCreate(0, "ELM_OB_TXT", OBJ_TEXT, 0, rightTime, activeOB.isBullish ? activeOB.obHigh : activeOB.obLow);
   ObjectSetString(0, "ELM_OB_TXT", OBJPROP_TEXT, "OB M1");
   ObjectSetInteger(0, "ELM_OB_TXT", OBJPROP_COLOR, ColorOB);
   ObjectSetInteger(0, "ELM_OB_TXT", OBJPROP_ANCHOR, ANCHOR_LEFT);
   ObjectSetInteger(0, "ELM_OB_TXT", OBJPROP_FONTSIZE, 8);
}

void DrawTradeLevels()
{
   if(!EnableVisualBoxes || !activeSR.isValid) return;

   ObjectDelete(0, "ELM_ENTRY");
   ObjectDelete(0, "ELM_SL");
   ObjectDelete(0, "ELM_TP");
   ObjectDelete(0, "ELM_ENTRY_TXT");
   ObjectDelete(0, "ELM_SL_TXT");
   ObjectDelete(0, "ELM_TP_TXT");

   datetime leftTime, rightTime;
   GetPOIDrawRange(activePOI, leftTime, rightTime);

   DrawShortLine("ELM_ENTRY", leftTime, rightTime, activeSR.entryPrice, ColorEntry, STYLE_DASH, 1, false);
   DrawShortLine("ELM_SL",    leftTime, rightTime, activeSR.slPrice,    ColorSL,    STYLE_DASH, 1, false);
   DrawShortLine("ELM_TP",    leftTime, rightTime, activeSR.tpPrice,    ColorTP,    STYLE_DASH, 1, false);

   ObjectCreate(0, "ELM_ENTRY_TXT", OBJ_TEXT, 0, rightTime, activeSR.entryPrice);
   ObjectSetString(0, "ELM_ENTRY_TXT", OBJPROP_TEXT, "ENTRY");
   ObjectSetInteger(0, "ELM_ENTRY_TXT", OBJPROP_COLOR, ColorEntry);
   ObjectSetInteger(0, "ELM_ENTRY_TXT", OBJPROP_ANCHOR, ANCHOR_LEFT);
   ObjectSetInteger(0, "ELM_ENTRY_TXT", OBJPROP_FONTSIZE, 8);

   ObjectCreate(0, "ELM_SL_TXT", OBJ_TEXT, 0, rightTime, activeSR.slPrice);
   ObjectSetString(0, "ELM_SL_TXT", OBJPROP_TEXT, "SL");
   ObjectSetInteger(0, "ELM_SL_TXT", OBJPROP_COLOR, ColorSL);
   ObjectSetInteger(0, "ELM_SL_TXT", OBJPROP_ANCHOR, ANCHOR_LEFT);
   ObjectSetInteger(0, "ELM_SL_TXT", OBJPROP_FONTSIZE, 8);

   ObjectCreate(0, "ELM_TP_TXT", OBJ_TEXT, 0, rightTime, activeSR.tpPrice);
   ObjectSetString(0, "ELM_TP_TXT", OBJPROP_TEXT, "TP");
   ObjectSetInteger(0, "ELM_TP_TXT", OBJPROP_COLOR, ColorTP);
   ObjectSetInteger(0, "ELM_TP_TXT", OBJPROP_ANCHOR, ANCHOR_LEFT);
   ObjectSetInteger(0, "ELM_TP_TXT", OBJPROP_FONTSIZE, 8);
}


void RedrawVisuals()
{
   if(!EnableVisualBoxes) return;

   if(State == STATE_WAIT_ANY_TAP)
   {
      DrawAllWatchPOIs();
      return;
   }

   if(activePOI.isValid)
      DrawActivePOI();

   if(activeOB.isValid)
      DrawOBBox();

   if(activeSR.isValid)
      DrawTradeLevels();
}

//+------------------------------------------------------------------+
// Blocked zone helpers
//==================== PART 3/8 ====================
// lines 700-1050
//+------------------------------------------------------------------+
datetime H1CloseTime()
{
   datetime h1open = iTime(_Symbol, PERIOD_H1, 0);
   if(h1open <= 0)
      return TimeCurrent() + PeriodSeconds(PERIOD_H1);

   return h1open + PeriodSeconds(PERIOD_H1);
}

bool IsZoneMatch(double zH1,double zL1,double mid1,string type1,ENUM_TIMEFRAMES tf1,
                 double zH2,double zL2,double mid2,string type2,ENUM_TIMEFRAMES tf2)
{
   if(type1 != type2) return false;
   if(tf1   != tf2)   return false;

   double tol = D2P(2.0);

   bool midClose = (MathAbs(mid1 - mid2) <= tol);
   bool overlap  = (MathMin(zH1, zH2) - MathMax(zL1, zL2)) >= -tol;

   return (midClose || overlap);
}

void CleanupBlockedZones()
{
   for(int i=0; i<ELM_MAX_BLOCKED_ZONES; i++)
   {
      if(blockedZones[i].valid && TimeCurrent() >= blockedZones[i].untilTime)
         InitBlocked(blockedZones[i]);
   }
}

bool IsBlockedZone(POI_DATA &poi)
{
   CleanupBlockedZones();

   for(int i=0; i<ELM_MAX_BLOCKED_ZONES; i++)
   {
      if(!blockedZones[i].valid) continue;

      if(IsZoneMatch(poi.zoneHigh, poi.zoneLow, poi.midLevel, poi.poiType, poi.poiTF,
                     blockedZones[i].zoneHigh, blockedZones[i].zoneLow, blockedZones[i].midLevel,
                     blockedZones[i].poiType, blockedZones[i].poiTF))
      {
         return true;
      }
   }

   return false;
}

void BlockZoneUntilH1Close(POI_DATA &poi, string why)
{
   if(!poi.isValid)
      return;

   CleanupBlockedZones();

   int slot = -1;
   for(int i=0; i<ELM_MAX_BLOCKED_ZONES; i++)
   {
      if(!blockedZones[i].valid)
      {
         slot = i;
         break;
      }
   }

   if(slot < 0)
      slot = 0;

   blockedZones[slot].zoneHigh  = poi.zoneHigh;
   blockedZones[slot].zoneLow   = poi.zoneLow;
   blockedZones[slot].midLevel  = poi.midLevel;
   blockedZones[slot].poiType   = poi.poiType;
   blockedZones[slot].poiTF     = poi.poiTF;
   blockedZones[slot].untilTime = H1CloseTime();
   blockedZones[slot].valid     = true;

   SetRejectLabel("Block zone until H1 close | " + poi.poiType + " " + EnumToString(poi.poiTF) + " | " + why);
}

TP_PLAN BuildTPPlan(bool isBuy, double entry, double sl)
{
   TP_PLAN plan;
   plan.tp1        = 0.0;
   plan.tp2        = 0.0;
   plan.usePartial = false;
   plan.valid      = false;

   double slDist = MathAbs(entry - sl);
   if(slDist <= 0.0)
      return plan;

   double hardMinDist = slDist * MinRRHard;
   double idealDist   = slDist * TargetRRIdeal;
   double partialDist = slDist * PartialRR;

   // cari liquidity ketat, bukan swing kecil
   double liqIdeal = FindNearestLiquidityStrict(isBuy, entry, idealDist);
   double liqHard  = FindNearestLiquidityStrict(isBuy, entry, hardMinDist);

   // kalau ada target ideal >= 5R
   if(liqIdeal > 0.0)
   {
      plan.tp1        = liqIdeal;
      plan.tp2        = 0.0;
      plan.usePartial = false;
      plan.valid      = true;
      return plan;
   }

   // kalau tidak ada ideal, tapi ada target minimal >= 2R
   if(liqHard > 0.0)
   {
      if(UsePartialAt2R)
      {
         plan.tp1        = isBuy ? (entry + partialDist) : (entry - partialDist);
         plan.tp2        = liqHard;
         plan.usePartial = true;
         plan.valid      = true;
         return plan;
      }

      plan.tp1        = liqHard;
      plan.tp2        = 0.0;
      plan.usePartial = false;
      plan.valid      = true;
      return plan;
   }

   return plan;
}

bool IsFVGInvalidatedByBreak(POI_DATA &poi, double bid, double ask)
{
   if(!poi.isValid || poi.poiType != "FVG")
      return false;

   double buf = D2P(POIZoneBufDollar) * 0.5;

   // bullish FVG invalid kalau ditembus ke bawah
   if(poi.isBullish)
   {
      if(bid < poi.zoneLow - buf)
         return true;
   }
   else
   {
      // bearish FVG invalid kalau ditembus ke atas
      if(ask > poi.zoneHigh + buf)
         return true;
   }

   return false;
}

bool IsLiquidityHigh(double level, int idx, MqlRates &r[])
{
   double tol = 1.0; // sesuaikan
   int matches = 0;

   for(int j = idx + 1; j < idx + 15 && j < ArraySize(r); j++)
   {
      if(MathAbs(r[j].high - level) <= tol)
         matches++;
   }

   // equal highs atau swing high jelas
   return (matches >= 1);
}

bool IsLiquidityLow(double level, int idx, MqlRates &r[])
{
   double tol = 1.0;
   int matches = 0;

   for(int j = idx + 1; j < idx + 15 && j < ArraySize(r); j++)
   {
      if(MathAbs(r[j].low - level) <= tol)
         matches++;
   }

   return (matches >= 1);
}

bool IsUnsweptHigh(double level, int idx, MqlRates &r[])
{
   for(int j = 0; j < idx; j++)
   {
      if(r[j].high > level)
         return false;
   }
   return true;
}

bool IsUnsweptLow(double level, int idx, MqlRates &r[])
{
   for(int j = 0; j < idx; j++)
   {
      if(r[j].low < level)
         return false;
   }
   return true;
}

bool IsClearSwingHigh(double &h[], int i, int sw)
{
   int total = ArraySize(h);
   for(int j = i - sw; j <= i + sw; j++)
   {
      if(j < 0 || j >= total || j == i)
         continue;
      if(h[j] >= h[i])
         return false;
   }
   return true;
}

bool IsClearSwingLow(double &l[], int i, int sw)
{
   int total = ArraySize(l);
   for(int j = i - sw; j <= i + sw; j++)
   {
      if(j < 0 || j >= total || j == i)
         continue;
      if(l[j] <= l[i])
         return false;
   }
   return true;
}

bool HasEqualHigh(double &h[], int i, double buf, int maxGap = 12)
{
   int total = ArraySize(h);
   int endAt = MathMin(total - 1, i + maxGap);

   for(int j = i + 2; j <= endAt; j++)
   {
      if(MathAbs(h[i] - h[j]) <= buf)
         return true;
   }
   return false;
}

bool HasEqualLow(double &l[], int i, double buf, int maxGap = 12)
{
   int total = ArraySize(l);
   int endAt = MathMin(total - 1, i + maxGap);

   for(int j = i + 2; j <= endAt; j++)
   {
      if(MathAbs(l[i] - l[j]) <= buf)
         return true;
   }
   return false;
}

bool IsUnsweptHigh(double &h[], int i, double level, double buf)
{
   // index lebih kecil = candle lebih baru
   for(int j = 0; j < i; j++)
   {
      if(h[j] > level + buf)
         return false;
   }
   return true;
}

bool IsUnsweptLow(double &l[], int i, double level, double buf)
{
   for(int j = 0; j < i; j++)
   {
      if(l[j] < level - buf)
         return false;
   }
   return true;
}

//+------------------------------------------------------------------+
// Freshness
//+------------------------------------------------------------------+
bool ZoneTouchedInRange(ENUM_TIMEFRAMES tf, double zH, double zL, int startShift, int countBars)
{
   if(countBars <= 0)
      return false;

   double h[], l[];
   ArraySetAsSeries(h, true);
   ArraySetAsSeries(l, true);

   if(CopyHigh(_Symbol, tf, startShift, countBars, h) < countBars)
      return true;
   if(CopyLow (_Symbol, tf, startShift, countBars, l) < countBars)
      return true;

   double buf = D2P(POIZoneBufDollar) * 0.5;

   for(int j = 0; j < countBars; j++)
   {
      if(l[j] < (zH - buf) && h[j] > (zL + buf))
         return true;
   }

   return false;
}

bool IsFreshZone(ENUM_TIMEFRAMES tf, double zH, double zL, int formedIdx)
{
   if(formedIdx <= 0)
      return true;

   // cek semua candle yang lebih baru sejak zona terbentuk sampai sekarang
   return !ZoneTouchedInRange(tf, zH, zL, 0, formedIdx);
}


bool IsZoneStillFreshForTap(ENUM_TIMEFRAMES tf, double zH, double zL)
{
   int checkBars = MathMax(1, FreshZoneBars);
   return !ZoneTouchedInRange(tf, zH, zL, 1, checkBars);
}

//+------------------------------------------------------------------+
// Session / position
//+------------------------------------------------------------------+
double GetFloatingPLEA()
{
   double total = 0.0;

   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0)
         continue;

      if(!PositionSelectByTicket(ticket))
         continue;

      string sym = PositionGetString(POSITION_SYMBOL);
      long magic = PositionGetInteger(POSITION_MAGIC);

      if(sym != _Symbol || magic != MagicNumber)
         continue;

      total += PositionGetDouble(POSITION_PROFIT);
   }

   return total;
}

void GetEAClosedStats(double &profitSum, double &lossSum, int &winCount, int &lossCount, double &netProfit)
{
   profitSum = 0.0;
   lossSum   = 0.0;
   winCount  = 0;
   lossCount = 0;
   netProfit = 0.0;

   if(!HistorySelect(0, TimeCurrent()))
      return;

   int totalDeals = HistoryDealsTotal();
   for(int i = 0; i < totalDeals; i++)
   {
      ulong deal = HistoryDealGetTicket(i);
      if(deal == 0)
         continue;

      string sym = HistoryDealGetString(deal, DEAL_SYMBOL);
      long magic = HistoryDealGetInteger(deal, DEAL_MAGIC);
      long entry = HistoryDealGetInteger(deal, DEAL_ENTRY);

      if(sym != _Symbol || magic != MagicNumber)
         continue;

      if(entry != DEAL_ENTRY_OUT)
         continue;

      double profit = HistoryDealGetDouble(deal, DEAL_PROFIT);
      double swap   = HistoryDealGetDouble(deal, DEAL_SWAP);
      double comm   = HistoryDealGetDouble(deal, DEAL_COMMISSION);
      double net    = profit + swap + comm;

      netProfit += net;

      if(net >= 0.0)
      {
         profitSum += net;
         winCount++;
      }
      else
      {
         lossSum += MathAbs(net);
         lossCount++;
      }
   }
}

bool IsInSession()
{
   MqlDateTime dt;
   TimeToStruct(TimeGMT(), dt);
   return (dt.hour >= SessionStartHour && dt.hour < SessionEndHour);
}

bool HasOpenPosition()
{
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(posInfo.SelectByIndex(i))
      {
         if(posInfo.Symbol() == _Symbol && posInfo.Magic() == MagicNumber)
            return true;
      }
   }
   return false;
}
bool HasZoneBeenTouchedBeforeTime(POI_DATA &poi, datetime untilTime, ENUM_TIMEFRAMES checkTF = PERIOD_M5)
{
   if(!poi.isValid)
      return false;

   if(poi.poiType != "FVG")
      return false;

   if(poi.originTime <= 0)
      return false;

   if(untilTime <= poi.originTime)
      return false;

   MqlRates rates[];
   ArraySetAsSeries(rates, true);

   int bars = CopyRates(_Symbol, checkTF, 0, 2000, rates);
   if(bars <= 0)
      return false;

   double trigger = PreferredTapLevel(poi); // FVG = 50%

   for(int i = bars - 1; i >= 0; i--)
   {
      if(rates[i].time <= poi.originTime)
         continue;

      if(rates[i].time >= untilTime)
         continue;

      bool touched = (rates[i].low <= trigger && rates[i].high >= trigger);
      if(touched)
         return true;
   }

   return false;
}

//==================== PART 4/8 ====================
// lines 1050-1400
//+------------------------------------------------------------------+

bool IsSameZoneOverlap(POI_DATA &a, POI_DATA &b, double minOverlapRatio = 0.30)
{
   if(!a.isValid || !b.isValid)
      return false;

   double top     = MathMin(a.zoneHigh, b.zoneHigh);
   double bottom  = MathMax(a.zoneLow,  b.zoneLow);
   double overlap = top - bottom;

   if(overlap <= 0.0)
      return false;

   double sizeA = a.zoneHigh - a.zoneLow;
   double sizeB = b.zoneHigh - b.zoneLow;
   double base  = MathMin(sizeA, sizeB);

   if(base <= 0.0)
      return false;

   return ((overlap / base) >= minOverlapRatio);
}

//+------------------------------------------------------------------+
// Watch helpers
//+------------------------------------------------------------------+
void ClearWatchPOIs()
{
   watchCount = 0;
   for(int i = 0; i < ELM_MAX_WATCH_POIS; i++)
      InitWatch(watchPOIs[i]);
}

bool AddWatchDirect(POI_DATA &poi)
{
   if(!poi.isValid)
      return false;
      
if(poi.poiType == "FVG")
{
   if(IsFVGInvalidatedByBreak(poi, GetBid(), GetAsk()))
   {
      BlockZoneUntilH1Close(poi, "FVG broken through opposite side");
      return false;
   }
}

   if(watchCount >= ELM_MAX_WATCH_POIS)
      return false;

   if(IsBlockedZone(poi))
      return false;

   double price = GetBid();

   // FVG sekali sudah disentuh sebelum sekarang = jangan dipakai lagi
   if(poi.poiType == "FVG")
   {
      if(HasZoneBeenTouchedBeforeTime(poi, TimeCurrent(), PERIOD_M5))
      {
         BlockZoneUntilH1Close(poi, "FVG already touched");
         return false;
      }
   }

   if(!IsWithinMaxDist(poi.midLevel, price))
      return false;

   double newDist = MathAbs(price - poi.midLevel);

   for(int i = 0; i < watchCount; i++)
   {
      if(!watchPOIs[i].valid)
         continue;

      POI_DATA existing = watchPOIs[i].poi;

      // exact sama -> jangan dobel
      if(IsZoneMatch(existing.zoneHigh, existing.zoneLow, existing.midLevel,
                     existing.poiType, existing.poiTF,
                     poi.zoneHigh, poi.zoneLow, poi.midLevel, poi.poiType, poi.poiTF))
      {
         return false;
      }

      // overlap kuat di TF yang sama -> cegah FVG berubah jadi SBR/RBS/OCL seenaknya
      if(existing.poiTF == poi.poiTF && IsSameZoneOverlap(existing, poi))
      {
         // kalau yang lama FVG dan yang baru bukan FVG -> pertahankan FVG lama
         if(existing.poiType == "FVG" && poi.poiType != "FVG")
            return false;

         // kalau yang baru FVG dan yang lama bukan FVG -> ganti pakai FVG
         if(existing.poiType != "FVG" && poi.poiType == "FVG")
         {
            watchPOIs[i].poi   = poi;
            watchPOIs[i].dist  = newDist;
            watchPOIs[i].valid = true;
            return true;
         }

         // kalau sama-sama FVG overlap, pilih yang lebih dekat ke harga sekarang
         if(existing.poiType == "FVG" && poi.poiType == "FVG")
         {
            if(newDist < watchPOIs[i].dist)
            {
               watchPOIs[i].poi  = poi;
               watchPOIs[i].dist = newDist;
            }
            return true;
         }

         // selain itu, jangan tambah zone overlap baru
         return false;
      }
   }

   watchPOIs[watchCount].poi   = poi;
   watchPOIs[watchCount].dist  = newDist;
   watchPOIs[watchCount].valid = true;
   watchCount++;
   return true;
}


void DrawShortLine(string name,
                   datetime t1,
                   datetime t2,
                   double price,
                   color clr,
                   ENUM_LINE_STYLE style = STYLE_SOLID,
                   int width = 1,
                   bool back = false)
{
   ObjectDelete(0, name);

   ObjectCreate(0, name, OBJ_TREND, 0, t1, price, t2, price);
   ObjectSetInteger(0, name, OBJPROP_COLOR, clr);
   ObjectSetInteger(0, name, OBJPROP_STYLE, style);
   ObjectSetInteger(0, name, OBJPROP_WIDTH, width);
   ObjectSetInteger(0, name, OBJPROP_RAY_LEFT, false);
   ObjectSetInteger(0, name, OBJPROP_RAY_RIGHT, false);
   ObjectSetInteger(0, name, OBJPROP_BACK, back);
}

void GetPOIDrawRange(POI_DATA &poi, datetime &leftTime, datetime &rightTime)
{
   leftTime = iTime(_Symbol, poi.poiTF, 1);
   if(leftTime <= 0)
      leftTime = TimeCurrent() - PeriodSeconds(poi.poiTF) * 2;

   rightTime = TimeCurrent() + PeriodSeconds((ENUM_TIMEFRAMES)_Period) * 2;
}

bool AddTempPOI(WATCH_POI &tempArr[], int &tempCount, POI_DATA &poi)
{
   if(!poi.isValid) return false;
   if(tempCount >= ELM_MAX_TEMP_POIS) return false;
   if(IsBlockedZone(poi)) return false;
   if(poi.poiType == "FVG")
{
   if(HasZoneBeenTouchedBeforeTime(poi, TimeCurrent(), PERIOD_M5))
      return false;
}

   double price = GetBid();

   if(!IsWithinMaxDist(poi.midLevel, price))
      return false;

   for(int i=0; i<tempCount; i++)
   {
      if(!tempArr[i].valid) continue;

      if(IsZoneMatch(tempArr[i].poi.zoneHigh, tempArr[i].poi.zoneLow, tempArr[i].poi.midLevel,
                     tempArr[i].poi.poiType, tempArr[i].poi.poiTF,
                     poi.zoneHigh, poi.zoneLow, poi.midLevel, poi.poiType, poi.poiTF))
      {
         return false;
      }
   }

   tempArr[tempCount].poi   = poi;
   tempArr[tempCount].dist  = MathAbs(price - poi.midLevel);
   tempArr[tempCount].valid = true;
   tempCount++;
   return true;
}

string GetPOIDisplayName(POI_DATA &poi, bool dirBuy)
{
   if(poi.poiType == "SR")
      return dirBuy ? "SUPPORT" : "RESISTANCE";

   if(poi.poiType == "RBS")
      return "RBS";

   if(poi.poiType == "SBR")
      return "SBR";

   if(poi.poiType == "FVG")
      return "FVG50%";

   if(poi.poiType == "OCL")
      return "OCL";

   return poi.poiType;
}

void SortTempPOIs(WATCH_POI &arr[], int count)
{
   for(int i = 0; i < count - 1; i++)
   {
      for(int j = i + 1; j < count; j++)
      {
         bool swapNeeded = false;

         if(arr[j].dist < arr[i].dist)
            swapNeeded = true;
         else if(MathAbs(arr[j].dist - arr[i].dist) <= _Point)
         {
            int tfJ = TFPriority(arr[j].poi.poiTF);
            int tfI = TFPriority(arr[i].poi.poiTF);

            if(tfJ > tfI)
               swapNeeded = true;
            else if(tfJ == tfI)
            {
               int tpJ = TypePriority(arr[j].poi.poiType);
               int tpI = TypePriority(arr[i].poi.poiType);
               if(tpJ > tpI)
                  swapNeeded = true;
            }
         }

         if(swapNeeded)
         {
            WATCH_POI tmp = arr[i];
            arr[i] = arr[j];
            arr[j] = tmp;
         }
      }
   }
}

//+------------------------------------------------------------------+
// POI finders
//+------------------------------------------------------------------+
POI_DATA FindBestOCL(ENUM_TIMEFRAMES tf)
{
   POI_DATA best;
   InitPOI(best);

   double c[], o[], h[], l[];
   ArraySetAsSeries(c, true);
   ArraySetAsSeries(o, true);
   ArraySetAsSeries(h, true);
   ArraySetAsSeries(l, true);

   int bars = OCL_Lookback + 5;

   if(CopyClose(_Symbol, tf, 0, bars, c) < bars) return best;
   if(CopyOpen (_Symbol, tf, 0, bars, o) < bars) return best;
   if(CopyHigh (_Symbol, tf, 0, bars, h) < bars) return best;
   if(CopyLow  (_Symbol, tf, 0, bars, l) < bars) return best;

   double price    = GetBid();
   double buf      = D2P(POIZoneBufDollar) * 2.0;
   double maxDist  = (MaxPOIDistDollar > 0.0) ? D2P(MaxPOIDistDollar) : DBL_MAX;
   double bestDist = DBL_MAX;

   for(int i = 1; i < OCL_Lookback; i++)
   {
      if(i + 1 >= bars)
         break;

      bool oldBear = (c[i + 1] < o[i + 1]);
      bool oldBull = (c[i + 1] > o[i + 1]);
      bool newBear = (c[i] < o[i]);
      bool newBull = (c[i] > o[i]);

      double lvl   = 0.0;
      bool bullish = false;

      if(oldBear && newBear)
      {
         lvl = c[i + 1];
         bullish = false;
      }
      else if(oldBull && newBull)
      {
         lvl = c[i + 1];
         bullish = true;
      }

      if(lvl <= 0.0) continue;

      double dist = MathAbs(price - lvl);
      if(dist > maxDist) continue;
      if(!IsFreshZone(tf, lvl + buf, lvl - buf, i)) continue;
      if(dist >= bestDist) continue;

      bestDist         = dist;
      best.zoneHigh    = lvl + buf;
      best.zoneLow     = lvl - buf;
      best.midLevel    = lvl;
      best.midLevel50  = lvl;
      best.isBullish   = bullish;
      best.isValid     = true;
      best.poiType     = "OCL";
      best.isSRType    = true;
      best.edgeLevel   = lvl;
      best.poiTF       = tf;
   }

   return best;
}

POI_DATA FindBestFVG(ENUM_TIMEFRAMES tf)
{
   POI_DATA best;
   InitPOI(best);

   double h[], l[];
   datetime t[];

   ArraySetAsSeries(h, true);
   ArraySetAsSeries(l, true);
   ArraySetAsSeries(t, true);

   if(CopyHigh(_Symbol, tf, 0, 60, h) < 60) return best;
   if(CopyLow (_Symbol, tf, 0, 60, l) < 60) return best;
   if(CopyTime(_Symbol, tf, 0, 60, t) < 60) return best;

   double price    = GetBid();
   double minFVG   = D2P(FVGMinDollar);
   double maxDist  = (MaxPOIDistDollar > 0.0) ? D2P(MaxPOIDistDollar) : DBL_MAX;
   double bestDist = DBL_MAX;

   for(int i = 2; i < 58; i++)
   {
      // Dengan array series:
      // i+2 = candle lebih lama
      // i+1 = candle tengah
      // i   = candle lebih baru

      // Bullish FVG:
      // low candle terbaru > high candle lama
      if(l[i] > h[i + 2] && (l[i] - h[i + 2]) >= minFVG)
      {
         double gH   = l[i];
         double gL   = h[i + 2];
         double mid  = gL + (gH - gL) * 0.5;
         double dist = MathAbs(price - mid);

         if(dist > maxDist) continue;
         if(!IsFreshZone(tf, gH, gL, i)) continue;
         if(dist >= bestDist) continue;

         bestDist        = dist;
         best.zoneHigh   = gH;
         best.zoneLow    = gL;
         best.midLevel   = mid;
         best.midLevel50 = mid;
         best.isBullish  = true;
         best.isValid    = true;
         best.poiType    = "FVG";
         best.isSRType   = false;
         best.edgeLevel  = mid;
         best.poiTF      = tf;
         best.originTime = t[i];
      }

      // Bearish FVG:
      // high candle terbaru < low candle lama
      if(h[i] < l[i + 2] && (l[i + 2] - h[i]) >= minFVG)
      {
         double gH   = l[i + 2];
         double gL   = h[i];
         double mid  = gL + (gH - gL) * 0.5;
         double dist = MathAbs(price - mid);

         if(dist > maxDist) continue;
         if(!IsFreshZone(tf, gH, gL, i)) continue;
         if(dist >= bestDist) continue;

         bestDist        = dist;
         best.zoneHigh   = gH;
         best.zoneLow    = gL;
         best.midLevel   = mid;
         best.midLevel50 = mid;
         best.isBullish  = false;
         best.isValid    = true;
         best.poiType    = "FVG";
         best.isSRType   = false;
         best.edgeLevel  = mid;
         best.poiTF      = tf;
         best.originTime = t[i];
      }
   }

   return best;
}


POI_DATA FindBestRBS(ENUM_TIMEFRAMES tf)
{
   POI_DATA best;
   InitPOI(best);

   double h[], l[], c[];
   ArraySetAsSeries(h, true);
   ArraySetAsSeries(l, true);
   ArraySetAsSeries(c, true);

   int bars = RBS_SBR_Lookback + 10;

   if(CopyHigh (_Symbol, tf, 0, bars, h) < bars) return best;
   if(CopyLow  (_Symbol, tf, 0, bars, l) < bars) return best;
   if(CopyClose(_Symbol, tf, 0, bars, c) < bars) return best;

   double price    = GetBid();
   double buf      = D2P(POIZoneBufDollar) * 3.0;
   double srBuf    = D2P(POIZoneBufDollar) * 4.0;
   double bestDist = DBL_MAX;

   double cands[];
   ArrayResize(cands, bars);
   int cnt = 0;
   for(int i=2; i<bars; i++) cands[cnt++] = c[i];

   for(int k=0; k<cnt; k++)
   {
      double lv = cands[k];
      if(lv <= 0.0) continue;

      bool dup=false;
      for(int m=0; m<k; m++)
      {
         if(MathAbs(cands[m]-lv) < srBuf)
         {
            dup=true; break;
         }
      }
      if(dup) continue;

      int resTouch=0;
      int breakIdx=-1;

      for(int i=bars-1; i>=2; i--)
      {
         if(c[i] < lv && h[i] >= lv - srBuf)
            resTouch++;

         if(c[i] > lv && resTouch >= MinSRTouches && breakIdx < 0)
            breakIdx = i;
      }

      if(resTouch < MinSRTouches || breakIdx < 0) continue;
      if(!IsFreshZone(tf, lv + buf, lv - buf, breakIdx)) continue;
      if(!IsWithinMaxDist(lv, price)) continue;

      double dist = MathAbs(price - lv);
      if(dist >= bestDist) continue;

      bestDist         = dist;
      best.zoneHigh    = lv + buf;
      best.zoneLow     = lv - buf;
      best.midLevel    = lv;
      best.midLevel50  = lv;
      best.isBullish   = true;
      best.isValid     = true;
      best.poiType     = "RBS";
      best.isSRType    = true;
      best.edgeLevel   = lv;
      best.poiTF       = tf;
   }

   return best;
}

POI_DATA FindBestSBR(ENUM_TIMEFRAMES tf)
{
   POI_DATA best;
   InitPOI(best);

   double h[], l[], c[];
   ArraySetAsSeries(h, true);
   ArraySetAsSeries(l, true);
   ArraySetAsSeries(c, true);

   int bars = RBS_SBR_Lookback + 10;

   if(CopyHigh (_Symbol, tf, 0, bars, h) < bars) return best;
   if(CopyLow  (_Symbol, tf, 0, bars, l) < bars) return best;
   if(CopyClose(_Symbol, tf, 0, bars, c) < bars) return best;

   double price    = GetBid();
   double buf      = D2P(POIZoneBufDollar) * 3.0;
   double srBuf    = D2P(POIZoneBufDollar) * 4.0;
   double bestDist = DBL_MAX;

   double cands[];
   ArrayResize(cands, bars);
   int cnt = 0;
   for(int i=2; i<bars; i++) cands[cnt++] = c[i];

   for(int k=0; k<cnt; k++)
   {
      double lv = cands[k];
      if(lv <= 0.0) continue;

      bool dup=false;
      for(int m=0; m<k; m++)
      {
         if(MathAbs(cands[m]-lv) < srBuf)
         {
            dup=true; break;
         }
      }
      if(dup) continue;

      int supTouch=0;
      int breakIdx=-1;

      for(int i=bars-1; i>=2; i--)
      {
         if(c[i] > lv && l[i] <= lv + srBuf)
            supTouch++;

         if(c[i] < lv && supTouch >= MinSRTouches && breakIdx < 0)
            breakIdx = i;
      }

      if(supTouch < MinSRTouches || breakIdx < 0) continue;
      if(!IsFreshZone(tf, lv + buf, lv - buf, breakIdx)) continue;
      if(!IsWithinMaxDist(lv, price)) continue;

      double dist = MathAbs(price - lv);
      if(dist >= bestDist) continue;

      bestDist         = dist;
      best.zoneHigh    = lv + buf;
      best.zoneLow     = lv - buf;
      best.midLevel    = lv;
      best.midLevel50  = lv;
      best.isBullish   = false;
      best.isValid     = true;
      best.poiType     = "SBR";
      best.isSRType    = true;
      best.edgeLevel   = lv;
      best.poiTF       = tf;
   }

   return best;
}

POI_DATA FindBestSR(ENUM_TIMEFRAMES tf)
{
   POI_DATA best;
   InitPOI(best);

   double c[], o[], h[], l[];
   ArraySetAsSeries(c, true);
   ArraySetAsSeries(o, true);
   ArraySetAsSeries(h, true);
   ArraySetAsSeries(l, true);

   int bars = SwingLookback * 3;
   if(bars < 20) bars = 20;

   if(CopyClose(_Symbol, tf, 0, bars, c) < bars) return best;
   if(CopyOpen (_Symbol, tf, 0, bars, o) < bars) return best;
   if(CopyHigh (_Symbol, tf, 0, bars, h) < bars) return best;
   if(CopyLow  (_Symbol, tf, 0, bars, l) < bars) return best;

   double price = GetBid();
   double buf   = D2P(POIZoneBufDollar) * 3.0;
   double srBuf = D2P(POIZoneBufDollar) * 5.0;

   double avgBody = 0.0;
   for(int i=1; i<bars; i++)
      avgBody += MathAbs(c[i]-o[i]);
   avgBody /= (bars-1);

   double bestLv   = 0.0;
   double bestD    = DBL_MAX;
   bool   bestBull = false;
   double bestWick = 0.0;

   for(int i=2; i<bars-1; i++)
   {
      double bodyOld = MathAbs(c[i+1]-o[i+1]);
      double bodyNew = MathAbs(c[i]-o[i]);

      if(bodyOld < avgBody * 0.3 || bodyNew < avgBody * 0.3)
         continue;

      bool oldBear = (c[i+1] < o[i+1]);
      bool oldBull = (c[i+1] > o[i+1]);
      bool newBull = (c[i]   > o[i]);
      bool newBear = (c[i]   < o[i]);

      double lvl = 0.0;
      bool bullish = false;

      if(oldBear && newBull)
      {
         lvl = c[i+1];
         bullish = true;
      }
      else if(oldBull && newBear)
      {
         lvl = c[i+1];
         bullish = false;
      }

      if(lvl <= 0.0) continue;

      double dist = MathAbs(price - lvl);
      if(dist >= bestD) continue;
      if(!IsWithinMaxDist(lvl, price)) continue;
      if(!IsFreshZone(tf, lvl + buf, lvl - buf, i)) continue;

      double wickExt = bullish ? MathMin(l[i+1], l[i]) : MathMax(h[i+1], h[i]);

      bestD    = dist;
      bestLv   = lvl;
      bestBull = bullish;
      bestWick = wickExt;
   }

   double cands[];
   ArrayResize(cands, bars);
   int cnt = 0;
   for(int i=2; i<bars; i++) cands[cnt++] = c[i];

   for(int k=0; k<cnt; k++)
   {
      double lv = cands[k];
      if(lv <= 0.0) continue;

      bool dup=false;
      for(int m=0; m<k; m++)
      {
         if(MathAbs(cands[m]-lv) < srBuf)
         {
            dup=true; break;
         }
      }
      if(dup) continue;

      int touches=0;
      int recentTouchIdx=-1;
      double wickLow=lv;
      double wickHigh=lv;

      for(int i=1; i<bars; i++)
      {
         bool touch = (l[i] <= lv + srBuf && h[i] >= lv - srBuf);
         if(touch)
         {
            touches++;
            if(recentTouchIdx < 0 || i < recentTouchIdx)
               recentTouchIdx = i;

            wickLow  = MathMin(wickLow, l[i]);
            wickHigh = MathMax(wickHigh, h[i]);
         }
      }

      if(touches < MinSRTouches || recentTouchIdx < 1) continue;

      double dist = MathAbs(price - lv);
      if(dist >= bestD) continue;
      if(!IsWithinMaxDist(lv, price)) continue;
      if(!IsFreshZone(tf, lv + buf, lv - buf, recentTouchIdx)) continue;

      bool bullish = (price > lv);
      double wickExt = bullish ? wickLow : wickHigh;

      bestD    = dist;
      bestLv   = lv;
      bestBull = bullish;
      bestWick = wickExt;
   }

   if(bestLv > 0.0)
   {
      double mid50 = bestBull
                     ? bestLv - (bestLv - bestWick) * 0.5
                     : bestLv + (bestWick - bestLv) * 0.5;

      best.zoneHigh      = bestBull ? (bestLv + buf)   : (bestWick + buf);
      best.zoneLow       = bestBull ? (bestWick - buf) : (bestLv - buf);
      best.midLevel      = bestLv;
      best.midLevel50    = mid50;
      best.isBullish     = bestBull;
      best.isValid       = true;
      best.poiType       = "SR";
      best.isSRType      = true;
      best.edgeLevel     = bestBull ? best.zoneLow : best.zoneHigh;
      best.poiTF         = tf;
   }

   return best;
}

//+------------------------------------------------------------------+
// Reset / daily
//+------------------------------------------------------------------+
//==================== PART 6/8 ====================
// lines 1750-2100
//+------------------------------------------------------------------+
void CheckDailyReset()
{
   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);

   datetime today = StringToTime(StringFormat("%04d.%02d.%02d 00:00", dt.year, dt.mon, dt.day));

   if(today > LastTradeDate)
   {
      TradesToday   = 0;
      LastTradeDate = today;
      Print("Daily reset.");
   }
}

void ResetAll()
{
   InitPOI(activePOI);
   InitOB(activeOB);
   InitSR(activeSR);
   ClearWatchPOIs();

   PendingTicket       = 0;
   TapTime             = 0;
   TapM1BarOpen        = 0;
   OB_Timeout          = 0;
   SR_Deadline         = 0;
   lastM5CheckBar      = 0;
   lastM1CheckBar      = 0;
   M5SkipCount         = 0;

   State               = STATE_SCAN_HTF;
   ForceHTFRescan      = true;
   WatchlistDirty      = false;
   lastWatchRefreshM1  = 0;

   SetStatusLabel("SCAN_HTF");

   if(EnableVisualBoxes)
   {
      ClearWatchObjects();
      ClearSetupObjects();
   }
}

//+------------------------------------------------------------------+
// Init / Tick / Deinit
//+------------------------------------------------------------------+
int OnInit()
{
   int digits = (int)SymbolInfoInteger(_Symbol, SYMBOL_DIGITS);

   if(digits == 2)            Pip = 0.10;
   else if(digits == 3)       Pip = 1.0;
   else if(digits == 4 || digits == 5) Pip = 10.0 * _Point;
   else                       Pip = 10.0 * _Point;

   double tickVal  = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   double tickSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   PipVal = (tickSize > 0.0) ? (Pip / tickSize) * tickVal : tickVal;

   trade.SetExpertMagicNumber(MagicNumber);
   trade.SetDeviationInPoints(20);
   trade.SetTypeFilling(ORDER_FILLING_IOC);

   for(int i=0; i<ELM_MAX_BLOCKED_ZONES; i++)
      InitBlocked(blockedZones[i]);

   ResetAll();

   Print("Elmethod EA initialized | Symbol:", _Symbol,
         " Digits:", digits,
         " Pip:", DoubleToString(Pip, _Digits),
         " PipVal:", DoubleToString(PipVal, 2));
   if(GainReferenceBalance > 0.0)
      StartBalanceRef = GainReferenceBalance;
   else
      StartBalanceRef = AccountInfoDouble(ACCOUNT_BALANCE);

   CreateStatsPanel();
   UpdateStatsPanel();

   return(INIT_SUCCEEDED);
}

void OnTick()
{
UpdateStatsPanel();

if(!EAEnabled)
   return;
   
   CheckDailyReset();
   CleanupBlockedZones();
   RedrawVisuals();

   if(UseSessionFilter && !IsInSession())
      return;

   if(TradesToday >= MaxTradesPerDay)
      return;

   if(HasOpenPosition())
   {
      RedrawVisuals();
      return;
   }

   switch(State)
   {
      case STATE_SCAN_HTF:     ScanHTF_POI();   break;
      case STATE_WAIT_ANY_TAP: CheckAnyTap();   break;
      case STATE_WAIT_M1_OB:   CheckM1_OB();    break;
      case STATE_WAIT_M5_SR:   CheckM5_SR();    break;
      case STATE_ORDER_PLACED: ManagePending(); break;
   }

   RedrawVisuals();
}

void OnDeinit(const int reason)
{
   DeleteStatsPanel();
   ClearAllVisualObjects();
   Print("Elmethod EA deinitialized");
}

void OnChartEvent(const int id,
                  const long &lparam,
                  const double &dparam,
                  const string &sparam)
{
   if(id == CHARTEVENT_OBJECT_CLICK)
   {
      if(sparam == PANEL_BTN_EA)
      {
         EAEnabled = !EAEnabled;
         UpdateStatsPanel();
         Print("EA Toggle: ", (EAEnabled ? "ON" : "OFF"));
      }
   }
}


//+------------------------------------------------------------------+
// STEP 1: build watchlist
// W1 -> D1 -> H4 -> H1
// H1 OCL diprioritaskan dulu
// max total 5
//+------------------------------------------------------------------+
void ScanHTF_POI()
{
   datetime barM1 = CurrentM1BarOpen();
   if(!ForceHTFRescan && barM1 == LastScanM1Bar)
      return;

   LastScanM1Bar   = barM1;
   ForceHTFRescan  = false;
   WatchlistDirty  = false;

   ClearWatchPOIs();

   WATCH_POI temp[ELM_MAX_TEMP_POIS];
   int tempCount = 0;
   for(int i=0; i<ELM_MAX_TEMP_POIS; i++)
      InitWatch(temp[i]);

   // reserve H1 OCL first
   POI_DATA h1OCL;
   InitPOI(h1OCL);
   if(UseH1)
      h1OCL = FindBestOCL(PERIOD_H1);

   // scan higher TF first
   ENUM_TIMEFRAMES allTFs[4] = { PERIOD_W1, PERIOD_D1, PERIOD_H4, PERIOD_H1 };
   bool tfEnabled[4]         = { UseWeekly, UseDaily, UseH4, UseH1 };

   for(int t=0; t<4; t++)
   {
      if(!tfEnabled[t]) continue;
      ENUM_TIMEFRAMES tf = allTFs[t];

      POI_DATA poi;

      // avoid double-add H1 OCL here because reserved separately
      if(!(tf == PERIOD_H1))
      {
         poi = FindBestOCL(tf); AddTempPOI(temp, tempCount, poi);
      }

      poi = FindBestFVG(tf); AddTempPOI(temp, tempCount, poi);
      poi = FindBestSR(tf);  AddTempPOI(temp, tempCount, poi);
      poi = FindBestRBS(tf); AddTempPOI(temp, tempCount, poi);
      poi = FindBestSBR(tf); AddTempPOI(temp, tempCount, poi);
   }

   SortTempPOIs(temp, tempCount);

   // slot 1 reserved for H1 OCL if valid
   if(h1OCL.isValid)
      AddWatchDirect(h1OCL);

   for(int i=0; i<tempCount && watchCount < ELM_MAX_WATCH_POIS; i++)
   {
      if(temp[i].valid)
         AddWatchDirect(temp[i].poi);
   }

   if(watchCount <= 0)
   {
      SetRejectLabel("Tidak ada watch POI valid.");
      SetStatusLabel("SCAN_HTF | no POI");
      return;
   }

   lastWatchRefreshM1 = CurrentM1BarOpen();

   SortTempPOIs(watchPOIs, watchCount); // safe because struct same shape
   DrawAllWatchPOIs();

   SetStatusLabel("WAIT_ANY_TAP | watchCount=" + IntegerToString(watchCount));
   State = STATE_WAIT_ANY_TAP;

   if(DebugScan)
   {
      for(int i=0; i<watchCount; i++)
      {
         if(!watchPOIs[i].valid) continue;
         DebugPrint("WATCH " + IntegerToString(i+1) + " | " +
                    watchPOIs[i].poi.poiType + " " + EnumToString(watchPOIs[i].poi.poiTF) +
                    " | mid=" + DoubleToString(watchPOIs[i].poi.midLevel, 2) +
                    " | dist$=" + DoubleToString(P2D(watchPOIs[i].dist), 2));
      }
   }
}

//+------------------------------------------------------------------+
// STEP 2: first tapped POI wins
//+------------------------------------------------------------------+
void CheckAnyTap()
{
   // rebuild watchlist hanya di fase tunggu tap
   if(WatchlistDirty || NeedRefreshWatchlistByM1())
   {
      if(DebugScan)
      {
         string why = WatchlistDirty ? "dirty" : "new M1 bar";
         DebugPrint("Rebuild watchlist because: " + why);
      }

      WatchlistDirty = false;
      ForceHTFRescan = true;
      State = STATE_SCAN_HTF;
      return;
   }

   if(watchCount <= 0)
   {
      ResetAll();
      return;
   }

   SetStatusLabel("WAIT_ANY_TAP | watchCount=" + IntegerToString(watchCount));

   double bid = GetBid();
   double ask = GetAsk();

   double hi0 = iHigh(_Symbol, PERIOD_M1, 0);
   double lo0 = iLow (_Symbol, PERIOD_M1, 0);

   if(hi0 <= 0.0) hi0 = ask;
   if(lo0 <= 0.0) lo0 = bid;

   int    tappedIdx   = -1;
   double bestScore   = DBL_MAX;
   int    bestTFPrio  = -1;
   int    bestTypePrio= -1;
   int    validRemain = 0;

for(int i = 0; i < watchCount; i++)
{
   if(!watchPOIs[i].valid)
      continue;

   POI_DATA poi = watchPOIs[i].poi;

   if(poi.poiType == "FVG")
   {
      if(HasZoneBeenTouchedBeforeTime(poi, TimeCurrent(), PERIOD_M5))
      {
         watchPOIs[i].valid = false;
         BlockZoneUntilH1Close(poi, "FVG already touched");
         WatchlistDirty = true;
         continue;
      }
   }

   if(poi.poiType == "FVG" && IsFVGInvalidatedByBreak(poi, bid, ask))
   {
      watchPOIs[i].valid = false;
      BlockZoneUntilH1Close(poi, "FVG broken through opposite side");
      WatchlistDirty = true;
      continue;
   }


      // invalidasi kasar kalau harga sudah terlalu jauh lewat zone
      if(( ResolvePOIDirectionByLocation(poi, bid) && bid < poi.zoneLow - D2P(100.0)) ||
         (!ResolvePOIDirectionByLocation(poi, bid) && ask > poi.zoneHigh + D2P(100.0)))
      {
         watchPOIs[i].valid = false;
         WatchlistDirty = true;
         continue;
      }

      validRemain++;

bool forBuy = (poi.poiType == "FVG") ? poi.isBullish
                                     : ResolvePOIDirectionByLocation(poi, bid);
      double tapScore = DBL_MAX;

      if(!IsPOITappedNow(poi, bid, ask, hi0, lo0, forBuy, tapScore))
         continue;

      if(!IsZoneStillFreshForTap(poi.poiTF, poi.zoneHigh, poi.zoneLow))
      {
         BlockZoneUntilH1Close(poi, "tap stale");
         watchPOIs[i].valid = false;
         WatchlistDirty = true;
         continue;
      }

      int tfPrio   = TFPriority(poi.poiTF);
      int typePrio = TypePriority(poi.poiType);

      bool takeThis = false;

      if(tappedIdx < 0)
      {
         takeThis = true;
      }
      else if(tapScore < bestScore - (_Point * 2.0))
      {
         takeThis = true;
      }
      else if(MathAbs(tapScore - bestScore) <= (_Point * 2.0))
      {
         if(tfPrio > bestTFPrio)
            takeThis = true;
         else if(tfPrio == bestTFPrio && typePrio > bestTypePrio)
            takeThis = true;
      }

      if(takeThis)
      {
         tappedIdx    = i;
         bestScore    = tapScore;
         bestTFPrio   = tfPrio;
         bestTypePrio = typePrio;
      }
   }

   if(tappedIdx >= 0)
   {
activePOI = watchPOIs[tappedIdx].poi;

if(activePOI.poiType != "FVG")
   activePOI.isBullish = ResolvePOIDirectionByLocation(activePOI, bid);


      // final direction harus ikut lokasi zone terhadap harga saat tap
      activePOI.isBullish = ResolvePOIDirectionByLocation(activePOI, bid);

      TapTime      = TimeCurrent();
      TapM1BarOpen = iTime(_Symbol, PERIOD_M1, 0);
      OB_Timeout   = TapTime + (OB_TimeoutMin * 60);

      ClearWatchPOIs();
      ClearWatchObjects();
      DrawActivePOI();

      State = STATE_WAIT_M1_OB;
      SetStatusLabel("WAIT_M1_OB | tapped " + activePOI.poiType + " " + EnumToString(activePOI.poiTF));

      Print("Tapped first: ", activePOI.poiType, " ", EnumToString(activePOI.poiTF),
            " Mid:", DoubleToString(activePOI.midLevel, 2),
            " Dir:", (activePOI.isBullish ? "BUY" : "SELL"));
      return;
   }

   if(validRemain <= 0)
   {
      SetRejectLabel("Semua watch POI invalid. Rescan.");
//==================== PART 7/8 ====================
// lines 2100-2450
//+------------------------------------------------------------------+
      ResetAll();
   }
}

//+------------------------------------------------------------------+
// STEP 3: lock POI, wait OB+FVG max 5 candle M1
// OB buy  = last bearish candle before bullish displacement + bullish FVG
// OB sell = last bullish candle before bearish displacement + bearish FVG
//+------------------------------------------------------------------+
void CheckM1_OB()
{
   if(!activePOI.isValid)
   {
      ResetAll();
      return;
   }

   // kalau FVG ini ternyata sudah pernah disentuh SEBELUM tap sekarang, buang
   if(activePOI.poiType == "FVG")
   {
      if(HasZoneBeenTouchedBeforeTime(activePOI, TapTime, PERIOD_M5))
      {
         SetRejectLabel("FVG already touched before this tap.");
         BlockZoneUntilH1Close(activePOI, "FVG already touched");
         ResetAll();
         return;
      }
   }

   SetStatusLabel("WAIT_M1_OB | " + activePOI.poiType + " " + EnumToString(activePOI.poiTF));

int barsPassed = CountM1BarsSinceTap();

if(barsPassed > M1ConfirmBarsMax)
{
   SetRejectLabel("OB timeout by M1 bars: " + IntegerToString(barsPassed));
   BlockZoneUntilH1Close(activePOI, "no OB+FVG after 50% tap in " + IntegerToString(M1ConfirmBarsMax) + " M1 bars");
   ResetAll();
   return;
}


   datetime curM1Bar[];
   ArraySetAsSeries(curM1Bar, true);

   if(CopyTime(_Symbol, PERIOD_M1, 0, 1, curM1Bar) < 1)
      return;

   if(curM1Bar[0] == lastM1CheckBar)
      return;

   lastM1CheckBar = curM1Bar[0];

   double h[], l[], c[], o[];
   datetime bt[];

   ArraySetAsSeries(h, true);
   ArraySetAsSeries(l, true);
   ArraySetAsSeries(c, true);
   ArraySetAsSeries(o, true);
   ArraySetAsSeries(bt, true);

   int bars = OB_Lookback + 10;

   if(CopyHigh (_Symbol, PERIOD_M1, 0, bars, h ) < bars) return;
   if(CopyLow  (_Symbol, PERIOD_M1, 0, bars, l ) < bars) return;
   if(CopyClose(_Symbol, PERIOD_M1, 0, bars, c ) < bars) return;
   if(CopyOpen (_Symbol, PERIOD_M1, 0, bars, o ) < bars) return;
   if(CopyTime (_Symbol, PERIOD_M1, 0, bars, bt) < bars) return;

   double minFVG = D2P(M1_FVGMinDollar);

   for(int ob = 3; ob < bars; ob++)
   {
      if(ob - 2 < 0)
         continue;

      // candle yang sepenuhnya selesai sebelum tap dibuang
      if((bt[ob] + PeriodSeconds(PERIOD_M1)) <= TapTime)
         continue;

      int newer1 = ob - 1;
      int newer2 = ob - 2;

      if(activePOI.isBullish)
      {
         // BUY = last bearish candle before bullish displacement + bullish FVG
         if(!(c[ob] < o[ob]))
            continue;

         bool displacement = (c[newer1] > o[newer1] || c[newer2] > o[newer2]);
         if(!displacement)
            continue;

         if(!(l[newer2] > h[ob]))
            continue;

         double fvgSz = l[newer2] - h[ob];
         if(fvgSz < minFVG)
            continue;

         activeOB.obHigh    = h[ob];
         activeOB.obLow     = l[ob];
         activeOB.isBullish = true;
         activeOB.isValid   = true;
         activeOB.obTime    = bt[ob];

         Print("OB BUY M1 valid | barsPassed=", barsPassed,
               " OBTime:", TimeToString(bt[ob]),
               " FVG:", DoubleToString(fvgSz, 2));

         DrawOBBox();
         State = STATE_WAIT_M5_SR;
         SetStatusLabel("WAIT_M5_SR | OB BUY valid");
         return;
      }
      else
      {
         // SELL = last bullish candle before bearish displacement + bearish FVG
         if(!(c[ob] > o[ob]))
            continue;

         bool displacement = (c[newer1] < o[newer1] || c[newer2] < o[newer2]);
         if(!displacement)
            continue;

         if(!(h[newer2] < l[ob]))
            continue;

         double fvgSz = l[ob] - h[newer2];
         if(fvgSz < minFVG)
            continue;

         activeOB.obHigh    = h[ob];
         activeOB.obLow     = l[ob];
         activeOB.isBullish = false;
         activeOB.isValid   = true;
         activeOB.obTime    = bt[ob];

         Print("OB SELL M1 valid | barsPassed=", barsPassed,
               " OBTime:", TimeToString(bt[ob]),
               " FVG:", DoubleToString(fvgSz, 2));

         DrawOBBox();
         State = STATE_WAIT_M5_SR;
         SetStatusLabel("WAIT_M5_SR | OB SELL valid");
         return;
      }
   }
}


//+------------------------------------------------------------------+
// STEP 4: M5 entry
// candle ke-3 tetap wajib via pending deadline
//+------------------------------------------------------------------+
void CheckM5_SR()
{
   if(!activeOB.isValid)
   {
      ResetAll();
      return;
   }

   SetStatusLabel("WAIT_M5_SR");

   if(activeSR.isValid)
      return;

   datetime curBar[];
   ArraySetAsSeries(curBar, true);
   if(CopyTime(_Symbol, PERIOD_M5, 0, 1, curBar) < 1)
      return;
   if(curBar[0] == lastM5CheckBar)
      return;
   lastM5CheckBar = curBar[0];

if(M5SkipCount > 3)
{
   SetRejectLabel("M5 entry timeout.");
   BlockZoneUntilH1Close(activePOI, "M5 no confirmation after 50% tap");
   ResetAll();
   return;
}


   double h[], l[], c[], o[];
   datetime t[];

   ArraySetAsSeries(h, true);
   ArraySetAsSeries(l, true);
   ArraySetAsSeries(c, true);
   ArraySetAsSeries(o, true);
   ArraySetAsSeries(t, true);

   if(CopyHigh (_Symbol, PERIOD_M5, 0, 6, h) < 6) return;
   if(CopyLow  (_Symbol, PERIOD_M5, 0, 6, l) < 6) return;
   if(CopyClose(_Symbol, PERIOD_M5, 0, 6, c) < 6) return;
   if(CopyOpen (_Symbol, PERIOD_M5, 0, 6, o) < 6) return;
   if(CopyTime (_Symbol, PERIOD_M5, 0, 6, t) < 6) return;

   double slBuf = D2P(SLBufferDollar);

   if(activeOB.isBullish)
   {
      bool c1Bear = (c[2] < o[2]);
      bool c2Bull = (c[1] > o[1]);
      if(!c1Bear || !c2Bull)
         return;

      double entry    = o[1];
      double worstLow = MathMin(l[1], l[2]);
      double slPrice  = worstLow - slBuf;
      double slDist   = entry - slPrice;

      if(slDist <= 0.0)
      {
         SetRejectLabel("M5 Buy reject: SL >= Entry.");
         return;
      }

double minTPDist = slDist * MinRR;
double tpPrice   = FindNearestLiquidityStrict(true, entry, minTPDist);

      if(tpPrice <= entry)
      {
         SetRejectLabel("M5 Buy reject: TP tidak ditemukan.");
         return;
      }

      activeSR.entryPrice = entry;
      activeSR.slPrice    = slPrice;
      activeSR.tpPrice    = tpPrice;
      activeSR.isBullish  = true;
      activeSR.isValid    = true;
      activeSR.setupTime  = TimeCurrent();
      M5SkipCount         = 0;

      Print("M5 BUY | Entry:", DoubleToString(entry, 2),
            " SL:", DoubleToString(slPrice, 2),
            " TP:", DoubleToString(tpPrice, 2),
            " RR:", DoubleToString((tpPrice - entry) / slDist, 1));

      PlaceLimitOrder(t[0]);
   }
   else
   {
      bool c1Bull = (c[2] > o[2]);
      bool c2Bear = (c[1] < o[1]);
      if(!c1Bull || !c2Bear)
         return;

      double entry     = o[1];
      double worstHigh = MathMax(h[1], h[2]);
      double slPrice   = worstHigh + slBuf;
      double slDist    = slPrice - entry;

      if(slDist <= 0.0)
      {
         SetRejectLabel("M5 Sell reject: SL <= Entry.");
         return;
      }

double minTPDist = slDist * MinRR;
double tpPrice   = FindNearestLiquidityStrict(false, entry, minTPDist);


      if(tpPrice >= entry)
      {
         SetRejectLabel("M5 Sell reject: TP tidak ditemukan.");
         return;
      }

      activeSR.entryPrice = entry;
      activeSR.slPrice    = slPrice;
      activeSR.tpPrice    = tpPrice;
      activeSR.isBullish  = false;
      activeSR.isValid    = true;
      activeSR.setupTime  = TimeCurrent();
      M5SkipCount         = 0;

      Print("M5 SELL | Entry:", DoubleToString(entry, 2),
            " SL:", DoubleToString(slPrice, 2),
            " TP:", DoubleToString(tpPrice, 2),
            " RR:", DoubleToString((entry - tpPrice) / slDist, 1));

      PlaceLimitOrder(t[0]);
   }
}

//+------------------------------------------------------------------+
// TP liquidity
//+------------------------------------------------------------------+
double FindNearestLiquidityStrict(bool forBuy, double fromPrice, double minDist)
{
   ENUM_TIMEFRAMES tfs[4];
   tfs[0] = PERIOD_M5;
   tfs[1] = PERIOD_M15;
   tfs[2] = PERIOD_H1;
   tfs[3] = PERIOD_H4;

   double bestLv = 0.0;
   double bestD  = DBL_MAX;
   int    bestScore = -1;

   double eqBuf = D2P(EQH_EQL_BufDollar);
   int sw = 3;

   for(int t = 0; t < 4; t++)
   {
      double h[], l[];
      ArraySetAsSeries(h, true);
      ArraySetAsSeries(l, true);

      int bars = MathMax(EQH_EQL_Lookback, 80);

      if(CopyHigh(_Symbol, tfs[t], 0, bars, h) < bars) continue;
      if(CopyLow (_Symbol, tfs[t], 0, bars, l) < bars) continue;

      for(int i = sw + 2; i < bars - sw - 2; i++)
      {
         double level = 0.0;
         double dist  = 0.0;
         int score    = -1;

         if(forBuy)
         {
            level = h[i];
            if(level <= fromPrice)
               continue;

            dist = level - fromPrice;
            if(dist < minDist)
               continue;

            bool eq    = HasEqualHigh(h, i, eqBuf);
            bool swing = IsClearSwingHigh(h, i, sw);
            bool fresh = IsUnsweptHigh(h, i, level, eqBuf);

            if(!fresh)
               continue;

            if(eq)         score = 3;   // BSL / EQH prioritas tertinggi
            else if(swing) score = 2;   // swing high jelas
            else           continue;
         }
         else
         {
            level = l[i];
            if(level >= fromPrice)
               continue;

            dist = fromPrice - level;//==================== PART 8/8 ====================
// lines 2450-2695
//+------------------------------------------------------------------+
            if(dist < minDist)
               continue;

            bool eq    = HasEqualLow(l, i, eqBuf);
            bool swing = IsClearSwingLow(l, i, sw);
            bool fresh = IsUnsweptLow(l, i, level, eqBuf);

            if(!fresh)
               continue;

            if(eq)         score = 3;   // SSL / EQL prioritas tertinggi
            else if(swing) score = 2;   // swing low jelas
            else           continue;
         }

         if(score > bestScore || (score == bestScore && dist < bestD))
         {
            bestScore = score;
            bestD     = dist;
            bestLv    = level;
         }
      }
   }

   if(bestLv > 0.0)
      Print("TP Strict Liquidity:", DoubleToString(bestLv, 2),
            " dist:", DoubleToString(bestD, 2),
            " score:", bestScore);
   else
      Print("TP Strict Liquidity: tidak ditemukan.");

   return bestLv;
}

//+------------------------------------------------------------------+
// Order / pending
//+------------------------------------------------------------------+
double CalculateLot(double entry, double sl)
{
   double bal  = AccountInfoDouble(ACCOUNT_BALANCE);
   double risk = bal * RiskPercent / 100.0;

   double slDist = MathAbs(entry - sl);
   if(slDist <= 0.0)
      return MinLot;

   double tv   = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   double ts   = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   double step = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);

   if(ts <= 0.0 || tv <= 0.0 || step <= 0.0)
      return MinLot;

   double lot = risk / ((slDist / ts) * tv);

   lot = MathFloor(lot / step) * step;
   lot = MathMax(MinLot, MathMin(MaxLot, lot));

   return NormalizeDouble(lot, 2);
}

void PlaceLimitOrder(datetime candle3Open)
{
   if(!activeSR.isValid)
      return;

   double lot = CalculateLot(activeSR.entryPrice, activeSR.slPrice);
   if(lot <= 0.0)
   {
      SetRejectLabel("Lot invalid.");
      ResetAll();
      return;
   }

   double e  = NormalizeDouble(activeSR.entryPrice, _Digits);
   double s  = NormalizeDouble(activeSR.slPrice, _Digits);
   double tp = NormalizeDouble(activeSR.tpPrice, _Digits);

   if(activeSR.isBullish && s >= e)
   {
      SetRejectLabel("SL tidak valid (SL>=Entry).");
      ResetAll();
      return;
   }

   if(!activeSR.isBullish && s <= e)
   {
      SetRejectLabel("SL tidak valid (SL<=Entry).");
      ResetAll();
      return;
   }

   bool ok = false;

   if(activeSR.isBullish)
      ok = trade.BuyLimit(lot, e, _Symbol, s, tp, ORDER_TIME_GTC, 0, "Elmethod_Buy");
   else
      ok = trade.SellLimit(lot, e, _Symbol, s, tp, ORDER_TIME_GTC, 0, "Elmethod_Sell");

   if(ok)
   {
      PendingTicket = trade.ResultOrder();
      SR_Deadline   = candle3Open + PeriodSeconds(PERIOD_M5);
      State         = STATE_ORDER_PLACED;
      SetStatusLabel("ORDER_PLACED");

      Print("ORDER ", (activeSR.isBullish ? "BUY" : "SELL"),
            " E:", DoubleToString(e, _Digits),
            " SL:", DoubleToString(s, _Digits),
            " TP:", DoubleToString(tp, _Digits),
            " Lot:", DoubleToString(lot, 2),
            " | Deadline:", TimeToString(SR_Deadline));

      DrawTradeLevels();
   }
   else
   {
      SetRejectLabel("ORDER GAGAL: " + trade.ResultRetcodeDescription());
      ResetAll();
   }
}

bool PlaceFallbackMarketOrder()
{
   if(!activeSR.isValid)
      return false;

   double entry = activeSR.isBullish ? GetAsk() : GetBid();
   double slDist = FallbackSL_Pips * Pip;
   double tpDist = FallbackTP_Pips * Pip;

   double sl = 0.0;
   double tp = 0.0;

   if(activeSR.isBullish)
   {
      sl = entry - slDist;
      tp = entry + tpDist;
   }
   else
   {
      sl = entry + slDist;
      tp = entry - tpDist;
   }

   entry = NormalizeDouble(entry, _Digits);
   sl    = NormalizeDouble(sl, _Digits);
   tp    = NormalizeDouble(tp, _Digits);

   double lot = CalculateLot(entry, sl);
   if(lot <= 0.0)
   {
      SetRejectLabel("Fallback market gagal: lot invalid.");
      return false;
   }

   bool ok = false;

   if(activeSR.isBullish)
      ok = trade.Buy(lot, _Symbol, 0.0, sl, tp, "Elmethod_Fallback_Buy");
   else
      ok = trade.Sell(lot, _Symbol, 0.0, sl, tp, "Elmethod_Fallback_Sell");

   if(ok)
   {
      Print("Fallback MARKET ", (activeSR.isBullish ? "BUY" : "SELL"),
            " | Entry:", DoubleToString(entry, _Digits),
            " SL:", DoubleToString(sl, _Digits),
            " TP:", DoubleToString(tp, _Digits),
            " Lot:", DoubleToString(lot, 2));

      activeSR.entryPrice = entry;
      activeSR.slPrice    = sl;
      activeSR.tpPrice    = tp;

      DrawTradeLevels();
      return true;
   }

   SetRejectLabel("Fallback market gagal: " + trade.ResultRetcodeDescription());
   return false;
}

void ManagePending()
{
   if(PendingTicket == 0)
   {
      ResetAll();
      return;
   }

   SetStatusLabel("ORDER_PLACED");

   if(HasOpenPosition())
   {
      TradesToday++;
      Print("TEREKSEKUSI! Trades:", TradesToday);
      PendingTicket = 0;
      ResetAll();
      return;
   }

   bool exists = false;
   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(orderInfo.SelectByIndex(i))
      {
         if(orderInfo.Ticket() == PendingTicket)
         {
            exists = true;
            break;
         }
      }
   }

   if(!exists)
   {
      PendingTicket = 0;
      ResetAll();
      return;
   }

   if(TimeCurrent() >= SR_Deadline)
   {
      trade.OrderDelete(PendingTicket);
      PendingTicket = 0;

      if(UseM5FallbackMarketOn4th)
      {
         SetRejectLabel("Limit miss candle ke-3, fallback market candle ke-4.");
         
         if(PlaceFallbackMarketOrder())
         {
            State = STATE_ORDER_PLACED;
            return;
         }
      }

      SetRejectLabel("Order batal: candle ke-3 M5 tutup tanpa fill.");
      ResetAll();
      return;
   }
}

//+------------------------------------------------------------------+
