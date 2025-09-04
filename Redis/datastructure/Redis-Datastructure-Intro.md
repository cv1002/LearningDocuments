# Intro
- [Intro](#intro)
  - [Redis 五种基本数据类型](#redis-五种基本数据类型)

## Redis 五种基本数据类型
Redis 共有 5 种基本数据类型：String（字符串）、List（列表）、Set（集合）、Hash（散列）、Zset（有序集合）。

这 5 种数据类型是直接提供给用户使用的，是数据的保存形式，其底层实现主要依赖这 8 种数据结构：
- SimpleDynamicString（SDS，简单动态字符串）
- LinkedList（双向链表）
- Dict（哈希表/字典）
- SkipList（跳跃表）
- Intset（整数集合）
- ZipList（压缩列表）
- QuickList（快速列表）

Redis 5 种基本数据类型对应的底层数据结构实现如下表所示：

| String | List                         | Hash          | Set          | Zset              |
| ------ | ---------------------------- | ------------- | ------------ | ----------------- |
| SDS    | LinkedList/ZipList/QuickList | Dict、ZipList | Dict、Intset | ZipList、SkipList |

Redis 3.2 之前，List 底层实现是 LinkedList 或者 ZipList。 Redis 3.2 之后，引入了 LinkedList 和 ZipList 的结合 QuickList，List 的底层实现变为 QuickList。从 Redis 7.0 开始， ZipList 被 ListPack 取代。


你可以在 Redis 官网上找到 Redis 数据类型/结构非常详细的介绍：
- [Redis Data Structures](https://redis.io/technology/data-structures/)

未来随着 Redis 新版本的发布，可能会有新的数据结构出现，通过查阅 Redis 官网对应的介绍，你总能获取到最靠谱的信息。
