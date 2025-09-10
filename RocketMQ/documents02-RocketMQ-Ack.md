# RocketMQ Ack
- [RocketMQ Ack](#rocketmq-ack)
  - [生产者消息发送确认机制](#生产者消息发送确认机制)
    - [机制说明](#机制说明)
    - [源码解析](#源码解析)
  - [客户端消息确认机制](#客户端消息确认机制)
    - [机制说明](#机制说明-1)
    - [源码解析](#源码解析-1)
    - [Broker 端的确认处理](#broker-端的确认处理)
  - [示例代码](#示例代码)
    - [生产者](#生产者)
      - [生产者同步发送确认](#生产者同步发送确认)
      - [生产者异步发送确认](#生产者异步发送确认)
    - [消费者](#消费者)
      - [消费者消费确认](#消费者消费确认)
  - [总结](#总结)

## 生产者消息发送确认机制

### 机制说明
- 同步发送: 生产者发送消息后阻塞等待 `Broker` 返回 `SendResult`，包含消息状态（`SEND_OK`、`FLUSH_DISK_TIMEOUT` 等）。
- 异步发送: 通过回调函数 `SendCallback` 处理成功或异常。
- 单向发送: 不关心发送结果，无确认机制。
- 重试机制: 默认同步发送重试 2 次（可配置 `retryTimesWhenSendFailed`）。

### 源码解析

- 核心类: `DefaultMQProducerImpl`
- 关键方法: `sendDefaultImpl()`

```java
private SendResult sendDefaultImpl(Message msg, final CommunicationMode communicationMode, ...) {
    // 选择消息队列（负载均衡）
    MessageQueue mq = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
    // 实际发送逻辑
    sendResult = this.sendKernelImpl(msg, mq, communicationMode, ...);
    // 处理结果或异常，触发重试
}
```

- 发送结果处理: 根据 `communicationMode` 处理同步/异步逻辑。
- 异常处理: 网络异常或 `Broker` 不可用时触发重试。


## 客户端消息确认机制

### 机制说明
- PushConsumer: 客户端监听消息，消费完成后返回状态: 
  - ConsumeConcurrentlyStatus.CONSUME_SUCCESS: 确认消费成功。
  - ConsumeConcurrentlyStatus.RECONSUME_LATER: 消费失败，触发重试。

```java
consumer.registerMessageListener((MessageListenerConcurrently) (msg, context) -> {
   System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msg);
   return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
```

- 重试策略: 消息失败后进入重试队列（`%RETRY%`），默认最多重试 16 次，之后进入死信队列（`%DLQ%`）。
- 顺序消费: 需实现 `MessageListenerOrderly`，消费失败时暂停当前队列消费。

```java
public enum ConsumeOrderlyStatus {
    /**
     * Success consumption
     */
    SUCCESS,
    /**
     * Rollback consumption(only for binlog consumption)
     */
    @Deprecated
    ROLLBACK,
    /**
     * Commit offset(only for binlog consumption)
     */
    @Deprecated
    COMMIT,
    /**
     * Suspend current queue a moment
     */
    SUSPEND_CURRENT_QUEUE_A_MOMENT;
}
```

### 源码解析
- 核心类: `ConsumeMessageConcurrentlyService`
- 关键方法: `processConsumeResult()`

```java
public void processConsumeResult(ConsumeConcurrentlyStatus status, ...) {
    if (status == CONSUME_SUCCESS) {
        // 更新消费进度
        this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(...);
    } else {
        // 发送重试消息到 Broker
        sendMessageBack(msg, delayLevel);
    }
}
```

- ACK 提交: 消费成功后更新本地消费进度（`RemoteBrokerOffsetStore`）。
- 重试处理: 调用 `sendMessageBack()` 将消息发回 `Broker` 并延迟重试。

### Broker 端的确认处理

- 消息存储: `Broker` 将消息持久化到 `CommitLog` 后返回 `ACK`。
- 消费进度管理: 通过 `ConsumerOffsetManager` 记录消费进度。
- 重试队列: `Broker` 维护 `SCHEDULE_TOPIC` 处理延迟重试消息。

## 示例代码

### 生产者

#### 生产者同步发送确认
```java
SendResult sendResult = producer.send(msg);
System.out.println("Send Status: " + sendResult.getSendStatus());
```

#### 生产者异步发送确认
```java
producer.send(msg, new SendCallback() {
  @Override
  public void onSuccess(SendResult sendResult) {
      // do something
  }

  @Override
  public void onException(Throwable e) {
      // do something
  }
});
System.out.printf("%s%n", sendResult);
```

### 消费者

#### 消费者消费确认
```java
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
    try {
        // 处理消息
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    } catch (Exception e) {
        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
    }
});
```

## 总结
- 生产者确认: 通过同步/异步回调确保消息发送到 Broker。
- 消费者确认: 通过返回状态或手动 ACK 控制消息重试逻辑。
- 可靠性保障: 结合重试队列、消费进度持久化和死信队列实现端到端可靠性。

源码中的关键逻辑集中在 `DefaultMQProducerImpl`（发送）和 `ConsumeMessageConcurrentlyService`（消费），通过状态机管理消息生命周期。
