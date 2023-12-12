## Trading Platform Components

### Servers
1. Access Server
    
    * Developed by Visual Studio 2022

    * Functionality: 
        
        - receive all requests from terminals
        - store information for every connections(login, group, configuration, etc)
        - communication with the main trade server via REST API which provided by the main trade server
        - communication with history server via Websocket protocol
        
    * Components:
        
        - AccessServer.cpp: main part for this server.  
        In this part, it makes the threads for history server and terminals. Access server use port 9001 for terminals and this port 9001 must allowed incoming connection in firewall settings.
        - WebsocketServer.*: These files are used for receive the requests from terminals and response to them.        `WebsocketServer` class has `onMessage` function which is invoked for requests from terminals. This function parse the requests and send the request to trade server or history server.  
        `AppendHistoryMessageToQueue` function will be registered in 'HistoryConnector' and receive all message from history server in any time and send them to the clients in `handle_history_message` of thread event loop.
        - HistoryConnector.*: These files are used for communicating with history server as websocket client mode.  
        To make the connection with history server, it send login message in `on_open` part.
        It can send a message to history server in `sendMessage` function and receive messages from history server in function handler by set in `set_message_handler` part which use the function `AppendHistoryMessageToQueue` of `WebsocketServer`
        
2. Main Trade Server

    * Developed by Visual Studio 2022

    * Functionality: 
        
        - receive all requests via RESTful API from access server
        - store all information in PostgreSQL database.
        - handle trading operation(creating new order, making deals and positions, close positions, charging commission, etc)
        - manage the client's account and admin, managers (creating new account, editing account information)
        - receive the real-time prices of symbols from history server and calculate the profit&loss, margin, etc.
        
    * Components:
        
        - MainTradeServer.cpp: main part for this server.  
        In this part, it makes a connection for history server to receive the market prices of symbols then prepare `TradeRequestManager` for processing trade requests and `AdminManager` for managing accounts.   
        At last it starts the service for REST API. This server use port 9002 for REST API.

        - RestAPI.h, RestAPI.cpp: These files are used for REST API.
        All requests are received via this api in POST mode.  
        For security, this api use API KEY and AUTH_TOKEN for every login. Currently API_KEY is `3Erg45DF76f3` and AUTH_TOKEN is generated in case of the client log in at first time.

        - TradeRequestManager.*: All requests, which received via REST API, are processed in this manager. This manager includes `TradeAccountManager`and `OrderManager`, `DealManager`, `PositionManager`, `SymbolManager`.  Every manager store their information to PostgreSQL database.   
        The `handle_trade` function will be running as thread for handling trades according to events.    

        - TradeAccountManager.*: These files are used for managing the trade accounts.  

        - OrderManager.*: This is used for handling trading order such as creating new order, modifying order, closing position order.
        After making a order, then it will processed in `DealManager`.

        - DealManager.*: This is used for handling deals such as BUY deal or SELL deal. 
        These deals will processed in `PositionManager`.

        - PositionManager.*: This is used for handling trading positions.

3. History Server
    
    * Developed by Visual Studio 2022

    * Functionality: 
        
        - receive and store all real-time prices from quote provider.        
        - manage symbol configuration.
        - store and providing the 1 minute bar of price history.
        - provide all price update to the other servers.
        
    * Components:

        - HistoryServer.cpp: main part of history server
        - SymbolManager.*: used for symbol configuration of trading platform.
        history server filter the symbol prices by this setting and transmit them to other servers.
        - QuoteReceiver.*: used for receiving price from other external system or liquidity providers.
        - WebsocketServer.*: used for communicating other servers via Websocket protocal. Access server and trade server will be connected to this.
    
### Terminals
All terminals connect to Access Server by using Websocket protocol.

1. Admin terminal
    
    This terminal is used by super admin for trading platform.
    The admin of platform can set and change the platform configuration.

    * Developed by Qt5(above 5.12)

    * Functionality: 
        - account management.
        - group management.
        - symbol management.

2. Manager terminal

    This terminal is used by manager of trading platform.
    The Manager(or Dealer) of platform can be handle all trades of the clients such as make a trade( buy or sell), managing accounts balance(deposit or withdrawal), managing account info, etc.

    * Developed by Qt5(above 5.12)

    * Functionality: 
        - trading management.
        - balance management.
        - dealing management.
        - account management.
    
3. Client terminal

    This terminal is the desktop application for trading clients.

    * Developed by Qt5(above 5.12)

    * Functionality: 
        - draw chart
        - show prices in market watch
        - buy or sell the financial instruments.
        - show trading history
4. Web teriminal

    This terminal is web-based version for trading clients. The functionals are same as the client terminal.  

    Developed by React, This is only frontend part.

5. Mobile terminal

    This terminal is mobile app for trading clients. The functionals are same as the client terminal.  

    Developed by React Native.

## Protocols

Each protocol has JSON format and communicate via Websocket.

1. Client terminal protocol
    Access Server and Client terminal intercommunicate via WebSocket.

    Access Server acts as WebSocket Server and Client terminal as WebSocket Client.

    #### Login

    User must login with account id and password.
    Only after login successfully, connection has to be established.  
    
    **req_code = res_code = 1**

    1. Mobile app send message with (req_code, id, password)  

        id: login id, password: main password for login
        
    2. Access Server check the verification for id, password

    3. Access Server send message verification result  
        * success   
        {res_code: 1, state: 1, login: uint64, name: string, country: string, email: string, group: string, phone: string, balance: double, credit: double, leverage: int}

        * fail   
        {res_code: 1, state: 0}        

    #### Real-time financial price
    Access Server broadcast real-time price for each symbol to client terminal continuously. 

    (res_code, symbol, timestamp, bid, ask, bid_high, bid_low, ask_high, ask_low, open_price, close_price, price_change, digits, status_code)  
    **res_code = 2**

    #### Price history
    
    For drawing chart, Mobile app requests the price history.  
    **req_code = res_code = 3**
        
    1. Request format:   
        
        (req_code, from, to, symbol, interval)

    2. Response format  
        
        (res_code, from, to, symbol, interval, array)
        
    * **from**: timestamp datatype  
    * **to**: timestamp datatype  
    * **symbol**: financial symbol
    * **interval**: interval for chart bar (unit: minutes)
    * **array**: array of chart bar [{datetime: *INT64*, open: *double*, close: *double*, high: *double*, low: *double*},...]

    #### Trade request
    Mobile app requests to Access Server for creating new order or modifying an order, closing an open position, getting position list.      
    **req_code = 4**

    1. New order *type = 0*  
    (req_code, type, order_id(0), order_type, price, volume, s_price, sl, tp, login, symbol, t_time, exp_mins)
        
    2. Modify order *type = 1*  
    (req_code, type, order_id(!=0), price, s_price, sl, tp, login, t_time, exp_mins)

    3. Modify position *type = 2*  
    (req_code, type, order_id(position ticket), sl, tp, login)
    
    4. Delete order *type = 3*  
    (req_code, type, order_id(!=0), login)

    5. Close position *type = 4*  
    (req_code, type, order_id(position_ticket), login)

    6. Position list *type = 5*  
    (req_code, type, login)

    * **order_type**: new order type  
        0: BUY  
        1: SELL  
        2: BUY_LIMIT  
        3: SELL_LIMIT  
        4: BUY_STOP
        5: SELL_STOP  
        6: BUY_STOP_LIMIT  
        7: SELL_STOP_LIMIT
    * **price**: order price
    * **volume**: lot * 10000
    * **s_price**: limit price for stop limit order
    * **sl**: stop loss
    * **tp**: take profit
    * **t_time**: type for expiratation time  
        0: GTC(Good Till Canceled)  
        1: Today(Intraday)  
        2: Specified  
    * **exp_mins**: expired minutes from now for *Specified* type of expiration time

    #### Open Positions

    When mobile app request the open positions or create a new position via client terminal or web terminal, Access Server send a open position list to mobile app.  
    **res_code = 4**
    
    (res_code, array, balance, equity, credit, margin)
    
    array: [{"symbol":"EURUSD", "ticket":1354, "time":1634234, "type":"buy" or "sell", "volume":100, "price":1.09234, "sl":1.08000, "tp":1.09999, "csize":1.0, "r_profit":1.0}, ...]
    
    * symbol : financial symbol
    * ticket : position ticket
    * time : position openning time    
    * type : position type (ref: order_type)
    * volume: volume for lot
    * price : open price
    * sl: stop loss
    * tp: take profit
    * csize: contract size
    * r_profit: profit rate

    #### Trade history
    **req_code = res_code = 5**

    Mobile app requests the Trading history for current logged account.  
        (req_code, from, to, login)

    Access Server responses as trading history with positions and orders, deals.
        (res_code, positions, orders, deals)
        
    ##### position structure
    * time
    * time_c: position closed time
    * ticket
    * type
    * volume
    * symbol
    * price_o
    * price_c
    * sl
    * tp
    * profit
    * swap
    * fee
    * margin_rate
    * digits

    ##### order structure

    * symbol
    * ticket
    * time
    * type
    * volume
    * price
    * sl
    * tp
    * state   
        STARTED: 0  
        PLACED: 1  
        CANCELED: 2  
        PARTIAL: 3  
        FILLED: 4  
        REJECTED: 5  
        EXPIRED: 6          
        (below Order state from gateway)
        REQUEST_ADD: 7  
        REQUEST_MODIFY: 8  
        REQUEST_CANCEL: 9  
        
    * s_price (optional)
    * digits

    ##### deal structure
        
    * time
    * ticket
    * order
    * symbol
    * type
    * direction
    * volume
    * price
    * commission
    * fee
    * swap
    * profit
    * digits

    #### Pending Orders
    
    When client place a new pending order via client terminal or web terminal, Access Server send the pending order list to mobile app.  
    **res_code = 6**

    format:  {res_code:6, array:[...]}
        
    * array: [{"symbol":"EURUSD", "ticket":1354, "time":1634234, "type": 2~7, "volume":100, "price":1.09234, "s_price":1.09234 "sl":1.08000, "tp":1.09999}, ...]


    #### Trade Response
    Mobile app receives the status code of trading request.
    {res_code:7, status:..., res_ord:1234, msg:"..."}
    
    * res_code = 7,
    * status : return code
    {
    
        10009: 'Request fulfilled',
        
        10019: 'Not enough money',
        
        10018: 'Market is closed',
        
        10006: 'Request rejected',
        
        10008: 'An order placed as a result of the request',
        
        10014: 'Invalid Volume'
    }
    * res_ord : result order ticket
    * msg: result return message
        

    #### Symbol Configuration List

    Just after successful login, server send the current symbol configuration to the mobile client as JSON array. **res_code = 8**  

    fromat: {res_code:8, array:[...]}

    * array: [{"symbol":"EURUSD", "path":"-", "category":"-", "desc": "description", "digits":5, "min_vol":100, "step_vol":100, "max_vol":1000000}, ...]



2. manager terminal protocol

    #### Login

    **req_code = res_code = 1**

    1. Client should send message with 
        (req_code, id, password, type)  
        id: login id, password: main password for login, type: manager or admin(manager is 10, admin is 20)
        
    2. Server should send message verification result (res_code, state)  
        state: 1, success; 0, otherwise  

    #### Real-time tick price
    (res_code, symbol, timestamp, bid, ask, bid_high, bid_low, ask_high, ask_low, open_price, close_price, price_change, digits, status_code)  
    **res_code = 2**

    #### Account list
    **req_code = res_code = 9**
        (req_code, dealer)
        (res_code, array)
        * **array**: [{login: *INT64*, name: *string*, balance: *double*, profit: *double*, email: *string*, country: *string*, phone: *string*, group: *string*, agent: *INT64*},...]


    #### Trade request
    **req_code = 4**
    **res_code = 7**
    1. New order *type = 0*  
    (req_code, login, dealer, type, order_id(0), order_type, price, volume, s_price, sl, tp, login, symbol, t_time, exp_mins)
        
    2. Modify order *type = 1*  
    (req_code, login, dealer, type, order_id(!=0), price, s_price, sl, tp, login, t_time, exp_mins)
    
    3. Modify position *type = 2*  
    (req_code, login, dealer, type, order_id(position ticket), sl, tp)
    
    4. Delete order *type = 3*  
    (req_code, login, dealer, type, order_id(!=0))

    5. Close position *type = 4*  
    (req_code, login, dealer, type, order_id(position_ticket), price)
    
    6. Position list *type = 5*  
    (req_code, type, login, dealer)

    7. Close position *type = 6*  
    (req_code, login, dealer, type, order_id(position_ticket))


    * **order_type**: new order type  
        0: BUY  
        1: SELL  
        2: BUY_LIMIT  
        3: SELL_LIMIT  
        4: BUY_STOP
        5: SELL_STOP  
        6: BUY_STOP_LIMIT  
        7: SELL_STOP_LIMIT
    * **price**: order price
    * **volume**: lot * 10000
    * **s_price**: limit price for stop limit order
    * **sl**: stop loss
    * **tp**: take profit
    * **t_time**: type for expiratation time  
        0: GTC(Good Till Canceled)  
        1: Today(Intraday)  
        2: Specified  
    * **exp_mins**: expired minutes from now for *Specified* type of expiration time

    answer: (res_code, type, status)
    
    #### Deal Operation
    
    **req_code = res_code = 10**
    
    1. balance *type = 0*  
    (req_code, type, balance, comment, login, dealer)
    deposit case: balance > 0
    withdraw case: balance < 0
        
    2. credit *type = 1*  
    (req_code, type, login, amount, comment, dealer)
    
    3. charge *type = 2*  
    (req_code, type, login, amount, comment, dealer)
    
    (res_code, balance, credit, equity, margin, array, login)
    * **array**: [{ticket,type, profit, comment, login, time}]

    #### Account Operation
    
    **req_code = 11**
    
    1. (req_code, login, dealer, type, name, email, password, phone, country, group, agent)
        
        type==0: create, type==1: update, type==2: delete, type==3 check
        
    2. (res_code, state, login)
        
        state==0 false, state==1 true
        
    #### Trade Response (is same as open positions and pending orders of common user's protocol)
    
    ##### Positions
    
        rescode = 4
    
    ##### Orders
    
        rescode = 6
        
    #### Group Management( Create, Update, List)
    
    **req_code = res_code = 12**

    ##### request structure

        { req_code: 12, type: 0~2, id: 0 or current_id, name: "forex"}
        
    ##### response structure

        { res_code:12, array: [{id: *INT64*, name: *string*}] }


    #### Manager account management
    
    **req_code = res_code = 13**

    ##### request structure

        { req_code: 13, type: 0~4, login: 0~1999, name:"manager1", password: "password", rights: 1}
        
    1. create manager
        type = 0, login = 0, rights:(0 None, 1 Admin, 2 Manager)
    2. update manager
        type = 1, login = *updated login*
    3. delete manager
        type = 2, login = *deleted login*
    4. list managers
        type = 3

    ##### response structure
        { res_code: 13, array:[{login: 1000, name:"" , group_id: 2 , group_name:"managers" , rights: 1,}]}
        
    #### Dealing
    **res_code = 14**
        {res_code: 14, for_login, by_dealer, request, answer, time}

    #### Journal
    **res_code = 15**
        {res_code: 15, server:"Trade", message:"close ...", time}
        
    ##### Dealing Operation
    
    **req_code = res_code = 16**    
     {req_code, type, ...}

    1. type = 0
    
        {req_code, type, dealing}
        dealing == 1: start dealing, dealing == 0: stop dealing
    
    2. type = 1
    
        {res_code, type, login, ticket, order_type, price, volume, sl, tp, symbol, deal_log}          
        
        deal_log: dealing message.
        
    3. type = 2
    
        {req_code, type, status, price, ticket}
    
        status == 0: reject, status == 1: confirm  
        ticket: order ticket  
        
        This is dealing response  
    
        {res_code, type, login, symbol, ticket, order_id, time, pos_type, volume, price, sl, tp, csize}
        ticket: positon ticket

3. Admin terminal portocol

    #### Login

    **req_code = res_code = 1**

    1. Client should send message with 
        (req_code, id, password, type)  
        id: login id, password: main password for login, type: manager or admin(manager is 10, admin is 100)
        
    2. Server should send message verification result (res_code, state)  
        state: 1, success; 0, otherwise  

    #### Account list
    **req_code = res_code = 9**
        (req_code, dealer)
        (res_code, array)
        * **array**: [{login: *INT64*, name: *string*, balance: *double*, profit: *double*, email: *string*, country: *string*, phone: *string*, group: *string*, leverage},...]

        
    #### Group Management( Create, Update, List)
    **req_code = res_code = 104**

    ##### request structure

        { 
            req_code: 104,
            type: 0~2,
            id: 0 or current_id,
            name: "deal", 
            currency, digits, risk_management, margin_call_level, stop_out_level, in, margin_checkbox: 1, 
            symbols: 
                [{symbol:"forex/EURUSD", ...}, {symbol:"forex/", default_spread:0(or 1), spread_diff: 12, balance_diff: 4, default_vol:0(or 1), min_vol, max_vol, step_vol, default_margin: 0(or 1), init_margin, hedged_margin, maintain_margin }], 
            commissions: 
            [
                {
                    name: "",
                    desc: "",
                    symbol: "*",
                    currency: "",
                    mode: 0~2,
                    range: 0~2,
                    charge: 0~2,
                    entry: 0~2,
                    action: 0~2,
                    profit: 0~2,
                    reason: (ref:EnCommReasonFlags),
                    tiers: [{ mode: 0~7, type: 0~1, value: double, min: double, max: double, from: double, to: double, currency: ""}, {...}]
                },
                ...
            ]
                
        }
        
    ##### response structure

    { res_code:104, array: [{id: *INT64*, name: *string*, currency, digits, risk_management, margin_call_level, stop_out_level, in, margin_checkbox: 1, symbols: [{symbol:"forex/EURUSD", ...}, {symbol:"forex/", default_spread:0(or 1), spread_diff: 12, balance_diff: 4, default_vol:0(or 1), min_vol, max_vol, step_vol, default_margin: 0(or 1), init_margin, hedged_margin, maintain_margin }], commissions: [{ name: "", desc: "", symbol: "*", currency: "", mode: 0~2, range: 0~2, charge: 0~2, entry: 0~2, action: 0~2, profit: 0~2, reason: (ref:EnCommReasonFlags), tiers: [{ mode: 0~7, type: 0~1, value: double, min: double, max: double, from: double, to: double, currency: ""}, {...}]}, ...]}]}

    #### Manager account management
    **req_code = res_code = 103**

    ##### request structure

    { req_code: 103, type: 0~4, login: 0~1999, name:"manager1", password: "password", groups:["real","demo"]}
    
    1. create manager
        type = 0, login = 0
    2. update manager
        type = 1, login = *updated login*
    3. delete manager
        type = 2, login = *deleted login*
    4. list managers
        type = 3

    ##### response structure

    { res_code: 103, array:[{login: 1000, name:"" , group_id: 2, groups:["real","demo"]}]}

    #### History
    **req_code = res_code = 101**

    ##### request structure

    { req_code: 101, from, to, type}  

    type==1: order, type==2: deal
        
    ##### response structure

    { res_code: 101, type, array:[{order or deal}]}
        
    #### Symbol Operation
    **req_code = 102**

    {req_code, type, ...}

    1. type = 0 (symbol add)

        {req_code, type, symbol: 'EURUSD', path: 'forex', sym_type, desc, digits, base_cur: 'EUR', profit_cur: 'USD', margin_cur: 'EUR', base_digits, profit_digits, margin_digits, c_size: 100000, min_vol, max_vol, step_vol, init_margin: 0.0, hedged_margin: 100000, maintain_margin: 0.0, spread, spread_balance}  
        {res_code, type, array: [{symbol: 'EURUSD', path: 'forex', sym_type, desc, digits, base_cur: 'EUR', profit_cur: 'USD', margin_cur: 'EUR', base_digits, profit_digits, margin_digits, c_size: 100000, min_vol, max_vol, step_vol, init_margin: 0.0, hedged_margin: 100000, maintain_margin: 0.0, spread, spread_balance}]}  

    2. type = 1 (symbol edit)

        {req_code, type, symbol: 'EURUSD', sym_type, descr, digits, base_cur: 'EUR', profit_cur: 'USD', margin_cur: 'EUR', base_digits, profit_digits, margin_digits, c_size: 100000, min_vol, max_vol, step_vol, init_margin: 0.0, hedged_margin: 100000, maintain_margin: 0.0, spread, spread_balance}        

        {res_code, type, array: [{symbol: 'EURUSD', path: 'forex', sym_type, desc, digits, base_cur: 'EUR', profit_cur: 'USD', margin_cur: 'EUR', base_digits, profit_digits, margin_digits, c_size: 100000, min_vol, max_vol, step_vol, init_margin: 0.0, hedged_margin: 100000, maintain_margin: 0.0, spread, spread_balance}]}  
        
    3. type = 2 (symbol delete)

        {req_code, type, symbol, path}  

        {res_code, type, symbol, path}  


## Deployment
1. PostgreSQL Installation
2. Platform Installation
3. Setting Firewall config
