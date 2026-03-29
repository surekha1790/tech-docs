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

Here **TransactionContext** plays key role. 

#### When one order arrives, multiple things can happen:

   - Order validation
   - Multiple partial fills
   - Several trades generated 
   - Updates to multiple price levels 
   - Order state transitions 
   - Multiple outbound events

#### All of that must be:

   - Atomic (all or nothing)
   - Consistent (no partial side effects)
   - Ordered (events must reflect reality)

The transaction context provides that guarantee.

### Example Without a Transaction Context (Bad)

#### Imagine this happens:

   - Order matches 3 resting orders
   - 2 trade events are emitted
   - ME crashes before:
     - Final order state event 
     - Snapshot update
   - Result:
     - Kafka has partial events 
     - State reconstruction becomes inconsistent 
     - Downstream systems see a broken sequence

❌ This is unacceptable in a trading system.

### What the Transaction Context Does
1. Groups All Side Effects

   - For a single incoming command, the transaction context tracks:
     - In-memory state mutations 
     - Generated domain events 
     - Outgoing responses (Execution Reports)
     - Nothing is published externally until the transaction completes successfully.

2. Enables Atomic Event Publication

   - Events are:
     - Collected in the context 
     - Validated
     - Published as a batch or in strict order
   - If something fails:
     - Context is discarded 
     - No events are published 
     - In-memory state is rolled back (or never committed)

3. Guarantees Event Ordering

   - All events from one command:
     - Share a transaction boundary 
     - Are published sequentially 
     - Preserve causal ordering 
     - This is critical for:
       - Kafka consumers (OQS, MDA, SA)
       - Regulatory replay 
       - Deterministic recovery