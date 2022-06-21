# apache-kafka v3.x

## Kafka Theory
- Pull Model
- Message in a partition gets an id, called Offset. Offset read is possible unlike JMS
- Kafka logs are immutable, so topics can be easily replicated
- Traditional message Q vs kafka
  - Event gets deleted from Traditional Q or topic once consumed. Kafka persists until event expires(Default 1 week)
  - Traditional Q FIFO, offset seek not possible
- Kafka Message Format
  - [Format](kafkaMessageFormat.png)
  - Key can be numeric or string or anything 
  - If event has no key, events distributed across partition in round-robin fashion. If event has key, hash(key)%partitionSize => decides which partition that event goes into
    - Murmur2 algo for Hashing
    partition# = Math.abs(Utils.murmur2(keyBytes)) % (partitionSize)
    - Producers decide which partition data is written into by using Key
  - No key Advt: Even distribution across partition
  - Having key Advt:Ordering of events of similar key in a partition ,as events with same key goes into same partition always
- Kafka serializer/Deserializer
  - Producer can send in any format and kafka only accepts byte format, That's why kafka serializer. Example, 
    IntegerSerializer, StringSerializer
  - `Best Practice`: In a topic life cycle, don't change the format of data in that topic, else would break consumers 
- Consumer Group
  - Consumers in a group read from `Exclusive` partitions of a topic
  - [Multiple CG per Topic](MultipleConsumerGroupsPerTopic.PNG)
  - Consumer has a property called "group.id"
- Consumer Offset
  - Kafka has a mechanism to tell till which offset a consumer of a group has read the partition. It's stored in an 
    internal kafka topic called `__consumer_offsets`
  - How it works 
    - Consumer from a group `periodically` commits offset while consuming data and Kafka writes the offset to the 
      `__consumer_offsets` topic
    - If a consumer dies, another is spawn, it can resume reading from the same offset where previous consumer died.
- Consumer Delivery semantics (When consumer commits offset)
  - `At most once`: Commits offset once message `received`
    - Implication
      - if processing fails, message lost, can not be re-read
    - High throughput and less latency
  - `At least once`(Preferred): Commits offset once message `received + processed`
    - Implication
      - Possibility of reading the message more than once.
      - `Best practice` Make the consumer processing idempotent,i.e. Processing same message again should not impact system
    - Moderate throughput and Moderate latency
    - java sdk by default uses this
  - `Exactly once`
    - Requirement -> Deliver only once and no Data loss
    - Make consumer idempotent(Notice difference vs At least once, there processing is idempotent),i.e. maintain a 
      state at consumer end and filter out the duplicate events.
    - Low throughput and High latency
- Kafka Cluster & broker
  - Each kafka server or node -> Broker. If more than one broker, Kafka is called cluster
  - Broker contains partition of different topic,i.e. n partitions of a topic can be distributed in different 
    brokers. This is horizontal scaling.
  ![img.png](HorizontalScaling.png)
- Kafka Broker discovery(As different broker has data of different topic-partition, How client how know this info and 
  connect to appropriate broker)
  - Kafka client needs to connect to any kafka broker and the rest follow. This "any broker" is `Bootstrap broker`. 
    All brokers eligible to be bootstrap broker
  - [kafka Broker Discovery](KafkaBrokerDiscovery.png)
- Topic replication factor
  - For redundancy/replication. If a broker goes down, then another broker can have to data of partition
  - [Topic Replication](TopicReplication.png)
  - If brokers# < replication factor, Kafka throws error. `Partitions reassignment failed due to replication factor: 45 larger than available brokers: 21` 
  - `Topic Availability`: Let's assume replication factor < brokers#, for N replication factor, topic can withstand N-1 
    broker failures.
    Example, 2 topics , Each of 2 partitions, Replication Factor=2 & brokers#=2, Then if one broker goes down , the 
    2nd broker becomes leader for all partition of both topics
- Partition Leader
  - Now we have replicas of partition, only one broker can be a leader of a particular partition
  - Rules
    - Producer writes to `only` broker that's leader of partition
    - Consumer reads from `by default` broker that's leader of partition. But kafka 2.4+ version, consumer can fetch data from ISR partition
  - SO, each partition has leader and ISR(in-sync-replica) or OSR(out-sync-replica)/if replication ha not happened
- Producer Delivery Semantics(How producer can choose to rcv ACK from broker)
  - ACKS=0, producer won't wait for ACK(Possible data loss)
  - ACKS=1, producer wait for ACK from leader broker(limited data loss), so producer can retry
  - ACKS=-1 or all(Default), producer wait for ACK from all brokers, Ensurers replication happens before ACK(No data 
    loss) 
- Apache Zookeeper
  - ZK is [:face_with_spiral_eyes: centralized, yet distributed](https://medium.com/nakamo-to/whats-the-difference-between-decentralized-and-distributed-1b8de5e7f5a4) service which acts like 
    distributed system coordinator, discovery/naming service
  - Kafka Dependency on ZK
    - Kafka 2.x can't work w/o ZK, even for single broker setup
    - Kafka 3.x can work w/o ZK, using `Kafka Raft(KIP-500)`, i.e. KRaft
    - Kafka 4.x won't have ZK
    - Managing kafka Broker
      - Until 4.x, should not use kafka w/o ZK in Production. KRaft in 3.x not prod-ready yet
    - Managing Kafka Client
      - Since 2.2 all clients/CLI already migrated to leverage brokers instead of ZK, so don't use ZK for managing 
        clients, i.e. for a producer or consumer to connect, it just needs to know any broker + topic name. [Refer](KafkaBrokerDiscovery.png)  
  - [ZK also is distributed, has leader/follower nodes](ZK_Cluster.PNG)
  - Roles of ZK managing Broker & Clients
    - Wrt kafka Broker
      - Broker Leader Election
      - ACL and quotas stored
      - State of brokers - Keeps polling brokers
      - Broker/Topic Registry: Can Find all broker/partition/topic details from zookeeper
    - Wrt Kafka Client
      - < v0.10 used to store consumer offset
      - Consumers register to ZK. So, can find all consumers in cluster
- Why KRaft
  - ZK scaling issue when cluster has >1L partitions
  - Difficult to manage two system from admin point of view
  - Single security model if ZK removed
  - [ZK Centralized vs KRaft Decentralized](KRaft_Architecture.PNG), i.e. one of broker acts as quorum leader

## Kafka Installation
- [Refer PDF for flow](Install+Kafka+Diagram.pdf)
- [Refer for Different platform installation](https://www.conduktor.io/kafka/starting-kafka)
- Windows
  - Kafka not supposed to on native windows, run either on VM or Docker

## Kafka CLI
- Use --bootstrap-server instead of --zookeeper, as all CLI commands upgraded to work w/o ZK
- [Refer](udemy-part1/1-kafka-cli)
- Producing to a non-existing topic, Kafka creates a topic for you with replication factor=1, partition=1
- Kafka consumer by default reads from end of topic, to read from beginning specify `--from-beginning`
- If a consumer reading w/o group, Kafka creates a CG with name `console-consumer-[0-9+]`. But unlike actual groups, 
  if consumer die, this special CG is removed so consumer offset not maintained for this CG
- [How Consumer Group looks](CG_Describe.png)
- Offset seek/Tweaking so that consumer can read differently than ordered reading
  - Done at CG command using `kafka-consumer-groups`, not consumer command
  - `--reset-offsets --to-earliest --execute --topic topic.name` (Will re-read from beginning for each partition of 
    that topic)
  - `--reset-offsets --shift-by 2 --execute --topic topic.name` (Will skip 2 offsets for each partition of that topic)

## Kafka SDK
- Official SDK is java SDK, i.e. apache-kafka-clients
- Examples of the course
  - Producer async send with or w/o keys
  - Producer with callback and exception handler
  - `StickyPartitioner`
    - Ideally when no key for a message, messages should be sent ot partitions in a round-robin fashion, But for 
      optimization purpose, messages are sent batches if sent with less time gap to same partition. This is 
      StickyPartitioner behavior. 
    - How to break this behavior ?? (Just for testing, but sticky is better and performant)
      - Just add a sleep between sending messages, It would send to partition in round-robin
  - Consumer with :infinity: loop polling and no graceful exit
  - [Consumer with :infinity: loop polling and graceful exit](./udemy-part2/kafka-basics/src/main/java/io/conduktor/demos/kafka/ConsumerDemoWithShutdown.java)
  - `Partition re-balancing` with multiple consumers of a CG[1 topic, 3 partitions]
    - Happens when partition added/removed or consumer added/removed of a CG
    - Steps
      - 1>Run one consumer , it's assigned all 3 partitions
      - 2>Start another instance of same consumer, partition balancing happens. Consumer1 log shows that consumer 
        leaves group, partitions revoked, then 2 partitions assigned to consumer, and Consumer2 might get 1 partition.
        Consumer2 consumes starting from leftover offsets.
  - `Partition re-balancing` Strategies (This section proves why re-balancing is an overhead)
    - `parition.assignment.strategy` property
    - 2 types
      - Eager Rebalance (Stop-the-world rebalance)
        - When consumer added, stop all consumers, remove all partition assignments, reassign partitions to all consumers
        - Examples : RangeAssignor(Default), RoundRobin, StickyAssignor
      - Cooperative Rebalance(Incremental rebalance)
        - Reassigns only subset of partitions to consumers, meanwhile other untouched consumers can continue polling
        - Not stop-the-world
        - :face_with_spiral_eyes: Rebalancing can happen in multiple iterations until stable assignment is attained, 
          hence called incremental rebalancing
        - - Examples : CooperativeStickyAssignor
  - How to make a consumer read from same partition by using `Static group membership`
    - :metal: `Static group membership` => Static assignment of consumer to partition
    - Each consumer has a group member id, in case of re-balancing, the consumer leaves the group(Refer above section), as given new group member ID.
    - Instead, we can define static group member id while creating consumer using `group.instance.id` property
    - If a static member consumer is down, it has upto `session.timeout.ms` millisecond to join back and get back same 
      previous partition assignments
      - If timeout expires, partition will be reassigned to another consumer
    - `Best Practice`: Useful when consumer is stateful or maintains cache
  - Consumer offset commit
    - Auto-Commit
      - By default, java sdk uses at-least at periodic interval, iff `enable.auto.commit`= true & `auto.commit.
        interval.ms = 5000`and poll() used, the auto-commits every 5 sec. commitAsync() called behind the scenes.
    - Manual Commit
      - If `enable.auto.commit`= false, then you have to manually call commitAsync() or commitSync()
  - [Advanced consumer examples](./udemy-part2/kafka-basics/src/main/java/io/conduktor/demos/kafka/advanced)

## Kafka Realtime best practices
- Choose right partition# at the start
  - Why : If changed, key-partition assignment would change
  - How to choose
    - Partition helps in parallelism of consumers, implies better throughput
    - Faster /high volume producers, create more topics
    - If more brokers, keep more partition for horizontal scaling
    - Con : More Partition means more election for leader election
    - `Note` :Don't create partition for each user or customer, if you do, those will in millions of number and of 
      no use. Basically you want customer or user data to be ordered , so use key as "user_id" and even 10 partitions will help achieve ordering requirement
  - Guidelines from Kafka
    - With ZK
      - Total partition# <= 2L per cluster
      - Total partition# <= 4k per broker
    - With KRaft
      - Millions of partitions can be supported
- Choose right Replication Factor at the start
  - Why: If changed, more replication , more usage of n/w resources
  - How to choose
    - At least 2, preferred 3, max 4
    - More replication factor, more latency if producers uses acks=all(which is default)
    - More replication factor, more availability
- Topic naming convention
  - 