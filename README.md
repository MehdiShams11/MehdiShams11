#include <Trade\Trade.mqh>

// تنظیمات ورودی
input int maPeriod = 50;               // دوره میانگین متحرک
input double tradeSizePercent = 1.0;   // درصد حجم معامله از بالانس
input double riskRewardRatio = 2.0;    // نسبت ریسک به ریوارد
input int takeProfitPips = 50;         // حد سود به صورت پیپ
input ENUM_TIMEFRAMES timeFrame = PERIOD_H1; // تایم‌فریم معاملاتی

CTrade trade;

double movingAverage[];
int ma_handle;

//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
int OnInit() {
    // ایجاد هندل برای میانگین متحرک
    ma_handle = iMA(_Symbol, timeFrame, maPeriod, 0, MODE_SMA, PRICE_CLOSE);
    
    if (ma_handle == INVALID_HANDLE) {
        Print("Error creating moving average handle");
        return INIT_FAILED;
    }

    return INIT_SUCCEEDED;
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason) {
    Print("Deinitializing expert");
}

//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick() {
    // محاسبه میانگین متحرک
    CopyBuffer(ma_handle, 0, 0, 3, movingAverage);

    double closePrice = SymbolInfoDouble(_Symbol, SYMBOL_BID);
    double tradeVolume = AccountInfoDouble(ACCOUNT_BALANCE) * tradeSizePercent / 100.0 / closePrice;

    // بررسی برخورد کندل با خط میانگین متحرک
    if (iClose(_Symbol, timeFrame, 1) > movingAverage[1] && iOpen(_Symbol, timeFrame, 1) <= movingAverage[1]) {
        // کندل صعودی
        double stopLoss = iLow(_Symbol, timeFrame, 2) - (_Point * 10); // حد ضرر زیر کندل قبلی
        double takeProfit = closePrice + (takeProfitPips * _Point);
        if (!trade.Buy(tradeVolume, _Symbol, 0, stopLoss, takeProfit)) {
            Print("Error opening buy position: " + IntegerToString(trade.ResultRetcode()));
        }
    } else if (iClose(_Symbol, timeFrame, 1) < movingAverage[1] && iOpen(_Symbol, timeFrame, 1) >= movingAverage[1]) {
        // کندل نزولی
        double stopLoss = iHigh(_Symbol, timeFrame, 2) + (_Point * 10); // حد ضرر بالای کندل قبلی
        double takeProfit = closePrice - (takeProfitPips * _Point);
        if (!trade.Sell(tradeVolume, _Symbol, 0, stopLoss, takeProfit)) {
            Print("Error opening sell position: " + IntegerToString(trade.ResultRetcode()));
        }
    }

    // بررسی و تنظیم تریلینگ استاپ بر اساس نسبت ریسک به ریوارد
    for (int i = PositionsTotal() - 1; i >= 0; i--) {
        if (PositionSelect(i)) {
            ulong ticket = PositionGetTicket(i);
            double entryPrice = PositionGetDouble(POSITION_PRICE_OPEN);
            double currentStopLoss = PositionGetDouble(POSITION_SL);
            double risk = MathAbs(entryPrice - currentStopLoss);
            double reward = risk * riskRewardRatio;

            if (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY) {
                double trailStop = entryPrice + reward;
                if (closePrice >= trailStop) {
                    double newStopLoss = closePrice - (_Point * 10);
                    double newTakeProfit = closePrice + (takeProfitPips * _Point); // تعریف newTakeProfit
                    if (!trade.PositionModify(ticket, newStopLoss, newTakeProfit)) {
                        Print("Error modifying position: " + IntegerToString(trade.ResultRetcode()));
                    }
                }
            } else if (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_SELL) {
                double trailStop = entryPrice - reward;
                if (closePrice <= trailStop) {
                    double newStopLoss = closePrice + (_Point * 10);
                    double newTakeProfit = closePrice - (takeProfitPips * _Point); // تعریف newTakeProfit
                    if (!trade.PositionModify(ticket, newStopLoss, newTakeProfit)) {
                        Print("Error modifying position: " + IntegerToString(trade.ResultRetcode()));
                    }
                }
            }
        }
    }
}
