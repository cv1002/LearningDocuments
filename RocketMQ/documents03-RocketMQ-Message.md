# RocketMQ Message
- [RocketMQ Message](#rocketmq-message)
  - [顺序消息](#顺序消息)
  - [延迟消息](#延迟消息)
    - [基于延迟级别设置延迟消息](#基于延迟级别设置延迟消息)
    - [示例代码](#示例代码)
  - [批量消息](#批量消息)
  - [事务消息](#事务消息)

## 顺序消息



## 延迟消息
当消息写入到Broker后，不能立刻被消费者消费，需要等待指定的时长后才可被消费处理的消息，称为延时消息。

Message可以通过三种方式设置延迟时间，一类是设置延迟时长，一类是直接设置投递时间戳。

```java
public void setDelayTimeLevel(int level) {
    this.putUserProperty(MessageConst.PROPERTY_TIMER_DELAY_LEVEL, String.valueOf(level));
}
public void setDelayTimeSec(long sec) {
    this.putUserProperty(MessageConst.PROPERTY_TIMER_DELAY_SEC, String.valueOf(sec));
}
public void setDelayTimeMs(long timeMs) {
    this.putUserProperty(MessageConst.PROPERTY_TIMER_DELAY_MS, String.valueOf(timeMs));
}
public void setDeliverTimeMs(long timeMs) {
    this.putUserProperty(MessageConst.PROPERTY_TIMER_DELAY_MS, String.valueOf(timeMs));
}
```

### 基于延迟级别设置延迟消息
Rocket MQ 默认支持18个等级的延迟消息，延时等级定义在RocketMQ服务端的MessageStoreConfig类中的如下变量中：

```java
// MessageStoreConfig.java
private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";
```
发消息时，设置delayLevel等级即可：msg.setDelayLevel(level)。level有以下三种情况：
- level == 0
  - 消息为非延迟消息
- 1<=level<=maxLevel
  - 消息延迟特定时间，例如 level == 1，延迟 1s
- level > maxLevel
  - 则level == maxLevel，例如 level == 20，延迟 2h

例如指定的延时等级为3，则表示延迟时长为10s，延迟等级是从1开始计数的。

不使用定时器，使用Rocket MQ的延迟消息是可以实现定时任务的功能，但是也是在一些特定的情景之下。下面介绍2种场景的使用：

1. 电商交易系统的订单超时未支付，自动取消订单
2. 活动场景，发起活动后，同时发送一条延时消息，延时时间为设置本次活动周期的时间，当活动结束后，这条消息正好可以被消费者消费，然后进行后期的处理，比如活动结算

### 示例代码
```java
// 固定延迟级别
Message message = new Message(TOPIC, ("Hello scheduled message " + i).getBytes(StandardCharsets.UTF_8));
message.setDelayTimeLevel(3); // 10s

// 自定义延迟时间
Message message = new Message(TOPIC, ("Hello scheduled message " + i).getBytes(StandardCharsets.UTF_8));
message.setDelayTimeMs(System.currentTimeMillis() + 10_000L); // 10s
```

## 批量消息
生产者消息比较多的时候，可以多条消息合并成一个批量消息，减少网络IO，提升性能。

示例代码：
```java
List<Message> messages = new ArrayList<>(MESSAGE_COUNT);
for (int i = 0; i < MESSAGE_COUNT; i++) {
    messages.add(new Message(TOPIC, TAG, "OrderID" + i, ("Hello world " + i).getBytes(StandardCharsets.UTF_8)));
}
ListSplitter splitter = new ListSplitter(messages);
while (splitter.hasNext()) {
    List<Message> listItem = splitter.next();
    SendResult sendResult = producer.send(listItem);
    System.out.printf("%s\n", sendResult);
}
```

> **注意点**: 批量消息的使用非常简单，但是要注意 RocketMQ 做了限制。同一批消息的 Topic 必须相同，另外不支持延迟消息。
>
> 还有批量消息的大小爱=不要超过 1M，太大则需要自行分割。
>
> 另外，当前版本也在尝试实现一种自动的消息分隔机制。目前没有放到Example中。
> 
> 详见 `org.apache.rocketmq.client.producer.ProduceAccumulatorTest`的`testProduceAccumulator_async`和`testProduceAccumulator_sync`方法。
>
> 基于客户端内部一个新增的`ProduceAccumulator`组件。

## 事务消息
通过 RocketMQ 的事务机制，来保证上下游数据一致性。

具体实现思路: 
1. 生产者将消息发送至 Apache RocketMQ 服务端
2. Apache RocketMQ 服务端将消息持久化后，向生产者发送确认消息，此时消息标记为半事务消息
3. 生产者开始执行本地事务逻辑
4. 生产者根据本地事务结果向服务端提交二次确认结果（Commit 或是 Rollback）
   - 如果提交结果为 Commit，此时消息最终被消费
   - 如果提交结果为 Rollback，此时消息最终被删除，不会被消费
5. 在断网或生产者应用重启的特殊情况下，若服务端未收到二次确认结果，或二次确认结果为 Unknown 状态，经过固定时间后，服务端将对该消息发起消息回查
6. 生产者收到消息回查后，需要检查对应消息的本地事务执行状态
   - 如果本地事务执行成功，确认提交该消息
   - 如果本地事务执行失败，确认回滚该消息
7. 如果消息被确认提交，此时消息被消费者消费，如果消息被确认回滚，此时消息不会被消费者消费
8. 如果消息回查次数达到一定阈值仍未成功，此时 RocketMQ 将丢弃该消息

**值得注意的是，rocketmq并不会无休止的的信息事务状态回查，默认回查15次，如果15次回查还是无法得知事务状态，rocketmq默认回滚该消息**

