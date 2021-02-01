## Kafka

Kafka是一个分布式的基于发布/订阅模式的消息队列(Message Queue)

### Kafka与ZooKeeper

Kafka依赖于ZooKeeper，ZooKeeper给Kafka提供组成集群的功能，存储一些必要的集群信息，帮助kafka的消费者存储消费到的位置信息offset（kafka v0.9之后，offset又存到kafka了，这样可以减小与ZooKeeper的交互频率）。

### Kafka性能为什么这么高？

* 分布式，分区，可以并发读写

* 顺序写磁盘

  Kafka的producer生产数据，要写入到log文件中，写的过程是一直追加到文件末端，为顺序写。顺序写之所以快，是因为省去了大量磁头寻址的时间。

* 零复制技术

### Kafka的消费模式

基于消费者主动拉取的模式

优点：消费者可以自己来控制消费速率(如果是生产者推送模式，则是由生产者控制速率，则有可能出现推送速率过高，消费者消费能力不足，从而产生异常)。

缺点：不管有没有消息，消费者都会对服务器进行长轮询，当没有新消息时，这种轮询就浪费了服务器资源。

#### 对基本的消费者主动拉取模式的优化

针对于以上方案的长轮询缺点，采用服务端主动推送和客户端拉取相结合的方案，即：由服务端主动向客户端推送新消息到来的通知，然后由客户端收到通知后自己主动拉取实际数据。这个方案也有缺点：当客户端宕机，服务端也就通知不到它了。

### Broker

Broker是Kafka集群中单个Kafka服务实例，Kafka集群由多个Broker共同组成集群。

#### 消息存储方式

Broker在物理上把 topic 分成一个或多个 partition（对应 server.properties 中的 num.partitions=3 配置），以轮询topic下的多个partition的方式来均匀存储数据，每个 partition 在物理上对应一个文件夹（该文件夹存储该 patition 的所有消息和索引文件）

#### 存储策略

无论消息是否被消费，kafka 都会保留所有消息。有两种策略可以删除旧数据：
* 基于时间：log.retention.hours=168
* 基于大小：log.retention.bytes=1073741824

需要注意的是，因为 Kafka 读取特定消息的时间复杂度为 O(1)，即与文件大小无关，所以这里删除过期文件与提高 Kafka 性能无关。

### 主题（Topic）与分区（Partition）

消息发送时都被发送到一个 topic，topic的本质就是一个目录，而 topic 是由一些 Partition Logs(分区日志, 也即kafka的消息数据)组成。

每个Partition中的消息都是有序的，生产的消息被不断追加到相应的 Partition Logs，其中的每一个消息都被赋予了一个唯一的 offset 值。

#### 分区的原因

* 方便在集群中扩展，每个 Partition 可以通过调整以适应它所在的机器，而一个 topic
又可以有多个 Partition 组成，因此整个集群就可以适应任意大小的数据了；
* 可以提高并发，因为可以以 Partition 为单位读写了。

#### 对消息数据分区的策略

1. 指定了 patition，则直接使用；
2. 未指定 patition 但指定 key，通过对 key 的 value 进行 hash 出一个 patition；
3. patition 和 key 都未指定，使用轮询选出一个 patition。

### 副本（Replication）

在集群中Topic是有分区的，Partition也有单独的Leader和Follower，只能连Leader生产和消费，Follower仅提供备份作用。

副本是对应于Partition的数据。副本数不是备份数。若设置为1，实际上只有1份数据；若设置为2，则有1份备份数据（Follower存在于该Topic的Leader以外的某一台机器上，该Leader宕机时将发挥作用）。

### 生产者

![生产者](../src/kafka/producer.png)

1. producer 先从 zookeeper 的 "/brokers/.../state"节点找到该 partition 的 leader
2. producer 将消息发送给该 leader
3. leader 将消息写入本地 log
4. followers 从 leader pull 消息，写入本地 log 后向 leader 发送 ACK
5. leader 收到所有 ISR 中的 replication 的 ACK 后，增加 HW（high watermark，最后 commit 
的 offset）并向 producer 发送 ACK

### 消费者

同一个分区下的某个topic是不能被同一个消费者组里的多个成员同时消费的，即一个分区下的某个topic只能被同一个消费者组里的某一个成员消费。

当消费者组的成员数大于分区数时，就是资源浪费；当消费者组的成员数等于分区数时，即是最大消费能力的配置。

#### 分区分配策略

一个consumer group中有多个consumer，一个topic有多个partition，所以必然会涉及到partition的分配问题，即确定哪个partition由哪个consumer来消费。消费者组的成员有增加删除的时候会触发策略重新分配。

* RoundRobin: 按消费者组订阅的多个topic所涉及的全部partition，视为一个整体，轮询分配。

  优点: 比较均匀的分配。

  缺点: 要保证该消费者组里面的所有消费者订阅的topic是一样的（如果不一样，RoundRobin仍然会把partition均匀的分配给各个消费者，与期望的"在同一个组内按设定的规则消费指定的不同的topic"不符）。

* Range: 逐个topic分配。假设有A、B这2个topic，A、B都有3个partition (0,1,2)，有一个2个消费成员的消费者组同时订阅了A、B，按范围分配，3除以2不能整除，partition 0和1分配给第一个消费者，partition 2分配给第2个消费者，这样分配的结果是给第一个消费者分配了4个partition，第二个消费者2个partition。

  优点: 消费者组里面的消费者可以订阅不同的主题。缺点: 分配不均匀。

案例：假设有Topic1和Topic2，分别都有3个对应的Partition（0，1，2），消费者A、B在同一个消费者组，消费者C在单独的消费者组，A订阅了Topic1，B订阅了Topic1和Topic2，C订阅了Topic1。

![生产者](../src/kafka/consume_example.png)

* 若选用了RoundRobin，Group1的分配中，会把Topic1和Topic2总共6个Partition作为整体轮询分配给A和B，这样就会导致A也消费了Topic2，显然与期望不符。
* 若选用了Range，先考虑Topic1的分配，Group2只有1个成员，Partition（0，1，2）都分配给C，在Group1中，Partition（0，1）分配给A，Partition（2）分配给B；再考虑Topic2的分配，由于Group1中只有B订阅了它，所以Topic2的Partition（0，1，2）都分配给了B；这样分配的结果是A订阅了2个Partition，B订阅了4个Partition，分配不均匀。

#### offset的维护

由这3个关键字段作为key：consumer group、topic、partition，value即是相应的offset值。

#### 消费者消费消息时，怎么知道消息在哪个Partition呢？

由上文可知，消费者组按分区分配策略来消费，分配时确定了消费者消费哪些Partition。

当消费者组成员变动时，也会重新分配Partition，消费者从kafka取回该Partition相应的offset接着消费（由上文的3个关键字段确定了唯一的offset）。