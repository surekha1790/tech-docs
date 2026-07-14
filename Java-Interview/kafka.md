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
  
   
