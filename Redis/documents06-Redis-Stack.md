# Stack
- [Stack](#stack)
  - [介绍](#介绍)
  - [Redis JSON](#redis-json)
    - [Redis JSON 命令](#redis-json-命令)
    - [JSON 支持两种查询语法](#json-支持两种查询语法)
  - [Redis 查询引擎](#redis-查询引擎)
  - [Bloom Filter](#bloom-filter)
    - [介绍](#介绍-1)
  - [Cuckoo Filter](#cuckoo-filter)

## 介绍
Redis Stack是一个基于Redis的开源项目，提供了全文搜索、文档数据库、向量数据库、遥测、身份和资源管理等多种功能。

## Redis JSON
RedisJSON是Redis的一个扩展模块，提供了对JSON数据的原生支持。

RedisJSON的特性: 
- 完全支持JSON标准
- 所有JSON值类型的类型化原子操作
- 文档以二进制数据形式存储在树结构中，允许快速访问子元素
- 用于选择/更新文档内部元素的JSONPath语法（参见JSONPath语法）
- 与其他 Redis 数据类型（如 Streams）结合使用

### Redis JSON 命令
```shell
# 创建 JSON 字符串
JSON.SET bike $ '"Hyperion"'
# 查询 JSON
JSON.GET bike $
# 追加字符串
JSON.STRAPPEND bike $ '" (Enduro bikes)"'
# 数字可以被递增和相乘
JSON.NUMINCRBY crashes $ 1
JSON.NUMMULTBY crashes $ 24
# 删除 JSON 中的 Key
JSON.DEL user $.address
# 查看 user 对象的所有 Key
JSON.OBJKEYS user
# 查看数据类型
JSON.TYPE user
JSON.TYPE user $.age
JSON.TYPE user $.name
# 对 bool 类型，对换 true 和 false
JSON.TOGGLE
```

### JSON 支持两种查询语法
- JSONPath 语法
- 第一版 JSON 中的旧版路径语法。

JSONPath 语法:
| 语法元素         | 描述                                                                                                                                                                                              |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| $                | 根（最外层 JSON 元素），路径的起点。                                                                                                                                                              |
| . 或 []          | 选择子元素。                                                                                                                                                                                      |
| ..               | 递归下降遍历 JSON 文档。                                                                                                                                                                          |
| *                | 通配符，返回所有元素。                                                                                                                                                                            |
| []               | 下标运算符，访问数组元素。                                                                                                                                                                        |
| [,]              | 联合运算符，选择多个元素。                                                                                                                                                                        |
| [start:end:step] | 数组切片，其中 start、end 和 step 是索引值。您可以从切片中省略值（例如，[3:]、[:8:2]）以使用默认值：start 默认为第一个索引，end 默认为最后一个索引，step 默认为 1。使用 [*] 或 [:] 选择所有元素。 |
| ?()              | 过滤 JSON 对象或数组。支持比较运算符(==, !=, <, <=, >, >=, =~)、逻辑运算符(&&,                                                                                                                    |  | )和括号((, )). |
| ()               | 脚本表达式。                                                                                                                                                                                      |
| @                | 当前元素，用于过滤或脚本表达式中。                                                                                                                                                                |

## Redis 查询引擎
使用 Redis 查询引擎进行 Redis 数据的搜索和查询

Redis 查询引擎通过以下搜索和查询功能提供增强的 Redis 使用体验：
- 丰富的查询语言
- 对 JSON 和哈希文档的增量索引
- 向量搜索
- 全文搜索
- 地理空间查询
- 聚合功能

Redis 查询引擎的功能使你可以将 Redis 用作：
- 文档数据库
- 向量数据库
- 二级索引
- 搜索引擎


传统的SCAN搜索方式，只能简单的过滤Key。如果想要做一些复杂的搜索，就力不从心了。

比如在电商场景中，我们通常会用Redis来缓存商品信息，但是如果要做按品牌、型号、价格等等各种条件过滤商品的场景，Redis就不够用了。以往我们会选择将商品数据导入到MongoDB或者ElasticSearch这样的搜索引擎进行复杂过滤。

而Redis提供了RedisSearch插件，基本就可以认为是ElasticSearch这类搜索引擎的平替。大部分ES能够实现的搜索功能，在Redis里就能直接进行。这样就极大的减少了数据迁移带来的麻烦。

既然要做搜索，那就需要有能够支持搜索的数据结构。 Redis的哪些数据结构能够支持结构化查询呢？只有HASH和JSON。

## Bloom Filter
### 介绍
布隆过滤器是一种快速检索一个元素是否在一个海量集合中的算法。

布隆过滤器可以告诉我们 **某样东西一定不存在或者可能存在**，也就是说布隆过滤器说这个数不存在则一定不存在，反之则可能存在。

作用：
- 解决Redis缓存穿透问题（面试重点）
- 邮件过滤，使用布隆过滤器来做邮件黑名单过滤
- 对爬虫网址进行过滤，爬过的不再爬
- 解决新闻推荐过的不再推荐(类似抖音刷过的往下滑动不再刷到)
- HBase\RocksDB\LevelDB等数据库内置布隆过滤器，用于判断数据是否存在，可以减少数据库的IO请求

## Cuckoo Filter
布隆过滤器最大的问题是无法删除数据。因此，后续诞生了很多布隆过滤器的改进版本。Cuckoo Filter 布谷鸟过滤器就是其中一种。

相比于布隆过滤器，Cuckoo Filter可以删除数据。而且基于相同的集合和误报率，Cuckoo Filter通常占用空间更少，算法实现也更复杂。

不过他同样有误判。有可能将一个不在集合中的元素错误的判断成在集合中。

布隆过滤器的误报率通过调整位数组的大小和哈希函数来控制，而CuckooFilter的误报率受指纹大小和桶大小控制。
