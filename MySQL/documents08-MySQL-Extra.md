# Extra
- [Extra](#extra)
  - [系统参数](#系统参数)
  - [InnoDB参数](#innodb参数)
  - [binlog参数](#binlog参数)
  - [如何判断当前数据库的内存是否已经瓶颈了](#如何判断当前数据库的内存是否已经瓶颈了)

## 系统参数

```ini
max_connections=3000
max_user_connections=2980
interactive_timeout=300
sort_buffer_size=4M
join_buffer_size=4M
```

## InnoDB参数
```ini
# InnoDB线程并发数，默认0，表示不受限制，若要设置，最好与服务器CPU核心数相同，或者服务器核心数的2倍
# 如果超过并发数，则需要排队
# 这个值不宜太大，否则锁争用严重
innodb_thread_concurrency=64

# InnoDB存储引擎BufferPool缓存大小，一般为物理内存的60%～70%
# **内存大小直接反应数据库的性能**
innodb_buffer_pool_size=40G

# 行锁定时间，默认50s，根据公司业务而定，没有标准值
innodb_lock_wait_timeout=10
```

## binlog参数
```ini
# write和fsync的时机，可以由参数sync_binlog控制，默认是1。
# 为0的时候，表示每次提交事务都只write，由系统自行判断什么时候执行fsync。虽然性能得到提升，但是机器宕机，page cache里面的 binlog 会丢失。
# 为了安全起见，可以设置为1，表示每次提交事务都会执行fsync，就如同 redo log 日志刷盘流程 一样。
# 最后还有一种折中方式，可以设置为N(N>1)，表示每次提交事务都write，但累积N个事务后才fsync。
# 在出现 IO 瓶颈的场景里，将sync_binlog设置成一个比较大的值，可以提升性能。
# 同样的，如果机器宕机，会丢失最近N个事务的 binlog 日志。
sync_binlog=1
```

## 如何判断当前数据库的内存是否已经瓶颈了
```SQL
show global status like 'innodb%read%';
```

当前服务器的参数: 
- innodb_buffer_pool_reads: 表示从物理磁盘读取页的次数
- innodb_buffer_pool_read_ahead: 预读的次数
- innodb_buffer_pool_read_ahead_evicted: 预读的页，但是没有被读取就从缓冲池中被替换的页的数量，一般用来判断预读的效率
- innodb_buffer_pool_read_requests: 从缓冲池中读取页的次数
- innodb_data_read: 总共读入的字节数
- innodb_data_reads: 发起读取请求的次数，每次读取可能需要读取多个页

```
# 缓冲池读取次数 / 总读取次数
缓冲池命中率 = innodb_buffer_pool_read_requests / (innodb_buffer_pool_read_requests + innodb_buffer_pool_read_ahead + innodb_buffer_pool_reads)

# 读取字节数 / 读取次数
平均每次读取的字节数 = innodb_data_read / innodb_data_reads
```