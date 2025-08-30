# Explain
- [Explain](#explain)
  - [Explain作用](#explain作用)
  - [Explain使用示例](#explain使用示例)
  - [执行计划的输出格式](#执行计划的输出格式)
  - [执行计划详解](#执行计划详解)
    - [table](#table)

## Explain作用
Explain是用于分析慢速SQL语句的工具。

Explain可以让我们了解数据库优化器的内部决策过程。清晰的看到：

1. 数据库选择了哪张表作为驱动表
2. 哪些索引被使用了，哪些被忽略了
3. 是否出现了全表扫描、文件排序或临时表操作
4. 每个步骤预估会扫描多少行数据

> Explain只显示执行计划，不执行真正的查询，不会修改数据，也不会消耗大量资源
> MySQL 8.0 开始，可以使用Explain Analyze来查看实际运行过程（包括真实行数和耗时），这比传统的Explain更精确，但会真正的执行SQL

## Explain使用示例
MySQL中，Explain的使用非常简单，在SQL查询前加上Explain关键字。
```SQL
-- 单表查询
explain select * from users where user_id=100;

-- 连表查询
explain select u.name, o.amount
from
  users u join orders o
on
  u.user_id = o.user_id
where
  o.amount > 500;
```

执行后，MySQL不会真正的运行SQL，而是返回一张表，表示执行计划。

这张表告诉我们：
- 优化器如何打算如何执行这条语句
- 选择了哪些索引
- 预计会扫描多少行数据
- 是否需要排序、临时表等额外操作

## 执行计划的输出格式
标准的Explain是输出一张表格，不同版本（5.7雨与8.）字段略有差别，但核心字段基本一致。

字段简介：
- id: 查询的执行顺序标识（数值越大优先级越高）
- select_type: 查询类型（SIMPLE、PRIMARY、SUBQUERY等）
- table: 当前访问的表名或别名
- partitions: 涉及的分区信息（若表使用了分区功能）
- type: 连接类型，性能由好到坏排序 `system > const > eq_req > ref > range > index > ALL` 
- possible_keys: 优化器认为可用的索引
- key: 实际选择的索引
- key_len: 使用索引的长度
- ref: 索引比较时使用的列或常量
- rows: 预计需要扫描的行数
- filtered: 条件过滤后剩余的行数比例
- Extra: 额外信息（是否用临时表、是否派序、是否覆盖索引等）

## 执行计划详解
### table
table字段表示当前操作涉及到的表名或者别名。

以当前SQL为例：
```SQL
explain select u.name, o.amount
from
  users u join orders o
on
  u.user_id = o.user_id
where
  o.amount > 500;
```

可能输出如下：
| id  | select_type | table | type   | possible_keys | key        | key_len | ref          | rows | Extra       |
| --- | ----------- | ----- | ------ | ------------- | ---------- | ------- | ------------ | ---- | ----------- |
| 1   | SIMPLE      | o     | ref    | idx_amount    | idx_amount | 4       | const        | 1000 | Using where |
| 1   | SIMPLE      | u     | eq_ref | PRIMARY       | PRIMARY    | 4       | db.p.user_id | 1    | Using index |

> 解析：
> - 第一行 o 表示优化器先扫描 orders 表
> - 第二行 u 表示基于第一行结果，通过索引关联扫描 users 表
> - **重要提示**：多表 join 中，table字段的顺序并不总是 SQL 中写的顺序，而是优化器决定的执行顺序。优化器会选择**成本最低**的表先执行
>
> 
> 实践案例：
> - 假设 orders 表有 100 万行，users表有 10 万行，如果优化器先扫描 users 表，再 join orders，会导致巨量行扫描。
> - 使用 explain 可以确认**优化器选择了正确的驱动表**


