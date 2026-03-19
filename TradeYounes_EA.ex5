//+------------------------------------------------------------------+
//|  TradeYounes AI — Expert Advisor v2.0                            |
//|  Fetches signals from Railway API                                 |
//|  Validates license every 24h — stops automatically if expired    |
//|  Manages TP1→BE / TP2→Trail / TP3→Close                         |
//+------------------------------------------------------------------+
#property copyright "TradeYounes AI — tradeyounes.com"
#property version   "2.00"
#property strict

#include <Trade\Trade.mqh>
#include <Trade\PositionInfo.mqh>

//──────────────────────────────────────────────────────────────────
// INPUTS
//──────────────────────────────────────────────────────────────────
input string   LicenseKey      = "";    // Your Elite License Key (TY-XXXX-XXXX)
input string   ServerURL       = "https://web-production-80604.up.railway.app";
input double   RiskPercent     = 1.0;  // Risk per trade (% of balance)
input int      CheckInterval   = 60;   // Seconds between signal checks
input int      Slippage        = 10;   // Max slippage (points)
input bool     EnableTrading   = true; // Master on/off switch
input bool     PushAlerts      = true; // MT5 push notifications

//──────────────────────────────────────────────────────────────────
// GLOBALS
//──────────────────────────────────────────────────────────────────
CTrade        Trade;
CPositionInfo PosInfo;

datetime g_lastSignalCheck = 0;
datetime g_lastValidation  = 0;
bool     g_licenseValid    = false;
string   g_processedIDs[];

//──────────────────────────────────────────────────────────────────
// INIT
//──────────────────────────────────────────────────────────────────
int OnInit()
{
   if(LicenseKey == "") {
      Alert("❌ No license key entered.\nEnter your Elite license key in EA inputs.");
      return INIT_PARAMETERS_INCORRECT;
   }

   Trade.SetDeviationInPoints(Slippage);
   Trade.SetTypeFilling(ORDER_FILLING_IOC);
   Trade.LogLevel(LOG_LEVEL_ERRORS);

   Print("🔐 Validating license...");
   g_licenseValid = ValidateLicense();
   if(!g_licenseValid) return INIT_FAILED;

   g_lastValidation  = TimeCurrent();
   g_lastSignalCheck = 0;

   string msg = "✅ TradeYounes EA ONLINE\n"
                "Account: #" + IntegerToString(AccountInfoInteger(ACCOUNT_LOGIN)) + "\n"
                "Balance: "  + DoubleToString(AccountInfoDouble(ACCOUNT_BALANCE),2)
                             + " " + AccountInfoString(ACCOUNT_CURRENCY) + "\n"
                "Risk: "     + DoubleToString(RiskPercent,1) + "% per trade";
   Print(msg);
   if(PushAlerts) SendNotification("🤖 TradeYounes EA ONLINE | Acct #"
                                   + IntegerToString(AccountInfoInteger(ACCOUNT_LOGIN)));
   return INIT_SUCCEEDED;
}

//──────────────────────────────────────────────────────────────────
// DEINIT
//──────────────────────────────────────────────────────────────────
void OnDeinit(const int reason)
{
   Print("TradeYounes EA stopped. Reason: ", reason);
   if(PushAlerts) SendNotification("⚠️ TradeYounes EA OFFLINE");
}

//──────────────────────────────────────────────────────────────────
// MAIN TICK
//──────────────────────────────────────────────────────────────────
void OnTick()
{
   if(!EnableTrading) return;
   datetime now = TimeCurrent();

   // ── Re-validate every 24h ─────────────────────────────────────
   if(now - g_lastValidation >= 86400) {
      g_licenseValid = ValidateLicense();
      if(!g_licenseValid) {
         Alert("❌ License expired. Renew at tradeyounes.com");
         if(PushAlerts) SendNotification("❌ TradeYounes EA: License expired. EA stopped.");
         ExpertRemove();
         return;
      }
      g_lastValidation = now;
   }

   if(!g_licenseValid) return;

   // ── Manage open positions ──────────────────────────────────────
   ManagePositions();

   // ── Fetch new signals ──────────────────────────────────────────
   if(now - g_lastSignalCheck >= CheckInterval) {
      FetchAndExecute();
      g_lastSignalCheck = now;
   }
}

//──────────────────────────────────────────────────────────────────
// LICENSE VALIDATION — POST /ea/validate
//──────────────────────────────────────────────────────────────────
bool ValidateLicense()
{
   string url  = ServerURL + "/ea/validate";
   string acct = IntegerToString(AccountInfoInteger(ACCOUNT_LOGIN));
   string body = "{\"license_key\":\"" + LicenseKey
               + "\",\"account_number\":\"" + acct + "\"}";

   char   postData[], result[];
   string respHeaders = "Content-Type: application/json\r\n";
   StringToCharArray(body, postData, 0, StringLen(body));

   int code = WebRequest("POST", url, respHeaders, 10000, postData, result, respHeaders);

   if(code == -1) {
      int err = GetLastError();
      if(err == 4014) {
         Print("⚠️  Add URL to allowed list:\n"
               "Tools → Options → Expert Advisors → Allow WebRequest\n"
               "URL: ", ServerURL);
         Alert("⚠️  Allow WebRequest for:\n" + ServerURL +
               "\nTools → Options → Expert Advisors");
      }
      // 48h grace period if server temporarily down
      if(g_licenseValid && TimeCurrent() - g_lastValidation < 172800) {
         Print("⚠️  Server unreachable — grace period active");
         return true;
      }
      return false;
   }

   string resp = CharArrayToString(result);

   if(code != 200) {
      Print("❌ License HTTP ", code, ": ", resp);
      return false;
   }

   if(StringFind(resp, "\"valid\":true") >= 0) {
      Print("✅ License valid");
      return true;
   }

   // Extract reason from JSON
   string reason = JsonStr(resp, "reason");
   string message = JsonStr(resp, "message");

   if(message != "") Print("❌ ", message);
   else              Print("❌ License rejected: ", reason);

   if(PushAlerts && message != "") SendNotification("❌ " + message);
   return false;
}

//──────────────────────────────────────────────────────────────────
// FETCH SIGNALS — GET /ea/pending
//──────────────────────────────────────────────────────────────────
void FetchAndExecute()
{
   string url = ServerURL + "/ea/pending?license=" + LicenseKey;
   char   result[], dummy[];
   string headers = "";

   int code = WebRequest("GET", url, "", 10000, dummy, result, headers);

   if(code == -1 || code != 200) {
      if(code == 403) Print("⚠️  EA: License rejected during signal fetch");
      else if(code != -1) Print("⚠️  EA: Signal fetch HTTP ", code);
      return;
   }

   string resp = CharArrayToString(result);
   if(StringFind(resp, "\"count\":0") >= 0) return;

   ParseSignals(resp);
}

//──────────────────────────────────────────────────────────────────
// PARSE SIGNALS ARRAY
//──────────────────────────────────────────────────────────────────
void ParseSignals(string json)
{
   int pos = StringFind(json, "\"signals\":[");
   if(pos < 0) return;
   pos += 11;

   int depth = 0, objStart = -1;
   for(int i = pos; i < StringLen(json); i++) {
      ushort ch = StringGetCharacter(json, i);
      if(ch == '{') { if(depth==0) objStart=i; depth++; }
      else if(ch == '}') {
         depth--;
         if(depth == 0 && objStart >= 0) {
            ProcessSignal(StringSubstr(json, objStart, i-objStart+1));
            objStart = -1;
         }
      }
      else if(ch == ']' && depth == 0) break;
   }
}

void ProcessSignal(string obj)
{
   string signal_id = JsonStr(obj, "signal_id");
   string action    = JsonStr(obj, "action");
   string symbol    = JsonStr(obj, "symbol");
   string type_str  = JsonStr(obj, "type");
   double entry     = JsonDbl(obj, "entry");
   double sl        = JsonDbl(obj, "sl");
   double tp1       = JsonDbl(obj, "tp1");
   double tp2       = JsonDbl(obj, "tp2");
   double tp3       = JsonDbl(obj, "tp3");
   double risk_pct  = JsonDbl(obj, "risk_percent");
   int    score     = (int)JsonDbl(obj, "score");

   if(signal_id=="" || action!="OPEN_TRADE") return;
   if(IsProcessed(signal_id))               return;

   MarkProcessed(signal_id);

   if(entry==0 || sl==0 || tp1==0) {
      Print("⚠️  Invalid levels — ", signal_id); return;
   }
   if(score < 8) {
      Print("ℹ️  Score ", score, "/10 below threshold — skipped"); return;
   }

   double lot = CalcLot(symbol, entry, sl, risk_pct>0 ? risk_pct : RiskPercent);
   if(lot <= 0) { Print("❌ Lot calc failed — ", symbol); return; }

   bool ok = PlaceOrder(symbol, type_str, entry, sl, tp1, tp2, tp3, lot, signal_id);
   string msg = ok
      ? "🟡 ORDER: " + symbol + " " + type_str + " lot=" + DoubleToString(lot,2) + " score=" + IntegerToString(score)
      : "❌ ORDER FAILED: " + symbol + " err=" + IntegerToString(GetLastError());
   Print(msg);
   if(PushAlerts) SendNotification(msg);
}

//──────────────────────────────────────────────────────────────────
// LOT SIZE — (balance × risk%) / (sl_points × pip_value)
//──────────────────────────────────────────────────────────────────
double CalcLot(string symbol, double entry, double sl, double risk_pct)
{
   double balance   = AccountInfoDouble(ACCOUNT_BALANCE);
   double risk_usd  = balance * risk_pct / 100.0;
   double point     = SymbolInfoDouble(symbol, SYMBOL_POINT);
   double sl_pts    = MathAbs(entry - sl) / point;
   if(sl_pts <= 0) return 0;

   double tick_val  = SymbolInfoDouble(symbol, SYMBOL_TRADE_TICK_VALUE);
   double tick_size = SymbolInfoDouble(symbol, SYMBOL_TRADE_TICK_SIZE);
   if(tick_val<=0 || tick_size<=0) return 0;

   double pip_val = (tick_val / tick_size) * point;
   double lot     = risk_usd / (sl_pts * pip_val);

   double step = SymbolInfoDouble(symbol, SYMBOL_VOLUME_STEP);
   lot = MathFloor(lot/step)*step;
   lot = MathMax(SymbolInfoDouble(symbol,SYMBOL_VOLUME_MIN),
         MathMin(SymbolInfoDouble(symbol,SYMBOL_VOLUME_MAX), lot));

   Print("💰 ", symbol, " | bal=", DoubleToString(balance,0),
         " risk=", DoubleToString(risk_pct,1), "% ($", DoubleToString(risk_usd,2),
         ") sl_pts=", DoubleToString(sl_pts,0), " → lot=", DoubleToString(lot,2));
   return lot;
}

//──────────────────────────────────────────────────────────────────
// PLACE ORDER
//──────────────────────────────────────────────────────────────────
bool PlaceOrder(string symbol, string type_str, double entry, double sl,
                double tp1, double tp2, double tp3,
                double lot, string signal_id)
{
   if(!SymbolSelect(symbol, true)) { Print("❌ Symbol not found: ", symbol); return false; }

   string comment = "TY_" + signal_id;
   double ask     = SymbolInfoDouble(symbol, SYMBOL_ASK);
   double bid     = SymbolInfoDouble(symbol, SYMBOL_BID);
   double point   = SymbolInfoDouble(symbol, SYMBOL_POINT);

   if(type_str=="BUY_LIMIT" || type_str=="BUY") {
      if(ask <= entry + point*5) return Trade.Buy(lot, symbol, 0, sl, tp1, comment);
      else                       return Trade.BuyLimit(lot, entry, symbol, sl, tp1, 0, 0, comment);
   }
   if(type_str=="SELL_LIMIT" || type_str=="SELL") {
      if(bid >= entry - point*5) return Trade.Sell(lot, symbol, 0, sl, tp1, comment);
      else                       return Trade.SellLimit(lot, entry, symbol, sl, tp1, 0, 0, comment);
   }
   Print("❌ Unknown type: ", type_str);
   return false;
}

//──────────────────────────────────────────────────────────────────
// MANAGE POSITIONS — TP1→BE | TP2→Trail | TP3→Close
//──────────────────────────────────────────────────────────────────
void ManagePositions()
{
   for(int i = PositionsTotal()-1; i >= 0; i--) {
      if(!PosInfo.SelectByIndex(i)) continue;
      if(StringFind(PosInfo.Comment(), "TY_") != 0) continue;

      string symbol  = PosInfo.Symbol();
      ulong  ticket  = PosInfo.Ticket();
      double open_pr = PosInfo.PriceOpen();
      double current = PosInfo.PriceCurrent();
      double sl      = PosInfo.StopLoss();
      double point   = SymbolInfoDouble(symbol, SYMBOL_POINT);
      int    digits  = (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS);

      ENUM_POSITION_TYPE pt = PosInfo.PositionType();
      double risk = MathAbs(open_pr - sl);
      if(risk <= 0) continue;

      double tp1, tp2, tp3;
      if(pt == POSITION_TYPE_BUY) {
         tp1 = open_pr + risk;
         tp2 = open_pr + risk * 2.5;
         tp3 = open_pr + risk * 4.0;
      } else {
         tp1 = open_pr - risk;
         tp2 = open_pr - risk * 2.5;
         tp3 = open_pr - risk * 4.0;
      }

      string msg = "";

      if(pt == POSITION_TYPE_BUY) {
         if(current >= tp3) {
            Trade.PositionClose(ticket);
            msg = "🏆 TP3 FULL | " + symbol + " +" + DoubleToString((current-open_pr)*PipMult(symbol),0) + " pips";
         } else if(current >= tp2 && sl < tp1 - point) {
            Trade.PositionModify(ticket, tp1, tp3);
            msg = "🎯🎯 TP2 HIT | " + symbol + " | SL → " + DoubleToString(tp1,digits);
         } else if(current >= tp1 && sl < open_pr - point) {
            Trade.PositionModify(ticket, open_pr + point*2, tp2);
            msg = "🎯 TP1 HIT | " + symbol + " | SL → Breakeven";
         }
      } else {
         if(current <= tp3) {
            Trade.PositionClose(ticket);
            msg = "🏆 TP3 FULL | " + symbol + " +" + DoubleToString((open_pr-current)*PipMult(symbol),0) + " pips";
         } else if(current <= tp2 && sl > tp1 + point) {
            Trade.PositionModify(ticket, tp1, tp3);
            msg = "🎯🎯 TP2 HIT | " + symbol + " | SL → " + DoubleToString(tp1,digits);
         } else if(current <= tp1 && sl > open_pr + point) {
            Trade.PositionModify(ticket, open_pr - point*2, tp2);
            msg = "🎯 TP1 HIT | " + symbol + " | SL → Breakeven";
         }
      }

      if(msg != "") {
         Print(msg);
         if(PushAlerts) SendNotification(msg);
      }
   }
}

//──────────────────────────────────────────────────────────────────
// JSON HELPERS (no external library needed)
//──────────────────────────────────────────────────────────────────
string JsonStr(string json, string key) {
   int ki = StringFind(json, "\"" + key + "\":\"");
   if(ki < 0) return "";
   int vs = ki + StringLen(key) + 4;
   int ve = StringFind(json, "\"", vs);
   if(ve < 0) return "";
   return StringSubstr(json, vs, ve-vs);
}

double JsonDbl(string json, string key) {
   int ki = StringFind(json, "\"" + key + "\":");
   if(ki < 0) return 0;
   int vs = ki + StringLen(key) + 2;
   if(StringGetCharacter(json,vs)=='"') vs++;
   string val = "";
   for(int i=vs; i<MathMin(vs+25,StringLen(json)); i++) {
      ushort c = StringGetCharacter(json,i);
      if(c==',' || c=='}' || c=='"' || c==']') break;
      val += ShortToString(c);
   }
   return StringToDouble(StringTrimRight(StringTrimLeft(val)));
}

double PipMult(string s) {
   if(StringFind(s,"XAU")>=0||StringFind(s,"XAG")>=0) return 10.0;
   if(StringFind(s,"JPY")>=0) return 100.0;
   return 10000.0;
}

bool IsProcessed(string id) {
   for(int i=0;i<ArraySize(g_processedIDs);i++) if(g_processedIDs[i]==id) return true;
   return false;
}
void MarkProcessed(string id) {
   int n=ArraySize(g_processedIDs);
   ArrayResize(g_processedIDs,n+1);
   g_processedIDs[n]=id;
}
//+------------------------------------------------------------------+
