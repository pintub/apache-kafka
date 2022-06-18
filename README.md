# apache-kafka v3.x

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
  - ACKS=all, producer wait for ACK from all brokers, Ensurers replication happens before ACK(No data loss) 
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