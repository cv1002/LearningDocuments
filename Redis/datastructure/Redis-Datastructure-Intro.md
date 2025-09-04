# Intro
- [Intro](#intro)
  - [Redis 五种基本数据类型](#redis-五种基本数据类型)
  - [Redis 三种特殊数据类型](#redis-三种特殊数据类型)

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

## Redis 三种特殊数据类型

除了 5 种基本的数据类型之外，Redis 还支持 3 种特殊的数据类型：
- Bitmap
- HyperLogLog
- GEO

三种类型的简单介绍: 
- Bitmap
  - 你可以将 Bitmap 看作是一个存储二进制数字（0 和 1）的数组，数组中每个元素的下标叫做 offset（偏移量）。通过 Bitmap, 只需要一个 bit 位来表示某个元素对应的值或者状态，key 就是对应元素本身 。我们知道 8 个 bit 可以组成一个 byte，所以 Bitmap 本身会极大的节省储存空间。
- HyperLogLog
  - Redis 提供的 HyperLogLog 占用空间非常非常小，只需要 12k 的空间就能存储接近 2^64 个不同元素。不过，HyperLogLog 的计数结果并不是一个精确值，存在一定的误差（标准误差为 0.81% ）。
- Geospatial index
  - Geospatial index（地理空间索引，简称 GEO） 主要用于存储地理位置信息，基于 Sorted Set 实现。
