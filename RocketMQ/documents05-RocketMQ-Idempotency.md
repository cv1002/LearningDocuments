# Idempotency
- [Idempotency](#idempotency)
  - [幂等性问题的根源](#幂等性问题的根源)
  - [RocketMQ 幂等性实现方案](#rocketmq-幂等性实现方案)
    - [唯一业务标识法](#唯一业务标识法)
      - [实现步骤](#实现步骤)
        - [生产者附加唯一标识](#生产者附加唯一标识)
        - [消费者检查唯一键](#消费者检查唯一键)
    - [数据库唯一约束](#数据库唯一约束)
      - [实现示例](#实现示例)
        - [消费时通过数据库事务实现](#消费时通过数据库事务实现)
    - [Redis原子操作](#redis原子操作)
      - [实现示例](#实现示例-1)
  - [高级技巧](#高级技巧)
    - [布隆过滤器优化](#布隆过滤器优化)
    - [消息指纹去重](#消息指纹去重)
    - [RocketMQ事务消息结合幂等](#rocketmq事务消息结合幂等)
    - [监控与验证](#监控与验证)
      - [幂等校验失败监控 使用Metrics统计重复率](#幂等校验失败监控-使用metrics统计重复率)
      - [自动化测试方案](#自动化测试方案)
  - [最佳实践总结](#最佳实践总结)
  - [实施建议：](#实施建议)

## 幂等性问题的根源
重复消费可能由以下原因引发：

- 生产者重试：网络抖动时消息可能重复发送
- Broker 主从切换：未同步的消息可能被重新投递
- 消费者重试：消费失败后消息重新入队

## RocketMQ 幂等性实现方案

### 唯一业务标识法
核心逻辑：通过业务唯一键（如订单ID）标识消息唯一性

#### 实现步骤

##### 生产者附加唯一标识
```java
Message msg = new Message(
    "OrderTopic",
    "Order_123456".getBytes(), // 业务唯一键作为消息Key
    "OrderBody".getBytes()
);
```

##### 消费者检查唯一键
```java
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
    for (MessageExt msg : msgs) {
        String orderId = msg.getKeys(); // 提取唯一键
        if (!isProcessed(orderId)) { // 检查是否已处理
            processOrder(orderId); // 处理业务逻辑
            markAsProcessed(orderId); // 标记为已处理
        }
    }
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
```

### 数据库唯一约束
适用场景：强一致性要求的场景（如支付系统）

#### 实现示例
```java
CREATE TABLE order_records (
    order_id VARCHAR(64) PRIMARY KEY,  -- 唯一键约束
    status TINYINT,
    ...
);
```

##### 消费时通过数据库事务实现
```java

@Transactional
public void processOrder(MessageExt msg) {
    String orderId = msg.getKeys();
    if (orderDao.exists(orderId)) return; // 已存在则跳过
    
    orderDao.insert(orderId);            // 插入唯一记录
    // 执行业务操作...
}
```

### Redis原子操作
适用场景：高频低延迟场景（如秒杀系统）

#### 实现示例
```java
public boolean checkIdempotent(String messageId) {
    String redisKey = "msg:" + messageId;
    // SETNX + EXPIRE 原子操作
    return redisTemplate.opsForValue()
        .setIfAbsent(redisKey, "1", Duration.ofMinutes(30));
}
```

## 高级技巧

### 布隆过滤器优化
```java
// 初始化布隆过滤器（需预估数据量）
BloomFilter<String> filter = BloomFilter.create(
    Funnels.stringFunnel(Charset.defaultCharset()), 
    1_000_000,  // 预计元素数量
    0.01         // 误判率
);

// 消费时快速过滤
if (filter.mightContain(orderId)) {
    // 可能重复，走详细检查流程
    checkRedisOrDB(orderId);
} else {
    filter.put(orderId);
    processOrder(orderId);
}
```

### 消息指纹去重
```java
String messageFingerprint = DigestUtils.md5Hex(
    msg.getTopic() + 
    msg.getTags() + 
    new String(msg.getBody())
);
```

### RocketMQ事务消息结合幂等

### 监控与验证

#### 幂等校验失败监控 使用Metrics统计重复率
```java
Meter duplicateMeter = Metrics.meter("message.duplicate");
if (isDuplicate) {
    duplicateMeter.mark();
}
```

#### 自动化测试方案
```python
# 使用Pytest模拟重复消息
def test_idempotence():
send_message(order_id)
result1 = consume_message()
send_message(order_id) # 故意重复发送
result2 = consume_message()
assert result1 == result2
```

## 最佳实践总结

设计原则：
- 前置过滤（快速失败）
- 最终一致性兜底
- 幂等范围控制（按消息/按操作）

## 实施建议：
```shell
# 启用RocketMQ消息轨迹追踪
export NAMESRV_ADDR=localhost:9876
mqadmin updateSubGroup -c DefaultCluster -g YourGroup -t true
```

通过结合业务特性选择合适方案，可将消息重复率控制在10^-6级别。实际场景中推荐采用Redis+DB二级校验的混合模式，兼顾性能与可靠性。

