# Intro
- [Intro](#intro)
  - [RocketMQ \& Kafka \& RabbitMQ \& Pulsar](#rocketmq--kafka--rabbitmq--pulsar)
  - [Architecture](#architecture)
    - [nameserver 名字服务](#nameserver-名字服务)
    - [broker 代理服务](#broker-代理服务)
    - [生产者 Producer](#生产者-producer)
    - [消费者 Consumer](#消费者-consumer)

## RocketMQ & Kafka & RabbitMQ & Pulsar

| 组件     | 优点                                                                         | 缺点                                     | 场景                         |
| -------- | ---------------------------------------------------------------------------- | ---------------------------------------- | ---------------------------- |
| Kafka    | 吞吐量非常大，性能非常好，集群高可用                                         | 可能丢数据，可能重复消费，功能比较单一   | 日志分析、大数据采集         |
| RabbitMQ | 消息可靠性高，功能全面。                                                     | Erlang语言不方便定制。吞吐量较低。       | 企业内部小规模服务调用       |
| Pulsar   | 基于Bookkeeper构建，消息可靠性高                                             | 周边生态还有差距，目前使用的公司比较少。 | 企业内部大规模服务调用       |
| RocketMQ | 高吞吐、高性能、高可用。功能全面。客户端协议丰富。使用Java语言开发方便定制。 | 服务加载比较慢                           | 几乎全场景，特别适合金融场景 |

## Architecture

### nameserver 名字服务
NameServer是一个简单的 Topic 路由注册中心，支持 Topic、Broker 的动态注册与发现。

主要包括两个功能：
- **Broker管理** NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；
- **路由信息管理** 每个NameServer将保存关于 Broker 集群的整个路由信息和用于客户端查询的队列信息。Producer和Consumer通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。

NameServer通常会有多个实例部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。当某个NameServer因某种原因下线了，客户端仍然可以向其它NameServer获取路由信息。


### broker 代理服务
Broker主要负责消息的存储、投递和查询以及服务高可用保证。

NameServer几乎无状态节点，因此可集群部署，节点之间无任何信息同步。Broker部署相对复杂。

在 Master-Slave 架构中，Broker 分为 Master 与 Slave。一个Master可以对应多个Slave，但是一个Slave只能对应一个Master。Master 与 Slave 的对应关系通过指定相同的BrokerName，不同的BrokerId 来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。

> 部署模型小结
> - 每个 Broker 与 NameServer 集群中的所有节点建立长连接，定时注册 Topic 信息到所有 NameServer。
> - Producer 与 NameServer 集群中的其中一个节点建立长连接，定期从 NameServer 获取Topic路由信息，并向提供 Topic 服务的 Master 建立长连接，且定时向 Master 发送心跳。Producer 完全无状态。
> - Consumer 与 NameServer 集群中的其中一个节点建立长连接，定期从 NameServer 获取 Topic 路由信息，并向提供 Topic 服务的 Master、Slave 建立长连接，且定时向 Master、Slave发送心跳。Consumer 既可以从 Master 订阅消息，也可以从Slave订阅消息。

### 生产者 Producer
发布消息的角色。Producer通过 MQ 的负载均衡模块选择相应的 Broker 集群队列进行消息投递，投递的过程支持快速失败和重试。

### 消费者 Consumer
消息消费的角色。
- 支持以推（push），拉（pull）两种模式对消息进行消费。
- 同时也支持集群方式和广播方式的消费。
- 提供实时消息订阅机制，可以满足大多数用户的需求。

