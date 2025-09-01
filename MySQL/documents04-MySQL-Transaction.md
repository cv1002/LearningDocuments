# Transaction
- [Transaction](#transaction)
  - [什么是Transaction/事务](#什么是transaction事务)
  - [MVCC机制](#mvcc机制)
    - [脏读、幻读、不可重复读](#脏读幻读不可重复读)
  - [ReadView机制](#readview机制)
    - [快照读和当前读的区别](#快照读和当前读的区别)
    - [ReadView机制如何实现MVCC](#readview机制如何实现mvcc)
      - [ReadView](#readview)
  - [事务ID](#事务id)

## 什么是Transaction/事务
事务是一组操作，这一组操作要么全都成功要么全都失败。事务是为了保证数据最终的一致性。
- 事务的ACID特性：
- A Atomicity 原子性。事务操作要么同时成功、要么同时失败。原子性由undo log实现。
- C Consistency 一致性。使用事务的最终目的，由其他三个特性以及业务代码的正确逻辑来实现。
- I Isolation 隔离性。在事务并发执行时，他们内部的操作不能互相干扰，隔离性由MySQL的各种锁以及MVCC机制来实现。
- D Durability 持久性。一旦提交了事务，他对数据库的改变就应该是永久性的。持久性由redo log来实现。

## MVCC机制
- RU Read Uncommitted 读未提交
- RC Read Commited 读已提交
- RR Repeatable Read 可重复读
- Serializable 串行

> - RC解决脏读
> - RR解决不可重复读

### 脏读、幻读、不可重复读
- 脏读
  - 读到别的事务还没提交的数据
  - 隔离级别改成RC或更高可以解决
- 不可重复读
  - 读到别的事务修改好的数据
  - 隔离级别改成RR或更高可以解决
- 幻读
  - 读到别的事务新添加的数据
  - 隔离级别改成RR能解决一部份，通过间隙锁
  - 隔离级别改成Serializable可以解决


## ReadView机制
### 快照读和当前读的区别
快照读又叫普通读，也就是利用MVCC机制读取快照中的数据。不加锁的简单的SELECT 都属于快照读，比如这样：
```SQL
SELECT * FROM user WHERE ...
```
- 快照读是基于MVCC实现的，提高了并发的性能，降低开销
- 大部分业务代码中的读取都属于快照读

当前读读取的是记录的最新版本，读取时会对读取的记录进行加锁, 其他事务就有可能阻塞。加锁的 SELECT，或者对数据进行增删改都会进行当前读。比如：
```SQL
SELECT * FROM user LOCK IN SHARE MODE; # 共享锁
SELECT * FROM user FOR UPDATE; # 排他锁
INSERT INTO user values ... # 排他锁
DELETE FROM user WHERE ... # 排他锁
UPDATE user SET ... # 排他锁
```
update、delete、insert语句虽然没有select, 但是它们也会先进行读取，而且只能读取最新版本。

### ReadView机制如何实现MVCC
MySQL innoDB引擎支持一条数据保存多个历史版本。通过undo log实现。
> Undo Log保存了数据的各个历史版本，用于数据的回滚，保证事务的一致性。
#### ReadView
ReadView是事务在使用MVCC机制进行快照读操作时产生的一致性视图。

- READ COMMITTED 语句级快照，在每一次进行普通SELECT操作前都会生成一个ReadView。
- REPEATABLE READ 事务级快照，只在第一次进行普通SELECT操作前生成一个ReadView，之后的查询操作都重复使用这个ReadView就好了。

## 事务ID
事务ID并不是开启事务时就产生了，只有执行第一次修改操作或加排他锁操作的语句，事务才会真正启动，才会向MySQL申请真正的事务ID，MySQL内部严格按照事务的启动顺序来分派事务ID。

查询操作并不算事务的启动，查询操作会产生一个trx_id，但这并不是真正的事务ID，而是当前trx变量的地址加上2的48次方得来的。

一旦事务真正启动，便会真正分配一个事务ID。



