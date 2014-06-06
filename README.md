docs
====

>---
- **[Rpc](#rpc)**
- **[Attention](#attention)**
- **[Public Functions](#public-functions)**
	- [Ticker](#ticker) 
	- [OrderBook](#orderbook)
	- [LastTrade](#lasttrade)
	- [History](#history)
	- [HistorySince](#historysince)
- **[Private Functions](#private-functions)**
	- [ApiNewBidOrder](#apinewbidorder)
	- [ApiNewAskOrder](#apinewaskorder)
	- [ApiCancelBidOrder](#apicancelbidorder)
	- [ApiCancelAskOrder](#apicancelaskorder)
	- [ApiCurrentOrders](#apicurrentorders)
	- [ApiHistoryTrades](#apihistorytrades)

- **[Parameters](#parameters)**
	- [trading pair](#trading-pair)
	- [order type](#ordertype)
	- [operation](#operation)
- **[Sign](#sign)**
- **[Domains](#domains)**
- **[Uri](#uri)**
- **[SDK](#sdk)**

>---

### Rpc 
We use the rpc framework [Hprose](https://github.com/hprose/) to provide our api service which both support http and tcp protocol. 

Later we will also release the restful api and it only supports http protocol.

If you want to trade frequently, rpc api on tcp protocol should be better. :) Supporting high frequency trading is what UniLTC are proud of.

Please download [Hprose](https://github.com/hprose/), import hprose in your code, then you can invoke functions and get results. Simple as it is! 

### Attention
**Please be noted that**
* All data related to **rate**, **volume**, **total**(rate\*volume) and **balance** you get from the api are multipled by a **div**, since we are processing data using **int64** in backend in Golang. So you should divide the data by a specific **div** before any calculation.
	* div_rate = 10000 = 1e4
	* div_volume = 10000 = 1e4
	* div_total = 100000000 = 1e8
	* div_balance = 100000000 = 1e8
* All function names are case sensitive.
* Please spend 5 minutes getting familiar with [Hprose](https://github.com/hprose/) before you develop your own app. Although it is not restful api which you may be much more familiar with, hprose worth you 5 minutes since it really eases your job. You can focus on the logic, but not the communication which hprose perfectly does for you.

### Public Functions 

#### Ticker

Parameters:
* trading pair 

Return data:
* ticker
* error

Datatype
```go
	type ticker struct {
		Timestamp int64	// timestamp of ticker last updated
		Last, High, Low, Buy, Sell, Volume float64 // last 24 hours data
		Average float64 // last 24 hours average rate
	}
```

Example
```go
ticker, err := stub.Ticker("LTC_USD")
```

#### OrderBook

Parameters
* trading pair

Return data
* orderBook 
* error

More details

*By default, the max length of order array is 100. You can get no more than 100 bid or ask orders by this function.*

Datatype
```go
	// type of order
	type order struct {
		OrderId int64
		OrderType string
		Rate, Volume, Total float64
	}

	// type of depth 
	type depth map[string][]order

	// type returned by the function
	type orderBook struct {
		Depth depth
		Timestamp int64	// timestamp of order book that is last updated
	}
```

Example
```go
	depth, err := stub.OrderBook("LTC_USD")
	// print bid orders
	for _, v := range depth.Depth["bid"] {
		log.Printf("%v", v)
	}
```

#### LastTrade

Parameters
* trading pair

Return Data
* trade
* error

Datatype
```go
	type trade struct {
		Timestamp int64
		Type string
		Rate, Volume, Total float64
	}
```

Example
```go
	lastTrade, err := stub.LastTrade("LTC_BTC")
	log.Println(lastTrade.Timestamp)
```

#### History

Parameters
* trading pair
* lastN (it specifies how many history trades you want to fetch)

Return Data
* []trade (an array of trades)
* error

More details

*Max length of the trade array is 1000*

Example
```go
	// get the last 50 trades
	historyTrades, err := stub.History("LTC_USD", 50)
	// do something...
```

#### HistorySince

Parameters
* trading pair
* timestamp (it specifies from what time you want to fetch the trade data)

Return Data
* []trade (an array of trades)
* error

More details

*Max length of the trade array is 1000 and the unixnano timestamp is 19-digit integer*

Example
```go
	// get the trades since the same time of yesterday
	historySince, err := stub.HistorySince("LTC_USD", time.Now().AddDate(0, 0, -1).UnixNano())
	// do something...
```

### Private Functions 

#### ApiNewBidOrder 

Parameters
* api-key
* sign
* trading pair
* rate
* amount

Return Data

#### ApiNewAskOrder 

Parameters
* api-key
* sign
* trading pair
* rate
* amount

Return Data

#### ApiCancelBidOrder

Parameters
* api-key
* sign
* trading pair
* order id

Return Data
* error

#### ApiCancelAskOrder

Parameters
* api-key
* sign
* trading pair
* order id

Return Data
* error

#### ApiCurrentOrders

Parameters
* api-key
* sign
* trading pair

Return Data
* depth (refer to public api)
* error

#### ApiHistoryTrades

Parameters
* api-key
* sign
* trading pair

Return Data
* []trade (refer to public api, an array of trades)
* error

More details

*Max length of the trade array is 200* 

### Parameters

#### trading pair 
* LTC_BTC
* LTC_USD
* LTC_EUR

#### ordertype
* bid
* ask

#### operation
* newbid
* newask
* cancelbid
* cancelask
* currentorders
* historytrades

### Sign 

The sign is derived from operation and parameters from left to right. They are signed by your own secret key according to Scrypt algorithm, and then base64 encoded. 

Scrypt:
* n = 1 << 14 = 2 ^ 14 = 16384
* r = 1 << 3 = 2 ^ 3 = 8
* p = 1 << 0 = 2 ^ 0 = 1
* key length = 1 << 7 = w ^ 7 = 128

Example:

if you want to place a new bid order with rate 10 USD/LTC and volume 13 LTC, then you should invoke the function like this:
	
```go
	// scrypt 
	n := 16384
	r := 8
	p := 1
	keyLen := 128

	// data to encrypt
	operation := "newbid"	// specify it according to operations
	pair := "LTC_USD"	// trading pairs
	rate := 1e5		// please check the div_rate
	vol := 13e4		// please check the div_vol
	dataToEncrypt := fmt.Sprintf("%v%v%v%v", operation, pair, rate, vol) // the data being signed

	scryptData := scryptEncrypt(dataToEncrypt, "your-secret-key", n, r, p, keyLen)
	encryptedRes := base64.Encode(scryptData)
	filled, notFilled, err := NewBidOrder("your-key", encryptedRes, "LTC_USD", 1e5, 13e4)
```

### Domains

choose on your own for shortest response time

* usapi.uniltc.com (api server located in US)
* londonapi.uniltc.com (api server located in London)

### Uri 
<table>
<tr>
<th>protocol</th>
<th>uri</th>
</tr>
<tr>
<tr><td>http 	</td>		<td>http://$domain_name/</td></tr>
<tr><td>https 	</td>		<td>https://$domain_name/</td></tr>
<tr><td>tcp 	</td>		<td>tcp://$domain_name:9090/</td></tr>
<tr><td>tcp secure 	</td>	<td>tcp://$domain_name:9091/</td></tr>
<tr><td>websocket </td>		<td>ws://$domain_name/</td></tr>
<tr><td>websocket secure</td>	<td>wss://$domain_name/</td></tr>
</tr>
</table>

### SDK 
<table>
<tr>
<th>language</th>
<th>http supported</th>
<th>tcp supported</th>
<th>websocket supported</th>
<th>sdk url</th>
</tr>
<tr>
<tr><td>golang</td>	<td>yes</td>	<td>yes</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/golang/">click me</a></td></tr>
<tr><td>java</td>	<td>yes</td>	<td>yes</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
<tr><td>dotnet</td>	<td>yes</td>	<td>yes</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
<tr><td>php</td>	<td>yes</td>	<td>no</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
<tr><td>python</td>	<td>yes</td>	<td>no</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
<tr><td>ruby</td>	<td>yes</td>	<td>yes</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
<tr><td>nodejs</td>	<td>yes</td>	<td>no</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
<tr><td>js</td>		<td>yes</td>	<td>no</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
<tr><td>html5</td>	<td>yes</td>	<td>no</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
<tr><td>perl</td>	<td>yes</td>	<td>no</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
<tr><td>aauto</td>	<td>yes</td>	<td>no</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
<tr><td>Dlang</td>	<td>yes</td>	<td>no</td>	<td>no</td>	<td><a href="https://github.com/UniLTC-API/">unavailable</a></td></tr>
</tr>
</table>

