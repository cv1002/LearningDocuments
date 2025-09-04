# Bitmap
- [Bitmap](#bitmap)
  - [介绍](#介绍)
  - [常用命令](#常用命令)
  - [应用场景](#应用场景)

## 介绍

根据官网介绍：
> - Bitmaps are not an actual data type, but a set of bit-oriented operations defined on the String type which is treated like a bit vector. Since strings are binary safe blobs and their maximum length is 512 MB, they are suitable to set up to 2^32 different bits
> - Bitmap 不是 Redis 中的实际数据类型，而是在 String 类型上定义的一组面向位的操作，将其视为位向量。由于字符串是二进制安全的块，且最大长度为 512 MB，它们适合用于设置最多 2^32 个不同的位。

Bitmap 存储的是连续的二进制数字（0 和 1），通过 Bitmap, 只需要一个 bit 位来表示某个元素对应的值或者状态，key 就是对应元素本身 。我们知道 8 个 bit 可以组成一个 byte，所以 Bitmap 本身会极大的节省储存空间。

你可以将 Bitmap 看作是一个存储二进制数字（0 和 1）的数组，数组中每个元素的下标叫做 offset（偏移量）。

## 常用命令
| 命令                                  | 介绍                                                             |
| ------------------------------------- | ---------------------------------------------------------------- |
| SETBIT key offset value               | 设置指定 offset 位置的值                                         |
| GETBIT key offset                     | 获取指定 offset 位置的值                                         |
| BITCOUNT key start end                | 获取 start 和 end 之间值为 1 的元素个数                          |
| BITOP operation destkey key1 key2 ... | 对一个或多个 Bitmap 进行运算，可用运算符有 AND, OR, XOR 以及 NOT |

## 应用场景
**需要保存状态信息（0/1 即可表示）的场景**

- 举例：用户签到情况、活跃用户情况、用户行为统计（比如是否点赞过某个视频）。
- 相关命令：SETBIT、GETBIT、BITCOUNT、BITOP。

