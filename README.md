# MAY_30

```python

from ibapi.client import EClient
from ibapi.wrapper import EWrapper
from ibapi.contract import Contract
from ibapi.order import Order
import threading
import time
import pandas as pd
import pytz

tickers=['DAX']
F=1

class TradingApp(EWrapper, EClient):
    def __init__(self):
        EClient.__init__(self,self)
        self.Expiries=[]
        self.expiries={}
        self.underlyingPrice={}
        self.df_data={}
        self.data={}
        self.variables={}
        for ticker in tickers:
            self.expiries[ticker]={}
        
        
    def tickPrice(self, reqId, tickType, price, attrib):
        super().tickPrice(reqId, tickType, price, attrib)
        print(tickType,price)
        if tickType == 68:
            if reqId < 100:
                    self.underlyingPrice[tickers[reqId]] = price
                    streaming_event.set()  
                    

    def historicalData(self, reqId, bar):
        if reqId not in self.data:
            
            self.data[reqId] = [{
                 "date":bar.date,
                 "open":bar.open,
                                 "high":bar.high,
                                 "low":bar.low,
                                 "close":bar.close}]
        else:
            self.data[reqId].append({
                 "date":bar.date,
                 "open":bar.open,
                                 "high":bar.high,
                                 "low":bar.low,
                                 "close":bar.close})
    def historicalDataEnd(self, reqId: int, start: str, end: str):
            super().historicalDataEnd(reqId, start, end)
            print("HistoricalDataEnd. ReqId:", reqId, "from", start, "to", end)
            self.df_data[tickers[reqId]] = pd.DataFrame(self.data[reqId])
            PH=max(self.df_data[tickers[reqId]]['high'])
            PL=min(self.df_data[tickers[reqId]]['low'])
            PC=self.underlyingPrice[tickers[reqId]]

            SD= F * 20*(PH-PL)/(PH+PL) 
            TSC= PC + SD*PC/100
            TSP= PC - SD*PC/100 
            
            self.variables[tickers[reqId]]={'PH':PH,'PL':PL,'PC':PC,'SD':SD,'TSC':TSC,'TSP':TSP}
            print(self.variables[tickers[reqId]])




            hist_event.set()


    def contractDetails(self, reqId, contractDetails):
        strike=str(contractDetails.contract.strike)
        expiry=str(contractDetails.contract.lastTradeDateOrContractMonth)
        
        if expiry not in  self.expiries[tickers[reqId]] :
            self.expiries[tickers[reqId]][expiry]=[strike]
        else:
            self.expiries[tickers[reqId]][expiry].append(strike) 


    def stop_streaming(self,reqId):
                super().cancelMktData(reqId)
                #print('streaming stopped for ',reqId)  

            
    def contractDetailsEnd(self, reqId):
        super().contractDetailsEnd(reqId)
        contract_event.set()
        

        


def websocket_con():
    app.run()
app = TradingApp()      
app.connect("127.0.0.1", 7497, clientId=1)
con_thread = threading.Thread(target=websocket_con, daemon=True)
con_thread.start()
time.sleep(3) 
print('h')

def usTechOpt(symbol,sec_type="OPT",currency="EUR",exchange="EUREX"):
    contract = Contract()
    contract.symbol = symbol
    contract.secType = sec_type
    contract.currency = currency
    contract.exchange = exchange
    contract.right = "C"
    # contract.strike = 16000
    # contract.lastTradeDateOrContractMonth = "20230602"
    # contract.multiplier = 1
    return contract 






contract_event = threading.Event()
def Contracts_Download():
    for ticker in tickers:
        contract_event.clear() 
        app.reqContractDetails(tickers.index(ticker), usTechOpt(ticker)) 
        contract_event.wait() 
Contracts_Download()
print('Succefuuly Dowloaded Contracts')





# ST='DAX'
# WKS=0
# F=1
# NR=1


def usTechStk(symbol,sec_type="IND",currency="EUR",exchange="EUREX"):
    contract = Contract()
    contract.symbol = symbol
    contract.secType = sec_type
    contract.currency = currency
    contract.exchange = exchange
    return contract 






streaming_event = threading.Event()
def streamStockLtp(tickers):
    print('LTP Streaming Started')
    for ticker in tickers:
        app.reqMarketDataType(3)
        app.reqMktData(reqId=tickers.index(ticker), 
                        contract=usTechStk(ticker),
                        genericTickList="",
                        snapshot=False,
                        regulatorySnapshot=False,
                        mktDataOptions=[])
        # time.sleep(1)
        streaming_event.wait()
        app.stop_streaming(tickers.index(ticker))
       

streamThread = threading.Thread(target=streamStockLtp, args=(tickers,))
streamThread.start()
time.sleep(3) 
print(app.underlyingPrice)






hist_event=threading.Event()
def hist_Data(tickers):
     for ticker in tickers:
        app.reqMarketDataType(3)

        app.reqHistoricalData(reqId=tickers.index(ticker), 
                      contract=usTechStk('DAX'),
                      endDateTime='',
                      durationStr='1 Y',
                      barSizeSetting='1 day',
                      whatToShow='TRADES',
                      useRTH=1,
                      formatDate=1,
                      keepUpToDate=0,
                      chartOptions=[])	 
        hist_event.wait()
        app.stop_streaming(tickers.index(ticker))



histThread = threading.Thread(target=hist_Data, args=(tickers,))
histThread.start()
time.sleep(3) 




print(app.variables)

import time

WKS=1

#creating object of the limit order class - will be used as a parameter for other function calls
def mktOrder(direction,quantity):
    order = Order()
    order.action = direction
    order.orderType = "MKT"
    order.totalQuantity = quantity
    return order


def usTechOptt(symbol,right,expiry,strike,sec_type="OPT",currency="EUR",exchange="EUREX"):
    contract = Contract()
    contract.symbol = symbol
    contract.secType = sec_type
    contract.currency = currency
    contract.exchange = exchange
    contract.strike = strike #update me
    contract.right = right
    contract.lastTradeDateOrContractMonth = expiry
    return contract



for ticker in tickers:
    expiries=sorted(list(app.expiries['DAX'].keys()))
    expiry_found=expiries[WKS]
    strikes=sorted(app.expiries[ticker][expiry_found])
    TSP=float(app.variables[ticker]['TSP'])
    found_strike=None
    minn=float('inf')
    # print(strikes)
    for strike in strikes:
        strike=float(strike)
        if abs(strike-TSP)<minn:
         found_strike=strike
         minn=abs(strike-TSP)
    print(expiry_found,found_strike)
    app.placeOrder(100000,usTechOptt("DAX","P",str(expiry_found),float(found_strike)),mktOrder("BUY",1)) 
    time.sleep(5) 

         

    
    



         

```
