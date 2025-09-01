# TOC
- [TOC](#toc)
- [Kafka 分区的目的？](#kafka-分区的目的)
- [你知道 Kafka 是如何做到消息的有序性？](#你知道-kafka-是如何做到消息的有序性)
- [Kafka Producer 的执行过程？](#kafka-producer-的执行过程)
- [讲一下你使用 Kafka Consumer 消费消息时的线程模型，为何如此设计？](#讲一下你使用-kafka-consumer-消费消息时的线程模型为何如此设计)
- [请谈一谈 Kafka 数据一致性原理](#请谈一谈-kafka-数据一致性原理)
- [ISR、OSR、AR 是什么？](#isrosrar-是什么)
- [LEO、HW、LSO、LW等分别代表什么](#leohwlsolw等分别代表什么)
- [数据传输的事务有几种？](#数据传输的事务有几种)
- [Kafka 消费者是否可以消费指定分区消息？](#kafka-消费者是否可以消费指定分区消息)
- [Kafka消息是采用Pull模式，还是Push模式？](#kafka消息是采用pull模式还是push模式)
- [Kafka 高效文件存储设计特点](#kafka-高效文件存储设计特点)
- [Kafka创建Topic时如何将分区放置到不同的Broker中](#kafka创建topic时如何将分区放置到不同的broker中)
- [谈一谈 Kafka 的再均衡](#谈一谈-kafka-的再均衡)
- [Kafka 是如何实现高吞吐率的？](#kafka-是如何实现高吞吐率的)
- [Kafka 缺点？](#kafka-缺点)
- [Kafka 新旧消费者的区别](#kafka-新旧消费者的区别)
- [Kafka 分区数可以增加或减少吗？为什么？](#kafka-分区数可以增加或减少吗为什么)
- [Consumer和Partition的数量](#consumer和partition的数量)
- [ISR 与 OSR 转换、ISR 集合中的副本才允许选举为leader](#isr-与-osr-转换isr-集合中的副本才允许选举为leader)
- [什么是 HW 和 LEO](#什么是-hw-和-leo)
- [写入Partition的方式](#写入partition的方式)
- [kafka消息保证机制](#kafka消息保证机制)
- [Kafka数据为什么可能出现重复消费](#kafka数据为什么可能出现重复消费)

# Kafka 分区的目的？
分区对于 Kafka 集群的好处是：实现负载均衡。分区对于消费者来说，可以提高并发度，提高效率。

# 你知道 Kafka 是如何做到消息的有序性？
kafka 中的每个 partition 中的消息在写入时都是有序的，而且单独一个 partition 只能由一个消费者去消费，可以在里面保证消息的顺序性。但是分区之间的消息是不保证有序的。

# Kafka Producer 的执行过程？
1. Producer生产消息
2. 从Zookeeper找到Partition的Leader
3. 推送消息
4. 通过ISR列表通知给Follower
5. Follower从Leader拉取消息，并发送ack
6. Leader收到所有副本的ack，更新Offset，并向Producer发送ack，表示消息写入成功

# 讲一下你使用 Kafka Consumer 消费消息时的线程模型，为何如此设计？
Thread-Per-Consumer Model，这种多线程模型是利用Kafka的topic分多个partition的机制来实现并行：

每个线程都有自己的consumer实例，负责消费若干个partition。各个线程之间是完全独立的，不涉及任何线程同步和通信，所以实现起来非常简单。

# 请谈一谈 Kafka 数据一致性原理
- 一致性就是说不论是老的 Leader 还是新选举的 Leader，Consumer 都能读到一样的数据。
- 假设分区的副本为3，其中副本0是 Leader，副本1和副本2是 follower，并且在 ISR 列表里面。虽然副本0已经写入了 Message4，但是 Consumer 只能读取到 Message2。因为所有的 ISR 都同步了 Message2，只有 High Water Mark 以上的消息才支持 Consumer 读取，而 High Water Mark 取决于 ISR 列表里面偏移量最小的分区，对应于上图的副本2，这个很类似于木桶原理。
这样做的原因是还没有被足够多副本复制的消息被认为是"不安全"的，如果 Leader 发生崩溃，另一个副本成为新 Leader，那么这些消息很可能丢失了。如果我们允许消费者读取这些消息，可能就会破坏一致性。试想，一个消费者从当前 Leader（副本0） 读取并处理了 Message4，这个时候 Leader 挂掉了，选举了副本1为新的 Leader，这时候另一个消费者再去从新的 Leader 读取消息，发现这个消息其实并不存在，这就导致了数据不一致性问题。
- 当然，引入了 High Water Mark 机制，会导致 Broker 间的消息复制因为某些原因变慢，那么消息到达消费者的时间也会随之变长（因为我们会先等待消息复制完毕）。延迟时间可以通过参数 replica.lag.time.max.ms 参数配置，它指定了副本在复制消息时可被允许的最大延迟时间。

# ISR、OSR、AR 是什么？
- ISR：In-Sync Replicas，所有与leader副本保持一定程度同步的副本（包括leader副本在内）组成ISR（In-Sync Replicas）
- OSR：Out-of-Sync Replicas，与leader副本同步滞后过多的副本（不包括leader副本）组成OSR（Out-of-Sync Replicas）
- AR：Assigned Replicas 所有副本，AR = ISR + OSR
  - 消息会先发送到leader副本，然后follower副本才能从leader副本中拉取消息进行同步，同步期间内follower副本相对于leader副本而言会有一定程度的滞后。
  - 前面所说的"一定程度的同步"是指可忍受的滞后范围，这个范围可以通过参数进行配置。
  - 在正常情况下，所有的 follower 副本都应该与 leader 副本保持一定程度的同步，即 AR=ISR，OSR集合为空。

# LEO、HW、LSO、LW等分别代表什么
- LEO：是 LogEndOffset 的简称，代表当前日志文件中下一条
- HW：水位或水印（watermark）一词，也可称为高水位(high watermark)，通常被用在流式处理领域（比如Apache Flink、Apache Spark等），以表征元素或事件在基于时间层面上的进度。在Kafka中，水位的概念反而与时间无关，而是与位置信息相关。严格来说，它表示的就是位置信息，即位移（offset）。取 partition 对应的 ISR中 最小的 LEO 作为 HW，consumer 最多只能消费到 HW 所在的位置上一条信息。
- LSO：是 LastStableOffset 的简称，对未完成的事务而言，LSO 的值等于事务中第一条消息的位置(firstUnstableOffset)，对已完成的事务而言，它的值同 HW 相同
- LW：Low Watermark 低水位, 代表 AR 集合中最小的 logStartOffset 值。

# 数据传输的事务有几种？
数据传输的事务定义通常有以下三种级别：
1. 最多一次: 消息不会被重复发送，最多被传输一次，但也有可能一次不传输
2. 最少一次: 消息不会被漏发送，最少被传输一次，但也有可能被重复传输.
3. 精确的一次（Exactly once）: 不会漏传输也不会重复传输,每个消息都传输被

# Kafka 消费者是否可以消费指定分区消息？
- Kafa consumer消费消息时，向broker发出fetch请求去消费特定分区的消息。
- consumer指定消息在日志中的偏移量（offset），就可以消费从这个位置开始的消息。
- customer拥有了offset的控制权，可以向后回滚去重新消费之前的消息，这是很有意义的。

# Kafka消息是采用Pull模式，还是Push模式？
- Kafka最初考虑的问题是，customer应该从brokes拉取消息还是brokers将消息推送到consumer，也就是pull还push。在这方面，Kafka遵循了一种大部分消息系统共同的传统的设计：producer将消息推送到broker，consumer从broker拉取消息。
- 一些消息系统比如Scribe和Apache Flume采用了push模式，将消息推送到下游的consumer。这样做有好处也有坏处：由broker决定消息推送的速率，对于不同消费速率的consumer就不太好处理了。消息系统都致力于让consumer以最大的速率最快速的消费消息，但不幸的是，push模式下，当broker推送的速率远大于consumer消费的速率时，consumer恐怕就要崩溃了。最终Kafka还是选取了传统的pull模式。
- Pull模式的另外一个好处是consumer可以自主决定是否批量的从broker拉取数据。Push模式必须在不知道下游consumer消费能力和消费策略的情况下决定是立即推送每条消息还是缓存之后批量推送。如果为了避免consumer崩溃而采用较低的推送速率，将可能导致一次只推送较少的消息而造成浪费。Pull模式下，consumer就可以根据自己的消费能力去决定这些策略。
- Pull有个缺点是，如果broker没有可供消费的消息，将导致consumer不断在循环中轮询，直到新消息到t达。为了避免这点，Kafka有个参数可以让consumer阻塞知道新消息到达(当然也可以阻塞知道消息的数量达到某个特定的量这样就可以批量发

# Kafka 高效文件存储设计特点
- Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。
- 通过索引信息可以快速定位message和确定response的最大大小。
- 通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作。
- 通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小

# Kafka创建Topic时如何将分区放置到不同的Broker中
- 副本因子不能大于 Broker 的个数；
- 第一个分区（编号为0）的第一个副本放置位置是随机从 brokerList 选择的；
- 其他分区的第一个副本放置位置相对于第0个分区依次往后移。也就是如果我们有5个 Broker，5个分区，假设第一个分区放在第四个 Broker 上，那么第二个分区将会放在第五个 Broker 上；第三个分区将会放在第一个 Broker 上；第四个分区将会放在第二个 Broker 上，依次类推；
- 剩余的副本相对于第一个副本放置位置其实是由 nextReplicaShift 决定的，而这个数也是随机产生的

# 谈一谈 Kafka 的再均衡
- 在Kafka中，当有新消费者加入或者订阅的topic数发生变化时，会触发Rebalance(再均衡：在同一个消费者组当中，分区的所有权从一个消费者转移到另外一个消费者)机制，Rebalance顾名思义就是重新均衡消费者消费。Rebalance的过程如下：
- 第一步：所有成员都向coordinator发送请求，请求入组。一旦所有成员都发送了请求，coordinator会从中选择一个consumer担任leader的角色，并把组成员信息以及订阅信息发给leader。
- 第二步：leader开始分配消费方案，指明具体哪个consumer负责消费哪些topic的哪些partition。一旦完成分配，leader会将这个方案发给coordinator。coordinator接收到分配方案之后会把方案发给各个consumer，这样组内的所有成员就都知道自己应该消费哪些分区了。
- 所以对于Rebalance来说，Coordinator起着至关重要的作用

# Kafka 是如何实现高吞吐率的？
Kafka是分布式消息系统，需要处理海量的消息，Kafka的设计是把所有的消息都写入速度低容量大的硬盘，以此来换取更强的存储能力，但实际上，使用硬盘并没有带来过多的性能损失。kafka主要使用了以下几个方式实现了超高的吞吐率：

- 顺序读写
- 零拷贝
- 文件分段
- 批量发送
- 数据压缩

# Kafka 缺点？
- 由于是批量发送，数据并非真正的实时；
- 对于mqtt协议不支持；
- 不支持物联网传感数据直接接入；
- 仅支持统一分区内消息有序，无法实现全局消息有序；
- 监控不完善，需要安装插件；
- 依赖zookeeper进行元数据管理；

# Kafka 新旧消费者的区别
- 旧的 Kafka 消费者 API 主要包括：SimpleConsumer（简单消费者） 和 ZookeeperConsumerConnectir（高级消费者）。SimpleConsumer 名字看起来是简单消费者，但是其实用起来很不简单，可以使用它从特定的分区和偏移量开始读取消息。高级消费者和现在新的消费者有点像，有消费者群组，有分区再均衡，不过它使用 ZK 来管理消费者群组，并不具备偏移量和再均衡的可操控性。
- 现在的消费者同时支持以上两种行为，所以为啥还用旧消费者 API 呢？

# Kafka 分区数可以增加或减少吗？为什么？
- 我们可以使用 bin/kafka-topics.sh 命令对 Kafka 增加 Kafka 的分区数据，但是 Kafka 不支持减少分区数。
- Kafka 分区数据不支持减少是由很多原因的，比如减少的分区其数据放到哪里去？是删除，还是保留？删除的话，那么这些没消费的消息不就丢了。如果保留这些消息如何放到其他分区里面？追加到其他分区后面的话那么就破坏了 Kafka 单个分区的有序性。如果要保证删除分区数据插入到其他分区保证有序性，那么实现起来逻辑就会非常复杂。

# Consumer和Partition的数量
1. 如果consumer比partition多，会存在浪费，因为一个partition不允许并发处理，所以consumer数量不建议大于partition数量。
2. 如果consumer比partition少，一个consumer会对应多个partitions，这里要合理分配consumer数和partition数，否则会导致partition里面的数据被取的不均匀。最好partition数是consumer数的整数倍。所以partition数很重要。
3. 如果consumer从多个partition中读到数据，不保证数据间的顺序，kafka只保证一个partition上有序，多个partition的顺序则不一定。
4. 增减consumer、broker、partition会导致rebalance，所以rebalance之后consumer对应额的partition会变化

# ISR 与 OSR 转换、ISR 集合中的副本才允许选举为leader
- leader副本负责维护和跟踪ISR集合中所有follower副本的滞后状态，当follower副本落后太多或失效时，leader副本会把它从ISR集合中剔除。
- 如果OSR集合中有follower副本"追上"了leader副本，那么leader副本会把它从OSR集合转移至ISR集合。
- 默认情况下，当leader副本发生故障时，只有在ISR集合中的副本才有资格被选举为新的leader，而在OSR集合中的副本则没有任何机会（不过这个原则也可以通过修改相应的参数配置来改变）。

# 什么是 HW 和 LEO
- HW是High Watermark的缩写，俗称高水位，它标识了一个特定的消息偏移量（offset），消费者只能拉取到这个offset之前的消息。

# 写入Partition的方式
一个 Topic 有多个 Partition，那么，向一个 Topic 中发送消息的时候，具体是写入哪个 Partition 呢？有3种写入方式。

1. 使用 Partition Key 写入特定 Partition
- Producer 发送消息的时候，可以指定一个 Partition Key，这样就可以写入特定 Partition 了。
- Partition Key 可以使用任意值，例如设备ID、User ID。
- Partition Key 会传递给一个 Hash 函数，由计算结果决定写入哪个 Partition。
- 所以，有相同 Partition Key 的消息，会被放到相同的 Partition。
- 例如使用 User ID 作为 Partition Key，那么此 ID 的消息就都在同一个 Partition，这样可以保证此类消息的有序性。
- 这种方式需要注意 Partition 热点问题。
- 例如使用 User ID 作为 Partition Key，如果某一个 User 产生的消息特别多，是一个头部活跃用户，那么此用户的消息都进入同一个 Partition 就会产生热点问题，导致某个 Partition 极其繁忙。

2. 由 kafka 决定
- 如果没有使用 Partition Key，Kafka 就会使用轮询的方式来决定写入哪个 Partition。
- 这样，消息会均衡的写入各个 Partition。
- 但这样无法确保消息的有序性。

3. 自定义规则
- Kafka 支持自定义规则，一个 Producer 可以使用自己的分区指定规则。

# kafka消息保证机制

1. 至少一次。即一条消息至少被消费一次，消息不可能丢失，但是可能会被重复消费。
- 消费者读取消息，先处理消息，在保存消费进度。消费者拉取到消息，先消费消息，然后在保存偏移量，当消费者消费消息后还没来得及保存偏移量，则会造成消息被重复消费。如下图所示：

2. 至多一次。即一条消息最多可以被消费一次，消息不可能被重复消费，但是消息有可能丢失。
- 消费者读取消息，先保存消费进度，在处理消息。消费者拉取到消息，先保存了偏移量，当保存了偏移量后还没消费完消息，消费者挂了，则会造成未消费的消息丢失。

3. 正好一次。即一条消息正好被消费一次，消息不可能丢失也不可能被重复消费。
- 正好消费一次的办法可以通过将消费者的消费进度和消息处理结果保存在一起。只要能保证两个操作是一个原子操作，就能达到正好消费一次的目的。通常可以将两个操作保存在一起，比如 HDFS 中。

# Kafka数据为什么可能出现重复消费

- 重复数据一般是在消费者重启后发生（一般是消费系统宕机、rebalance的时候或者消费者速度很慢，导致一个session周期内未完成消费会出现）。
- 消费者不是说消费完一条数据就立马提交 offset的，而是定时定期提交一次 offset。
- 消费者如果再准备提交 offset，但是还没提交 offset的时候，消费者进程重启了，那么此时已经消费过的消息的 offset并没有提交，kafka也就不知道你已经消费了，就会导致重复消费。

> 解决方案：
- 生产者（ack=all 代表至少成功发送一次) 。
- offset手动提交，业务逻辑成功处理后，提交offset。
- 主键或者唯一索引的方式落库，避免重复数据。
- 代码逻辑判断，如果存在虽然消费但是不写入。
