### 1. High-level Kafka Architecture

At a glance, Kafka is a distributed, partitioned, replicated log system.

Core components:

    Producers  →  Kafka Cluster  →  Consumers
    |
    Topics
    |
    Partitions
    |
    Replicas

Kafka itself runs as a cluster of brokers.

### 2. Kafka Cluster & Brokers
   Broker
   * A broker is a Kafka server
     * Each broker:
       * Stores data (messages)
       * Serves producer writes
       * Serves consumer reads
     * Cluster
       * A Kafka cluster = multiple brokers
       * Each broker has:
          * broker.id
       * Data is distributed across brokers, not stored centrally

#### Why this matters:

* Horizontal scalability
* Fault tolerance
* High throughput
### 3. Topics (Logical Data Streams)

A topic is a logical category or stream of messages.

    Examples:
    
    orders
    payments
    user-events

#### Key points:

* Topics are append-only

* Messages are immutable

Kafka does NOT delete messages after consumption (unlike queues)
### 4. Partitions (The Heart of Kafka)

Each topic is split into partitions.

    Topic: orders
    ├── Partition 0
    ├── Partition 1
    └── Partition 2
What is a partition?

* An ordered, immutable log
* Messages are stored sequentially
* Each message has an offset

Example:

    Partition 0:
    offset 0 → msgA
    offset 1 → msgB
    offset 2 → msgC
#### Why partitions matter
* Parallelism
* Multiple producers write in parallel
* Multiple consumers read in parallel
* Scalability
* Ordering guarantee

    Ordering is guaranteed within a partition, not across partitions

### 5. Producers
   * Role
   * Producers publish messages to topics
 #### How Kafka decides the partition
   * Key present
     * Same key → same partition (hash(key) % partitions)
     * Guarantees order per key
  * No key
     Round-robin or sticky partitioner
#### Producer guarantees
  * At-most-once
  * At-least-once
  * Exactly-once (with idempotent producers + transactions)
### 6. Replication & Fault Tolerance

Each partition is replicated across brokers.

#### Partition 0:
    Leader → Broker 1
    Follower → Broker 2
    Follower → Broker 3
#### Leader
* Handles all reads & writes
* Only leader accepts producer writes

#### Followers
* Replicate data from leader
* Serve as backups
#### ISR (In-Sync Replicas)
* Set of replicas fully caught up with leader
* Only ISR members can become leader

#### If leader fails:

* Kafka elects a new leader from ISR
* No data loss (if acks=all)

### 7. ZooKeeper vs KRaft (Important)
 
#### Old Architecture (ZooKeeper-based)

##### ZooKeeper handled:

* Broker metadata
* Leader election
* Controller election

##### Problems:

* Operational complexity
* Scaling limits
* New Architecture (KRaft mode – Kafka Raft)

##### Kafka now uses KRaft (Raft consensus protocol):

* Kafka manages metadata internally
* No ZooKeeper dependency
* Dedicated controller nodes

##### Benefits:

* Simpler ops
* Faster recovery
* Better scalability
### 8. Consumers
##### Consumer Groups
   * Consumers belong to a consumer group
   * Each partition is consumed by only one consumer per group
   * Topic: orders (3 partitions)

         Group A:
         Consumer 1 → Partition 0
         Consumer 2 → Partition 1
         Consumer 3 → Partition 2
##### Rebalancing Occurs when:

* Consumer joins/leaves
* Partitions increase
* Broker failure

Kafka reassigns partitions automatically

### 9. Offsets & Message Tracking

Kafka tracks consumption using offsets.

    Offset = position in partition
    Stored in internal topic: __consumer_offsets

#### Consumers:

* Pull data (Kafka is pull-based)
* Commit offsets:
   * Automatically
   * Manually (preferred for control)

#### This enables:

* Replay messages
* Time-travel reads
* Exactly-once processing
### 10. Storage Layer (Why Kafka is Fast)
#### Log Segments

##### Partitions are broken into segments:

    partition.log
    ├── segment-00001.log
    ├── segment-00002.log

#### Each segment has:

* Index file
* Time index file
* Zero-Copy Optimization

#### Kafka uses:

* OS page cache
* sendfile() system call

#### Result:

* Very high throughput
* Minimal JVM memory usage
### 11. Retention & Cleanup

#### Kafka retains data based on:

* Time (e.g., 7 days)
* Size (e.g., 100 GB)

#### Cleanup policies:

* Delete
    * Old segments removed
* Compact
  * Keeps latest value per key 
  * Used for changelog / state topics
### 12. End-to-End Data Flow
    Producer
    ↓
    Partition Leader (append log)
    ↓
    Followers replicate
    ↓
    Ack to producer
    ↓
    Consumer pulls data
    ↓
    Consumer commits offset
### 13. Why Kafka Scales So Well
 * Partition-based parallelism
 * Sequential disk writes
 * Pull-based consumers
 * Zero-copy IO 
 * No broker-side state for consumers
### 14. When Kafka is NOT a Good Fit
 * Strict global ordering needed
 * Low-latency (<5ms) messaging
 * Simple task queues (RabbitMQ fits better)