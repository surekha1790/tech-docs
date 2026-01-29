#### Flow

DXXL -> IS

#### DXXL : 
Listing/Delisting/updating instruments is done in this service. Pulls instruments from GOA/S3 aws bucket.

#### IS:
Associating trading groups, parameter sets for instruments. Saved down to database by issuing security Id.
And events out LLM messages to ME and then picked up by MDA to Trep. Then MDGW subscribed to TREP and fetch the data.

#### OEGW:

Takes care of orders. Orders comes from Market Makers in fix messages. OEGW receives fix messages and encode them 
into SBE messages and sends them to ME for matching. Matching Engine validates the order and create order placed/rejected/cancelled
event. This event will be converted to SBE message and send back to OEGW. 
The final result (Execution Report) will be sent to Market Maker in fix message.

#### ME:
Receive SBE orders/Quotes messages from OEGW. Validates them and places quotes/orders then events out back to OEGW in SBE format.

Use LLM RCMS cluster to elect leader/member
Quotes comes from Market Makers to OEGW and flows through ME and evented out to MDA
ME follows CQRS pattern, all the events emitted will be stored in Kafka topic and OQS will be consumed by OQS to write onto the database.
Snapshots each book in 3sec interval periodically.

##### Note:
every instrument listed in the system is assigned with a security Id and saved in database.
Last character of the security id will decide on which tier it should go.

#### CB: 
Tracks volatility to halt instruments

#### MDA: 
Adapter between ME and Trep. ME publishes trade events and instrument events which will be published to TREP. MM can subscribe to
TREP and consume the data.
#### MDGW: 
produce output data from the books from ME

#### SA: 
Third party Scila market abuse adapter

#### OQS:
Consumes events from ME and writes orders, trades, execution reports. Persists data to database.

#### Example Fix Message:
8=FIX.4.2|9=176|35=D|49=SENDER|56=TARGET|34=2|52=20240319-14:35:00|11=12345|21=1|55=AAPL|54=1|
38=100|40=2|44=150.50|10=128|. 
It includes a header (version, target), body (symbol, price, quantity), and trailer (checksum)


