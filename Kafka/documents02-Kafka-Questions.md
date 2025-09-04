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
  - [消费者故障与再均衡（Consumer Failure and Rebalancing）](#消费者故障与再均衡consumer-failure-and-rebalancing)
  - [偏移量提交失败（Offset Commit Failure）](#偏移量提交失败offset-commit-failure)
  - [生产者重试（Producer Retries）](#生产者重试producer-retries)
  - [消费处理时间过长（Long Processing Time）](#消费处理时间过长long-processing-time)
  - [解决方案](#解决方案)
  - [Kafka事务（Exactly-Once Semantics）](#kafka事务exactly-once-semantics)
- [配置Partition数量多少比较好](#配置partition数量多少比较好)
  - [越多的partition可以提供更高的吞吐量](#越多的partition可以提供更高的吞吐量)
  - [越多的分区需要打开更多的文件句柄](#越多的分区需要打开更多的文件句柄)
  - [越多的partition意味着需要更多的内存](#越多的partition意味着需要更多的内存)
  - [越多的partition会导致更长时间的恢复期](#越多的partition会导致更长时间的恢复期)
  - [总结](#总结)
- [Kafka Rebalance 会有什么样的问题？如何降低Rebalance的负面影响？](#kafka-rebalance-会有什么样的问题如何降低rebalance的负面影响)
- [](#)

# Kafka 分区的目的？
分区对于 Kafka 集群的好处是: 实现负载均衡。分区对于消费者来说，可以提高并发度，提高效率。

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
Thread-Per-Consumer Model，这种多线程模型是利用Kafka的topic分多个partition的机制来实现并行: 

每个线程都有自己的consumer实例，负责消费若干个partition。各个线程之间是完全独立的，不涉及任何线程同步和通信，所以实现起来非常简单。

# 请谈一谈 Kafka 数据一致性原理
- 一致性就是说不论是老的 Leader 还是新选举的 Leader，Consumer 都能读到一样的数据。
- 假设分区的副本为3，其中副本0是 Leader，副本1和副本2是 follower，并且在 ISR 列表里面。虽然副本0已经写入了 Message4，但是 Consumer 只能读取到 Message2。因为所有的 ISR 都同步了 Message2，只有 High Water Mark 以上的消息才支持 Consumer 读取，而 High Water Mark 取决于 ISR 列表里面偏移量最小的分区，对应于上图的副本2，这个很类似于木桶原理。
这样做的原因是还没有被足够多副本复制的消息被认为是"不安全"的，如果 Leader 发生崩溃，另一个副本成为新 Leader，那么这些消息很可能丢失了。如果我们允许消费者读取这些消息，可能就会破坏一致性。试想，一个消费者从当前 Leader（副本0） 读取并处理了 Message4，这个时候 Leader 挂掉了，选举了副本1为新的 Leader，这时候另一个消费者再去从新的 Leader 读取消息，发现这个消息其实并不存在，这就导致了数据不一致性问题。
- 当然，引入了 High Water Mark 机制，会导致 Broker 间的消息复制因为某些原因变慢，那么消息到达消费者的时间也会随之变长（因为我们会先等待消息复制完毕）。延迟时间可以通过参数 replica.lag.time.max.ms 参数配置，它指定了副本在复制消息时可被允许的最大延迟时间。

# ISR、OSR、AR 是什么？
- ISR: In-Sync Replicas，所有与leader副本保持一定程度同步的副本（包括leader副本在内）组成ISR（In-Sync Replicas）
- OSR: Out-of-Sync Replicas，与leader副本同步滞后过多的副本（不包括leader副本）组成OSR（Out-of-Sync Replicas）
- AR: Assigned Replicas 所有副本，AR = ISR + OSR
  - 消息会先发送到leader副本，然后follower副本才能从leader副本中拉取消息进行同步，同步期间内follower副本相对于leader副本而言会有一定程度的滞后。
  - 前面所说的"一定程度的同步"是指可忍受的滞后范围，这个范围可以通过参数进行配置。
  - 在正常情况下，所有的 follower 副本都应该与 leader 副本保持一定程度的同步，即 AR=ISR，OSR集合为空。

# LEO、HW、LSO、LW等分别代表什么
- LEO: 是 LogEndOffset 的简称，代表当前日志文件中下一条
- HW: 水位或水印（watermark）一词，也可称为高水位(high watermark)，通常被用在流式处理领域（比如Apache Flink、Apache Spark等），以表征元素或事件在基于时间层面上的进度。在Kafka中，水位的概念反而与时间无关，而是与位置信息相关。严格来说，它表示的就是位置信息，即位移（offset）。取 partition 对应的 ISR中 最小的 LEO 作为 HW，consumer 最多只能消费到 HW 所在的位置上一条信息。
- LSO: 是 LastStableOffset 的简称，对未完成的事务而言，LSO 的值等于事务中第一条消息的位置(firstUnstableOffset)，对已完成的事务而言，它的值同 HW 相同
- LW: Low Watermark 低水位, 代表 AR 集合中最小的 logStartOffset 值。

# 数据传输的事务有几种？
数据传输的事务定义通常有以下三种级别: 
1. 最多一次: 消息不会被重复发送，最多被传输一次，但也有可能一次不传输
2. 最少一次: 消息不会被漏发送，最少被传输一次，但也有可能被重复传输.
3. 精确的一次（Exactly once）: 不会漏传输也不会重复传输,每个消息都传输被

# Kafka 消费者是否可以消费指定分区消息？
- Kafa consumer消费消息时，向broker发出fetch请求去消费特定分区的消息。
- consumer指定消息在日志中的偏移量（offset），就可以消费从这个位置开始的消息。
- customer拥有了offset的控制权，可以向后回滚去重新消费之前的消息，这是很有意义的。

# Kafka消息是采用Pull模式，还是Push模式？
- Kafka最初考虑的问题是，customer应该从brokes拉取消息还是brokers将消息推送到consumer，也就是pull还push。在这方面，Kafka遵循了一种大部分消息系统共同的传统的设计: producer将消息推送到broker，consumer从broker拉取消息。
- 一些消息系统比如Scribe和Apache Flume采用了push模式，将消息推送到下游的consumer。这样做有好处也有坏处: 由broker决定消息推送的速率，对于不同消费速率的consumer就不太好处理了。消息系统都致力于让consumer以最大的速率最快速的消费消息，但不幸的是，push模式下，当broker推送的速率远大于consumer消费的速率时，consumer恐怕就要崩溃了。最终Kafka还是选取了传统的pull模式。
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
- 在Kafka中，当有新消费者加入或者订阅的topic数发生变化时，会触发Rebalance(再均衡: 在同一个消费者组当中，分区的所有权从一个消费者转移到另外一个消费者)机制，Rebalance顾名思义就是重新均衡消费者消费。Rebalance的过程如下: 
- 第一步: 所有成员都向coordinator发送请求，请求入组。一旦所有成员都发送了请求，coordinator会从中选择一个consumer担任leader的角色，并把组成员信息以及订阅信息发给leader。
- 第二步: leader开始分配消费方案，指明具体哪个consumer负责消费哪些topic的哪些partition。一旦完成分配，leader会将这个方案发给coordinator。coordinator接收到分配方案之后会把方案发给各个consumer，这样组内的所有成员就都知道自己应该消费哪些分区了。
- 所以对于Rebalance来说，Coordinator起着至关重要的作用

# Kafka 是如何实现高吞吐率的？
Kafka是分布式消息系统，需要处理海量的消息，Kafka的设计是把所有的消息都写入速度低容量大的硬盘，以此来换取更强的存储能力，但实际上，使用硬盘并没有带来过多的性能损失。kafka主要使用了以下几个方式实现了超高的吞吐率: 

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
- 旧的 Kafka 消费者 API 主要包括: SimpleConsumer（简单消费者） 和 ZookeeperConsumerConnectir（高级消费者）。SimpleConsumer 名字看起来是简单消费者，但是其实用起来很不简单，可以使用它从特定的分区和偏移量开始读取消息。高级消费者和现在新的消费者有点像，有消费者群组，有分区再均衡，不过它使用 ZK 来管理消费者群组，并不具备偏移量和再均衡的可操控性。
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
- 消费者读取消息，先处理消息，在保存消费进度。消费者拉取到消息，先消费消息，然后在保存偏移量，当消费者消费消息后还没来得及保存偏移量，则会造成消息被重复消费。如下图所示: 

2. 至多一次。即一条消息最多可以被消费一次，消息不可能被重复消费，但是消息有可能丢失。
- 消费者读取消息，先保存消费进度，在处理消息。消费者拉取到消息，先保存了偏移量，当保存了偏移量后还没消费完消息，消费者挂了，则会造成未消费的消息丢失。

3. 正好一次。即一条消息正好被消费一次，消息不可能丢失也不可能被重复消费。
- 正好消费一次的办法可以通过将消费者的消费进度和消息处理结果保存在一起。只要能保证两个操作是一个原子操作，就能达到正好消费一次的目的。通常可以将两个操作保存在一起，比如 HDFS 中。

# Kafka数据为什么可能出现重复消费
Kafka 数据出现重复消费的主要原因是: 消费者处理消息后未能及时提交offset，导致在发生消费者宕机、 rebalance（再均衡）等情况后，消费者从旧的offset 位置重新拉取数据。
## 消费者故障与再均衡（Consumer Failure and Rebalancing）
- 这是最常见的原因之一。当消费者组（Consumer Group）中的某个消费者实例崩溃、长时间无响应或有新的消费者加入时，会触发消费者再均衡（Rebalancing）。在再均衡期间，分区（Partition）的所有权会在消费者之间重新分配。
- 如果在消费者处理完一条消息但尚未提交其偏移量（Offset）时发生再均衡，那么接管该分区的新消费者将从上一次提交的偏移量处开始消费。这就导致了已经被前一个消费者处理过的消息被重复消费。

## 偏移量提交失败（Offset Commit Failure）
- 消费者在处理完消息后，需要向Kafka集群提交当前消费的偏移量，以标记消费进度。如果消费者成功处理了消息，但在提交偏移量时发生网络故障或其他瞬时错误，导致提交失败，那么在下一次拉取消息时，它仍会从上一次成功提交的偏移量处开始，从而造成数据重复。

## 生产者重试（Producer Retries）
- 在生产者端，如果发送消息后没有收到Kafka Broker的确认响应（ACK），例如由于网络波动，生产者为了保证消息不丢失，会进行重试。如果第一次发送的消息实际上已经成功写入，但ACK丢失，那么生产者的重试将导致同一条消息在分区中出现多次。

## 消费处理时间过长（Long Processing Time）
- 如果消费者处理一条消息所需的时间超过了其会话超时时间（session.timeout.ms）或两次拉取（poll）之间的最大间隔（max.poll.interval.ms），协调器（Coordinator）会认为该消费者已经"死亡"，并触发再均衡。这同样会导致消息被重新分配给另一个消费者，引发重复消费。

## 解决方案
- 实现幂等消费者
在消费者端实现幂等性是解决重复消费问题的关键。即使消息被重复投递，幂等的消费逻辑也能确保最终结果的正确性。以下是几种常见的实现方式: 

利用唯一业务标识符:  在消息中包含一个唯一的业务标识符（如订单ID、用户ID等）。消费者在处理消息前，先根据这个唯一标识符查询外部存储系统（如关系型数据库、Redis等）。如果记录已存在，则说明该消息已被处理过，直接跳过即可。

数据库唯一约束:  在将处理结果写入数据库时，可以利用数据库的唯一键（Primary Key）或唯一索引（Unique Index）约束。当重复处理消息并尝试插入相同的数据时，数据库会抛出异常，消费者可以捕获该异常并忽略它。例如，使用INSERT IGNORE或INSERT ... ON CONFLICT DO NOTHING等SQL语句。

事务性写入:  将消息的处理和偏移量的提交放在一个原子事务中。这通常需要将偏移量存储在支持事务的外部系统中（如关系型数据库）。消费者在处理消息和更新业务数据的同时，也在同一个数据库事务中更新偏移量。这样可以保证业务操作和偏移量更新的原子性。

- 精细化偏移量管理（Fine-Grained Offset Management）

默认情况下，Kafka消费者会自动提交偏移量（enable.auto.commit=true）。为了更精确地控制消费进度，建议关闭自动提交，改为手动提交偏移量。

同步提交（commitSync）:  在处理完一批消息后，调用commitSync方法同步提交偏移量。该方法会阻塞，直到偏移量提交成功或失败。这种方式虽然可靠，但会影响吞吐量。

异步提交（commitAsync）:  调用commitAsync方法进行异步提交，不会阻塞当前线程。为了处理提交失败的情况，通常会提供一个回调函数来记录错误。

通过手动提交，可以确保只有在消息被成功处理后才更新偏移量，从而降低重复消费的概率。

## Kafka事务（Exactly-Once Semantics）

对于需要最高级别保证的场景，可以使用Kafka 0.11.0版本之后引入的事务功能，实现端到端的"精确一次（Exactly-Once）"语义。这涉及到生产者、消费者和Broker的协同工作，确保在一个事务内，消息的生产、消费和偏移量的提交是原子性的。这虽然提供了最强的保证，但实现起来也相对复杂。

# 配置Partition数量多少比较好
Kafka的单个Partition效率非常高，但是Kafka的Partition设计是非常碎片化的，如果Partition文件过多，很容易严重影响Kafka的整体性能。

Partition的数量最好根据业务灵活调整，Partition数量设置的多一些，可以一定程度上增加Topic的吞吐量。但是过多的Partition数量会带来Partition索引的压力。因此需要根据业务情况来调整。

## 越多的partition可以提供更高的吞吐量

「单个partition是kafka并行操作的最小单元」。每个partition可以独立接收推送的消息以及被consumer消费，相当于topic的一个子通道，partition和topic的关系就像高速公路的车道和高速公路的关系一样，起始点和终点相同，每个车道都可以独立实现运输，不同的是kafka中不存在车辆变道的说法，入口时选择的车道需要从一而终。

kafka的吞吐量显而易见，在资源足够的情况下，partition越多速度越快。

这里提到的资源充足解释一下，假设我现在一个partition的最大传输速度为p，目前kafka集群共有三个broker，每个broker的资源足够支撑三个partition最大速度传输，那我的集群最大传输速度为33p=9p。

假设在不增加资源的情况下将partition增加到18个，每个partition只能以p/2的速度传输数据，因此传输速度上限还是9p，并不能再提升，因此吞吐量的设计需要考虑broker的资源上限。

kafka跟其他集群一样，可以横向扩展，再增加三个相同资源的broker，那传输速度即可达到18p。

## 越多的分区需要打开更多的文件句柄
在kafka的broker中，每个分区都会对照着文件系统的一个目录。

> 在kafka的数据日志文件目录中，每个日志数据段都会分配两个文件，一个索引文件和一个数据文件。因此，随着partition的增多，需要的文件句柄数急剧增加，必要时需要调整操作系统允许打开的文件句柄数。

## 越多的partition意味着需要更多的内存

在新版本的kafka中可以支持批量提交和批量消费，而设置了批量提交和批量消费后，每个partition都会需要一定的内存空间。

假设为100k，当partition为100时，producer端和consumer端都需要10M的内存；当partition为100000时，producer端和consumer端则都需要10G内存。
无限的partition数量很快就会占据大量的内存，造成性能瓶颈。

## 越多的partition会导致更长时间的恢复期

「kafka通过多副本复制技术，实现kafka的高可用性和稳定性」。每个partition都会有多个副本存在于多个broker中，其中一个副本为leader，其余的为follower。

「kafka集群其中一个broker出现故障时，在这个broker上的leader会需要在其他broker上重新选择一个副本启动为leader」，这个过程由kafka controller来完成，主要是从Zookeeper读取和修改受影响partition的一些元数据信息。

通常情况下，当一个broker有计划的停机，该broker上的partition leader会在broker停机前有次序的一一移走，假设移走一个需要1ms，10个partition leader则需要10ms，这影响很小，并且在移动其中一个leader的时候，其他九个leader是可用的。因此实际上每个partition leader的不可用时间为1ms。但是在宕机情况下，所有的10个partition

leader同时无法使用，需要依次移走，最长的leader则需要10ms的不可用时间窗口，平均不可用时间窗口为5.5ms，假设有10000个leader在此宕机的broker上，平均的不可用时间窗口则为5.5s。

更极端的情况是，当时的broker是kafka controller所在的节点，那需要等待新的kafka leader节点在投票中产生并启用，之后新启动的kafka leader还需要从zookeeper中读取每一个partition的元数据信息用于初始化数据。在这之前partition leader的迁移一直处于等待状态。

## 总结
通常情况下，越多的partition会带来越高的吞吐量，但是同时也会给broker节点带来相应的性能损耗和潜在风险，虽然这些影响很小，但不可忽略，因此需要根据自身broker节点的实际情况来设置partition的数量以及replica的数量。
例如我的集群部署在虚拟机里，12核cpu,就可以在kafka/config/sever.properties配置文件中，设置默认分区12，以后每次创建topic都是12个分区。

# Kafka Rebalance 会有什么样的问题？如何降低Rebalance的负面影响？

Rebalance 会有什么样的问题？
1. 重复消息。Rebalance后，Consumer重新分配Partition，这种情况下提交offset会失败，导致Kafka会重新投递消息，这会导致部分消息会被重复消费。
2. Stop the world。Rebalance期间，Group内的Consumer是停止消费的，因此不合预期的Rebalance会导致Group内的消费停止，这会影响消费效率。
3. Rebalance Storm。因为Group Coordinator在完成Rebalance时，会等待 max.poll.interval.ms，如果这时某个Consumer在处理poll()的批消息时，超过了这个时间，那么当Rebalance完成后，这个Consumer再次poll()，就又会触发一次Rebalance。那么如果这种超时响应是由于某短时间网络固定的延迟波动，那么就会导致频繁的rejoin，进而频繁的Rebalance，从而产生Rebalance Storm。

如何降低Rebalance的负面影响？

1. 设置合理的参数。设置 session.timeout.ms 给HeartBeat维活一定的兼容性；关注 max.poll.interval.ms 与自身Consumer消费消息的处理时长，及时调整参数或者优化Consumer消费逻辑；
2. 保证消息幂等，或者合理的消息滤重。Rebalance不可完全避免（正常的服务发布更新时的Consumer上下线，或者偶发的Consumer实例故障而触发Kafka故障转移），需要考虑将重复消息的影响降低；
3. 考虑 Static Membership。该机制可以通过设置 Consumer的 group.instance.id 来标识Consumer为 static member。而后的Rebalance时，会将原来的Partition分配给原来的Consumer。而且 Static Membership限制了Rebalance的触发情况，会大大降低Rebalance触发的概率
4. 升级Kafka版本，kafka2.4支持 Incremental Cooperative Rebalance，该Rebalance协议尝试将全局的Rebalance分解为多次小的Rebalance，降低Stop the world的影响。

#


