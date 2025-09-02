# SQL
- [SQL](#sql)
  - [In \& Exists](#in--exists)
    - [in 语句：只执行一次](#in-语句只执行一次)
    - [exists 语句：执行n次（外表行数）](#exists-语句执行n次外表行数)
    - [区别和应用场景](#区别和应用场景)
    - [not in 和 not exists](#not-in-和-not-exists)

## In & Exists

in 语法为：
```SQL
select *
from table_name
where col_name in (value1, value2,...);
```
- in 操作符允许在 where 子句中规定多个值。
- in 查询相当于多个 or 条件的叠加，比较好理解。
- in 查询就是先将子查询条件的记录全都查出来。
- in 查询的子条件返回结果必须只有一个字段。

exists 语法为：
```SQL
select *
from table_a a
where exists (select 1 from table_b b where b.id = b.id);
exists 对外表用 loop 逐条查询，每次查询都会查看 exists 的条件语句。
```
- 当 exists 里的条件语句能够返回记录行时（无论返回多少记录行，只要能返回），条件就为真，返回当前 loop 到的这条记录。
- 反之如果 exists 里的条件语句不能返回记录行，则当前 loop 到的这条记录被丢弃。
- exists 的条件就像一个 bool 条件，当能返回结果集则为 true，不能返回结果集则为 false。
- 当子查询为 select NULL 时， mysql 仍然认为它是 True。

### in 语句：只执行一次

确定给定的值是否与子查询或列表中的值相匹配。

in 在查询的时候，首先查询子查询的表，然后将内表和外表做一个笛卡尔积，然后按照条件进行筛选。

所以相对内表比较小的时候，in 的速度较快。

### exists 语句：执行n次（外表行数）

指定一个子查询，检测行的存在。

遍历循环外表，检查外表中的记录有没有和内表的的数据一致的。

匹配得上就放入结果集。

### 区别和应用场景

in 和 exists 的区别： 如果子查询得出的结果集记录较少，主查询中的表较大且又有索引时应该用 in， 反之如果外层的主查询记录较少，子查询中的表大，又有索引时使用 exists。

其实我们区分 in 和 exists 主要是造成了驱动顺序的改变（这是性能变化的关键），如果是 exists，那么以外层表为驱动表，先被访问，如果是 in ，那么先执行子查询，所以我们会以驱动表的快速返回为目标，那么就会考虑到索引及结果集的关系 ，另外 in 是不对 NULL 进行处理。

in 是把外表和内表作 hash 连接，而 exists 是对外表作 loop 循环，每次 loop 循环再对内表进行查询。一直以来认为 exists 比 in 效率高的说法是不准确的。

> In是内表驱动外表，Exists是外表驱动内表

### not in 和 not exists

如果查询语句使用 not in，那么内外表都进行全表扫描，没有用到索引；

而 not exists 的子查询依然能用到表上的索引。所以无论那个表大，用 not exists 都比not in 要快。



