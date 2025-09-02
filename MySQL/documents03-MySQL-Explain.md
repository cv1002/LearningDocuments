# Explain
- [Explain](#explain)
  - [Explain作用](#explain作用)
  - [Explain使用示例](#explain使用示例)
  - [执行计划的输出格式](#执行计划的输出格式)
  - [执行计划详解](#执行计划详解)
    - [table](#table)
    - [id](#id)
    - [select\_type](#select_type)
    - [partitions](#partitions)
    - [type](#type)
    - [possible\_keys 与 key](#possible_keys-与-key)
    - [key\_len](#key_len)
    - [ref](#ref)
    - [rows](#rows)
    - [filtered](#filtered)
    - [Extra](#extra)
    - [JSON格式执行计划](#json格式执行计划)
    - [Extended Explain](#extended-explain)
      - [功能介绍](#功能介绍)
      - [使用方式](#使用方式)
        - [示例: 索引未生效](#示例索引未生效)
  - [性能优化策略](#性能优化策略)
    - [避免全表扫描](#避免全表扫描)
      - [优化方法](#优化方法)
        - [创建合理的索引](#创建合理的索引)
        - [避免函数包装列](#避免函数包装列)
        - [合理选择驱动表](#合理选择驱动表)
        - [使用分区裁剪](#使用分区裁剪)
    - [消除文件排序](#消除文件排序)
      - [原因分析](#原因分析)
      - [优化方法](#优化方法-1)
        - [为排序字段创建索引](#为排序字段创建索引)
        - [联合索引覆盖 order by 列](#联合索引覆盖-order-by-列)
        - [尽量减少排序量](#尽量减少排序量)
    - [避免临时表](#避免临时表)
      - [原因分析](#原因分析-1)
      - [优化方法](#优化方法-2)
        - [使用索引覆盖](#使用索引覆盖)
        - [查询重写](#查询重写)
        - [合理分页](#合理分页)
    - [覆盖索引的构建](#覆盖索引的构建)
      - [原理](#原理)
      - [示例](#示例)
      - [优化建议](#优化建议)

## Explain作用
Explain是用于分析慢速SQL语句的工具。

Explain可以让我们了解数据库优化器的内部决策过程。清晰的看到: 

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

这张表告诉我们: 
- 优化器如何打算如何执行这条语句
- 选择了哪些索引
- 预计会扫描多少行数据
- 是否需要排序、临时表等额外操作

## 执行计划的输出格式
标准的Explain是输出一张表格，不同版本（5.7雨与8.）字段略有差别，但核心字段基本一致。

字段简介: 
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

以当前SQL为例: 
```SQL
explain select u.name, o.amount
from
  users u join orders o
on
  u.user_id = o.user_id
where
  o.amount > 500;
```

可能输出如下: 
| id  | select_type | table | type   | possible_keys | key        | key_len | ref          | rows | Extra       |
| --- | ----------- | ----- | ------ | ------------- | ---------- | ------- | ------------ | ---- | ----------- |
| 1   | SIMPLE      | o     | ref    | idx_amount    | idx_amount | 4       | const        | 1000 | Using where |
| 1   | SIMPLE      | u     | eq_ref | PRIMARY       | PRIMARY    | 4       | db.p.user_id | 1    | Using index |

> 解析: 
> - 第一行 o 表示优化器先扫描 orders 表
> - 第二行 u 表示基于第一行结果，通过索引关联扫描 users 表
> - **重要提示**: 多表 join 中，table字段的顺序并不总是 SQL 中写的顺序，而是优化器决定的执行顺序。优化器会选择**成本最低**的表先执行
>
> 
> 实践案例: 
> - 假设 orders 表有 100 万行，users表有 10 万行，如果优化器先扫描 users 表，再 join orders，会导致巨量行扫描。
> - 使用 explain 可以确认**优化器选择了正确的驱动表**
### id
- id 字段用于标识查询中每个 SELECT 的执行顺序和层级。它的值越大，优先级越高
- 在多表 JOIN 或子查询中，id 可以帮助我们判断哪一步先执行，哪一步后执行

以当前SQL为例: 
```SQL
explain select * from orders where order_id = 100;
```
可能输出如下: 
| id  | select_type | table  | type  | key     | rows | Extra       |
| --- | ----------- | ------ | ----- | ------- | ---- | ----------- |
| 1   | SIMPLE      | orders | const | PRIMARY | 1    | Using index |

> 解析: 
> - 只有一条 select，id 为 1
> - SIMPLE类型表示没有子查询或复杂操作

### select_type

### partitions

### type

### possible_keys 与 key

### key_len

### ref

### rows

### filtered

### Extra

### JSON格式执行计划

### Extended Explain
#### 功能介绍
- Extended Explain 可以让优化器显示查询重写之后的执行计划
- 与普通的Explain不同，它可以通过show warnings查看优化器对查询的重写信息
- 常用于调试复杂查询、子查询优化、索引未生效等问题
- MySQL移除了 Extended Explain 功能

#### 使用方式
```SQL
explain extended
select
  u.name, o.amount
from
  users u
join
  orders o
on
  u.id = o.user_id
where
  o.amount > 500;
```
执行后: 
```SQL
show warnings;
```
返回类似: 
```SQL
Note: select_rewritten as
select u.name, o.amount
from
  users as u
inner join
  orders as o
on
  u.id = o.user_id
where
  o.amount > 500;
```
解析: 
- select_rewritten 显示优化器对原查询的重写结果
- 优化器可能将复杂表达式展开、合并子查询、消除冗余列等
核心作用: 
- 调试索引未使用问题
  - 通过查看重写后的语句，分析优化器为什么不使用某个索引
  - 例如函数包装列或隐式类型转换可能导致索引失效
- 分析子查询优化
  - extended explain能显示子查询被重写为join或者临时表的情况
  - 有助于理解优化器执行顺序和成本
- 理解优化器执行逻辑
  - 比如where子句顺序调整、常量折叠、派生表展开等
  - 对调优复杂查询和索引设计非常有帮助

##### 示例: 索引未生效
```SQL
explain extended select * from orders where year(order_date) = 2023;
show warnings;
```
可能输出: 
```SQL
Note: select_rewritten as
select * from orders where (order_date >= '2023-01-01' and order_date < '2024-01-01')
```
解析: 
- 优化器将`year(order_date)`转换为范围条件
- 原始函数表达式可能导致索引无法直接使用
- 优化建议: 直接使用范围条件避免函数包装、提升索引利用率

## 性能优化策略
- 查询优化的核心技术是**减少扫描行数、提高索引命中率、消除额外操作（临时表、文件排序）**
- 通过 explain 可以直观的判断性能瓶颈，优化策略包括索引优化、查询重写、分区裁剪等

### 避免全表扫描
全表扫描（type=ALL）是性能最差的访问方式，尤其是大数据表上影响显著。
> 原因分析: 
> - 查询条件列没有索引
> - 查询使用函数或类型转换，导致索引失效
> - 多表 join 驱动表选择不合理

#### 优化方法
##### 创建合理的索引
- 单列索引或联合索引覆盖查询条件
- 优先保证常用 where 条件列和 join 列被索引覆盖
```SQL
create index idx_amount on orders(amount);

explain select * from orders where amount > 500;
```
##### 避免函数包装列
```SQL
-- 不推荐
select * from orders where YEAR(order_date) = 2023;

-- 推荐
select
  *
from
  orders
where
  order_date >= '2023-01-01'
and
  order_date < '2024-01-01'
```

##### 合理选择驱动表
- 小表优先驱动大表
- 避免全表扫描带动小表

##### 使用分区裁剪
- 对大表按日期、地区、或业务维度分区
- 查询只扫描部分分区，降低 rows

```SQL
create table orders (
  order_id int primary key,
  order_date date,
  amount decimal(10, 2)
)
partition by range(year(order_date)) (
  partition p2022 values less than (2023),
  partition p2023 values less than (2024)
);
```

### 消除文件排序
文件排序通常出现在 order by 或 group by 需要排序时，未能利用索引。
#### 原因分析
- 排序字段没有索引
- 索引顺序与 order by 列不匹配
- 联合排序（order by多列）未使用最左前缀索引
#### 优化方法
##### 为排序字段创建索引
```SQL
create index idx_amount on orders(amount);

explain select * from orders order by amount desc;
```
##### 联合索引覆盖 order by 列
```SQL
create index idx_user_amount on orders(user_id, amount);

explain select * from orders where user_id = 123 order by amount desc;
```
##### 尽量减少排序量
- 结合 where 条件，减少扫描行数
- 分页查询时使用索引列作为排序列，避免文件排序

### 避免临时表
临时表通常出现在 group by、distinct、union或子查询场景
#### 原因分析
- 查询字段未被索引覆盖
- 聚合或者排序导致 MySQL必须建立临时表
- 派生表或子查询未优化
#### 优化方法
##### 使用索引覆盖
##### 查询重写
- 子查询改为 join
- 提前聚合或者筛选
##### 合理分页


### 覆盖索引的构建
#### 原理
#### 示例
#### 优化建议

