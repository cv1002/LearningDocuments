# Log
- [Log](#log)
  - [WAL机制](#wal机制)
  - [redolog \& binlog \& undolog](#redolog--binlog--undolog)
    - [redolog](#redolog)
      - [redolog 日志文件组](#redolog-日志文件组)
    - [binlog](#binlog)

## WAL机制
WAL: Write Ahead Log 预写日志，是数据库系统中常见的一种手段，用于保证数据操作的原子性和持久性。

在计算机科学中，「预写式日志」（Write-ahead logging，缩写 WAL）是关系数据库系统中用于提供原子性和持久性（ACID 属性中的两个）的一系列技术。在使用 WAL 的系统中，所有的修改在提交之前都要先写入 log 文件中。

log 文件中通常包括 redo 和 undo 信息。这样做的目的可以通过一个例子来说明。假设一个程序在执行某些操作的过程中机器掉电了。在重新启动时，程序可能需要知道当时执行的操作是成功了还是部分成功或者是失败了。如果使用了 WAL，程序就可以检查 log 文件，并对突然掉电时计划执行的操作内容跟实际上执行的操作内容进行比较。在这个比较的基础上，程序就可以决定是撤销已做的操作还是继续完成已做的操作，或者是保持原样。

WAL 允许用 in-place 方式更新数据库。另一种用来实现原子更新的方法是 shadow paging，它并不是 in-place 方式。用 in-place 方式做更新的主要优点是减少索引和块列表的修改。ARIES 是 WAL 系列技术常用的算法。在文件系统中，WAL 通常称为 journaling。PostgreSQL 也是用 WAL 来提供 point-in-time 恢复和数据库复制特性。

WAL 机制的原理：
- 「修改并不直接写入到数据库文件中，而是写入到另外一个称为 WAL 的文件中；如果事务失败，WAL 中的记录会被忽略，撤销修改；如果事务成功，它将在随后的某个时间被写回到数据库文件中，提交修改。

WAL 的优点
1. 读和写可以完全地并发执行，不会互相阻塞（但是写之间仍然不能并发）。
2. WAL 在大多数情况下，拥有更好的性能（因为无需每次写入时都要写两个文件）。
3. 磁盘 I/O 行为更容易被预测。
4. 使用更少的 fsync()操作，减少系统脆弱的问题。

## redolog & binlog & undolog
### redolog
redo log（重做日志）是 InnoDB 存储引擎独有的，它让 MySQL 拥有了崩溃恢复能力。比如 MySQL 实例挂了或宕机了，重启时，InnoDB 存储引擎会使用 redo log 恢复数据，保证数据的持久性与完整性。

![RedoLog](assets/doc06/redolog.png)

redo log 支持顺序写，而InnoDB数据表文件ibd文件不支持顺序写，所以redo log写入速度比ibd文件快。因为顺序IO性能远高于随机IO。

数据在MySQL中存储是以页为单位，事务中的数据可能遍布在不同的页中，如果直接写入到对应的页中，是随机IO写入。

而redo log是通过顺序IO"追加"的方式写入到文件末尾，而且写入的内容也是物理日志，比如比如，某个事务将系统表空间中第10号页面中偏移量为 100 处的那个字节的值 1 改成 2等信息，日志占用空间也很小。

innodb_flush_log_at_trx_commit：这个参数控制redo log的写入策略，有三种可能取值
- 设置为0 表示每次提交事务时都只是把 redo log 留在redo log buffer中，数据库宕机可能丢失数据
- 设置为1 表示每次事务提交时都将redo log直接持久化到磁盘，数据最安全，不会因为数据库宕机丢失数据，但是效率稍微差一些，线上系统推荐这个设置
- 设置为2 表示每次事务提交时都只是把redo log写到操作系统的缓存page cache里，这种情况如果数据库宕机时不会丢失数据的，但是操作系统如果宕机了，page cache里的数据还没来得及写入磁盘文件的话就会丢失数据

InnoDB有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用操作系统函数 write 写到文件系统的 page cache，然后调用操作系统的 fsync 持久化到磁盘文件。

#### redolog 日志文件组

- 硬盘上存储的 redo log 日志文件不只一个，而是以一个日志文件组的形式出现的，每个的redo日志文件大小都是一样的。
- 比如可以配置为一组4个文件，每个文件的大小是 1GB，整个 redo log 日志文件组可以记录4G的内容。
- 它采用的是环形数组形式，从头开始写，写到末尾又回到头循环写，如下图所示。

![RedoLogFiles](assets/doc06/redolog-files.png)

- 在这个日志文件组中还有两个重要的属性，分别是 write pos、checkpoint
  - write pos 是当前记录的位置，一边写一边后移
  - checkpoint 是当前要擦除的位置，也是往后推移
- 每次刷盘 redo log 记录到日志文件组中，write pos 位置就会后移更新
- 每次 MySQL 加载日志文件组恢复数据时，会清空加载过的 redo log 记录，并把 checkpoint 后移更新
- write pos 和 checkpoint 之间的还空着的部分可以用来写入新的 redo log 记录

![RedoLogWrite](assets/doc06/redolog-write.png)

- 如果 write pos 追上 checkpoint ，表示日志文件组满了，这时候不能再写入新的 redo log 记录，MySQL 得停下来，清空一些记录，把 checkpoint 推进一下

![alt text](assets/doc06/redolog-write-full.png)

> MySQL 8.0.30 之前可以通过 innodb_log_files_in_group 和 innodb_log_file_size 配置日志文件组的文件数和文件大小
> 
> 但在 MySQL 8.0.30 及之后的版本中，这两个变量已被废弃，即使被指定也是用来计算 innodb_redo_log_capacity 的值。
> 
> 而日志文件组的文件数则固定为 32，文件大小则为 innodb_redo_log_capacity / 32 。

### binlog


