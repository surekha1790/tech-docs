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
              concurrency: 3
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
