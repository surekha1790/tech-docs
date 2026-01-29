#### CQRS Pattern Explained :

It's a pattern which separates the path that changes the system from the path that reads the system

CQRS is used between ME and OQS services

ME reads the events from OEGW, validate and match the orders but does not perform any DB actions because
it causes significant delay in process. So this model achieves ultra low latency.

ME emits events like 
 - orderPlaced
 - OrderCanceled
 - OrderRejected
 - TradeExecuted

#### Why CQRS Is a Perfect Fit Here 
1. Latency Isolation

   - ME never blocks on DB
   - Order matching stays microsecond-level
   - Read-heavy workloads don’t affect execution

2. Scalability

   - ME scales by instrument tier / security ID
   - OQS scales independently based on query load
   - Kafka acts as a buffer and fan-out mechanism

3. Fault Tolerance & Recovery

   #### If OQS goes down:
    - ME continues matching
    - Events stay safely in Kafka
    - OQS can replay events and rebuild state

   #### If ME restarts:
 
    - It restores state using:
    - Kafka event replay 
    - Periodic order book snapshots (every 3 seconds)