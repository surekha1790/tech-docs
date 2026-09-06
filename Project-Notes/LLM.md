What it is

Low Latency Messaging is a messaging library — a way for one program to send data to other programs. Same job as Kafka or RabbitMQ, but built for speed above everything else.

The one your diagram shows is most likely IBM's product, literally named MQ Low Latency Messaging and abbreviated LLM. Its competitors were Informatica Ultra Messaging (originally 29West) and TIBCO FTL.

To give you a sense of scale:

Approach	Typical time
Kafka	1–10 milliseconds
Normal TCP messaging	50–500 microseconds
LLM multicast	1–10 microseconds

That's roughly a thousand times faster than Kafka. In trading, that gap is the whole reason the product exists.

Why normal messaging is slow

Think about what Kafka does when you send a message:

Your program hands it to the Kafka client
Sent over the network to a broker (another machine)
Broker writes it to disk
Broker copies it to replica brokers
Broker acknowledges
Consumers connect and pull it

That's several network hops and a disk write. Excellent if you want the message stored forever and replayable. Terrible if you need it delivered in microseconds.

The four tricks LLM uses

1. No middleman. LLM has no broker. The sending program talks straight to the receiving programs over the network. One hop instead of three.

2. Multicast instead of one-to-one.

Normally, sending to 50 receivers means making 50 separate connections and sending 50 copies. Like phoning fifty people one at a time.

Multicast is like a radio broadcast. You send one copy onto the network, and the network switches duplicate it to everyone who's tuned in. Fifty receivers cost the sender exactly the same as one.

This is why it's perfect for market data — one price update needs to reach every consumer simultaneously.

3. UDP instead of TCP.

TCP is like registered post: guaranteed delivery, in order, with a signature. All those guarantees cost time — handshakes, acknowledgements, retries.

UDP is like dropping a postcard in a letterbox. Fast, but no promise it arrives.

LLM uses UDP for the speed, then adds its own lightweight reliability on top: it numbers every message, and if a receiver notices a gap in the numbers, it asks for just that one to be resent. Much cheaper than TCP's full machinery.

4. Kernel bypass. Normally your program asks the operating system to send data, which means handing off to the OS, copying buffers, waiting your turn. Kernel bypass lets the program talk to the network card directly, skipping the OS entirely. Saves several microseconds.

What you give up

Nothing is free, and knowing the costs is what makes you sound like you understand it rather than just having read about it.

No memory. Messages fly past and are gone. If your program wasn't listening at that moment, you missed it — there's no log to go back to. Kafka keeps everything for days; LLM keeps nothing.

Slow receivers lose data. If a receiver can't keep up, messages get dropped. The sender doesn't slow down for it, by design — one slow consumer must never slow down the whole venue.

NAK storms. If a packet gets lost and a hundred receivers all missed it, all hundred ask for a resend at once. That burst can make congestion worse. Real operational hazard with multicast.

Datacentre only. Multicast needs switches configured for it. It doesn't work across the internet, only inside a controlled network you own.

Why your system has both LLM and Kafka

This is the part worth understanding properly, because it explains a design choice that looks redundant until you see it.

LLM is the fast lane. Instrument events from ISS to the matching engine, and market data fan-out. Microseconds. No history. Get it there now.

Kafka is the memory. The Recovery Events log. Milliseconds, but it stores everything and you can replay it.

They're solving different problems. You couldn't use Kafka for the hot path — milliseconds is far too slow for an engine matching in microseconds. And you couldn't use LLM for recovery — it has no history to replay.

So the pattern is: LLM for delivery, Kafka for durability. Same events, two paths, two jobs.

If it comes up in an interview

LLM is Low Latency Messaging — brokerless UDP multicast middleware built for financial markets. Instead of going through a broker and hitting disk like Kafka, it sends one copy onto the network and lets the switches fan it out to every subscriber, with its own sequence-number-based retransmission layer instead of TCP. It's microseconds rather than milliseconds. The trade-off is no persistence and no replay, which is exactly why we also have Kafka — LLM handles delivery on the hot path, Kafka handles durability and recovery.

Two things to check internally, since they're plausible follow-ups: whether it's the IBM product or Informatica Ultra Messaging, and whether IG migrated — IBM has been withdrawing MQ LLM, with Confinity LLM V3 as the successor.
