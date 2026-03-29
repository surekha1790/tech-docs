#### Flow

DXXL -> IS

#### DXXL : 
Listing/Delisting/updating instruments is done in this service. Pulls instruments from GOA/S3 aws bucket.

#### IS:
Associating trading groups, parameter sets for instruments. Saved down to database by issuing security Id.
And events out LLM messages to ME and then picked up by MDA to Trep. Then MDGW subscribed to TREP and fetch the data.

#### OEGW:

Takes care of orders. Orders comes from Market Makers in fix messages. OEGW has a fix acceptor to accept fix messages.
OEGW receives fix messages and encode them into SBE messages and sends them to ME for matching. Matching Engine validates
the order and create order placed/rejected/cancelled event. This event will be converted to SBE message and send back to OEGW. 
The final result (Execution Report) will be sent to Market Maker in fix message.

#### ME:
Receive SBE orders/Quotes messages from OEGW. Validates them and places quotes/orders then events out back to OEGW in SBE format.

Use LLM RCMS cluster to elect leader/member
Quotes comes from Market Makers to OEGW and flows through ME and evented out to MDA
ME follows CQRS pattern, all the events emitted will be stored in Kafka topic and OQS will be consumed by OQS to write onto the database.
Snapshots each book in 3sec interval periodically.

ME stores all order books in memory to achieve low latency. It stores order book with buy, sell sides.
Each instrument has its own in-memory order book. It does not cause issues because kafka messages are source of records.
Messages can be replayed or snapshots can be recovered.
ME doesn't cause any memory issues because it works on tier based and each tier works on subset of instruments.
It stores only active orders per instruments. It discards filled, rejected, cancelled, traded events.

##### Note:
every instrument listed in the system is assigned with a security Id and saved in database.
Last character of the security id will decide on which tier it should go.

#### CB: 
Monitor price and volatility, abnormal market moments and halt trading.
CB send halt event to ME, ME halt instrument, orders, does not allow any new orders. This flows to
MDA -> TREP -> MDGW.
Also send to OEGW which sends execution reports for rejections.

#### MDA: 
Adapter between ME and Trep. ME publishes trade events and instrument events which will be published to TREP. MM can subscribe to
TREP and consume the data.
The Matching Engine publishes trade events and instrument state changes to MDA because ME is the system of record for market truth. 
MDA acts purely as an adapter to disseminate deterministic, ordered market events to TREP and downstream consumers, 
while keeping ME isolated from protocol and fan-out concerns

#### MDGW: 
produce output data from the books from ME.
MDGW consumes trade and market state events from TREP and publishes normalized,
read-only market data such as trades, top-of-book, and instrument status to downstream consumers. 
It never handles orders or client-specific information

#### Why MDA present in between ME and MDGW 
MDA exists to decouple the Matching Engine from the market data distribution layer.
It acts as an adapter and protective boundary, translating internal ME events into market-facing data, 
handling fan-out, buffering, and protocol concerns so the ME remains deterministic, low-latency, 
and isolated from downstream failures

#### SA: 
Third party Scila market abuse adapter

#### OQS:
Consumes events from ME and writes orders, trades, execution reports. Persists data to database.

#### Example Fix Message:
8=FIX.4.2|9=176|35=D|49=SENDER|56=TARGET|34=2|52=20240319-14:35:00|11=12345|21=1|55=AAPL|54=1|
38=100|40=2|44=150.50|10=128|. 
It includes a header (version, target), body (symbol, price, quantity), and trailer (checksum)


### What is Snapshot: 

Snapshot is a copy of order book at a point of time. This is main part of ME to continue matching without missing any data when 
it is restarted or crashed.
Snapshots are stores on local disc Ex: a folder is created with today's date and stores snapshots for all instruments order books.
When ME restarted it recovers these snapshots, and it also contains Kafka offset to read events. Once snapshots are recovered it consumes
Order events (published OEGW ) from the last offset.

### Order Book Details
An order book contains all active, unmatched orders for a single instrument, organized by side and price level 
with strict price–time priority, along with metadata such as best bid/ask and last trade, enabling deterministic 
matching and recovery via snapshots

- Order ID (ME-generated, unique)
- Security ID 
- Side (Buy / Sell)
- Price 
- Remaining Quantity 
- Original Quantity 
- Order Type 
  - Limit 
  - Market (usually not stored)
- Time priority 
- Sequence number / timestamp 
- Order state 
  - Active 
  - Partially filled 
- Trading group / participant 
- Order flags 
  - IOC / FOK 
  - Post-only