# RocketMQ Message Order
- [RocketMQ Message Order](#rocketmq-message-order)
  - [介绍](#介绍)
    - [生产者示例代码](#生产者示例代码)
    - [消费者示例代码](#消费者示例代码)

## 介绍
RocketMQ作为阿里巴巴开源的分布式消息队列,在保证消息顺序性方面提供了一种基于MessageQueueSelector的解决方案。其核心思路是将有序的消息写入特定的队列,从而使消费端固定消费某个队列时,就能够按顺序消费消息。

具体来说,RocketMQ中有两个重要概念:

- Topic: 逻辑上的消息主题
- MessageQueue: 物理上存储消息的队列

一个Topic包含多个MessageQueue,消息会根据其内容进行哈希计算,分配到不同的MessageQueue中。用户可以通过提供MessageQueueSelector,对特定类型的消息强制分配到同一个MessageQueue,从而保证顺序性。

### 生产者示例代码

```java
// 实例化消息生产者Producer
DefaultMQProducer producer = new DefaultMQProducer("unique_group_name");
// 设置NameServer的地址
producer.setNamesrvAddr("nameserver:9876");
// 启动Producer实例
producer.start();
// 创建消息，并指定Topic，Tag和消息体
Message msg = new Message("TopicTest", "TagA", "OrderID" + orderId, ("Hello RocketMQ " + i).getBytes());
// 发送有序消息
producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Integer orderId = (Integer) arg; // 订单ID作为选择器的参数
        int index = orderId % mqs.size(); // 根据订单ID计算MessageQueue索引
        return mqs.get(index); // 返回该索引对应的MessageQueue
    }
}, orderId);
```
通过上述代码,发送端可以将具有相同订单号的消息发送到同一个MessageQueue。

### 消费者示例代码

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("unique_group_name");
consumer.setNamesrvAddr("nameserver:9876");
consumer.subscribe("TopicTest", "TagA");
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
        context.setAutoCommit(true);
        for (MessageExt msg : msgs) {
            System.out.printf("Consumer: %s %n", new String(msg.getBody()));
        }
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```
消费端只需固定消费指定的MessageQueue,即可以保证消息按顺序被消费。

