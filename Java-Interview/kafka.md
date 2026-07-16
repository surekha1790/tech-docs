### Kafka Architecture
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

### What are atmost-once, atleast-once, exactly-once models
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
              concurrency: 3            **# concurrency means it creates 3 consumers to consume messages. If there are two such instances running then there will be 6 consumers**
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
                  
### Difference between "latest" & "earliest" 
- Earliest consumes messages from starting and latest consumes messages published after consumer joined the group.
- But this config is used only when there is no valid committed offset available to figure what was the last read message.
- Lets say kafka already published 100 messages. When server starts for the first time, it reads from 0 when config is earliest. When config is latest then it will consume from 100.
- Now, the consumer processed till 210 and committed offset. If it restarts then it reads from 210 irrespective of the config as committed offset is available.

### How to handle scenarios where messages processing fails or something wrong after processing before comitting 
- Scenario 1: Message is processed and failed during DB call
- How to handle: DefaultErrorHandle will pause consumer and retry failed message for n times. If it fails after retries then it moves it to dead letter topic.

- Scenario 2: Message is not in correct format and unable to deserialize it.
- How to handle: ErrorHandlingDeserializer pushes the message to dead letter topic

- Scenario 3: Consumer crash/rebalance before offset commit
- How to handle: After restart it automatically redeliver the message. Idempotency should be maintained.

- Scenario 4: Slow consumer where consumere does not consume message within max.poll.interval.ms time, so broker declares it as dead and rebalance it. So message will be republished
- How to handle: Idempotency implementation avoid duplicate updates. Config can be adjusted by increasing max poll time and decreasing max poll records.

### How does Kafka decide which partition a record goes to?
- Each topic splits into multiple partitions and record goes to exactly one record. Partition will be selected by producer.
- **Key present** → `partition = hash(key) % numPartitions`
- **Withot Key** → Kafka follows **Sticky partition**. Earlier it used to follow round robin way to push message in partitons when no key present but it cause low throughtput and less messages in each partition.
                   So, in sticky partition, one partition will be picked up and messages will be pushed into it until batch is full. Producer maintains a buffer for each partition, so when it is full then it switches to                      another partition.
- **Custom** → implement `Partitioner` for domain-specific routing (e.g. route by region).

### Why does partitioning matter, and how do you choose a key?
- Partition is to maintain parallelism and ordering.
- For example, same order id events goes to same partition so that they follow the order.

### How many partitions should a topic have?
- There is no particular number for it.
- Should be decided based on number of consumers in consumer group to achieve parallelism.
- More consumers than partitions make the consumer idle
- More partitions than consumer does not achieve great parallelism as multiple partitions are assigned to one consumer.
- **Rule of thumb:** `partitions = max(target_throughput / per_partition_throughput, desired_consumer_parallelism)`

### Can you change the partition count later? What breaks?
- It is possible to increase partition count later but it breaks few things.
- Old messages stays in the old partitions and new messages are distributed based on new partitions count
- So, there is possibility of missing order for the messages such as e-commerce sequence like order created, payment success, order confirmed, shipped, delivered etc.
- For such scenarios it causes issues as order will be disturbed due to new partitions and there is a possibility that order confirmed is process first than payment success.
- To avoid these, there are couple of ways
- **Create over partitions** : Create more partitions than requirement so there will be room to avoid scaling in future
- **New Partition** : Create new v2 version topic and publish new messages to new topic and consume from both topics, so old messages are ordered and consumed until it is finished.
- **Versioning** : Here we not only create a new topic with more partitions, we store versioned topic name in db along with the event(lets say order). So while publishing, will fetch the topic version and publish on to                       the same topic always, so we do not mess with the ordering even increase partitions

### How Kafka commits the offset ?
- When auto commit is on then kafka commit the offset in specific time intervals set by kafka irrespective of the message processing result.
- When auto commit is off then kafka listner takes care of the commit and it commits the offset after successfull processing of a batch before polling. If unsuccessfull process defaulterror handler will retry.
- It can be controlled more with config `spring.kafka.listener.ack-mode:`.
- **RECORD:** commit offset per record instead of batch.
- **MANUAL/MANUAL_IMMEDIATE:** Need to inject Acknowledgment in the listner and call ack.ack() manually to commit offset.

### How does Kafka achieve durability and fault tolerance
- Kafka maintains append only log on disk for each partition where it sequenctially append messages at the end
- Kafka also has replicas. One broker will be leader and remaining are followers where insyc replicas will be also be selected
- replicas store all messaged which are flowing through leader and in sync replica is the one which is elgible to be leader when leader crashes
- Producers can use acks=all to ensure messages are acknowledged only after replication to in-sync replicas
- kafka also has retention policy, based on which messages will be stored.

### What is the ISR, and how does it relate to `acks`?
- ISR (insync replica) which is same as leader which stores all messages flowing through leader
- To be eligible for becoming leader, it should be ISR
- acks config is to keep the brokers in sync.
- ack = 0: fire and forget, do not wait for ack
- ack = 1 : acks to only leader
- ack = all : leader waits until all replicas are in sync before ack

### Why the classic "RF=3, min.insync.replicas=2, acks=all"?
- min.insync.replicas=2 means there should be atleast replicas should be in sync with leader when acks=all.
- Anyone replica does not acknowledge the write then it is considered as failure
- If one broker is down for the partition then we still have data in another replica
- If both brokers are down then partition does not accept the data and throws NotEnoughReplicas

### What happens when a broker fails?
- When broker fails, Controller in Kraft mode will elect another leader from in sync replicas
- When broker returns it joins as follower, catches up and re enter ISR

### What is unclean leader election
- When all ISRs are lost and broker will be elected based on config unclean.leader.election.enable
- **True:**  even ISR is not available, kafka choose out of sync broker to maintain availability
- **False:** It waits until ISR is available. No data loss but no availability.  
  
