---
title: MySQL之binlog
categories:
  - 数据库
tags: MySQL
abbrlink: b70a2522
date: 2023-06-15 23:45:31
---



# MySQL之binlog

MySQL的binlog是由MySQL Server生成的，记录了所有的DDL和DML语句（不会记录查询类的操作，比如 SELECT 和 SHOW），以事件形式记录，还包含语句执行所消耗的时间，并且是事务安全型的。



## 开启binlog

[MySQL的binlog日志 - 马丁传奇 - 博客园 (cnblogs.com)](https://www.cnblogs.com/martinzhang/p/3454358.html)

```shell
vi编辑打开mysql配置文件
    # vi /usr/local/mysql/etc/my.cnf
    在[mysqld] 区块
    设置/添加 log-bin=mysql-bin  确认是打开状态(值 mysql-bin 是日志的基本名或前缀名)；

    重启mysqld服务使配置生效
    # pkill mysqld
    # /usr/local/mysql/bin/mysqld_safe --user=mysql &
```



### 查看binlog是否开启

```sql
show variables like 'log_%';

+----------------------------------------+-----------+
| Variable_name                          | Value     |
+----------------------------------------+-----------+
| log_bin                                | ON        | ------> ON表示已经开启binlog日志
+----------------------------------------+-----------+
```



### 常用binlog命令

1. 查看所有binlog日志列表

```sql
show master logs;
```

2. 查看master状态，即最后(最新)一个binlog日志的编号名称，及其最后一个操作事件pos结束点(Position)值

```sql
show master status;
```

3. 刷新log日志，自此刻开始产生一个新编号的binlog日志文件

```sql
flush logs;
```

4. 重置(清空)所有binlog日志

```sql
reset master;
```

5. 查看binlog内容

```sql
show binlog events [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count];
```

可选项说明：

```
IN 'log_name'   指定要查询的binlog文件名(不指定就是第一个binlog文件)
FROM pos        指定从哪个pos起始点开始查起(不指定就是从整个文件首个pos点开始算)
LIMIT [offset,] 偏移量(不指定就是0)
row_count       查询总条数(不指定就是所有行)
```

```sql
截取部分查询结果：
*************************** 20. row ***************************
Log_name: mysql-bin.000021  -----------------------------------------> 查询的binlog日志文件名
Pos: 11197 ----------------------------------------------------------> pos起始点:
Event_type: Query ---------------------------------------------------> 事件类型：Query
Server_id: 1 --------------------------------------------------------> 标识是由哪台服务器执行的
End_log_pos: 11308 --------------------------------------------------> pos结束点:11308(即：下行的pos起始点)
Info: use `zyyshop`; INSERT INTO `team2` VALUES (0,345,'asdf8er5') ---> 执行的sql语句
*************************** 21. row ***************************
Log_name: mysql-bin.000021
Pos: 11308 ----------------------------------------------------------> pos起始点:11308(即：上行的pos结束点)
Event_type: Query
Server_id: 1
End_log_pos: 11417
Info: use `zyyshop`; /*!40000 ALTER TABLE `team2` ENABLE KEYS */
*************************** 22. row ***************************
Log_name: mysql-bin.000021
Pos: 11417
Event_type: Query
Server_id: 1
End_log_pos: 11510
Info: use `zyyshop`; DROP TABLE IF EXISTS `type`
```



## binlog的格式

[2 万字 + 30 张图 ｜ 细聊 MySQL undo log、redo log、binlog 有什么用？ - 小林coding - 博客园 (cnblogs.com)](https://www.cnblogs.com/xiaolincoding/p/16396502.html)

### STATEMENT

每一条修改数据的 SQL 都会被记录到 binlog 中（相当于记录了逻辑操作，所以针对这种格式， binlog 可以称为逻辑日志），主从复制中 slave 端再根据 SQL 语句重现。但 STATEMENT 有动态函数的问题，比如你用了 uuid 或者 now 这些函数，你在主库上执行的结果并不是你在从库执行的结果，这种随时在变的函数会导致复制的数据不一致。

### ROW

记录行数据最终被修改成什么样了（这种格式的日志，就不能称为逻辑日志了），不会出现 STATEMENT 下动态函数的问题。但 ROW 的缺点是每行数据的变化结果都会被记录，比如执行批量 update 语句，更新多少行数据就会产生多少条记录，使 binlog 文件过大，而在 STATEMENT 格式下只会记录一个 update 语句而已。

###  MIXED

包含了 STATEMENT 和 ROW 模式，它会根据不同的情况自动使用 ROW 模式和 STATEMENT 模式。





## binlog刷盘

事务执行过程中，先把日志写到 binlog cache（Server 层的 cache），事务提交的时候，再把 binlog cache 写到 binlog 文件中。

MySQL 给 binlog cache 分配了一片内存，每个线程一个，参数 binlog_cache_size 用于控制单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。

在事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 文件中，并清空 binlog cache。如下图：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/binlogcache.drawio.png)

虽然每个线程有自己 binlog cache，但是最终都写到同一个 binlog 文件：

- 图中的 write，指的就是指把日志写入到 binlog 文件，但是并没有把数据持久化到磁盘，因为数据还缓存在文件系统的 page cache 里，write 的写入速度还是比较快的，因为不涉及磁盘 I/O。
- 图中的 fsync，才是将数据持久化到磁盘的操作，这里就会涉及磁盘 I/O，所以频繁的 fsync 会导致磁盘的 I/O 升高。

MySQL提供一个 sync_binlog 参数来控制数据库的 binlog 刷到磁盘上的频率：

- sync_binlog = 0 的时候，表示每次提交事务都只 write，不 fsync，后续交由操作系统决定何时将数据持久化到磁盘；
- sync_binlog = 1 的时候，表示每次提交事务都会 write，然后马上执行 fsync；
- sync_binlog =N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。

在MySQL中系统默认的设置是 sync_binlog = 0，也就是不做任何强制性的磁盘刷新指令，这时候的性能是最好的，但是风险也是最大的。因为一旦主机发生异常重启，在 binlog cache 中的所有 binlog 日志都会被丢失。

而当 sync_binlog 设置为 1 的时候，是最安全但是性能损耗最大的设置。因为当设置为1的时候，即使主机发生异常重启，也最多丢失 binlog cache 中未完成的一个事务，对实际数据没有任何实质性影响，就是对写入性能影响太大。

如果能容少量事务的 binlog 日志丢失的风险，为了提高写入的性能，一般会 sync_binlog 设置为 100~1000 中的某个数值。

### **组提交**

**MySQL 引入了 binlog 组提交（group commit）机制，当有多个事务提交的时候，会将多个 binlog 刷盘操作合并成一个，从而减少磁盘 I/O 的次数**，如果说 10 个事务依次排队刷盘的时间成本是 10，那么将这 10 个事务一次性一起刷盘的时间成本则近似于 1。

引入了组提交机制后，prepare 阶段不变，只针对 commit 阶段，将 commit 阶段拆分为三个过程：

- **flush 阶段**：多个事务按进入的顺序将 binlog 从 cache 写入文件（不刷盘）；
- **sync 阶段**：对 binlog 文件做 fsync 操作（多个事务的 binlog 合并一次刷盘）；
- **commit 阶段**：各个事务按顺序做 InnoDB commit 操作；

上面的**每个阶段都有一个队列**，每个阶段有锁进行保护，因此保证了事务写入的顺序，第一个进入队列的事务会成为 leader，leader领导所在队列的所有事务，全权负责整队的操作，完成后通知队内其他事务操作结束。

![每个阶段都有一个队列](http://keithlan.github.io/image/mysql_innodb_arch/commit_4.png)每个阶段都有一个队列

对每个阶段引入了队列后，锁就只针对每个队列进行保护，不再锁住提交事务的整个过程，可以看的出来，**锁粒度减小了，这样就使得多个阶段可以并发执行，从而提升效率**。
