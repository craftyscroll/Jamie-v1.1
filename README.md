// Jamie's hedge strategy

////////////// Comments and suggestions ///////////////////////

	// What happens when there is no opposite entry signal and the original position goess deeply negative? Probably need some other form of risk management
		// See for example GBP/USD, 4-hour bars, April-July 2012
	// You can experiment with normal bars by simply commenting out the function for Haiken Ashi bars below
	// For simplicity I've used peaks and valleys in the EMA as the initial entry signals. Simple enough to add other conditions or vary as you see fit.
	// I've used a simple approach to position sizing for the hedges: this can also be varied as desired.
	// I've set the profit objective (in pips) for the hedged positions to 5 - vary as you see fit.  
	// Currently, all hedged positions are closed when the profit objective is acheived. In order to keep the last (profitable) trade open, use 'return 0' for that trade in the TMF. You'd need to do so by firstly identifying the correct trade with an if() statement
	
////////////////////////////////////////////////////////////////

#define LongEntry AssetVar[0]
#define ShortEntry AssetVar[1]
#define lastTrade AssetVar[2] // 1 if initial hedged trade is long, -1 if initial hedged trade is short

var hedgeProfit; // profit to be obtained, in pips, from the hedged positions before closing them

	// Haiken Ashi Bars
int bar(vars Open,vars High,vars Low,vars Close)
{
  Close[0] = (Open[0]+High[0]+Low[0]+Close[0])/4;
  Open[0] = (Open[1]+Close[1])/2;
  High[0] = max(High[0],max(Open[0],Close[0]));
  Low[0] = min(Low[0],min(Open[0],Close[0]));
  return 8;
}

int hedgeControl() {
	if (TradeIsLong) LongEntry = TradePriceOpen; // first long entry
	if (TradeIsShort) ShortEntry = TradePriceOpen; // first short entry
		
	if (NumOpenLong >= 1 and NumOpenShort >= 1 and ProfitOpen/TradeUnits/PIP > 1) {
		exitLong(); exitShort();
		LongEntry = ShortEntry = lastTrade = 0; // reset asset vars
		return 1;
	}
}

int hedgeControl2() {
	if (NumOpenLong >= 1 and NumOpenShort >= 1 and ProfitOpen/TradeUnits/PIP > 1) {
		exitLong(); exitShort();
		LongEntry = ShortEntry = lastTrade = 0; // reset asset vars
		return 1;
	}
	
}


function run() {
	set(TICKS|LOGFILE);
	BarPeriod = 1440;
	StartDate = 2012;
	EndDate = 2012;
	Hedge = 2;
	
	if(is(INITRUN)) {
		LongEntry = ShortEntry = lastTrade = 0; // reset Algovars between runs
	}
	
	vars Open = series(priceOpen());
	vars High = series(priceHigh());
	vars Low = series(priceLow());
	vars Close = series(priceClose());
	
	vars Price = series(price());
	vars trend = series(EMA(Price, 21));
	
	var LotsStart = 1.25; //vary this to incrase/decrease the rate at which position sizing is increased
	
	//trade signals
	Lots = 1;

	if (crossOver(Price, trend) and NumOpenLong == 0) {
		if (NumOpenShort == 1 and NumWinningShort == 1) { // exit opposite position if in profit
			exitShort();
			}
		if (NumOpenShort == 1 and NumWinningShort == 0) {
			lastTrade = 1; // this acts like a switch and in this case signifies that the initial entry is short and the initial hedge is long
			Lots = 1; // how many lots to open? For a breakeven at 2x hedge range, the size of the hedged trade = size of original trade
			}
		enterLong(hedgeControl); 		
	}
	
	if (crossUnder(Price, trend) and NumOpenShort == 0) {
		if (NumOpenLong == 1 and NumWinningLong == 1) { //exit opposite position if in profit
			exitLong();	
		}
		if (NumOpenLong == 1 and NumWinningLong == 0 ) {
			lastTrade = -1; // initial entry is long, intial hedged trade is short
			Lots = 1;
		}
		enterShort(hedgeControl);		
	}
	
	// open additional hedges in the run function, ie max once per bar, exit heges via TMF
	
	if (lastTrade == 1) { // initial hedge was long, therefore hedge short when NumLong = NumShort, hedge long when NumShort > NumLong
		if (NumOpenLong > 0 and NumOpenShort >0 and NumOpenLong == NumOpenShort) {
			Entry = ShortEntry;
			Lots = pow(LotsStart, (NumOpenLong + NumOpenShort));
			enterShort(hedgeControl2);	// hedge short if price crosses the short entry
		}
		if (NumOpenLong > 0 and NumOpenShort > 0 and NumOpenShort > NumOpenLong) {
			Entry = LongEntry;
			Lots = pow(LotsStart, (NumOpenLong + NumOpenShort));
			enterLong(hedgeControl2);	// hedge short if price crosses the short entry
		}
	}
	
	if (lastTrade == -1) { //initial hedge was short, therefore hedge long when NumLong = NumShort, hedge short when NumLong > NumShort
		if (NumOpenLong > 0 and NumOpenShort > 0 and NumOpenLong == NumOpenShort) {
			Entry = LongEntry;
			Lots = pow(LotsStart, (NumOpenLong + NumOpenShort));
			enterLong(hedgeControl2); 	// hedge short if price crosses the short entry
		}
		if (NumOpenLong > 0 and NumOpenShort > 0 and NumOpenLong > NumOpenShort) {
			Entry = ShortEntry;
			Lots = pow(LotsStart, (NumOpenLong + NumOpenShort));
			enterShort(hedgeControl2); // hedge short if price crosses the short entry
		}
	}
	
	// close profitable single trade upon 2x candles in opposite direction
	if (NumOpenTotal == 1 and ProfitOpen > 0) {
		if (NumOpenLong == 1 and Close[0] < Open[0] and Close[1] < Open[1]) // long trade, two consecutive down candles
			exitLong();
		if (NumOpenShort == 1 and Close[0] > Open[0] and Close[1] > Open[1]) // short trade, two consective up candles
			exitShort();	
	}
	
	// print statements to check that script is functioning properly	
	printf("\nNumOpenTotal: %1d\n", NumOpenTotal);
	printf("\nHedgeRange: %1.3f\n", abs(LongEntry-ShortEntry));
	
	ColorUp = BLUE;
	ColorDn = MAGENTA; //BLACK;
	
	plot("EMA", trend, MAIN, BLACK);
	plot("lastTrade", lastTrade, NEW, RED);
	plot("lots", Lots, NEW, GREEN);
	plot("LongEntry", LongEntry, NEW, BLUE);
	plot("ShortEntry", ShortEntry, 0, RED);

}
