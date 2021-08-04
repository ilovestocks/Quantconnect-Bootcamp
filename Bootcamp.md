# Quantconnect-bootcamp-notes: Buy and Hold / Equities

### Set Starting Cash
In backtests you can set your starting capital using the ```self.SetCash(float cash)``` method. In live trading this is ignored and your brokerage cash is used instead. In paper trading we set the cash to a fictional $100,000 USD.

```python
class BootCampTask(QCAlgorithm):

    def Initialize(self):
        
        self.AddEquity("SPY", Resolution.Daily)
        
        # 1. Set Starting Cash 
        
        self.SetCash(25000)
        
    def OnData(self, data):
        pass
```

### Set Date Range

Backtesting uses the ```self.SetStartDate(int year, int month, int day)``` and ```self.SetEndDate(int year, int month, int day)``` methods to configure the backtest time range. 
If unspecified, the end date defaults to yesterday.

```python
class BootCampTask(QCAlgorithm):

    def Initialize(self):
        
        #1-2. Set Date Range
        self.SetStartDate(2017, 1, 1)
        self.SetEndDate(2017, 10, 31)
        self.AddEquity("SPY", Resolution.Daily)
        
    def OnData(self, data):
        pass

```

### Manually Selecting Data

The ```self.AddEquity()``` method is used for manually requesting assets. It takes a string ticker of the current asset name and the resolution. 
The ```AddEquity``` (and all other AddAsset methods) return a security object. This gives you a reference to the security object you can use later.

You can control the resolution of the data with the Resolution enum. It has the values Tick, Second, Minute, Hour and Daily. e.g. Resolution.Minute.

```python
class BootCampTask(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2017, 6, 1)
        self.SetEndDate(2017, 6, 15)
        
        # Manually Select Data
        self.spy = self.AddEquity("SPY", Resolution.Minute)
        self.iwm = self.AddEquity("IWM", Resolution.Minute)
        
    def OnData(self, data):
        pass
```
```python
// Complete Add Equity API - Including Default Parameters:
AddEquity(string ticker, Resolution resolution = Resolution.Minute, 
string market = Market.USA, bool fillDataForward = true, 
decimal leverage = 0m, bool extendedMarketHours = false)
```

### Set Data Normalization Mode

By default equity data in QuantConnect is Split and Dividend adjusted backwards in time to give smooth continuous prices. This allows easy use for indicators. 
Some algorithms need raw or partially adjusted price data. You can control this with the ```SetDataNormalizationMode()``` method. 
The DataNormalizationMode enum has the values Adjusted (default), Raw, SplitAdjusted, and TotalReturn. 
When data is set to Raw mode the dividends are paid as cash into your algorithm, and the splits are directly applied to your holding quantity.

```python
class BootCampTask(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2017, 6, 1)
        self.SetEndDate(2017, 6, 15)
        
        #1-2. Change the data normalization mode for SPY and set the leverage for the IWM Equity     
        self.spy = self.AddEquity("SPY", Resolution.Daily)
        self.iwm = self.AddEquity("IWM", Resolution.Daily)
    
        self.spy.SetDataNormalizationMode(DataNormalizationMode.Raw)
        # Applying the DataNormalization helper directly on the security object
        
        self.iwm.SetLeverage(1.0)
        
    def OnData(self, data):
        pass
    
```

### Checking Holdings

The algorithm Portfolio dictionary also has helper properties for quick look ups of things like: Invested, TotalUnrealizedProfit, TotalPortfolioValue, TotalMarginUsed.

Individual asset holdings are held in your Portfolio property. This can be accessed via the ```self.Portfolio[symbol]``` dictionary.

Entries in the Portfolio dictionary are SecurityHolding objects with many properties about your holdings, such as: Invested, Quantity, AveragePrice and UnrealizedProfit.
```python
class BootCampTask(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2017, 6, 1)
        self.SetEndDate(2017, 6, 2)
        
        #1. Update the AddEquity command to request IBM data
        self.ibm = self.AddEquity("IBM", Resolution.Daily)
        
    def OnData(self, data):
        
        #2. Display the Quantity of IBM Shares You Own
        self.Debug("Number of IBM Shares: " + str(self.Portfolio["IBM"].Quantity))
        
```

### Placing Orders

Market orders are filled immediately when the market is open. If you are using daily data, the order isn't processed until the next morning. Daily bars only arrive at your algorithm after the market has closed.

The average fill price of your asset is available in the Portfolio class. You can access it like this: ```Portfolio["IBM"].AveragePrice```. In backtesting this is a modelled price. In live trading this is taken from your brokerage fill event.
```python
class BootCampTask(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2017, 6, 1)
        self.SetEndDate(2017, 6, 15)

        #1,2. Select IWM minute resolution data and set it to Raw normalization mode
        self.iwm = self.AddEquity("IWM", Resolution.Minute)
        self.iwm.SetDataNormalizationMode(DataNormalizationMode.Raw)

    def OnData(self, data):

        #3. Place an order for 100 shares of IWM and print the average fill price
        #4. Debug the AveragePrice of IWM
        if not self.Portfolio.Invested:
            self.MarketOrder("IWM", 100)
            self.Debug(str(self.Portfolio["IWM"].AveragePrice))

```

# Buy and Hold with a Trailing Stop Loss

### Setting up a Stop Market Order

```python
class BootCampTask(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2018, 12, 1) # Set Start Date
        self.SetEndDate(2019, 4, 1) # Set End Date
        self.SetCash(100000) # Set Strategy Cash
        
        #1. Subscribe to SPY in raw mode
        self.spy = self.AddEquity("SPY", Resolution.Daily)
        self.spy.SetDataNormalizationMode(DataNormalizationMode.Raw)
        
    def OnData(self, data):
        
        if not self.Portfolio.Invested:
            #2. Create market order to buy 500 units of SPY
            self.MarketOrder("SPY", 500)
            
            #3. Create a stop market order to sell 500 units at 90% of the SPY current price
            # self.Securities is a collection of the securities objects
            # Close is the last close price
            self.StopMarketOrder("SPY", -500, 0.90 * self.Securities["SPY"].Close)

```

### Understanding Order Events

Order events are updates on the status of your order. 

Every order event is sent to the ```def OnOrderEvent()``` event handler, with information about the order status held in an ```OrderEvent``` object.

The ```OrderEvent``` object has a Status property with the ```OrderStatus``` enum values Submitted, PartiallyFilled, Filled, Canceled, and Invalid. 
It also contains an ```OrderId``` property which is a unique number representing the order.
```python
class BootCampTask(QCAlgorithm):

    def Initialize(self):
        
        self.SetStartDate(2018, 12, 1) 
        self.SetEndDate(2019, 4, 1) 
        self.SetCash(100000) 
        spy = self.AddEquity("SPY", Resolution.Daily)
        spy.SetDataNormalizationMode(DataNormalizationMode.Raw)
        self.lastOrderEvent = None
        
    def OnData(self, data):
    
        if not self.Portfolio.Invested:
            self.MarketOrder("SPY", 500)
            self.StopMarketOrder("SPY", -500, 0.9 * self.Securities["SPY"].Close)
        
    def OnOrderEvent(self, orderEvent):
        
        #1. Write code to only act on fills
        if orderEvent.Status == OrderStatus.Filled:
            #2. Save the orderEvent to lastOrderEvent, use Debug to print the event OrderId
            self.lastOrderEvent = orderEvent
            self.Debug(str(orderEvent.OrderId))
            # a unique number representing the order
```

### Identifying a Stop Loss Hit
When placing an order, QuantConnect returns an ```OrderTicket``` object which can be used to update an order's properties, request that it is cancelled, or fetch its OrderId.

The OrderId is stored on the orderEvent parameter passed into our OnOrderEvent() method. We can match the orderEvent.OrderId with the Id of the stop market order to see if our order has been filled.
```python
class BootCampTask(QCAlgorithm):
    
    # Order ticket for our stop order, Datetime when stop order was last hit
    # stopMarketTicket = None
    # stopMarketFillTime = datetime.min
    
    def Initialize(self):
        self.SetStartDate(2018, 12, 1)
        self.SetEndDate(2019, 4, 1)
        self.SetCash(100000)
        self.stopMarketTicket = None
        self.stopMarketFillTime = datetime.min
        spy = self.AddEquity("SPY", Resolution.Daily)
        spy.SetDataNormalizationMode(DataNormalizationMode.Raw)
        
    def OnData(self, data):
        
        #4. Check that at least 15 days (~2 weeks) have passed since we last hit our stop order
        if (self.Time - self.stopMarketFillTime).days < 15:
            return
        
        # If it is more than 15 days, run below
        if not self.Portfolio.Invested:
            self.MarketOrder("SPY", 500)
            
            #1. Create stop loss through a stop market order
            self.stopMarketTicket = self.StopMarketOrder("SPY", -500, 0.9 * self.Securities["SPY"].Close)
            
    def OnOrderEvent(self, orderEvent):
        
        # There will be many orderEvents and we are only interested in the Filled Orders
        if orderEvent.Status != OrderStatus.Filled:
            return
        
        # Printing the security fill prices.
        self.Debug(self.Securities["SPY"].Close)
        
        #2. Check if we hit our stop loss (Compare the orderEvent.Id with the stopMarketTicket.OrderId)
        #   It's important to first check if the ticket isn't null (i.e. making sure it has been submitted)
        if self.stopMarketTicket is not None and self.stopMarketTicket.OrderId == orderEvent.OrderId:
            #3. Store datetime
            # self.Time is the Current time of the backtest
            self.stopMarketFillTime = self.Time
            self.Debug(str(self.stopMarketFillTime))
```

### Creating a Trailing Stop Loss

Orders which are not filled immediately can be updated using their order ticket. To update an order you create an ```UpdateOrderFields``` object which contains all the properties you'd like to change.
```python
class BootCampTask(QCAlgorithm):
    
    # Order ticket for our stop order, Datetime when stop order was last hit
    stopMarketTicket = None
    stopMarketOrderFillTime = datetime.min
    highestSPYPrice = 0
    
    def Initialize(self):
        self.SetStartDate(2018, 12, 1)
        self.SetEndDate(2018, 12, 10)
        self.SetCash(100000)
        spy = self.AddEquity("SPY", Resolution.Daily)
        spy.SetDataNormalizationMode(DataNormalizationMode.Raw)
        
    def OnData(self, data):
        
        if (self.Time - self.stopMarketOrderFillTime).days < 15:
            return

        if not self.Portfolio.Invested:
            self.MarketOrder("SPY", 500)
            self.stopMarketTicket = self.StopMarketOrder("SPY", -500, 0.9 * self.Securities["SPY"].Close)
        
        else:
            
            #1. Check if the SPY price is higher that highestSPYPrice.
            if self.Securities["SPY"].Close > self.highestSPYPrice:
                
                #2. Save the new high to highestSPYPrice; then update the stop price to 90% of highestSPYPrice 
                self.highestSPYPrice = self.Securities["SPY"].Close
                updateFields = UpdateOrderFields()
                updateFields.StopPrice = self.highestSPYPrice * 0.9
                self.stopMarketTicket.Update(updateFields)
                
                #3. Print the new stop price with Debug()
                self.Debug("SPY: " + str(self.highestSPYPrice) + " Stop: " + str(updateFields.StopPrice))
                
    def OnOrderEvent(self, orderEvent):
        if orderEvent.Status != OrderStatus.Filled:
            return
        if self.stopMarketTicket is not None and self.stopMarketTicket.OrderId == orderEvent.OrderId: 
            self.stopMarketOrderFillTime = self.Time
```

### Visualizing the Stop Levels

The Plot() method can draw a line-chart with a single line of code. 

It takes three arguments, the name of the chart, the name of the series and the value you'd like to plot.
```python
class BootCampTask(QCAlgorithm):
    
    # Order ticket for our stop order, Datetime when stop order was last hit
    stopMarketTicket = None
    stopMarketOrderFillTime = datetime.min
    highestSPYPrice = -1
    
    def Initialize(self):
        self.SetStartDate(2018, 12, 1)
        self.SetEndDate(2018, 12, 10)
        self.SetCash(100000)
        spy = self.AddEquity("SPY", Resolution.Daily)
        spy.SetDataNormalizationMode(DataNormalizationMode.Raw)
        
    def OnData(self, data):
        
        # 1. Plot the current SPY price to "Data Chart" on series "Asset Price"
        self.Plot("Data Chart", "Asset Price", data["SPY"].Close)

        if (self.Time - self.stopMarketOrderFillTime).days < 15:
            return

        if not self.Portfolio.Invested:
            self.MarketOrder("SPY", 500)
            self.stopMarketTicket = self.StopMarketOrder("SPY", -500, 0.9 * self.Securities["SPY"].Close)
        
        else:
            
            #2. Plot the moving stop price on "Data Chart" with "Stop Price" series name
            self.Plot("Data Chart", "Stop Price", self.stopMarketTicket.Get(OrderField.StopPrice))
            
            if self.Securities["SPY"].Close > self.highestSPYPrice:
                
                self.highestSPYPrice = self.Securities["SPY"].Close
                updateFields = UpdateOrderFields()
                updateFields.StopPrice = self.highestSPYPrice * 0.9
                self.stopMarketTicket.Update(updateFields) 
            
    def OnOrderEvent(self, orderEvent):
        
        if orderEvent.Status != OrderStatus.Filled:
            return
        
        if self.stopMarketTicket is not None and self.stopMarketTicket.OrderId == orderEvent.OrderId: 
            self.stopMarketOrderFillTime = self.Time

```

# Momentum-Based Tactical Allocation
A tactical asset allocation (TAA) strategy allows us to move the portfolio between assets depending on the market conditions. We can use TAA to liquidate trend laggards and capture strong market trends for short-term profit.
### Setting Up Tactical Asset Allocation
Tracking multiple assets in your portfolio can easily be done with QuantConnect by simply calling AddEquity() twice. Data requested is passed into your OnData() method synchronized in time.
```python
class MomentumBasedTacticalAllocation(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2007, 8, 1)  # Set Start Date
        self.SetEndDate(2010, 8, 1)  # Set End Date
        
        #1. Subscribe to SPY -- S&P 500 Index ETF -- using daily resolution
        # Risky asset class
        self.spy = self.AddEquity("SPY", Resolution.Daily)
        
        #2. Subscribe to BND -- Vanguard Total Bond Market ETF -- using daily resolution
        # Safe asset class
        self.bnd = self.AddEquity("BND", Resolution.Daily)  
        
        #3. Set strategy cash to $3000
        self.SetCash(3000)  
        
    def OnData(self, data):
        pass
```

### Using a Momentum Percentage Indicator
Indicators describe asset momentum, trends, volume and volatility. The indicator MomentumPercent computes the n-period percent change in the security. The indicator will be automatically updated on the resolution frequency. Our MOMP() method has the parameters symbol, period, and resolution.
```python
class MomentumBasedTacticalAllocation(QCAlgorithm):

    spyMomentum = None
    bondMomentum = None

    def Initialize(self):
        self.SetStartDate(2007, 6, 1)  
        self.SetEndDate(2010, 6, 1)  
        self.SetCash(3000)   
        
        self.spy = self.AddEquity("SPY", Resolution.Daily)  
        self.bnd = self.AddEquity("BND", Resolution.Daily)  
        
        #1. Add 50-day Momentum Percent indicator for SPY
        self.spyMomentum = self.MOMP("SPY", 50, Resolution.Daily) 
        
        #2. Add 50-day Momentum Percent indicator for BND
        self.bondMomentum = self.MOMP("BND", 50, Resolution.Daily) 

    def OnData(self, data):
        pass


```

### Preparing an Indicator for Testing
We prepare our algorithm with a "fast-forward" system called "Warm Up" which automatically pumps historical data to an indicator using the SetWarmUp() method. Warm up requests should be sent in your algorithm Initialize() method.
```python
class MomentumBasedTacticalAllocation(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2007, 8, 1)  
        self.SetEndDate(2010, 8, 1)  
        self.SetCash(3000)  
        
        self.AddEquity("SPY", Resolution.Hour)  
        self.AddEquity("BND", Resolution.Hour) 
        
        self.spyMomentum = self.MOMP("SPY", 50, Resolution.Daily) 
        self.bondMomentum = self.MOMP("BND", 50, Resolution.Daily) 
        
        #1. Set SPY Benchmark
        # To measure the impact of our algorithm we should set an appropriate benchmark. 
        # A benchmark asset should be representative of the portfolio you are trading. 
        # The SetBenchmark() helper should be called in your Initialize method and takes a single parameter, ticker.
        self.SetBenchmark("SPY")  
        
        #2. Warm up algorithm for 50 days to populate the indicators prior to the start date
        self.SetWarmUp(50)
        # QUESTION: why not 50*24???
    
    def OnData(self, data):
        # You should validate indicators are ready before using them:
        if self.spyMomentum is None or self.bondMomentum is None or not self.bondMomentum.IsReady or not self.spyMomentum.IsReady:
            return
```

### Using our Signal to Flip

```python
class MomentumBasedTacticalAllocation(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2007, 8, 1) 
        self.SetEndDate(2010, 8, 1) 
        self.SetCash(3000)   
        
        self.spy = self.AddEquity("SPY", Resolution.Daily)  
        self.bnd = self.AddEquity("BND", Resolution.Daily)  
        
        self.spyMomentum = self.MOMP("SPY", 50, Resolution.Daily)
        self.bondMomentum = self.MOMP("BND", 50, Resolution.Daily) 
        
        self.SetBenchmark("SPY")  
        self.SetWarmUp(50) 
        
    def OnData(self, data):
        
        # Don't place trades until our indicators are warmed up:
        if self.IsWarmingUp:
            return
        
        #1. If SPY has more upward momentum than BND, then we liquidate our holdings in BND and allocate 100% of our equity to SPY
        if self.spyMomentum.Current.Value > self.bondMomentum.Current.Value:
            self.Liquidate("BND")
            self.SetHoldings("SPY", 1)

        #2. Otherwise we liquidate our holdings in SPY and allocate 100% to BND
        else:
            self.Liquidate("SPY")
            self.SetHoldings("BND", 1)

```

### Time Our Market Moves
Limiting the trades we make ensures we don't trade inefficiently (i.e. generate high fees for low value trades). We can use the algorithm self.Time property to track the current simulation time. In backtesting this fast-forwards through historical data, in live trading this is always the current time (now). 

Rebalancing frequently is expensive

Remember data is indexed by the end time of the bar, and the Time property will show the next day. 
You'll have to trade on Tuesdays as orders placed on Tuesday are filled on Wednesday.
```python
class MomentumBasedTacticalAllocation(QCAlgorithm):
    
    def Initialize(self):
        
        self.SetStartDate(2007, 8, 1) 
        self.SetEndDate(2010, 8, 1)  
        self.SetCash(3000)  
        
        self.spy = self.AddEquity("SPY", Resolution.Daily)  
        self.bnd = self.AddEquity("BND", Resolution.Daily)  
      
        self.spyMomentum = self.MOMP("SPY", 50, Resolution.Daily) 
        self.bondMomentum = self.MOMP("BND", 50, Resolution.Daily) 
       
        self.SetBenchmark(self.spy.Symbol)  
        self.SetWarmUp(50) 
  
    def OnData(self, data):
        
        if self.IsWarmingUp:
            return
        
        #1. Limit trading to happen once per week
        if self.Time.weekday() == 1:
            if self.spyMomentum.Current.Value > self.bondMomentum.Current.Value:
                self.Liquidate("BND")
                self.SetHoldings("SPY", 1)
                
            else:
                self.Liquidate("SPY")
                self.SetHoldings("BND", 1)

```

# Opening Range Breakout
Opening range breakout uses a defined period of time to set a price-range, and trades on leaving that range. 
### Creating a Consolidator
```python
# Receive consolidated data with a timedelta 
# parameter and OnDataConsolidated event handler 
self.Consolidate("SPY", timedelta(minutes=45), self.OnDataConsolidated)

# Receive consolidated data with a CalendarType
# parameter and OnDataConsolidated event handler 
self.Consolidate("SPY", CalendarType.Weekly, self.OnDataConsolidated)

# Receive consolidated data with a Resolution
# parameter and OnDataConsolidated event handler 
self.Consolidate("SPY", Resolution.Hour, TickType.Trade, self.OnDataConsolidated)
```
```python
class OpenRangeBreakout(QCAlgorithm):
    
    openingBar = None
    currentBar = None

    def Initialize(self):
        self.SetStartDate(2018, 7, 10) # Set Start Date  
        self.SetEndDate(2019, 6, 30) # Set End Date 
        self.SetCash(100000)  # Set Strategy Cash 
        
        # Subscribe to TSLA with Minute Resolution
        self.symbol = self.AddEquity("TSLA", Resolution.Minute)
        #1. Create our consolidator with a timedelta of 30 min
        self.Consolidate("TSLA", timedelta(minutes=30), self.OnDataConsolidated)
        
    def OnData(self, data):
        pass
    
    #2. Create a function OnDataConsolidator which saves the currentBar as bar 
    # Consolidators require an accompanying event handler to receive the output data. 
    # The consolidator event handlers are functions which are called when a new bar is created. 
    def OnDataConsolidated(self, bar):
        self.currentBar = bar

```
### Bar Data and Bar Time

```python
class OpeningRangeBreakout(QCAlgorithm):
    
    openingBar = None 
  
    def Initialize(self):
        self.SetStartDate(2018, 7, 10)  
        self.SetEndDate(2019, 6, 30)  
        self.SetCash(100000)  
        self.AddEquity("TSLA", Resolution.Minute)
        self.Consolidate("TSLA", timedelta(minutes=30), self.OnDataConsolidated)
        
    def OnData(self, data):
        pass
        
    def OnDataConsolidated(self, bar):
        #1. Check the time, we only want to work with the first 30min after Market Open
        if bar.Time.hour == 9 and bar.Time.minute == 30:
            #2. Save one bar as openingBar 
            self.openingBar = bar

```

### Using the Output of the Consolidator

```python
class OpeningRangeBreakout(QCAlgorithm):
    
    openingBar = None 
    
    def Initialize(self):
        self.SetStartDate(2018, 7, 10)  
        self.SetEndDate(2019, 6, 30)  
        self.SetCash(100000)
        self.AddEquity("TSLA", Resolution.Minute)
        self.Consolidate("TSLA", timedelta(minutes=30), self.OnDataConsolidated)
        
    def OnData(self, data):
        
        #1. If self.Portfolio.Invested is true, or if the openingBar is None, return
        if self.Portfolio.Invested or self.openingBar is None:
            return
        
        #2. Check if the close price is above the high price, if so go 100% long on TSLA 
        if data["TSLA"].Close > self.openingBar.High:
            self.SetHoldings("TSLA", 1)
        
        #3. Check if the close price is below the low price, if so go 100% short on TSLA
        elif data["TSLA"].Close < self.openingBar.Low:
            self.SetHoldings("TSLA", -1)
        
    def OnDataConsolidated(self, bar):
        if bar.Time.hour == 9 and bar.Time.minute == 30:
            self.openingBar = bar

```

### Scheduling Events
Scheduled events allow you to trigger code blocks for execution at specific times according to rules you set. We initialize scheduled events in the Initialize method so they are only executed once.

We use self.Schedule.On() to coordinate your algorithm activities and perform analysis at regular intervals while letting the trading engine take care of market holidays.

Scheduled events need a DateRules and TimeRules pair to set a specific time, and an action that you want to complete. When the event is triggered the action block (or function) is executed. We include our asset ticker in our EveryDay("ticker") object to specifify that we are calling days there is data for the ticker. 
```python
class OpeningRangeBreakout(QCAlgorithm):
    
    openingBar = None 
    
    def Initialize(self):
        self.SetStartDate(2018, 7, 10)  
        self.SetEndDate(2019, 6, 30)  
        self.SetCash(100000)
        self.AddEquity("TSLA", Resolution.Minute)
        self.Consolidate("TSLA", timedelta(minutes=30), self.OnDataConsolidated)
        
        #3. Create a scheduled event triggered at 13:30 calling the ClosePositions function
        self.Schedule.On(self.DateRules.EveryDay("TSLA"), self.TimeRules.At(13, 30), self.ClosePositions)
        
    def OnData(self, data):
        
        if self.Portfolio.Invested or self.openingBar is None:
            return
        
        if data["TSLA"].Close > self.openingBar.High:
            self.SetHoldings("TSLA", 1)

        elif data["TSLA"].Close < self.openingBar.Low:
            self.SetHoldings("TSLA", -1)  
         
    def OnDataConsolidated(self, bar):
        if bar.Time.hour == 9 and bar.Time.minute == 30:
            self.openingBar = bar
    
    #1. Create a function named ClosePositions(self)
    def ClosePositions(self):
        #2. Set self.openingBar to None, and liquidate TSLA 
        self.openingBar = None
        # It doesn't hurt even if you do not have TSLA
        self.Liquidate("TSLA")

```

# Liquid Universe Selection

### Setting Up a Coarse Universe Filter
We can add a universe with the method ```self.AddUniverse()```
Avoid survivorship bias

The method requires a filter function ```CoarseSelectionFilter``` which is passed an array of coarse data representing all stocks active on that trading day. Assets selected by our filter are automatically added to your algorithm in minute resolution.
```python
class LiquidUniverseSelection(QCAlgorithm):
    
    filteredByPrice = None
    coarse = None
    
    def Initialize(self):
        self.SetStartDate(2019, 1, 11)  
        self.SetEndDate(2019, 7, 1) 
        self.SetCash(100000)  
        
        #3. Add a Universe model using Coarse Fundamental Data and set the filter function 
        self.AddUniverse(self.CoarseSelectionFilter)
        
    #1. Add an empty filter function
    def CoarseSelectionFilter(self, coarse):
        # Every day, QC passes in the list of the currently traded securities into the coarse object
        #2. Save coarse as self.coarse and return an Unchanged Universe
        # If we don't have specific filtering instructions for our universe, 
        # we can use Universe.Unchanged which specifies that universe selection 
        # should not make changes on this iteration.
        
        self.coarse = coarse
        return Universe.Unchanged

```

### Coarse Fundamental Objects
Coarse is an array of ```CoarseFundamentalObjects```


The CoarseSelectionFilter function narrows the list of companies according to properties like price and volume. 
The filter needs to return a list of symbols. 
For example, we may want to narrow down our universe to liquid assets, or assets that pass a technical indicator filter. 
We can do all of this in the coarse selection function.

You can use the coarse fundamental data to create your own criteria ("factors") to perform your selection. 
Once you have your target criteria there are two key tools: sorting and filtering. 
Sorting lets you take the top and/or bottom ranked symbols according to your criteria, filtering allows you to narrow your selection range to eliminate some assets. 
In python this is accomplished by sort and list selection methods.
```python
def CoarseSelectionFilter(self, coarse):
    # 'coarse' is an array of CoarseFundamental data.
    # CoarseFundamental has properties Symbol, DollarVolume and Price.
    coarse[0].Symbol
    coarse[0].DollarVolume
    coarse[0].Price
```
```python
class LiquidUniverseSelection(QCAlgorithm):
    
    filteredByPrice = None

    def Initialize(self):
        self.SetStartDate(2019, 1, 11)  
        self.SetEndDate(2019, 7, 1) 
        self.SetCash(100000)  
        self.AddUniverse(self.CoarseSelectionFilter)
        
    def CoarseSelectionFilter(self, coarse):
        
        #1. Sort descending by daily dollar volume
        sortedByDollarVolume = sorted(coarse, key=lambda x: x.DollarVolume, reverse=True)  
        
        #2. Select only Symbols with a price of more than $10 per share
        self.filteredByPrice = [x.Symbol for x in sortedByDollarVolume if x.Price > 10]
        
        #3. Return the 8 most liquid Symbols from the filteredByPrice list
        self.filteredByPrice = self.filteredByPrice[:8]
        return self.filteredByPrice

```

### Tracking Security Changes
When the universe constituents change (securities are added or removed from the algorithm) the algorithm generates an ```OnSecuritiesChanged``` event which is passed information about the asset changes.
```python
class LiquidUniverseSelection(QCAlgorithm):
    
    filteredByPrice = None
    changes = None
    
    def Initialize(self):
        self.SetStartDate(2019, 1, 11)  
        self.SetEndDate(2019, 7, 1) 
        self.SetCash(100000)  
        self.AddUniverse(self.CoarseSelectionFilter)
        
    def CoarseSelectionFilter(self, coarse):
    
        sortedByDollarVolume = sorted(coarse, key=lambda x: x.DollarVolume, reverse=True)  
        
        filteredByPrice = [x.Symbol for x in sortedByDollarVolume if x.Price > 10]
       
        return filteredByPrice[:8]
    
    #1. Create a function OnSecuritiesChanged
    def OnSecuritiesChanged(self, changes):
        #2. Save securities changed as self.changes 
        self.changes = changes
        #3. Log the changes in the function 
        self.Log(f"OnSecuritiesChanged({self.Time}):: {changes}")

```

### Building Our Portfolio
The ```OnSecuritiesChanged``` event fires whenever we have changes to our universe. 
It receives a ```SecurityChanges``` object containing references to the added and removed securities.
```python
# List of securities entering the universe
changes.AddedSecurities
# List of securities leaving the universe
changed.RemovedSecurities
```

```python
class LiquidUniverseSelection(QCAlgorithm):
    
    filteredByPrice = None
    
    def Initialize(self):
        self.SetStartDate(2019, 1, 11)  
        self.SetEndDate(2019, 7, 1) 
        self.SetCash(100000)  
        self.AddUniverse(self.CoarseSelectionFilter)
        # Ignore this for now, we'll cover it in the next task.
        self.UniverseSettings.Resolution = Resolution.Daily 

    def CoarseSelectionFilter(self, coarse):
        sortedByDollarVolume = sorted(coarse, key=lambda x: x.DollarVolume, reverse=True) 
        filteredByPrice = [x.Symbol for x in sortedByDollarVolume if x.Price > 10]
        return filteredByPrice[:8]
   
    def OnSecuritiesChanged(self, changes):
        self.changes = changes
        self.Log(f"OnSecuritiesChanged({self.UtcTime}):: {changes}")
        
        #1. Liquidate removed securities
        for security in changes.RemovedSecurities:
            if security.Invested:
                self.Liquidate(security.Symbol)
        
        #2. We want 10% allocation in each security in our universe
        for security in changes.AddedSecurities:
            self.SetHoldings(security.Symbol, 0.1)

```

### Customizing Universe Settings
You can control the settings of assets added by universe selection with the UniverseSettings helper. This mostly used to control the Resolution, Leverage and Data Normalization Mode of assets in your universe.
```python
self.UniverseSettings.Resolution
self.UniverseSettings.Leverage
self.UniverseSettings.FillForward
self.UniverseSettings.MinimumTimeInUniverse
self.UniverseSettings.ExtendedMarketHours
```
```python
class LiquidUniverseSelection(QCAlgorithm):
    
    filteredByPrice = None
    
    def Initialize(self):
        self.SetStartDate(2019, 1, 11)  
        self.SetEndDate(2019, 7, 1) 
        self.SetCash(100000)  
        self.AddUniverse(self.CoarseSelectionFilter)
        self.UniverseSettings.Resolution = Resolution.Daily

        #1. Set the leverage to 2
        self.UniverseSettings.Leverage = 2
       
    def CoarseSelectionFilter(self, coarse):
        sortedByDollarVolume = sorted(coarse, key=lambda c: c.DollarVolume, reverse=True)
        filteredByPrice = [c.Symbol for c in sortedByDollarVolume if c.Price > 10]
        return filteredByPrice[:10] 

    def OnSecuritiesChanged(self, changes):
        self.changes = changes
        self.Log(f"OnSecuritiesChanged({self.Time}):: {changes}")
        
        for security in self.changes.RemovedSecurities:
            if security.Invested:
                self.Liquidate(security.Symbol)
        
        for security in self.changes.AddedSecurities:
            #2. Leave a cash buffer by setting the allocation to 0.18 instead of 0.2 
            # self.SetHoldings(security.Symbol, ...)
            self.SetHoldings(security.Symbol, 0.18)

```

# Fading The Gap

### Creating Scheduled Events
Scheduled events trigger a method at a specific date and time. Scheduled events have three parameters: a DateRule, a TimeRule, and an action parameter. Our action parameter should be set to the name of a method.

Our time rules ```AfterMarketOpen``` and ```BeforeMarketClose``` take a symbol, and minute period. Both trigger an event a for a specific symbol's market hours.
```python
class FadingTheGap(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2017, 11, 1)  
        self.SetEndDate(2018, 7, 1)  
        self.SetCash(100000)  
        self.AddEquity("TSLA", Resolution.Minute)
        
        #1. Create a scheduled event to run every day at "0 minutes" before 
        # TSLA market close that calls the method ClosingBar
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.BeforeMarketClose("TSLA", 0), self.ClosingBar) 
        
        #2. Create a scheduled event to run every day at 1 minute after 
        # TSLA market open that calls the method OpeningBar
        # Need to wait for 1 minute to get the minute bar
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.AfterMarketOpen("TSLA", 1), self.OpeningBar)
        
        #3. Create a scheduled event to run every day at 45 minutes after 
        # TSLA market open that calls the method ClosePositions
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.AfterMarketOpen("TSLA", 45), self.ClosePositions)
    
    #1. Create an empty method ClosingBar
    def ClosingBar(self):
        pass
    
    #2. Create an empty method OpeningBar
    def OpeningBar(self):
        pass
    
    #3. Create a method ClosePositions
        # Liquidate our position to limit our exposure and keep our holding period short 
    def ClosePositions(self):
        self.Liquidate()

```

### Creating a Rolling Window
A RollingWindow holds a set of the most recent entries of data. As we move from time t=0 forward, our rolling window will shuffle data further along to a different index until it leaves the window completely.

The length of the window is specified in parentheses after the data type. Keep in mind the window is indexed from 0 up to Count-1.
```python
class FadingTheGap(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2017, 11, 1) 
        self.SetEndDate(2018, 7, 1) 
        self.SetCash(100000)
        self.AddEquity("TSLA", Resolution.Minute)
        
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.BeforeMarketClose("TSLA", 0), self.ClosingBar)
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.AfterMarketOpen("TSLA", 1), self.OpeningBar)
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.AfterMarketOpen("TSLA", 45), self.ClosePositions) 
        
        #1. Save a RollingWindow with type TradeBar and length of 2 as self.window
        
        self.window = RollingWindow[TradeBar](2)
    
    def ClosingBar(self):
        #2. Add the final bar of TSLA to our rolling window
        self.window.Add(self.CurrentSlice["TSLA"])
 
    def OpeningBar(self):
        #3. If "TSLA" is in the current slice, add the current slice to the window
        # We update our rolling array with the Add() method, which adds a new element at the beginning of the window. 
        # The objects inserted must match the type of the RollingWindow.
        if "TSLA" in self.CurrentSlice.Bars:
            self.window.Add(self.CurrentSlice["TSLA"])
            # The latest data point is accessible with the algorithm self.CurrentSlice property. 
            # This is the same Slice object from OnData.
        
    def ClosePositions(self):
        self.Liquidate() 
    

```
### Accessing a Rolling Window
Our window is not full when we first create it. 

QuantConnect provides a shortcut to check the if the rolling window is full("ready"): ```window.IsReady``` which will return true when all slots of the window have data.

The object in the window with index [0] refers to the most recent item. The length-1 in the window is the oldest object.
```python
class FadingTheGap(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2017, 11, 1)
        self.SetEndDate(2018, 7, 1)
        self.SetCash(100000) 
        self.AddEquity("TSLA", Resolution.Minute)
        
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.BeforeMarketClose("TSLA", 0), self.ClosingBar) 
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.AfterMarketOpen("TSLA", 1), self.OpeningBar)
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.AfterMarketOpen("TSLA", 45), self.ClosePositions) 
        
        self.window = RollingWindow[TradeBar](2)
        
    def ClosingBar(self):
        self.window.Add(self.CurrentSlice["TSLA"])
    
    def OpeningBar(self):
        if "TSLA" in self.CurrentSlice.Bars:
            self.window.Add(self.CurrentSlice["TSLA"])
        
        #1. If our window is not full use return to wait for tomorrow
        if not self.window.IsReady:
            return
        
        #2. Calculate the change in overnight price
        # Calculate the change in overnight price using index values from 
        # our rolling window (ie. today's open - yesterday's close). Save the value to delta.
        delta = self.window[0].Open - self.window[1].Close
        
        #3. If delta is less than -$2.5, SetHoldings() to 100% TSLA
        # NOTE: this is an arbitrary parameter
        if delta < -2.5:
            self.SetHoldings("TSLA", 1)
        
    def ClosePositions(self):
        self.Liquidate()

```

### Reducing a Parameter

QuantConnect provides the ability to add indicators manually. 

When we create an indicator manually, we control what data goes into it. 

Each manual indicator is slightly different, but the standard deviation takes a name and period. 

```python
# Create an indicator StandardDeviation 
# for the symbol TSLA with a 30 "value" period 
self.volatilty = StandardDeviation("TSLA", 30)
self.rsi = RelativeStrengthIndex("MyRSI", 200)
self.ema = ExponentialMovingAverage("SlowEMA", 200)
```
```python
class FadingTheGap(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2017, 11, 1)
        self.SetEndDate(2018, 7, 1)
        self.SetCash(100000) 
        self.AddEquity("TSLA", Resolution.Minute)
        
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.BeforeMarketClose("TSLA", 0), self.ClosingBar) 
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.AfterMarketOpen("TSLA", 1), self.OpeningBar)
        self.Schedule.On(self.DateRules.EveryDay(), self.TimeRules.AfterMarketOpen("TSLA", 45), self.ClosePositions) 
        
        self.window = RollingWindow[TradeBar](2)
                
        #1. Create a manual Standard Deviation indicator to track recent volatility
        self.volatility = StandardDeviation("TSLA", 60)
        
    def OnData(self, data):
        if data["TSLA"] is not None: 
            #2. Update our standard deviation indicator manually with algorithm time and TSLA's close price
            # When we create a manual indicator object, we need to update it in order to create valid indicator values. 
            # All manual indicators have an Update method which takes a time and a value.
            
            # NOTE: When you're using the helper methods the updates are done automatically (e.g. RSI(), EMA()). 
            # When using a combination of OnData and Scheduled Events keep in mind that scheduled events trigger before our OnData events, so your indicator might not be updated yet.
            self.volatility.Update(self.Time, data["TSLA"].Close)
    
    def OpeningBar(self):
        if "TSLA" in self.CurrentSlice.Bars:
            self.window.Add(self.CurrentSlice["TSLA"])
        
        #3. Use IsReady to check if both volatility and the window are ready, if not ready 'return'
        if not self.window.IsReady or not self.volatility.IsReady:
            return
        
        delta = self.window[0].Open - self.window[1].Close
        
        #4. Save an approximation of standard deviations to our deviations variable by dividing delta by the current volatility value:
        #   Normally this is delta from the mean, but we'll approximate it with current value for this lesson. 
        deviations = delta / self.volatility.Current.Value 
        
        #5. SetHoldings to 100% TSLA if deviations is less than -3 standard deviations from the mean:
        if deviations < -3:
            self.SetHoldings("TSLA", 1)
        
    def ClosePositions(self):
        self.Liquidate()
        
    def ClosingBar(self):
        self.window.Add(self.CurrentSlice["TSLA"])

```

# 200-50 EMA Momentum Universe

### Laying Our Universe Foundation
The OnSecuritiesChanged event fires whenever we have changes to our universe. It receives a SecurityChanges object containing references to the added and removed securities. 

The AddedSecurities and RemovedSecurities properties are lists of security objects.
```python
class EMAMomentumUniverse(QCAlgorithm):
    
    def Initialize(self):
        self.SetStartDate(2019, 1, 7)
        self.SetEndDate(2019, 4, 1)
        self.SetCash(100000)
        self.UniverseSettings.Resolution = Resolution.Daily
        self.AddUniverse(self.CoarseSelectionFunction)
        self.selected = None
    
    def CoarseSelectionFunction(self, coarse):
        #1. Sort coarse by dollar volume
        sortedByDollarVolume = sorted(coarse, key=lambda c: c.DollarVolume, reverse=True)
        #2. Filter out the stocks less than $10 and return selected
        self.selected = [c.Symbol for c in sortedByDollarVolume if c.Price > 10][:10]
        return self.selected
        
    def OnSecuritiesChanged(self, changes): 
        #3. Liquidate securities leaving the universe
        for security in changes.RemovedSecurities:
            self.Liquidate(security.Symbol)
        #4. Allocate 10% holdings to each asset added to the universe
        for security in changes.AddedSecurities:
            self.SetHoldings(security.Symbol, 0.10)
        

```

### Grouping Data with Classes
The momentum strategy hypothesis is that stocks that have performed well in the past will continue to perform well. 

The EMA Momentum Universe selects stocks based on their fast and slow moving average values. An ```ExponentialMovingAverage()``` class is initialized with a number of bars. 

Its period is that bar count, multiplied by the period of the bars used.
```python
class EMAMomentumUniverse(QCAlgorithm):
    
    def Initialize(self):
        self.SetStartDate(2019, 1, 7)
        self.SetEndDate(2019, 4, 1)
        self.SetCash(100000)
        self.UniverseSettings.Resolution = Resolution.Daily
        self.AddUniverse(self.CoarseSelectionFunction)
    
    def CoarseSelectionFunction(self, coarse):
        sortedByDollarVolume = sorted(coarse, key=lambda c: c.DollarVolume, reverse=True) 
        selected = [c.Symbol for c in sortedByDollarVolume if c.Price > 10][:10]
        return selected
        
    def OnSecuritiesChanged(self, changes):  
        for security in changes.RemovedSecurities:
            self.Liquidate(security.Symbol) 
        for security in changes.AddedSecurities:
            self.SetHoldings(security.Symbol, 0.10)

#1. Create a class SelectionData
# So far our algorithms have all been part of one class. 
# We can create additional classes as a way to group together variables 
# for our universe selection and update any indicators all in a few lines of code.
class SelectionData():
    #2. Create a constructor that takes self 
    def __init__(self):
        #2. Save the fast and slow ExponentialMovingAverage
        self.slow = ExponentialMovingAverage(200)
        self.fast = ExponentialMovingAverage(50)
    
    #3. Check if our indicators are ready
    def is_ready(self):
        return self.slow.IsReady and self.fast.IsReady
    
    #4. Use the "indicator.Update" method to update the time and price of both indicators
    def update(self, time, price):
        self.fast.Update(time, price)
        self.slow.Update(time, price)    
    

```

### Applying Class In Universe
Dictionaries are a way of storing data which you can look up later with a "key". 
To perform EMA-universe selection on every security we'll need a SelectionData for every Symbol in our universe.
```python
class EMAMomentumUniverse(QCAlgorithm):
    
    def Initialize(self):
        self.SetStartDate(2019, 1, 7)
        self.SetEndDate(2019, 4, 1)
        self.SetCash(100000)
        self.UniverseSettings.Resolution = Resolution.Daily
        self.AddUniverse(self.CoarseSelectionFunction)
        
        #1. Create our dictionary and save it to self.averages
        self.averages = {}
    
    def CoarseSelectionFunction(self, universe):  
        selected = []
        universe = sorted(universe, key=lambda c: c.DollarVolume, reverse=True)  
        universe = [c for c in universe if c.Price > 10][:100]
        
        # Create loop to use all the coarse data
        for coarse in universe:  
            symbol = coarse.Symbol 
            
            #2. Check if we've created an instance of SelectionData for this symbol
            if symbol not in self.averages:
                #3. Create a new instance of SelectionData and save to averages[symbol]
                self.averages[symbol] = SelectionData()
            #4. Update the symbol with the latest coarse.AdjustedPrice data
            self.averages[symbol].update(self.Time, coarse.AdjustedPrice)
            
            #5. Check if 50-EMA > 200-EMA and if so append the symbol to selected list.
            if self.averages[symbol].fast > self.averages[symbol].slow:
                if self.averages[symbol].is_ready():
                    selected.append(symbol)
        return selected[:10]
        
    def OnSecuritiesChanged(self, changes):
        for security in changes.RemovedSecurities:
            self.Liquidate(security.Symbol)
       
        for security in changes.AddedSecurities:
            self.SetHoldings(security.Symbol, 0.10)
            
class SelectionData(object):
    def __init__(self):
        self.slow = ExponentialMovingAverage(200)
        self.fast = ExponentialMovingAverage(50)
    
    def is_ready(self):
        return self.slow.IsReady and self.fast.IsReady
    
    def update(self, time, price):
        self.fast.Update(time, price)
        self.slow.Update(time, price)

```

### Preparing Indicators with History
We need to prepare our indicators with data. 
With the QuantConnect History API, we request data for symbols as a pandas dataframe.
```python
class EMAMomentumUniverse(QCAlgorithm):
    
    def Initialize(self):
        self.SetStartDate(2019, 1, 7)
        self.SetEndDate(2019, 4, 1)
        self.SetCash(100000)
        self.UniverseSettings.Resolution = Resolution.Daily
        self.AddUniverse(self.CoarseSelectionFunction) 
        self.averages = {}
    
    def CoarseSelectionFunction(self, universe):  
        selected = []
        universe = sorted(universe, key=lambda c: c.DollarVolume, reverse=True)  
        universe = [c for c in universe if c.Price > 10][:100]
        # Now universe only contains 100 stocks, which is not demanding for the server to loop through
        for coarse in universe:  
            symbol = coarse.Symbol
            
            # Pump history into the symbol if the symbol is not found in the average dictionary
            if symbol not in self.averages:
                # 1. Call history to get an array of 200 days of history data
                history = self.History(symbol, 200, Resolution.Daily)
                
                #2. Adjust SelectionData to pass in the history result
                self.averages[symbol] = SelectionData(history) 
            # Simply update the symbol

            self.averages[symbol].update(self.Time, coarse.AdjustedPrice)
            
            if  self.averages[symbol].is_ready() and self.averages[symbol].fast > self.averages[symbol].slow:
                selected.append(symbol)
        # return only 10 stocks
        return selected[:10]
        
    def OnSecuritiesChanged(self, changes):
        for security in changes.RemovedSecurities:
            self.Liquidate(security.Symbol)
       
        for security in changes.AddedSecurities:
            self.SetHoldings(security.Symbol, 0.10)
            
class SelectionData():
    #3. Update the constructor to accept a history array
    def __init__(self, history):
        self.slow = ExponentialMovingAverage(200)
        self.fast = ExponentialMovingAverage(50)
        #4. Loop over the history data and update the indicators
        # The history DataFrame has two indexes; Symbol(0) and Time(1). 
        # You can access the time of a row by using row.Index[1].
        for bar in history.itertuples():
            # Only update for time and close
            self.fast.Update(bar.Index[1], bar.close)
            self.slow.Update(bar.Index[1], bar.close)
    
    def is_ready(self):
        return self.slow.IsReady and self.fast.IsReady
    
    def update(self, time, price):
        self.fast.Update(time, price)
        self.slow.Update(time, price)

```

# The Algorithm Framework

The QC Algorithm Framework is the foundation for building a robust and flexible investment strategy. The framework architecture makes it simple to reuse code and provides the scaffolding for good algorithm design.

The framework is a system that connects the inputs and outputs of five models to ultimately deliver filled orders based on your strategy. The five steps of the system are universe selection, alpha creation, portfolio construction, risk management, and finally execution of trades.
### Framework Overview
The data output of each module flows into the next predictably. 
Assets selected by the Universe Selection Model are fed into your Alpha Model to generate trade signals. The trade signals, or insights, are converted into Portfolio Targets by the Portfolio Construction Model. The Portfolio Targets hold a target share quantity we'd like the algorithm to hold. 
The Risk Management Model ensures our targets are still within safe risk parameters and adjusts the portfolio targets if required. To execute these targets efficiently the Execution model fills the targets efficiently over time.
```python
class FrameworkAlgorithm(QCAlgorithm):
    
    def Initialize(self):
        
        self.SetStartDate(2013, 10, 1)   
        self.SetEndDate(2013, 12, 1)    
        self.SetCash(100000)
        # The desired framework models should be set in your Initialize() method. 
        # Each model is a self contained class which can be imported to your algorithm.
        
        # Universe Models programmatically select assets to avoid selection bias.
        #1. Set the NullUniverseSelectionModel()
        self.SetUniverseSelection(NullUniverseSelectionModel())
        
        # Alpha Models generate predictions on assets in our universe.
        #2. Set the NullAlphaModel()
        self.SetAlpha(NullAlphaModel())
        
        # Portfolio Construction Models optimize the allocatation of resources for best return.
        #3. Set the NullPortfolioConstructionModel()
        self.SetPortfolioConstruction(NullPortfolioConstructionModel())

        # Risk Models monitor real-time risk in the portfolio targets.
        #4. Set the NullRiskManagementModel()
        self.SetRiskManagement(NullRiskManagementModel())

        # Execution Models efficiently break up orders and fill trades.
        #5. Set the NullExecutionModel()
        self.SetExecution(NullExecutionModel())

```

### Universe Selection Model
The Universe Selection Model systematically selects assets. It removes selection bias from the algorithm by filtering for assets to trade with programmed criteria.

Universe Models take in universe data, and return a list of symbol objects. QuantConnect provides dozens of premade universes for you to easily use in your algorithm.
```python
class FrameworkAlgorithm(QCAlgorithm):
    
    def Initialize(self):

        self.SetStartDate(2013, 10, 1)   
        self.SetEndDate(2013, 12, 1)    
        self.SetCash(100000)           
        
        #1. Create a SPY and BND Symbol object that gets passed to the Universe Selection Model
        self.symbols = [Symbol.Create("SPY", SecurityType.Equity, Market.USA), Symbol.Create("BND", SecurityType.Equity, Market.USA)]
        
        # The resolution of the assets added to the universe is configured by the self.UniverseSettings.Resolution property.
        #2. Set the resolution of the universe assets to daily resolution
        self.UniverseSettings.Resolution = Resolution.Daily
        #3. Set a universe using self.SetUniverseSelection(), and pass in a ManualUniverseSelectionModel() 
        # The simplest universe, the Manual Universe Selection Model, accepts a hard coded list of symbols. 
        # You can create Symbol objects with the Symbol.Create function.
        # initialized with the symbols list
        self.SetUniverseSelection(ManualUniverseSelectionModel(self.symbols))
        
        self.SetAlpha(NullAlphaModel())
        self.SetPortfolioConstruction(NullPortfolioConstructionModel())
        self.SetRiskManagement(NullRiskManagementModel())
        self.SetExecution(NullExecutionModel())

```

### Alpha Model
The Alpha Model generates predictions of the expected returns for assets in our universe.

Our Alpha Model consumes price data provided by the Universe Selection Model, and the results are emitted as trade signals called Insights. 

An Insight indicates the direction, magnitude, confidence, weight and period of our prediction.
```python
from datetime import timedelta

# Alpha models must inherit from the AlphaModel base class. 
# They should implement two functions; OnSecuritiesChanged() and Update().
class MOMAlphaModel(AlphaModel):
    
    def __init__(self):
        self.mom = []
    # When assets are added or removed from our universe they will trigger an OnSecuritiesChanged() event. 
    # We should use this event to setup any state required for our alpha calculations. 
    # The method provides the algorithm object, and a list of securities changes. 
    # For example, we could generate indicators for each symbol in this method and save the symbols and indicators to a dictionary.
    def OnSecuritiesChanged(self, algorithm, changes):
        
        # 1. Initialize a 14-day momentum indicator for each symbol
        for security in changes.AddedSecurities:
            symbol = security.Symbol
            self.mom.append({"symbol":symbol, "indicator":algorithm.MOM(symbol, 14, Resolution.Daily)})
    
    # The Update() method is called each time the algorithm receives data for subscribed securities. 
    # After analysis, the Alpha Model should emit insights for assets it would like to trade. 
    def Update(self, algorithm, data):

        #2. Sort the list of dictionaries by indicator in descending order
        ordered = sorted(self.mom, key=lambda kv: kv["indicator"].Current.Value, reverse=True)
        
        # We can return a set of Insights together as a group, which signal to the execution model that the insights need to be traded simultaneously.
        #3. Return a group of insights, emitting InsightDirection.Up for the first item of ordered, and InsightDirection.Flat for the second
        return Insight.Group(
            [
                Insight.Price(ordered[0]["symbol"], timedelta(1), InsightDirection.Up),
                Insight.Price(ordered[1]["symbol"], timedelta(1), InsightDirection.Flat)
            ])
        
class FrameworkAlgorithm(QCAlgorithm):
    
    def Initialize(self):

        self.SetStartDate(2013, 10, 1)   
        self.SetEndDate(2013, 12, 1)   
        self.SetCash(100000)           
        symbols = [Symbol.Create("SPY", SecurityType.Equity, Market.USA), Symbol.Create("BND", SecurityType.Equity, Market.USA)]
        self.UniverseSettings.Resolution = Resolution.Daily
        self.SetUniverseSelection(ManualUniverseSelectionModel(symbols))

        # Call the MOMAlphaModel Class 
        self.SetAlpha(MOMAlphaModel())

        self.SetPortfolioConstruction(NullPortfolioConstructionModel())
        self.SetRiskManagement(NullRiskManagementModel())
        self.SetExecution(NullExecutionModel())

```

### Portfolio Construction Model
The Portfolio Construction Model takes in an array of Insights and calculates the number of shares per asset to hold.

The Portfolio Construction Model receives Insight objects from the Alpha Model and uses them to create PortfolioTarget objects for the Execution Model. 

A PortfolioTarget is a quantity of shares of the asset we'd like to eventually hold.
```python
from datetime import timedelta
class MOMAlphaModel(AlphaModel): 
    def __init__(self):
        self.mom = []
    def OnSecuritiesChanged(self, algorithm, changes):
        for security in changes.AddedSecurities:
            symbol = security.Symbol
            self.mom.append({"symbol":symbol, "indicator":algorithm.MOM(symbol, 14, Resolution.Daily)})
    def Update(self, algorithm, data):
        ordered = sorted(self.mom, key=lambda kv: kv["indicator"].Current.Value, reverse=True)
        return Insight.Group([Insight.Price(ordered[0]['symbol'], timedelta(1), InsightDirection.Up), Insight.Price(ordered[1]['symbol'], timedelta(1), InsightDirection.Flat) ])
 
class FrameworkAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2013, 10, 1)  
        self.SetEndDate(2013, 12, 1)  
        self.SetCash(100000)         
        symbols = [Symbol.Create("SPY", SecurityType.Equity, Market.USA),  Symbol.Create("BND", SecurityType.Equity, Market.USA) ]
        self.UniverseSettings.Resolution = Resolution.Daily
        self.SetUniverseSelection(ManualUniverseSelectionModel(symbols))
        self.SetAlpha(MOMAlphaModel())
    
        #1. Set the Portfolio Model to an Equal Weighting Portfolio Construction Model
        # One simple existing portfolio model we can use is the Equal Weighting Portfolio Construction Model. 
        # It assigns an equal share of the portfolio to insights currently active. 
        # When an insight period ends the holdings in the asset will be liquidated. 
        # The model can be configured to rebalance at different periods to reduce churn. 
        # By default the model scans for expired insights daily.
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        
        self.SetRiskManagement(NullRiskManagementModel())
        self.SetExecution(NullExecutionModel())

```

### Risk Model
The Risk Management Model manages real-time market and portfolio risk. 
There are many potential applications for a risk model. 
For example, it may be used to ensure we are properly diversified or that we're limiting our drawdown.

The Risk Management model receives an array of Portfolio Targets, manages risk on the PortfolioTarget collection and returns final risk adjusted portfolio targets to the Execution Model. Only the changed targets are delivered back.
```python
from datetime import timedelta
class MOMAlphaModel(AlphaModel): 
    def __init__(self):
        self.mom = []
    def OnSecuritiesChanged(self, algorithm, changes):
        for security in changes.AddedSecurities:
            symbol = security.Symbol
            self.mom.append({"symbol":symbol, "indicator":algorithm.MOM(symbol, 14, Resolution.Daily)})
    def Update(self, algorithm, data):
        ordered = sorted(self.mom, key=lambda kv: kv["indicator"].Current.Value, reverse=True)
        return Insight.Group([Insight.Price(ordered[0]['symbol'], timedelta(1), InsightDirection.Up), Insight.Price(ordered[1]['symbol'], timedelta(1), InsightDirection.Flat) ])
 
         
class FrameworkAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2013, 10, 1)  
        self.SetEndDate(2013, 12, 1)   
        self.SetCash(100000)           
        symbols = [Symbol.Create("SPY", SecurityType.Equity, Market.USA),  Symbol.Create("BND", SecurityType.Equity, Market.USA)]
        self.UniverseSettings.Resolution = Resolution.Daily
        self.SetUniverseSelection(ManualUniverseSelectionModel(symbols))
        self.SetAlpha(MOMAlphaModel())
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        # The MaximumDrawdownPercentPerSecurity class seeks to mitigate portfolio risk by limiting the drawdown of a specific asset. 
        # Drawdown is the loss incurred from the peak value of an asset to the trough. 
        # The model is created with the maximum allowed drawdown per security.
        #1. Set the Risk Management handler to use a 2% maximum drawdown
        self.SetRiskManagement(MaximumDrawdownPercentPerSecurity(0.02))
        
        self.SetExecution(NullExecutionModel())

```

### Execution Model
The Execution Model is responsible for fulfilling the Portfolio Targets set by the Portfolio Construction Model. It seeks to find the optimal price and order size to efficiently fill orders.

Execution models are more commonly used for large volume trades. For example, imagine you have 100,000 trades of AAPL shares, the execution model breaks this order up into smaller pieces so it can be optimally filled.
```python
from datetime import timedelta
class MOMAlphaModel(AlphaModel): 
    def __init__(self):
        self.mom = []
    def OnSecuritiesChanged(self, algorithm, changes):
        for security in changes.AddedSecurities:
            symbol = security.Symbol
            self.mom.append({"symbol":symbol, "indicator":algorithm.MOM(symbol, 14, Resolution.Daily)})
    def Update(self, algorithm, data):
        ordered = sorted(self.mom, key=lambda kv: kv["indicator"].Current.Value, reverse=True)
        return Insight.Group([Insight.Price(ordered[0]['symbol'], timedelta(1), InsightDirection.Up), Insight.Price(ordered[1]['symbol'], timedelta(1), InsightDirection.Flat) ])
 
class FrameworkAlgorithm(QCAlgorithm):
    
    def Initialize(self):
        self.SetStartDate(2013, 10, 1)   
        self.SetEndDate(2013, 12, 1)    
        self.SetCash(100000)           
        symbols = [Symbol.Create("SPY", SecurityType.Equity, Market.USA),  Symbol.Create("BND", SecurityType.Equity, Market.USA)]
        self.UniverseSettings.Resolution = Resolution.Daily
        self.SetUniverseSelection(ManualUniverseSelectionModel(symbols))
        self.SetAlpha(MOMAlphaModel())
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        self.SetRiskManagement(MaximumDrawdownPercentPerSecurity(0.02))
        
        # The execution model receives an array of PortfolioTarget objects from the risk management model. 
        # The execution model must optimially fill these targets. 
        # The finaly output of the model is filled orders. 
        # The model is free to delay or spread out the fulfillment of orders as it sees fit.
        #1. Set the Execution Model to an Immediate Execution Model
        self.SetExecution(ImmediateExecutionModel())

```

# Pairs Trading with SMA
Pairs trading is a market neutral strategy, meaning the strategy returns are uncorrelated with market returns. 

A pairs trade is triggered when the difference between a pair of assets crosses an upper or lower threshold. 

The strategy's goal is to sell whichever is the expensive stock of the pair at the time and buy the cheaper stock.
### What is Pairs Trading?
In general, assets are considered a pair when the difference between the two assets mean revert, or are cointegrated. 

Cointegration is a statistical property of a time series. We expect the price difference between the pair to come back to a common long-run mean.

Another criteria for the pair is stationarity. This means the mean and variance of the series do not vary over time. Each asset price-time series could be non-stationary but the price difference between the pair should be stationary.
```python
from datetime import timedelta, datetime

class SMAPairsTrading(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2018, 7, 1)   
        self.SetEndDate(2019, 3, 31)
        self.SetCash(100000)
        
        #1. Using the ManualUniverseSelectionModel(), add the symbols "PEP" and "KO" 
        symbols = [Symbol.Create("PEP", SecurityType.Equity, Market.USA), 
                   Symbol.Create("KO", SecurityType.Equity, Market.USA)]
        self.AddUniverseSelection(ManualUniverseSelectionModel(symbols))
        #2. In Universe Settings, set the resolution to hour
        self.UniverseSettings.Resolution = Resolution.Hour
        
        self.AddAlpha(NullAlphaModel())
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        self.SetExecution(ImmediateExecutionModel())

```

### Creating a Synthetic Asset

The difference in price between our pair of assets is called the spread.

We can think of the spread as a synthetic asset, a combination of assets that have the financial effect becoming a brand new asset.

The spread can be shown as its own time series. The plot below shows the difference between assets.
```python
from datetime import timedelta, datetime

class SMAPairsTrading(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2018, 7, 1)   
        self.SetEndDate(2019, 3, 31)
        self.SetCash(100000)
        
        symbols = [Symbol.Create("PEP", SecurityType.Equity, Market.USA), Symbol.Create("KO", SecurityType.Equity, Market.USA)]
        self.AddUniverseSelection(ManualUniverseSelectionModel(symbols))
        self.UniverseSettings.Resolution = Resolution.Hour
        
        #1. Create an instance of the PairsTradingAlphaModel()
        self.AddAlpha(PairsTradingAlphaModel())
        
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        self.SetExecution(ImmediateExecutionModel())
         
class PairsTradingAlphaModel(AlphaModel): 
    # To model this new "synthetic asset" we create a spread indicator to track its value. 
    # Our new indicator will live inside our Pairs Trading Alpha Model.
    def __init__(self):
        #2. Initialize an empty list self.pair = [ ]
        self.pair = [ ]
    
    # We calculate our spread in our Update() method as the price difference between the two assets.
    def Update(self, algorithm, data): 
        
        #3. Set the price difference calculation to spread.
        spread = self.pair[1].Price - self.pair[0].Price 
        return []
    # The OnSecuritiesChanged() method is used to define the pair to trade.
    # We set our pairs in the OnSecuritiesChanged() method.
    def OnSecuritiesChanged(self, algorithm, changes):
        
        #4. Set self.pair to the changes.AddedSecurities changes
        self.pair = [x for x in changes.AddedSecurities]

```

### Monitoring for Price Deviance

```python
from datetime import timedelta, datetime

class SMAPairsTrading(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2018, 7, 1)   
        self.SetEndDate(2019, 3, 31)
        self.SetCash(100000)
        
        symbols = [Symbol.Create("PEP", SecurityType.Equity, Market.USA), Symbol.Create("KO", SecurityType.Equity, Market.USA)]
        self.AddUniverseSelection(ManualUniverseSelectionModel(symbols))
        self.UniverseSettings.Resolution = Resolution.Hour
        self.UniverseSettings.DataNormalizationMode = DataNormalizationMode.Raw
        self.AddAlpha(PairsTradingAlphaModel())
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        self.SetExecution(ImmediateExecutionModel()) 

class PairsTradingAlphaModel(AlphaModel):

    def __init__(self):
        self.pair = []
        # The absolute difference in price is too noisy to use as a trading signal. 
        # Because of this we will use a Simple Moving Average indicator to calculate mean price difference over a period of time
        #1. Create a 500-period Simple Moving Average Indicator monitoring the spread SMA 
        self.spreadMean = SimpleMovingAverage(500)
        
        #2. Create a 500-period Standard Deviation Indicator monitoring the spread Std 
        self.spreadStd = StandardDeviation(500)
    
    # We update our spread indicator with the latest data in the Update() method by using the arguments algorithm.Time and spread.
    def Update(self, algorithm, data):

        spread = self.pair[1].Price - self.pair[0].Price
        #3. Update the spreadMean indicator with the spread
        self.spreadMean.Update(algorithm.Time, spread)
        #4. Update the spreadStd indicator with the spread
        self.spreadStd.Update(algorithm.Time, spread)
        
        #5. Save our upper threshold and lower threshold
        upperthreshold = self.spreadMean.Current.Value + self.spreadStd.Current.Value
        lowerthreshold = self.spreadMean.Current.Value - self.spreadStd.Current.Value
        
        return []
    
    def OnSecuritiesChanged(self, algorithm, changes):
        self.pair = [x for x in changes.AddedSecurities]

```

### Taking Positions
We can emit an insight group for our pair of assets with a conditional statement that triggers an event when the spread is greater than the threshold.

When the current spread is greater than the upper threshold, it means the difference in asset prices is widening and will likely revert back to the mean. When this happens we go long one asset, and short the other
```python
from datetime import timedelta, datetime

class SMAPairsTrading(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2018, 7, 1)   
        self.SetEndDate(2019, 3, 31)
        self.SetCash(100000)
        
        symbols = [Symbol.Create("PEP", SecurityType.Equity, Market.USA), Symbol.Create("KO", SecurityType.Equity, Market.USA)]
        self.UniverseSettings.Resolution = Resolution.Hour
        self.UniverseSettings.DataNormalizationMode = DataNormalizationMode.Raw
        self.AddUniverseSelection(ManualUniverseSelectionModel(symbols))
        self.AddAlpha(PairsTradingAlphaModel())
        
        # The insights emitted from the Alpha Model are sent to our Portfolio Construction Model to determine our positions. 
        # We are taking a hedged position by offsetting risk from investing in one asset by taking a long position and selling a short position. 
        # This long-short balanced trade is automatically calculated by the EqualWeightingPortfolioConstructionModel.
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        self.SetExecution(ImmediateExecutionModel())
    
    # We can observe our positions taken at the close of each day by creating a new event handler OnEndOfDay(). 
    # This method lives in your algorithm class.
    #3. Use OnEndOfDay() to Log() your positions at the close of each trading day.
    def OnEndOfDay(self, symbol):
        self.Log("Taking a position of " + str(self.Portfolio[symbol].Quantity) + " units of symbol " + str(symbol))
    
class PairsTradingAlphaModel(AlphaModel):

    def __init__(self):
        self.pair = [ ]
        self.spreadMean = SimpleMovingAverage(500)
        self.spreadStd = StandardDeviation(500)
        # An insight is a prediction of the asset movements for a period of time. 
        # This is set with the period arguement of type TimeSpan.
        #1. Set self.period to a 2 hour timedelta 
        self.period = timedelta(hours=2)
        
    def Update(self, algorithm, data):

        spread = self.pair[1].Price - self.pair[0].Price
        self.spreadMean.Update(algorithm.Time, spread)
        self.spreadStd.Update(algorithm.Time, spread)
        
        upperthreshold = self.spreadMean.Current.Value + self.spreadStd.Current.Value
        lowerthreshold = self.spreadMean.Current.Value - self.spreadStd.Current.Value
        
        # In this case in Update() we take a long position on the first asset in the pair with InsightDirection.Up 
        # and we sell the second asset in the pair with InsightDirection.Down.
        #2. Emit an Insight.Group() if the spread is greater than the upperthreshold 
        if spread > upperthreshold:
            return Insight.Group(
                [
                    Insight.Price(self.pair[0].Symbol, self.period, InsightDirection.Up),
                    Insight.Price(self.pair[1].Symbol, self.period, InsightDirection.Down)
                ])
        
        #2. Emit an Insight.Group() if the spread is less than the lowerthreshold
        if spread < lowerthreshold:
            return Insight.Group(
                [
                    Insight.Price(self.pair[0].Symbol, self.period, InsightDirection.Down),
                    Insight.Price(self.pair[1].Symbol, self.period, InsightDirection.Up)
                ])
        
        # If the spread is not greater than the upper or lower threshold, do not return Insights
        return []
    
    def OnSecuritiesChanged(self, algorithm, changes):
        self.pair = [x for x in changes.AddedSecurities]

```

### Warming Our Pair Spread
Finally, we're going to prepare our spread deviation indicators by using a history call to ensure the algorithm can begin trading immediately. 
```python
from datetime import timedelta, datetime

class SMAPairsTrading(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2018, 7, 1)   
        self.SetEndDate(2019, 3, 31)
        self.SetCash(100000)
        
        symbols = [Symbol.Create("PEP", SecurityType.Equity, Market.USA), Symbol.Create("KO", SecurityType.Equity, Market.USA)]
        self.AddUniverseSelection(ManualUniverseSelectionModel(symbols))
        self.UniverseSettings.Resolution = Resolution.Hour
        self.UniverseSettings.DataNormalizationMode = DataNormalizationMode.Raw
        self.AddAlpha(PairsTradingAlphaModel())
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        self.SetExecution(ImmediateExecutionModel())
        
    def OnEndOfDay(self, symbol):
        self.Log("Taking a position of " + str(self.Portfolio[symbol].Quantity) + " units of symbol " + str(symbol))

class PairsTradingAlphaModel(AlphaModel):

    def __init__(self):
        self.pair = [ ]
        self.spreadMean = SimpleMovingAverage(500)
        self.spreadStd = StandardDeviation(500)
        self.period = timedelta(hours=2)
        
    def Update(self, algorithm, data):
        spread = self.pair[1].Price - self.pair[0].Price
        self.spreadMean.Update(algorithm.Time, spread)
        self.spreadStd.Update(algorithm.Time, spread) 
        
        upperthreshold = self.spreadMean.Current.Value + self.spreadStd.Current.Value
        lowerthreshold = self.spreadMean.Current.Value - self.spreadStd.Current.Value

        if spread > upperthreshold:
            return Insight.Group(
                [
                    Insight.Price(self.pair[0].Symbol, self.period, InsightDirection.Up),
                    Insight.Price(self.pair[1].Symbol, self.period, InsightDirection.Down)
                ])
        
        if spread < lowerthreshold:
            return Insight.Group(
                [
                    Insight.Price(self.pair[0].Symbol, self.period, InsightDirection.Down),
                    Insight.Price(self.pair[1].Symbol, self.period, InsightDirection.Up)
                ])

        return []
    
    def OnSecuritiesChanged(self, algorithm, changes):
        self.pair = [x for x in changes.AddedSecurities]
        
        # For example, in OnSecuritiesChanged() we can use list comprehension to iterate through each symbol 
        # in the pair and update it with 500-bars of previous data. 
        # History data for the pair is stored in a dataframe.
        #1. Call for 500 bars of history data for each symbol in the pair and save to the variable history
        history = algorithm.History([x.Symbol for x in self.pair], 500)
        # Using unstack we can reduce the history result pandas dataframe to just the close column.
        #2. Unstack the Pandas data frame to reduce it to the history close price
        history = history.close.unstack(level=0)
        #3. Iterate through the history tuple and update the mean and standard deviation with historical data 
        # tuple 0 is the time, tuple 2 is the second security, tuple 1 is the first security
        for tuple in history.itertuples():
            self.spreadMean.Update(tuple[0], tuple[2]-tuple[1])
            self.spreadStd.Update(tuple[0], tuple[2]-tuple[1])

```
# Liquid Value Stocks

We can easily buy and sell a security when a large volume of its shares are traded daily. A stock is liquid when it can be easily traded without significantly affecting the stock's price.

Value stocks are a category of stocks that seem to be trading for less than their intrinsic value. Value can be measured with fundamental data, which is information other than price itself, like PE Ratios, and Earnings.
### Requesting a Fundamental Universe

```python
from datetime import datetime
from QuantConnect.Data.UniverseSelection import * 
from Selection.FundamentalUniverseSelectionModel import FundamentalUniverseSelectionModel

class LiquidValueStocks(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 10, 1)
        self.SetEndDate(2017, 10, 1)
        self.SetCash(100000)
        self.AddAlpha(NullAlphaModel())
        
        #1. Create an instance of our LiquidValueUniverseSelectionModel and set to hourly resolution
        self.UniverseSettings.Resolution = Resolution.Hour
        self.AddUniverseSelection(LiquidValueUniverseSelectionModel())
        
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        self.SetExecution(ImmediateExecutionModel())
        
# To create a fundamental universe we extend from the FundamentalUniverseSelectionModel base class. 
# When extending from this class we must define the SelectCoarse() and SelectFine() methods. 
# We'll call our universe a LiquidValueUniverseSelectionModel.
# Define the Universe Model Class
class LiquidValueUniverseSelectionModel(FundamentalUniverseSelectionModel):
    
    def __init__(self):
        super().__init__(True, None)
    
    #2. Add an empty SelectCoarse() method with its parameters
    def SelectCoarse(self, algorithm, coarse):
        return Universe.Unchanged
    
    #2. Add an empty SelectFine() method with is parameters    
    def SelectFine(self, algorithm, fine):
        return Universe.Unchanged

```

### Selecting Liquid Stocks
To filter for securities based on fundamental data, we use a two-stage filter. 

We've used the first stage of the filter, the Coarse Universe Selection function, in previous lessons. 

The results of the coarse selection filter are passed into the second filter, the Fine Universe Selection function. 

It filters based on fundamental data and returns symbol objects.
```python
import datetime
from datetime import timedelta
from QuantConnect.Data.UniverseSelection import * 
from Selection.FundamentalUniverseSelectionModel import FundamentalUniverseSelectionModel

class LiquidValueStocks(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 10, 1)
        self.SetEndDate(2017, 10, 1)
        self.SetCash(100000)
        self.UniverseSettings.Resolution = Resolution.Hour
        self.AddUniverseSelection(LiquidValueUniverseSelectionModel())
        self.AddAlpha(NullAlphaModel())
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        self.SetExecution(ImmediateExecutionModel())

class LiquidValueUniverseSelectionModel(FundamentalUniverseSelectionModel):
    
    def __init__(self):
        super().__init__(True, None)
        self.lastMonth = -1 
    
    def SelectCoarse(self, algorithm, coarse):
        
        # We control how frequently we update our universe with our algorithm's Time property. 
        # The Time property has year, month, and day objects. 
        # We can access an index value of the object with an integer value (eg. self.lastMonth = -1). 
        # When it's not time to bring in new data, we return Universe.Unchanged.
        #1. If it isn't time to update data, return the previous symbols 
        if self.lastMonth == algorithm.Time.month:
            return Universe.Unchanged
        #2. Update self.lastMonth with current month to make sure only process once per month
        self.lastMonth = algorithm.Time.month
        
        #3. Sort symbols by dollar volume and if they have fundamental data, in descending order
        sortedByDollarVolume = sorted([x for x in coarse if x.HasFundamentalData], 
                                      key=lambda x: x.DollarVolume, reverse=True)
        
        #4. Return the top 100 Symbols by Dollar Volume 
        return [x.Symbol for x in sortedByDollarVolume[:100]]

```

### Accessing Fine Selection Properties

```python
from datetime import timedelta
from QuantConnect.Data.UniverseSelection import * 
from Selection.FundamentalUniverseSelectionModel import FundamentalUniverseSelectionModel

class LiquidValueStocks(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 10, 1)
        self.SetEndDate(2017, 10, 1)
        self.SetCash(100000)
        self.universe = None
        self.UniverseSettings.Resolution = Resolution.Hour
        self.AddUniverseSelection(LiquidValueUniverseSelectionModel())
        self.AddAlpha(NullAlphaModel())
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        self.SetExecution(ImmediateExecutionModel())
    
class LiquidValueUniverseSelectionModel(FundamentalUniverseSelectionModel):
    
    def __init__(self):
        self.lastMonth = -1 
        super().__init__(True, None)
    
    def SelectCoarse(self, algorithm, coarse):
        if self.lastMonth == algorithm.Time.month:
            return Universe.Unchanged
        self.lastMonth = algorithm.Time.month

        sortedByDollarVolume = sorted([x for x in coarse if x.HasFundamentalData], 
                                      key=lambda x: x.DollarVolume, reverse=True)

        return [x.Symbol for x in sortedByDollarVolume[:100]]
    # Each ticker has 900 fundamental data objects. 
    # The object's categories are: CompanyReferences, SecurityReferences, 
    # FinancialStatements, EarningsReports, OperationRatio, EarningRatios, ValuationRatios. 
    def SelectFine(self, algorithm, fine):
        #1. Sort yields per share
        sortedByYields = sorted(fine, key=lambda f: f.ValuationRatios.EarningYield, reverse=True)
        
        # Like coarse universe selection, we filter our fine universe of symbols based on "fine" objects. 
        # All fundamental data objects are updated once per month and a ticker's financial ratios are updated daily. 
        # We can then filter and sort based on the fundamental data values.
        
        #2. Take top 10 most profitable stocks -- and bottom 10 least profitable stocks
        # Save to the variable self.universe
        self.universe = sortedByYields[:10] + sortedByYields[-10:]
        
        #3. Return the symbol objects by iterating through self.universe with list comprehension
        return [f.Symbol for f in self.universe]

```

### Using Fundamental Data to Emit Insights
```python
from datetime import timedelta
from QuantConnect.Data.UniverseSelection import * 
from Selection.FundamentalUniverseSelectionModel import FundamentalUniverseSelectionModel

class LiquidValueStocks(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2017, 5, 15)
        self.SetEndDate(2017, 7, 15)
        self.SetCash(100000)
        self.UniverseSettings.Resolution = Resolution.Hour
        self.AddUniverseSelection(LiquidValueUniverseSelectionModel())
        
        #1. Create and instance of the LongShortEYAlphaModel
        self.AddAlpha(LongShortEYAlphaModel())
        
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel())
        self.SetExecution(ImmediateExecutionModel())

class LiquidValueUniverseSelectionModel(FundamentalUniverseSelectionModel):
    
    def __init__(self):
        super().__init__(True, None)
        self.lastMonth = -1 
        
    def SelectCoarse(self, algorithm, coarse):
        if self.lastMonth == algorithm.Time.month:
            return Universe.Unchanged
        self.lastMonth = algorithm.Time.month

        sortedByDollarVolume = sorted([x for x in coarse if x.HasFundamentalData],
                                      key=lambda x: x.DollarVolume, reverse=True)

        return [x.Symbol for x in sortedByDollarVolume[:100]]

    def SelectFine(self, algorithm, fine):
        sortedByYields = sorted(fine, key=lambda f: f.ValuationRatios.EarningYield, reverse=True)
        universe = sortedByYields[:10] + sortedByYields[-10:]
        return [f.Symbol for f in universe]

# Define the LongShortAlphaModel class  
class LongShortEYAlphaModel(AlphaModel):

    def __init__(self):
        self.lastMonth = -1

    def Update(self, algorithm, data):
        insights = []
        
        #2. If else statement to emit signals once a month 
        if self.lastMonth == algorithm.Time.month:
            return insights
        self.lastMonth = algorithm.Time.month
        
        # While we pick the securities with universe selection, we decide which we will long and short in the Alpha Model. 
        # Fundamental data selected through the Universe Model is not passed into the Alpha Model, 
        # but we may need to access fundamental data again in our Alpha Model. 
        # We can do this with the security.Fundamentals property.
        
        #3. For loop to emit insights with insight directions 
        # based on whether earnings yield is greater or less than zero once a month
        # the algorithm.ActiveSecurities.Values has active security collection
        for security in algorithm.ActiveSecurities.Values:
            direction = 1 if security.Fundamentals.ValuationRatios.EarningYield > 0 else -1 
            insights.append(Insight.Price(security.Symbol, timedelta(28), direction)) 
        return insights

```

# Tiingo Sentiment Analysis on Stocks
### Introducing Alternative Data

```python
#1. Import Tiingo Data 
from QuantConnect.Data.Custom.Tiingo import *
from datetime import datetime, timedelta
import numpy as np

class TiingoNewsSentimentAlgorithm(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 11, 1)
        self.SetEndDate(2017, 3, 1)  
        
        #2. Add AAPL and NKE symbols to a Manual Universe 
        # You can create Symbol objects with the Symbol.Create function, 
        # which accepts ticker, SecurityType and Market parameters.
        symbols = [Symbol.Create("AAPL", SecurityType.Equity, Market.USA), 
                   Symbol.Create("NKE", SecurityType.Equity, Market.USA)]
        
        # The simplest universe, the Manual Universe Selection Model, accepts a hard coded list of symbols. 
        # We add an instance of our ManualUniverseSelectionModel() in our Initialize() method. 
        # It is initialized with a list of symbols.
        self.SetUniverseSelection(ManualUniverseSelectionModel(symbols))
        
        # 3. Add an instance of the NewsSentimentAlphaModel
        self.SetAlpha(NewsSentimentAlphaModel())
        
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel()) 
        self.SetExecution(ImmediateExecutionModel()) 
        self.SetRiskManagement(NullRiskManagementModel())
        
# 4. Create a NewsSentimentAlphaModel class with Update() and OnSecuritiesChanged() methods
# Should copy this scaffolding from the documentations
class NewsSentimentAlphaModel(AlphaModel):
    def __init__(self): 
        pass
    def Update(self, algorithm, data):
        insights = []
        return insights
    def OnSecuritiesChanged(self, algorithm, changes):
        pass

```

### Adding Alternative Data

```python
from QuantConnect.Data.Custom.Tiingo import *
from datetime import datetime, timedelta
import numpy as np

class TiingoNewsSentimentAlgorithm(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 11, 1)
        self.SetEndDate(2017, 3, 1)  
        symbols = [Symbol.Create("AAPL", SecurityType.Equity, Market.USA), 
                   Symbol.Create("NKE", SecurityType.Equity, Market.USA)]
        self.SetUniverseSelection(ManualUniverseSelectionModel(symbols))
        self.SetAlpha(NewsSentimentAlphaModel())
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel()) 
        self.SetExecution(ImmediateExecutionModel()) 
        self.SetRiskManagement(NullRiskManagementModel())
        
class NewsSentimentAlphaModel(AlphaModel):
    
    def __init__(self):
        self.newsData = {} 
        
        # Assign polarity scores to words
        self.wordScores = {
            "bad": -0.5, "good": 0.5, "negative": -0.5, 
            "great": 0.5, "growth": 0.5, "fail": -0.5, 
            "failed": -0.5, "success": 0.5, "nailed": 0.5,
            "beat": 0.5, "missed": -0.5, "profitable": 0.5,
            "beneficial": 0.5, "right": 0.5, "positive": 0.5, 
            "large":0.5, "attractive": 0.5, "sound": 0.5, 
            "excellent": 0.5, "wrong": -0.5, "unproductive": -0.5, 
            "lose": -0.5, "missing": -0.5, "mishandled": -0.5, 
            "un_lucrative": -0.5, "up": 0.5, "down": -0.5,
            "unproductive": -0.5, "poor": -0.5, "wrong": -0.5,
            "worthwhile": 0.5, "lucrative": 0.5, "solid": 0.5
        } 
            
    def Update(self, algorithm, data):

        insights = []
        # We access alternative data with the Get() helper method. 
        # This helper returns a dictionary where each entry is a list of alternative data objects; keyed by the alt-data symbol. 
        # Like all data in QuantConnect, alternative data feeds also have a Symbol.
        # 2. Access TiingoNews and save to the variable news
        news = data.Get(TiingoNews) 
        
        for article in news.Values:
            # 3. Iterate through the article descriptions and save to the variable words
            # convert text to lowercase, and split the descriptions into a list of words
            words = article.Description.lower().split(" ")
            
            # 4. Assign a self.wordScore to the word if the word exists 
            # in self.wordScores and save to the variable self.score 
            self.score = sum([self.wordScores[word] for word in words if word in self.wordScores])
        return insights
    
    def OnSecuritiesChanged(self, algorithm, changes):
    # Alpha models should not assume a specific list of securities so we should request 
    # alternative data for our securities in the Alpha Model's OnSecuritiesChanged() method 
    # when new securities are added. This allows our alpha to be easily applied on other universes.
        for security in changes.AddedSecurities:
            # 1. When new assets are added to the universe
            # request news data for the assets and save to variable newsAsset
            newsAsset = algorithm.AddData(TiingoNews, security.Symbol)        

```

### Maintaining Algorithm Data

```python
from QuantConnect.Data.Custom.Tiingo import *
from datetime import datetime, timedelta
import numpy as np

class TiingoNewsSentimentAlgorithm(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 11, 1)
        self.SetEndDate(2017, 3, 1)  
        symbols = [Symbol.Create("AAPL", SecurityType.Equity, Market.USA), 
                   Symbol.Create("NKE", SecurityType.Equity, Market.USA)]
        self.SetUniverseSelection(ManualUniverseSelectionModel(symbols))
        self.SetAlpha(NewsSentimentAlphaModel())
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel()) 
        self.SetExecution(ImmediateExecutionModel()) 
        self.SetRiskManagement(NullRiskManagementModel())

#1. Create the NewsData() class with properties self.Symbol and self.Window.
class NewsData():
    def __init__(self, symbol):
        self.Symbol = symbol
        
        # A RollingWindow holds a set of the last n-items of data. 
        # The object in the window with index [0] refers to the most recent item. 
        # As we move from time t=0 forward, our rolling window will shuffle data further along to 
        # a different index until it leaves the window completely. 
        self.Window = RollingWindow[float](100) 

class NewsSentimentAlphaModel(AlphaModel):
    
    def __init__(self): 
        # Each symbol will have its own instance of our data-class. 
        # We should store these in our alpha in a dictionary indexed by symbol for easy access.
        self.newsData = {} 
        
        self.wordScores = {
            "bad": -0.5, "good": 0.5, "negative": -0.5, 
            "great": 0.5, "growth": 0.5, "fail": -0.5, 
            "failed": -0.5, "success": 0.5, "nailed": 0.5,
            "beat": 0.5, "missed": -0.5, "profitable": 0.5,
            "beneficial": 0.5, "right": 0.5, "positive": 0.5, 
            "large":0.5, "attractive": 0.5, "sound": 0.5, 
            "excellent": 0.5, "wrong": -0.5, "unproductive": -0.5, 
            "lose": -0.5, "missing": -0.5, "mishandled": -0.5, 
            "un_lucrative": -0.5, "up": 0.5, "down": -0.5,
            "unproductive": -0.5, "poor": -0.5, "wrong": -0.5,
            "worthwhile": 0.5, "lucrative": 0.5, "solid": 0.5
        } 
    
    def Update(self, algorithm, data):

        insights = []
        news = data.Get(TiingoNews) 

        for article in news.Values:
            words = article.Description.lower().split(" ")
            score = sum([self.wordScores[word] for word in words 
                         if word in self.wordScores])
            
        return insights
        
    def OnSecuritiesChanged(self, algorithm, changes):

        for security in changes.AddedSecurities:
            symbol = security.Symbol
            newsAsset = algorithm.AddData(TiingoNews, symbol)
            # 2. Create a new instance of the NewsData() and store in self.newsData[symbol]
            self.newsData[symbol] = NewsData(newsAsset.Symbol)
        
        # To make our algorithms efficient we should remove data requested when a security leaves the universe. 
        # We can do this in our OnSecuritiesChanged() method. 
        # First we should remove the reference in our data class storage, and then remove the alternative data itself.

        # To remove items from a dictionary we can use the pop(key, default) function that 
        # removes and returns last value from the dictionary, returning default if none is found.
        # 3. Remove news data once assets are removed from our universe
        for security in changes.RemovedSecurities:
            newsData = self.newsData.pop(security.Symbol, None)
            if newsData is not None:
                algorithm.RemoveSecurity(newsData.Symbol)

```

### Managing Insight Emission
```python
from QuantConnect.Data.Custom.Tiingo import *
from datetime import datetime, timedelta
import numpy as np

class TiingoNewsSentimentAlgorithm(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 11, 1)
        self.SetEndDate(2017, 3, 1)  
        symbols = [Symbol.Create("AAPL", SecurityType.Equity, Market.USA), 
                   Symbol.Create("NKE", SecurityType.Equity, Market.USA)]
        self.SetUniverseSelection(ManualUniverseSelectionModel(symbols))
        self.SetAlpha(NewsSentimentAlphaModel())
        self.SetPortfolioConstruction(EqualWeightingPortfolioConstructionModel()) 
        self.SetExecution(ImmediateExecutionModel()) 
        self.SetRiskManagement(NullRiskManagementModel())

class NewsData():
    def __init__(self, symbol):
        self.Symbol = symbol
        self.Window = RollingWindow[float](100)  
        
class NewsSentimentAlphaModel(AlphaModel):
    
    def __init__(self): 
        self.newsData = {}

        self.wordScores = {
            "bad": -0.5, "good": 0.5, "negative": -0.5, 
            "great": 0.5, "growth": 0.5, "fail": -0.5, 
            "failed": -0.5, "success": 0.5, "nailed": 0.5,
            "beat": 0.5, "missed": -0.5, "profitable": 0.5,
            "beneficial": 0.5, "right": 0.5, "positive": 0.5, 
            "large":0.5, "attractive": 0.5, "sound": 0.5, 
            "excellent": 0.5, "wrong": -0.5, "unproductive": -0.5, 
            "lose": -0.5, "missing": -0.5, "mishandled": -0.5, 
            "un_lucrative": -0.5, "up": 0.5, "down": -0.5,
            "unproductive": -0.5, "poor": -0.5, "wrong": -0.5,
            "worthwhile": 0.5, "lucrative": 0.5, "solid": 0.5
        } 
    
    def Update(self, algorithm, data):

        insights = []
        news = data.Get(TiingoNews) 

        for article in news.Values:
            words = article.Description.lower().split(" ")
            score = sum([self.wordScores[word] for word in words if word in self.wordScores])
            
            # Most alternative data related to US-Equities is mapped to an underlying tradable symbol. 
            # We can access that symbol with the Underlying property.
            #1. Get the underlying symbol and save to the variable symbol
            symbol = article.Symbol.Underlying 
            
            # Rolling windows can contain any kind of data which we add with Window.Add(). 
            # Our helper data class should store the rolling window for a specific symbol 
            # to keep relevant data together.
            #2. Add scores to the rolling window associated with its newsData symbol
            self.newsData[symbol].Window.Add(score)
            
            # Alternative data may be delivered at a high frequency. 
            # Emitting an insight for each data point results in a high volume of signals that 
            # may not be as effective as a lower volume of signals emitted for clusters of data.

            # When we emit signals based on aggregated data, individual data anomalies 
            # are less likely to skew our results. One way to aggregate data is by taking 
            # the sum of a rolling window. We can emit signals when aggregated data exceeds a threshold.
            #3. Sum the rolling window scores, save to sentiment
            # If sentiment aggregate score for the time period is greater than 5, emit an up insight
            sentiment = sum(self.newsData[symbol].Window)
            if sentiment > 5:
                insights.append(Insight.Price(symbol, timedelta(1), InsightDirection.Up, None, None))
           
        return insights
    
    def OnSecuritiesChanged(self, algorithm, changes):

        for security in changes.AddedSecurities:
            symbol = security.Symbol
            newsAsset = algorithm.AddData(TiingoNews, symbol)
            self.newsData[symbol] = NewsData(newsAsset.Symbol)

        for security in changes.RemovedSecurities:
            newsData = self.newsData.pop(security.Symbol, None)
            if newsData is not None:
                algorithm.RemoveSecurity(newsData.Symbol)

```

# Sector Weighted Portfolio Construction

A sector balanced portfolio strategy invests in assets across industries. Its goal is to expose the portfolio to less risk from a specific sector and generate more robust returns through a variety of market conditions.

We can use Morningstar fundamental data to select a universe of equities by sector. The asset classification data is stored in the AssetClassification property, with groupings by industry and sector.

Morningstar fundamental data maps equities to 11 asset sector classifications. Smaller industry groupings with similar operational characteristics make up these sector classifications. You can read more about Morningstar asset classifications here. We use the MoringstarSectorCode property to access sector classification codes.
### Selecting Universe By Sector

```python
from datetime import timedelta
from QuantConnect.Data.UniverseSelection import * 
from Selection.FundamentalUniverseSelectionModel import FundamentalUniverseSelectionModel

class SectorBalancedPortfolioConstruction(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 12, 28) 
        self.SetEndDate(2017, 3, 1) 
        self.SetCash(100000) 

        self.UniverseSettings.Resolution = Resolution.Hour
        #1. Set an instance of MyUniverseSelectionModel using self.SetUniverseSelection
        self.SetUniverseSelection(MyUniverseSelectionModel())
        
        self.SetAlpha(ConstantAlphaModel(InsightType.Price, InsightDirection.Up, timedelta(1), 0.025, None))
        self.SetExecution(ImmediateExecutionModel())

class MyUniverseSelectionModel(FundamentalUniverseSelectionModel):

    def __init__(self):
        super().__init__(True, None)

    def SelectCoarse(self, algorithm, coarse):
        filtered = [x for x in coarse if x.HasFundamentalData > 0 and x.Price > 0]
        sortedByDollarVolume = sorted(filtered, key=lambda x: x.DollarVolume, reverse=True)
        return [x.Symbol for x in sortedByDollarVolume][:100]

    # A common way to sort assets is by market capitalization, the market value of outstanding shares for a publicly-traded company. 
    # Calculate market cap by multiplying the number of shares outstanding by the closing price per share. 
    # We access a company's market cap on the FineFundamental object.
    def SelectFine(self, algorithm, fine):
        #2. Save the top 3 securities sorted by MarketCap for the Technology sector to the variable self.technology
        
        # Let's implement a sector weighted strategy. 
        # We'll build upon a pre-defined coarse selection method, which passes in 100 of the top securities sorted by dollar volume into our fine filter.
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.Technology]
        self.technology = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:3]

        #3. Save the top 2 securities sorted by MarketCap for the Financial Services sector to the variable self.financialServices
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.FinancialServices]
        self.financialServices = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:2]

        #4. Save the top 1 securities sorted by MarketCap for the Consumer Goods sector to the variable self.consumerDefensive
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.ConsumerDefensive]
        self.consumerDefensive = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:1]
        
        return [x.Symbol for x in self.technology + self.financialServices + self.consumerDefensive]

```

### Sector Weighting Portfolio Construction
```python
from datetime import timedelta
from QuantConnect.Data.UniverseSelection import * 
from Selection.FundamentalUniverseSelectionModel import FundamentalUniverseSelectionModel
from Portfolio.EqualWeightingPortfolioConstructionModel import EqualWeightingPortfolioConstructionModel

class SectorBalancedPortfolioConstruction(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 12, 28) 
        self.SetEndDate(2017, 3, 1) 
        self.SetCash(100000) 

        self.UniverseSettings.Resolution = Resolution.Hour
        self.SetUniverseSelection(MyUniverseSelectionModel())
        self.SetAlpha(ConstantAlphaModel(InsightType.Price, InsightDirection.Up, timedelta(1), 0.025, None))
        # 1. Set an instance of the MySectorWeightingPortfolioConstructionModel using self.SetPortfolioConstruction
        
        # Portfolio construction models (PCM) seek to generate targets for trade execution systems. 
        # These Portfolio Targets come from the Insights provided by the Alpha model.
        self.SetPortfolioConstruction(MySectorWeightingPortfolioConstructionModel(Resolution.Daily))
        
        self.SetExecution(ImmediateExecutionModel())

class MyUniverseSelectionModel(FundamentalUniverseSelectionModel):

    def __init__(self):
        super().__init__(True, None)

    def SelectCoarse(self, algorithm, coarse):
        filtered = [x for x in coarse if x.HasFundamentalData and x.Price > 0]
        sortedByDollarVolume = sorted(filtered, key=lambda x: x.DollarVolume, reverse=True)
        return [x.Symbol for x in sortedByDollarVolume][:100]

    def SelectFine(self, algorithm, fine):
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.Technology]
        self.technology = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:3]
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.FinancialServices]
        self.financialServices = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:2]
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.ConsumerDefensive]
        self.consumerDefensive = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:1]
        return [x.Symbol for x in self.technology + self.financialServices + self.consumerDefensive]

# The Sector Weighting Portfolio Construction Model (SWPCM) performs a specific kind of portfolio construction where capital assigned to each sector is fixed. 
# This minimizes risk exposure to a specific sector. The SWPCM needs to be used with a universe which selects assets based on fundamental data.

# In the SWPCM, insights for equities in a universe are grouped by their industry sector. 
# Each grouping is assigned an equal percentage of total portfolio value. 
# Each active sector in the universe is assigned equal weight.

class MySectorWeightingPortfolioConstructionModel(EqualWeightingPortfolioConstructionModel):
    
    # The SWPCM derives from the Equal Weighting Portfolio Construction Model (EWPCM). 
    # The coding simplification allows us to inherit most of the methods from the base class and only implement the DetermineTargetPercent() method.
    # 2. In the constructor, pass in the rebalance parameter and set it to daily resolution
    def __init__(self, rebalance = Resolution.Daily):
        
        # The rebalance parameter sets the rebalancing period of the SWPCM. 
        # The parameter accepts a timedelta, function, or Resolution object, which indicates the period between rebalancing.
        
        # The DetermineTargetPercent() method generates a dictionary of insights with the weight of the buying power to be assigned. 
        # The underlying classes of the EWPCM handle creating portfolio targets from those allocations.
        # 3. Initialize the underlying class by passing it through
        
        # When passing arguments to the underlying class, you use Python's super() function prefix, which provides access to the methods in the base class.
        super().__init__(rebalance)
        
        # 4. Initialize an empty dictionary self.symbolBySectorCode to group the securities by sector code
        self.symbolBySectorCode = dict()

```

### Tracking Security Changes
```python
from datetime import timedelta
from QuantConnect.Data.UniverseSelection import * 
from Selection.FundamentalUniverseSelectionModel import FundamentalUniverseSelectionModel
from Portfolio.EqualWeightingPortfolioConstructionModel import EqualWeightingPortfolioConstructionModel

class SectorBalancedPortfolioConstruction(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 12, 28) 
        self.SetEndDate(2017, 3, 1) 
        self.SetCash(100000) 
        self.UniverseSettings.Resolution = Resolution.Hour
        self.SetUniverseSelection(MyUniverseSelectionModel())
        self.SetAlpha(ConstantAlphaModel(InsightType.Price, InsightDirection.Up, timedelta(1), 0.025, None))
        self.SetPortfolioConstruction(MySectorWeightingPortfolioConstructionModel(Resolution.Daily))
        self.SetExecution(ImmediateExecutionModel())

class MyUniverseSelectionModel(FundamentalUniverseSelectionModel):

    def __init__(self):
        super().__init__(True, None)

    def SelectCoarse(self, algorithm, coarse):
        filtered = [x for x in coarse if x.HasFundamentalData and x.Price > 0]
        sortedByDollarVolume = sorted(filtered, key=lambda x: x.DollarVolume, reverse=True)
        return [x.Symbol for x in sortedByDollarVolume][:100]

    def SelectFine(self, algorithm, fine):
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.Technology]
        self.technology = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:3]
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.FinancialServices]
        self.financialServices = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:2]
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.ConsumerDefensive]
        self.consumerDefensive = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:1]
        return [x.Symbol for x in self.technology + self.financialServices + self.consumerDefensive]
        
class MySectorWeightingPortfolioConstructionModel(EqualWeightingPortfolioConstructionModel):

    def __init__(self, rebalance = Resolution.Daily):
        super().__init__(rebalance)
        self.symbolBySectorCode = dict()

    # By default, QuantConnect's universe selection model selects assets daily at midnight. 
    # When the universe constituents change, the algorithm generates an OnSecuritiesChanged event, which receives information about the asset changes. 
    # We can efficiently update these asset changes using the OnSecuritiesChanged() event handler. 
    # The data includes AddedSecurities and RemovedSecurities properties, which return a list of security objects.   
    def OnSecuritiesChanged(self, algorithm, changes):
        
        for security in changes.AddedSecurities:
            
            # Outside of the Universe Selection Model, fundamental data for securities is accessed using the Fundamentals property on the security object.
            #1. When new assets are added to the universe, save the Morningstar sector code 
            # for each security to the variable sectorCode
            
            # One way to store fundamental data is in a container such as a data class or dictionary indexed by the security symbol. 
            # To make our algorithms efficient, when an asset is added or removed to our universe, add or remove the security symbol and its fundamental data from the data storage.
            sectorCode = security.Fundamentals.AssetClassification.MorningstarSectorCode
            
            # 2. If the sectorCode is not in the self.symbolBySectorCode dictionary, create a new list 
            # and append the symbol to the list, keyed by sectorCode in the self.symbolBySectorCode dictionary 
            if sectorCode not in self.symbolBySectorCode:
                self.symbolBySectorCode[sectorCode] = list()
            self.symbolBySectorCode[sectorCode].append(security.Symbol) 
            
        for security in changes.RemovedSecurities:
            #3. For securities that are removed, save their Morningstar sector code to sectorCode
            sectorCode = security.Fundamentals.AssetClassification.MorningstarSectorCode
            
            #4. If the sectorCode is in the self.symbolBySectorCode dictionary
            if sectorCode in self.symbolBySectorCode:
                symbol = security.Symbol
                # If the symbol is in the dictionary's sectorCode list;
                if symbol in self.symbolBySectorCode[sectorCode]:
                    
                    # Then remove the corresponding symbol from the dictionary
                    self.symbolBySectorCode[sectorCode].remove(symbol)
                
        # We use the super() function to avoid using the base class name explicity
        super().OnSecuritiesChanged(algorithm, changes)


```

### Calculating Target Weight Percentages

The Sector Weighted Portfolio Construction Model (SWPCM) assigns each sector a fixed capital allocation. The capital assigned to each sector depends on the number of sectors in your universe.
```python
from datetime import timedelta
from QuantConnect.Data.UniverseSelection import * 
from Selection.FundamentalUniverseSelectionModel import FundamentalUniverseSelectionModel
from Portfolio.EqualWeightingPortfolioConstructionModel import EqualWeightingPortfolioConstructionModel

class SectorBalancedPortfolioConstruction(QCAlgorithm):

    def Initialize(self):
        self.SetStartDate(2016, 12, 28) 
        self.SetEndDate(2017, 3, 1) 
        self.SetCash(100000) 
        self.UniverseSettings.Resolution = Resolution.Hour
        self.SetUniverseSelection(MyUniverseSelectionModel())
        self.SetAlpha(ConstantAlphaModel(InsightType.Price, InsightDirection.Up, timedelta(1), 0.025, None))
        self.SetPortfolioConstruction(MySectorWeightingPortfolioConstructionModel(Resolution.Daily))
        self.SetExecution(ImmediateExecutionModel())

class MyUniverseSelectionModel(FundamentalUniverseSelectionModel):

    def __init__(self):
        super().__init__(True, None)

    def SelectCoarse(self, algorithm, coarse):
        filtered = [x for x in coarse if x.HasFundamentalData and x.Price > 0]
        sortedByDollarVolume = sorted(filtered, key=lambda x: x.DollarVolume, reverse=True)
        return [x.Symbol for x in sortedByDollarVolume][:100]

    def SelectFine(self, algorithm, fine):
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.Technology]
        self.technology = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:3]
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.FinancialServices]
        self.financialServices = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:2]
        filtered = [f for f in fine if f.AssetClassification.MorningstarSectorCode == MorningstarSectorCode.ConsumerDefensive]
        self.consumerDefensive = sorted(filtered, key=lambda f: f.MarketCap, reverse=True)[:1]
        return [x.Symbol for x in self.technology + self.financialServices + self.consumerDefensive]
    
# The Equal Weighting Portfolio Construction Model (EWPCM) base class tracks the currently active insights from the 
# algorithm alpha model and provides a list of currently active signals to the DetermineTargetPercent() method. 
# Active insights are insights from the alpha model, which have not expired and should receive a target allocation.
class MySectorWeightingPortfolioConstructionModel(EqualWeightingPortfolioConstructionModel):

    def __init__(self, rebalance = Resolution.Daily):
        super().__init__(rebalance)
        self.symbolBySectorCode = dict()
        self.result = dict()
    
    # The DetermineTargetPercent() method returns percent targets for each insight in the form of a dictionary of active insights and percent targets. 
    # Through inheritance from the EWPCM, the percent target is turned into a target number of shares for the execution model.    
    def DetermineTargetPercent(self, activeInsights):
        #1. Set the self.sectorBuyingPower before by dividing one by the length of self.symbolBySectorCode
        self.sectorBuyingPower = 1/len(self.symbolBySectorCode)
            
        for sector, symbols in self.symbolBySectorCode.items():
            #2. Search for the active insights in this sector. Save the variable self.insightsInSector
            self.insightsInSector = [insight for insight in activeInsights if insight.Symbol in symbols]
        
            #3. Divide the self.sectorBuyingPower by the length of self.insightsInSector to calculate the variable percent
            # The percent is the weight we'll assign the direction of the insight
            self.percent = self.sectorBuyingPower / len(self.insightsInSector)
        
            #4. For each insight in self.insightsInSector, assign each insight an allocation. 
            # The allocation is calculated by multiplying the insight direction by the self.percent 
            for insight in self.insightsInSector:
                self.result[insight] = insight.Direction * self.percent
        
        return self.result


    def OnSecuritiesChanged(self, algorithm, changes):
        for security in changes.AddedSecurities:
            sectorCode = security.Fundamentals.AssetClassification.MorningstarSectorCode
            if sectorCode not in self.symbolBySectorCode:
                self.symbolBySectorCode[sectorCode] = list()
            self.symbolBySectorCode[sectorCode].append(security.Symbol) 

        for security in changes.RemovedSecurities:
            sectorCode = security.Fundamentals.AssetClassification.MorningstarSectorCode
            if sectorCode in self.symbolBySectorCode:
                symbol = security.Symbol
                if symbol in self.symbolBySectorCode[sectorCode]:
                    self.symbolBySectorCode[sectorCode].remove(symbol)

        super().OnSecuritiesChanged(algorithm, changes)



```
