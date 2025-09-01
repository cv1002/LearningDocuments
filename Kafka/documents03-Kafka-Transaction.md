# Kafka事务
- [Kafka事务](#kafka事务)
  - [Reference](#reference)
  - [什么是 KAFKA 的事务机制](#什么是-kafka-的事务机制)
  - [知识总结](#知识总结)

## Reference
[一文读懂Kafka事务机制](https://zhuanlan.zhihu.com/p/411171789?spm=a2c6h.13046898.publish-article.8.20506ffawEKTMZ)

## 什么是 KAFKA 的事务机制
- KAFKA 的事务机制，是 KAFKA 实现端到端有且仅有一次语义（end-to-end EOS)的基础；
- KAFKA 的事务机制，涉及到 transactional producer 和 transactional consumer, 两者配合使用，才能实现端到端有且仅有一次的语义（end-to-end EOS)；
- 当然kakfa 的 producer 和 consumer 是解耦的，你也可以使用非 transactional 的 consumer 来消费 transactional producer 生产的消息，但此时就丢失了事务 ACID 的支持；
- 通过事务机制，KAFKA 可以实现对多个 topic 的多个 partition 的原子性的写入，即处于同一个事务内的所有消息，不管最终需要落地到哪个 topic 的哪个 partition, 最终结果都是要么全部写成功，要么全部写失败（Atomic multi-partition writes）；
- KAFKA的事务机制，在底层依赖于幂等生产者，幂等生产者是 kafka 事务的必要不充分条件；
- 事实上，开启 kafka事务时，kafka 会自动开启幂等生产者。

## 知识总结
- KAFKA 的事务机制，是 KAFKA 实现端到端有且仅有一次语义（end-to-end EOS)的基础；
- KAFKA 的事务机制，涉及到 transactional producer 和 transactional consumer, 两者配合使用，才能实现端到端有且仅有一次的语义（end-to-end EOS)；
- 通过事务机制，KAFKA 实现了对多个 topic 的多个 partition 的原子性的写入（Atomic multi-partition writes）；
- KAFKA的事务机制，在底层依赖于幂等生产者，幂等生产者是 kafka 事务的必要不充分条件：用户可以根据需要，配置使用幂等生产者但不开启事务；也可以根据需要开启 kafka事务，此时kafka 会使用幂等生产者；
- 为支持事务机制，KAFKA 引入了两个新的组件： Transaction Coordinator 和 Transaction Log，其中 transaction coordinator 是运行在每个 kafka broker 上的一个模块，是 kafka broker 进程承载的新功能之一（不是一个独立的新的进程）；而 transaction log 是 kakafa 的一个内部 topic；
- 为支持事务机制，kafka 将日志文件格式进行了扩展：日志中除了普通的消息，还有一种消息专门用来标志一个事务的结束，它就是控制消息 controlBatch,它有两种类型：commit和abort，分别用来表征事务已经成功提交或已经被成功终止。
- 开启了事务的生产者，生产的消息最终还是正常写到目标 topic 中，但同时也会通过 transaction coordinator 使用两阶段提交协议，将事务状态标记 transaction marker，也就是控制消息 controlBatch，写到目标 topic 中，控制消息共有两种类型 commit 和 abort，分别用来表征事务已经成功提交或已经被成功终止；
- 开启了事务的消费者，如果配置读隔离级别为 read-committed, 在内部会使用存储在目标 topic-partition 中的事务控制消息，来过滤掉没有提交的消息，包括回滚的消息和尚未提交的消息,从而确保只读到已提交的事务的 message；// 对于非事务生产者生产的消息，他们不处于事务中，使用 read-committed 隔离级别的开启了事务的消费者，可以正常读取这些消息；
- 开启了事务的消费者，过滤消息时，KAFKA consumer 不需要跟 transactional coordinator 进行 rpc 交互，因为 topic 中存储的消息，包括正常的数据消息和控制消息，包含了足够的元数据信息来支持消息过滤；
- 当然 kakfa 的 producer 和 consumer 是解耦的，你也可以使用非 transactional consumer 来消费 transactional producer 生产的消息，此时目标 topic-partition 中的所有消息都会被返回，不会进行过滤,此时也就丢失了事务 ACID 的支持；

