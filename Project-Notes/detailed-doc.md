

## 1. Quote flow (Market Maker side)

A **quote** is a two-sided price from one Market Maker for one instrument: a **bid** (I buy at X) and an **ask** (I sell at Y), sent together.

**Key rule:** each MM has only **one live quote per instrument**. A new quote does not add to the old one. It **replaces** it on both sides.

### Flow

1. MM sends the quote in FIX (Quote / Mass Quote) to the **OEGW**.
2. OEGW converts it to **SBE** and sends it to the **ME**.
3. ME removes the MM's old bid and ask, then inserts the new bid and ask.
4. If the new bid or ask **crosses** the other side, it trades right away.
5. ME sends an ack to OEGW (FIX back to the MM), the event to **Kafka**, and the new top-of-book to the **MDA → TREP → MDGW**.

### Example: quote replace

Turbo on DAX. Book before (MM1 is the issuer):

| Bids | Asks |
|---|---|
| MM1: 500 @ 9.98 | MM1: 500 @ 10.02 |
| Retail A: 100 @ 9.95 | Retail B: 200 @ 10.05 |

Top of book = **9.98 / 10.02**.

The DAX goes up, so MM1 sends a new quote: **bid 10.08 × 500, ask 10.12 × 500**.

The ME:

- removes MM1's 9.98 bid and 10.02 ask
- inserts bid 10.08, then checks it against the asks: the best ask is now Retail B at 10.05. **10.08 ≥ 10.05, so it crosses.** MM1 buys 200 @ 10.05 from Retail B (the resting order's price wins).
- the remaining bid of 300 @ 10.08 rests
- inserts ask 500 @ 10.12 (nothing to cross)

Book after:

| Bids | Asks |
|---|---|
| MM1: 300 @ 10.08 | MM1: 500 @ 10.12 |
| Retail A: 100 @ 9.95 | |

New top of book = **10.08 / 10.12**, which gets published to the market.

### Points to say in an interview

- A replace usually **loses time priority**, because the new quote goes to the back of its price level. Some venues keep priority if only the size goes down. Check what your ME does.
- Quotes update far more often than orders (the price follows the underlying all the time), so the replace path must be very cheap. That's why it uses an O(1) lookup by MM + instrument, then unlink and insert.
- On a **halt** (Circuit Breaker) or a **knock-out**, quotes are pulled or rejected.

## 2. Order flow (an order comes in and matches)

1. A broker sends a **New Order Single (35=D)** in FIX to the OEGW.
2. OEGW checks the session, converts to SBE, and sends it to the ME.
3. ME **validates** the order: is the instrument active, not halted, and is the price inside the band? If not, it is rejected.
4. ME **matches** using **price-time priority**: best opposite price first, and at the same price, the earliest order first.
5. Anything left over **rests** on the book, or is cancelled if it's IOC/FOK.
6. ME sends events to the OEGW (**Execution Report 35=8**) for both sides, to Kafka (OQS saves them to the DB), and trades to the MDA (the market sees the print).

### Example: order matching

Asks on the book:

| Asks | Time |
|---|---|
| MM1: 300 @ 10.12 | 1st |
| Retail C: 100 @ 10.12 | 2nd |
| MM2: 400 @ 10.15 | 3rd |

A retail **buy 350 @ 10.13 (limit)** comes in:

- The best ask is 10.12, and 10.12 ≤ 10.13, so it trades. MM1 came first, so the buyer gets **300 from MM1** (MM1 fully filled).
- 50 are still needed. Next in line at 10.12 is Retail C, so the buyer gets **50 from C**. C keeps 50 resting.
- The buy is done (300 + 50 = 350). MM2 at 10.15 is untouched because it is above the 10.13 limit.

Result: **2 trades at 10.12**, exec reports go to the buyer, MM1 and C, and the best ask is now **50 @ 10.12**.

If the buy had been **500 @ 10.13** instead: it fills 400 at 10.12, then the 100 left over **rests as a bid at 10.13**. That becomes the new best bid.

## Quote vs order in one line

- **Order**: one side, many per participant, each one stays until filled or cancelled.
- **Quote**: both sides, one per MM per instrument, and every new quote **replaces** the old one.

Both are matched in the same order book with the same price-time rules.
