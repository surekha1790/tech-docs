### * Kafka Architecture
- Kafka is a distributed system which is used to produce/consume messages.
- It will be in clustur environment where it contains multiple broker (nothing but server) to maintain backups in replicas.
- Each broker contains a topic name, on which messages are published and consumed by multiple consumers.
- Each topic contains partitions to distribute messages. Order maintained within the partition.
- Each partition contains offset to track the position of last read message
- To consume message concurrently, we can create a consumer group
- **Consumer Group:**
    - Consumers which shares the same name come under consumer group
    - Kafka assigns one partition for each consumer within the consumer group
    - No consumer in the consumer group read messages from other partitions assigned to different consumer
 - **Relicas:**
     - Leader broker contains messages and all messages will be duplicated in other brokers (followers)
     - ISR(In sync replica) become next leader when leader fails 
 - Earlier Zookeeper used to manager brokers, electing leader but it is additional operational overhead.
 - So, latest kafka replaced with kraft

### * What are atmost-once, atleast-once, exactly-once models
- **Atmost-once:** May lose, never duplicate
    * Messages are published 0 or 1 time
    * No duplicates but lost messages
    * Consume message, commit offset and process the message. If consumer could not process it then it is a loss.
    * Config:
      ```
      spring:
          kafka:
            producer:
              acks: "1"                 # or "0" for true fire-and-forget
              retries: 0                # no retries -> may drop, never re-send
              properties:
                enable.idempotence: false   # required: idempotence needs acks=all
            consumer:
              enable-auto-commit: true      # <-- the key line: commit on a timer
              auto-commit-interval: 1000    # ms
              auto-offset-reset: latest     # skip backlog; loss-tolerant
      ```
      * It is suitable for metrics, logs, telemetry
 - **Atleast-once:** No loss, Duplicates
     * Message are published 1 or more times
     * Once message is processed then offset will be committed
     * For unsuccessfull process, message will be processed again on kafka restart.
     * Idempotancy should be implemented to avoid processing duplicate message.
     * Services like orders, inventory should use this.
     * This is kafka's default and most common choice.
     * Add DefaultErrorHandler to retry the failed messages.
     * Config:
       ```
       spring:
          kafka:
            producer:
              acks: all
              retries: 3
              properties:
                enable.idempotence: true                       # dedupe producer retries
                max.in.flight.requests.per.connection: 5
            consumer:
              enable-auto-commit: false                        # Spring default; commit AFTER listener returns
              auto-offset-reset: earliest
              key-deserializer:   org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
              value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
              properties:
                spring.deserializer.key.delegate.class:   org.apache.kafka.common.serialization.StringDeserializer
                spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
            listener:
              ack-mode: record          # commit after EACH record (BATCH is the default; RECORD avoids re-doing a whole batch)
              concurrency: 3            # **concurrency means it creates 3 consumers to consume messages. If there are two such instances running then there will be 6 consumers**
       ```
- **Exactly-once:** No loss, no dup
    * Adds the idempotent producer + transactions + committed-read isolation.
    * Messages are consumed and processed only once.
    * Atomicity only within the kafka transaction manager, if there is a DB call or rest call then there is not guarante for atomicity.
    * Again, it is required to maintain idempotatency in DB
 
    ```
    spring:
      kafka:
        producer:
          acks: all
          transaction-id-prefix: ${spring.application.name}-tx-   # <-- enables Kafka transactions
          properties:
            enable.idempotence: true
        consumer:
          enable-auto-commit: false
          isolation-level: read_committed                          # <-- only see committed records
        listener:
          ack-mode: batch          # offsets are committed as part of the transaction
    ```
                  
### * Difference between "latest" & "earliest" 
- Earliest consumes messages from starting and latest consumes messages published after consumer joined the group.
- But this config is used only when there is no valid committed offset available to figure what was the last read message.
- Lets say kafka already published 100 messages. When server starts for the first time, it reads from 0 when config is earliest. When config is latest then it will consume from 100.
- Now, the consumer processed till 210 and committed offset. If it restarts then it reads from 210 irrespective of the config as committed offset is available.

### * How to handle scenarios where messages processing fails or something wrong after processing before comitting 
- Scenario 1: Message is processed and failed during DB call
- How to handle: DefaultErrorHandle will pause consumer and retry failed message for n times. If it fails after retries then it moves it to dead letter topic.
                 - auto commit is false: 
                     * If there is not default error handler then offset can not be commited, listner polls again and again for the same record infinitly (Partition Starvation).
                     * If the error is handled with try/catch block then it will swallow the error and commit the offset
                     * If we want to avoid retry and catch the exception then move the failed messages to DLT to investigate and process later.
                 - auto commit is true:
                     * Kafka commit the offset for interval time irrespective of the process status (failure/success) and data will be loss
                 It is recommended way to keep the auto commit offset to false and handle the ack by record/manually. Kafka by default acks batch wise.

- Scenario 2: Message is not in correct format and unable to deserialize it.
- How to handle: ErrorHandlingDeserializer pushes the message to dead letter topic

- Scenario 3: Consumer crash/rebalance before offset commit
- How to handle: After restart it automatically redeliver the message. Idempotency should be maintained.

- Scenario 4: Slow consumer where consumere does not consume message within max.poll.interval.ms time, so broker declares it as dead and rebalance it. So message will be republished
- How to handle: Idempotency implementation avoid duplicate updates. Config can be adjusted by increasing max poll time and decreasing max poll records.

### * How does Kafka decide which partition a record goes to?
- Each topic splits into multiple partitions and record goes to exactly one record. Partition will be selected by producer.
- **Key present** → `partition = hash(key) % numPartitions`
- **Withot Key** → Kafka follows **Sticky partition**. Earlier it used to follow round robin way to push message in partitons when no key present but it cause low throughtput and less messages in each partition.
                   So, in sticky partition, one partition will be picked up and messages will be pushed into it until batch is full. Producer maintains a buffer for each partition, so when it is full then it switches to                      another partition.
- **Custom** → implement `Partitioner` for domain-specific routing (e.g. route by region).

### * Why does partitioning matter, and how do you choose a key?
- Partition is to maintain parallelism and ordering.
- For example, same order id events goes to same partition so that they follow the order.

### * How many partitions should a topic have?
- There is no particular number for it.
- Should be decided based on number of consumers in consumer group to achieve parallelism.
- More consumers than partitions make the consumer idle
- More partitions than consumer does not achieve great parallelism as multiple partitions are assigned to one consumer.
- **Rule of thumb:** `partitions = max(target_throughput / per_partition_throughput, desired_consumer_parallelism)`

### * Can you change the partition count later? What breaks?
- It is possible to increase partition count later but it breaks few things.
- Old messages stays in the old partitions and new messages are distributed based on new partitions count
- So, there is possibility of missing order for the messages such as e-commerce sequence like order created, payment success, order confirmed, shipped, delivered etc.
- For such scenarios it causes issues as order will be disturbed due to new partitions and there is a possibility that order confirmed is process first than payment success.
- To avoid these, there are couple of ways
- **Create over partitions** : Create more partitions than requirement so there will be room to avoid scaling in future.
- **New Topic** : Create new v2 version topic and publish new messages to new topic and consume from both topics, so old messages are ordered and consumed until it is finished.
- **Versioning** : Here we not only create a new topic with more partitions, we store versioned topic name in db along with the event(lets say order). So while publishing, will fetch the topic version and publish on to                       the same topic always, so we do not mess with the ordering even increase partitions.

### * How Kafka commits the offset ?
- When auto commit is on then kafka commit the offset in specific time intervals set by kafka irrespective of the message processing result.
- When auto commit is off then kafka listner takes care of the commit and it commits the offset after successfull processing of a batch before polling. If unsuccessfull process defaulterror handler will retry.
- It can be controlled more with config `spring.kafka.listener.ack-mode:`.
- **RECORD:** commit offset per record instead of batch.
- **MANUAL/MANUAL_IMMEDIATE:** Need to inject Acknowledgment in the listner and call ack.ack() manually to commit offset.

### * How does Kafka achieve durability and fault tolerance
- Kafka maintains append only log on disk for each partition where it sequenctially append messages at the end.
- Kafka also has replicas. One broker will be leader and remaining are followers where insyc replicas will be also be selected.
- replicas store all messaged which are flowing through leader and in sync replica is the one which is elgible to be leader when leader crashes.
- Producers can use acks=all to ensure messages are acknowledged only after replication to in-sync replicas.
- kafka also has retention policy, based on which messages will be stored.

### * What is the ISR, and how does it relate to `acks`?
- ISR (insync replica) which is same as leader which stores all messages flowing through leader.
- To be eligible for becoming leader, it should be ISR.
- acks config is to keep the brokers in sync.
- ack = 0: fire and forget, do not wait for ack.
- ack = 1 : acks to only leader.
- ack = all : leader waits until all replicas are in sync before ack.

### * Why the classic "RF=3, min.insync.replicas=2, acks=all"?
- min.insync.replicas=2 means there should be atleast replicas should be in sync with leader when acks=all.
- Anyone replica does not acknowledge the write then it is considered as failure.
- If one broker is down for the partition then we still have data in another replica.
- If both brokers are down then partition does not accept the data and throws NotEnoughReplicas.

### * What happens when a broker fails?
- When broker fails, Controller in Kraft mode will elect another leader from in sync replicas.
- When broker returns it joins as follower, catches up and re enter ISR.

### * What is unclean leader election
- When all ISRs are lost and broker will be elected based on config unclean.leader.election.enable.
- **True:**  even ISR is not available, kafka choose out of sync broker to maintain availability.
- **False:** It waits until ISR is available. No data loss but no availability.

### * What if a partition is piled up due to a bad key
- It is called as  hot partition/skewed key.
- Should use better and approriate key to avoid this situation.
- Increasing consumers does not help because one partition is assigned to one consumer only.
- Increasing partitions also does not help because problem is with key and always goes to same partition.
- Implement parallalism to process quickly.
- append shard to key but does not guanrantee the order.
- Chosing right key is the better choice.

### * What is a rebalance, what triggers it, and why is it painful?
- When consumers added or removed from consumer group then kafka rebalance partitons to assigne new consumer group.
- Ex: P0...P6, consumers C0..C1 (3 partition each) and now C3 joins so rebalance partitions to assign to new consumer (2 partitions each)
- There are a fews ways how kafka rebalnce it
- **Classic/Eager:** Stop all consumers, revoke partitions and rebalance to all consumers
- **Cooperative:** Neither stop consumers nor revoke all partitions. It just revoke only partitions which required to be balanced.
  
  ```
  spring:
  kafka:
    consumer:
      properties:
        partition.assignment.strategy: >
          org.apache.kafka.clients.consumer.CooperativeStickyAssignor
  ```
- **Static membership:** Usually kafka provides consumer id as consumer-1343, so when it restarts then id changes and kafka assumes it is new consumer and perform rebalance.
                      To avoid this, provide consumer group name using config `group.instance.id`, so it keeps the partitions to the consumer until session.timeout.ms
  ```
  spring:
  kafka:
    consumer:
      properties:
        group.instance.id: inventory-service-1
  ```
  ### * What are the partition assignment strategies?
  - **Range:** Partitions are sorted and distributed to consumers. Ex: P0..P6 , C0..C2. Each consumer get two partitions. It can be imbalance when multiple topics has different number of partitions.
    ```
    C1:
     orders P0-P3
     payments P0
    
    C2:
     orders P4-P6
     payments P1
    
    C3:
     orders P7-P9
     payments P2
    ```
 - **Round Robin:** Assignes each partition to each consumer in round robin way. Better balance but during rebalance many partitions will be moved.
 - **Sticky Assigner:** Keep balanced assignment and move only required partitions but stop the world problem.
 - **CooperativeStickyAssignment:** Sticky Assignment + incremental rabalance, balanced and move required only without stop the world.

### * How do you scale producer throughput?
- publish batch wise instead of writing each message. linger.ms is to wait before publishing another batch.
```
spring:
  kafka:
    producer:
      batch-size: 65536
      properties:
        linger.ms: 10
```
- compression.type to reduce size of the message
```
spring:
  kafka:
    producer:
      compression-type: snappy
```
- Increase producer buffer size, producer maintain buffer to store batch so increase buffer makes to store more messages in batch.
```
spring:
  kafka:
    producer:
      buffer-memory: 67108864
```
- Increase producer parallelsim, concurrently publish messages and kafkatemple is threadsafe

### * If you increase producer throughput, how do you prevent overwhelming consumers ?
- There are chances that producer is producing many messages and consumers are unable to process them.
- Monitor the situation first then take action accordingly.
- If there are less consumers than partitions then increase partitions, so that concurrency will be increased
- Back pressure the producer
    * provide batch size so that once it is full producer will wait naturally.
    * Use ratelimiter
    * Use internal queue
    * max.block.ms - wait for next buffer availability
- async/parallel processing
- May need to tweak the business logic where it is taking time like DB calls

### * How does Kafka scale storage and retention?
- Each partition is stored as appended log and divided into segments.
- Old segments will be deleted based on retention period or compact them.


  
