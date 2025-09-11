# TOC
- [TOC](#toc)
- [Flink 特点](#flink-特点)
- [对比 Storm / Spark Streaming / Trident / Flink](#对比-storm--spark-streaming--trident--flink)
- [Flink 如何实现 Exactly-Once 语义](#flink-如何实现-exactly-once-语义)
- [](#)

# Flink 特点

- 同时支持高吞吐、低延迟、高性能
- 支持事件事件概念
- 支持有状态计算
- 支持高度灵活的窗口操作
- 支持轻量级分布式快照 (CheckPoint) 实现的容错
- 基于 JVM 实现独立的内存管理

# 对比 Storm / Spark Streaming / Trident / Flink

Storm 是比较早的流式计算框架，后来又出现了 Spark Streaming 和 Trident，现在又出现了 Flink 这种优秀的实时计算框架，那么这几种计算框架到底有什么区别呢？

- 模型
  - Storm 和 Flink 是真正的一条一条处理数据；而 Trident（Storm 的封装框架）和 Spark Streaming 其实都是小批处理，一次处理一批数据（小批量）。
- API
  - Storm 和 Trident 都使用基础 API 进行开发，比如实现一个简单的 sum 求和操作；而 Spark Streaming 和 Flink 中都提供封装后的高阶函数，可以直接拿来使用，这样就
比较方便了。
- 保证次数
  - 在数据处理方面，Storm 可以实现至少处理一次，但不能保证仅处理一次，这样就会导致数据重复处理问题，所以针对计数类的需求，可能会产生一些误差；Trident 通过事务可以保证对数据实现仅一次的处理，Spark Streaming 和 Flink 也是如此。
- 容错机制
  - Storm和Trident可以通过ACK机制实现数据的容错机制，而Spark Streaming和 Flink 可以通过 CheckPoint 机制实现容错机制。
- 状态管理
  - Storm 中没有实现状态管理，Spark Streaming 实现了基于 DStream 的状态管理，而 Trident 和 Flink 实现了基于操作的状态管理。
- 延时
  - 表示数据处理的延时情况，因此 Storm 和 Flink 接收到一条数据就处理一条数据，其数据处理的延时性是很低的；而 Trident 和 Spark Streaming 都是小型批处理，它们数据处理的延时性相对会偏高。
- 吞吐量
  - Storm 的吞吐量其实也不低，只是相对于其他几个框架而言较低；Trident 属于中等；而 Spark Streaming 和 Flink 的吞吐量是比较高的。

# Flink 如何实现 Exactly-Once 语义
> 一致性保证语义：
> 
> - At-most-once：每条数据消费至多一次，处理延迟低
> - At-least-once：每条数据消费至少一次，一条数据可能存在重复消费
> - Exactly-once：每条数据都被消费且仅被消费一次，仿佛故障从未发生

Checkpoint机制能够保证作业出现 fail-over 后可以从最新的快照进行恢复，即分布式快照机制可以保证 Flink 系统内部的"精确一次"处理。但是我们在实际生产系统中，Flink 会对接各种各样的外部系统，比如 Kafka、HDFS 等，一旦 Flink 作业出现失败，作业会重新消费旧数据，这时候就会出现重新消费的情况，也就是重复消费。

针对这种情况，Flink 1.4 版本引入了一个很重要的功能：两阶段提交，也就是 TwoPhaseCommitSinkFunction。两阶段搭配特定的 source 和 sink（特别是 0.11 版本 Kafka）使得"精确一次处理语义"成为可能。

> 在 Flink 中两阶段提交的实现方法被封装到了 TwoPhaseCommitSinkFunction 这个抽象类中，我们只需要实现其中的beginTransaction、preCommit、commit、abort 四个方法就可以实现"Exactly-once"的处理语义，实现的方式我们可以在官网中查到：
> 
> - BeginTransaction，在开启事务之前，我们在目标文件系统的临时目录中创建一个临时文件，后面在处理数据时将数据写入此文件；
> - PreCommit，在预提交阶段，刷写（flush）文件，然后关闭文件，之后就不能写入到文件了，我们还将为属于下一个检查点的任何后续写入启动新事务；
> - Commit，在提交阶段，我们将预提交的文件原子性移动到真正的目标目录中，请注意，这会增加输出数据可见性的延迟；
> - Abort，在中止阶段，我们删除临时文件。

# 



