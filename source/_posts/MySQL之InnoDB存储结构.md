---
title: MySQL之InnoDB存储结构
categories:
  - 数据库
tags: MySQL
abbrlink: ffd4a644
date: 2023-10-14 19:27:12
---
# MySQL之InnoDB存储结构

本文转载于[MySQL - InnoDB - 听雨危楼 - 博客园 (cnblogs.com)](https://www.cnblogs.com/Neeo/articles/13883976.html)

## InnoDB逻辑存储结构

### 表空间（table space）

在InnoDB存储引擎中，所有数据都存放在表空间(table space)中，表空间由段(segment)、区(extent)、页(page)、行（Row）组成，它们的关系如下图：
[![img](https://img2020.cnblogs.com/blog/1168165/202105/1168165-20210512162144068-1806137285.png)](https://img2020.cnblogs.com/blog/1168165/202105/1168165-20210512162144068-1806137285.png)

### 段（segment）
表空间由各个段组成，常见的段类型有：数据段、索引段、回滚段。
**由于InnoDB表采用的是聚簇索引，所以数据段可以看成是B+树的叶子节点，索引段可以看成是B+树的非索引节点。**

### 区（extend）
一个段由多个区组成，区由多个连续页组成，每个区的大小为1MB，默认情况下，每个页的大小为16KB：

```bash
mysql> select @@innodb_page_size;
+--------------------+
| @@innodb_page_size |
+--------------------+
|              16384 |
+--------------------+
1 row in set (0.00 sec)
```

即一个区中一共有64个连续页。用户可通过innodb_page_size参数设置每个页的大小。

**默认情况下，用户在创建一张InnoDB表后，该表对应的独立表空间文件为96KB，在每个段开始时会先用32个碎片页来存放数据，使用完这32个页后才是64个连续页的申请。**这么做是考虑到有些表的数据相对来说是比较少的，可以节省磁盘空间，因为申请64个页（即1个区）需要1MB空间。

### 页（page）
页是InnoDB磁盘管理的最小单位，默认大小为16KB。常见的页类型有：

- 数据页（B-tree Node）
- undo页（undo Log Page）
- 系统页（System Page）
- 事务数据页（Transaction system Page）
- 插入缓冲位图页（Insert Buffer Bitmap）
- 插入缓冲空闲列表页（Insert Buffer Free List）
- 未压缩的二进制大对象页（Uncompressed BLOB Page）
- 压缩的二进制大对象页（compressed BLOB Page）



### 行（Row）
InnoDB存储引擎将数据按行进行存放，每个页最多存放7992行记录（16KB除以2~200），InnoDB存储引擎提供了Compact、Redundant、Compressed、Dynamic四种格式来存放行记录数据，用户可通过命令`show table status like 'table_name'`来查看该属性。

其中Row_format就是行记录格式。





## InnoDB物理存储结构

最直观的物理存储结构，我们能看到的，就是MySQL的存储目录`data`：

```haskell
[root@cs ~]# ll /data/mysql
总用量 188468
-rw-r----- 1 mysql mysql       56 6月   8 09:25 auto.cnf
-rw-r----- 1 mysql mysql     7386 8月  17 17:06 cs.err
-rw-r----- 1 mysql mysql        4 9月  21 11:24 cs.pid
drwxr-x--- 2 mysql mysql       20 8月  17 18:00 data_test
-rw-r----- 1 mysql mysql      596 9月   1 03:11 ib_buffer_pool
-rw-r----- 1 mysql mysql 79691776 9月  22 16:35 ibdata1
-rw-r----- 1 mysql mysql 50331648 9月  22 16:35 ib_logfile0
-rw-r----- 1 mysql mysql 50331648 9月  15 16:25 ib_logfile1
-rw-r----- 1 mysql mysql 12582912 9月  22 16:35 ibtmp1
drwxr-x--- 2 mysql mysql     4096 6月   8 09:25 mysql
drwxr-x--- 2 mysql mysql     8192 6月   8 09:25 performance_schema
drwxr-x--- 2 mysql mysql       60 9月  17 21:22 pp
drwxr-x--- 2 mysql mysql      160 8月  26 00:02 school
drwxr-x--- 2 mysql mysql     8192 6月   8 09:25 sys
drwxr-x--- 2 mysql mysql     4096 9月  22 16:35 tt
drwxr-x--- 2 mysql mysql      172 9月  22 16:20 world
```

其中：

- `ib_logfile0 ~ ib_logfile1`：`REDO`日志文件，事务日志文件
- `ibdata1`：系统数据字典(统计信息)，`UNDO`表空间等数据
- `ibtmp1`：临时表空间磁盘位置，存储临时表

基于InnoDB的表都会建立：

- `frm`：存储表的字段信息
- `ibd`：表的数据行和索引

### 表空间

> https://dev.mysql.com/doc/refman/5.7/en/innodb-tablespace.html

MySQL在5.5版本(能够上生产环境，稳定的版本)才有了表空间(Table space)的概念，恰好这个时间节点是在被Oracle收购后.....

这里简单说说表空间。虽然MySQL将数据存储到段、区、页中，但是这其中还经历了一个逻辑层 ，物理层面的逻辑层！这里称之为表空间。

PS：一般提到表空间的概念，基本上都是说的是Oracle表空间，这里MySQL可能借鉴了......你有兴趣也可以参考[LVM](https://baike.baidu.com/item/LVM/6571177?fr=aladdin)的原理，都类似。

[![img](https://img2020.cnblogs.com/blog/1168165/202011/1168165-20201118110003189-1001840048.jpg)](https://img2020.cnblogs.com/blog/1168165/202011/1168165-20201118110003189-1001840048.jpg)

如上图左侧部分，如果MySQL将`ibd`文件直接从内存中写入到`sdb1`磁盘中，如果有一天该`sdb1`磁盘满了怎么办？一般来说可以将`sdb1`中的数据导入到磁盘空间更大的磁盘`sdb2`中，然后MySQL后续直接往`sdb2`中写入数据即可，直到有一天.........

MySQL为了解决这个问题，也可能是借鉴了Oracle.......在MySQL层和磁盘层中间再加上一层逻辑层，即表空间，MySQL将`ibd`文件都存储到`table space`层，后续的磁盘都可以动态的挂载到`table space`层..........

而在MySQL中的表空间，经过发展现在有共享表空间和独立表空间两个概念。

### 共享表空间

`Innodb`所有数据保存在一个单独的表空间里面，我们称这个表空间为共享表空间，毕竟所有表都在这个共享表空间里面嘛！
这个表空间可以由很多个文件组成，一个表可以跨多个文件存在，所以其大小限制不再是文件大小的限制，而是其自身的限制。从`Innodb`的官方文档中可以看到，其表空间的最大限制为64TB，也就是说，`Innodb`的单表限制基本上也在64TB左右了，当然这个大小是包括这个表的所有记录、索引和其他相关数据。

随着MySQL初始化时，默认的共享表空间也随之建立，即数据目录中的`ibdata1`文件(后续可以是多个)，并且初始就一个`ibdata1`文件，且文件的初始大小为12MB，然后随着数据量的推移，以64MB为单位增加。

可以通过以下参数来查看该`ibdata1`文件：

```sql
SELECT @@innodb_data_file_path;
-- SHOW VARIABLES LIKE 'innodb_data_file%';
SELECT @@innodb_autoextend_increment;
```

也可以`vim /etc/my.cnf`配置文件：

```ini
# ---------- 在默认的datadir目录下设置多个 ibdata 文件 ----------
[mysqld]
innodb_data_file_path=ibdata1:1G;ibdata2:12M:autoextend:max:512M    # 自增长到最大512MB
# innodb_data_file_path=ibdata1:1G;ibdata2:12M:autoextend	# 初始12M，自增长直到64TB
# 另外，自增长只针对后面的ibdata文件，上面的ibdata1就是写死的1GB大小了


# ---------- 在指定目录设置多个 ibdata 文件 ----------
innodb_data_home_dir= /data/mysql/mysql3307/data
innodb_data_file_path=ibdata1:1G;ibdata2:12M:autoextend      # 会在 innodb_data_home_dir 目录下设置多个 ibdata 文件


# ---------- 在指定目录设置多个 ibdata 文件 ----------
innodb_data_home_dir=        # innodb_data_home_dir的值保留为空，但该参数不能不写
innodb_data_file_path=ibdata1:12M;/data/mysql/mysql3307/data/ibdata2:12M:autoextend
# 如上例子会在 datadir 下创建 ibdata1；会在 /data/mysql/mysql3307/data/ 目录创建 ibdata2
```

还有另一种情况就是：当没有配置`innodb_data_file_path`时，默认`innodb_data_file_path = ibdata1:12M:autoextend`；当需要改为1G时，不能直接在配置文件把 ibdata1 改为 1G ；而应该再添加一个 `ibdata2:1G`，如`innodb_data_file_path = ibdata1:12M;ibdata2:1G:autoextend`。

另外，一般，对于共享表空间的设置，在MySQL初始化之前就设计好了，那么在MySQL初始化的时候，按照你的设置，就自动的建立好相应的文件了。

MySQL5.6版本，共享空间虽然得以保留，但也只用来存储：数据字典、`UNDO`、临时表

而在5.7版本，临时表被独立出去了....

8.0版本，`UNDO`也被独立出去了.....

点击以下链接查看架构演示：

> [5.6 InnoDB架构](https://dev.mysql.com/doc/refman/5.6/en/innodb-architecture.html)
> [5.7 Innodb架构](https://dev.mysql.com/doc/refman/5.7/en/innodb-architecture.html)
> [8.0 InnoDB架构](https://dev.mysql.com/doc/refman/8.0/en/innodb-architecture.html)

### 独立表空间

从5.6开始，默认表空间不再使用共享表空间，替换为独立表空间，主要存储的是用户数据。

存储特点：每个表都有自己的表空间，表空间内存放着`ibd`文件，`idb`文件被称之为表空间的数据文件，主要用来存储数据行以及索引信息；而基本的表结构和元数据存储在`frm`文件中。

除了上述的`frm`和`ibd`文件之外，还搭配的有：

- `Redo Log`，重做日志。
- `Undo Log`，存储在共享表空间中，回滚日志。
- `ibtmp`，临时表，存储在`JOIN`和`UNION`操作产生的临时数据，用完自动删除。

**独立表空间/共享表空间切换**

我们可以对表的元数据、数据、索引信息进行单独管理，那么也能单独对表空间进行管理，比如设置表使用共享表空间。

通过`innodb_file_per_table`参数来控制MySQL对表使用共享表空间还是独立表空间，而默认值`1`表示使用独立表空间(一个表就是一个独立的`idb`文件)，而改成`0`就是使用共享表空间。

```sql
mysql> select @@innodb_file_per_table;
+-------------------------+
| @@innodb_file_per_table |
+-------------------------+
|                       1 |
+-------------------------+
1 row in set (0.00 sec)
```

现在我们测试一下，将`innodb_file_per_table`的值改为`0`，然后新建一张表(修改操作对原来建立的表没有影响，只会影响修改值后新建的表)：

```sql
mysql> set global innodb_file_per_table=0;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@innodb_file_per_table;
+-------------------------+
| @@innodb_file_per_table |
+-------------------------+
|                       0 |
+-------------------------+
1 row in set (0.00 sec)

mysql> create table tt.t6(id int);
Query OK, 0 rows affected (0.03 sec)
```

现在，我们来看磁盘上的关于`tt`数据库下`t6`表的物理存储结构：

```kotlin
[root@cs ~]# ll /data/mysql/tt/t6*
-rw-r----- 1 mysql mysql 8556 10月 20 16:59 /data/mysql/tt/t6.frm
[root@cs ~]# ^C
[root@cs ~]# ll /data/mysql/tt/t5*       
-rw-r----- 1 mysql mysql  8608 9月  29 10:14 /data/mysql/tt/t5.frm
-rw-r----- 1 mysql mysql 98304 9月  29 10:16 /data/mysql/tt/t5.ibd
```

上述结果中，`t5`表是之前建立的表，它有`ibd`和`frm`两个文件分别存储数据、索引和元数据信息；而`innodb_file_per_table=0`后创建的`t6`表，它的物理存储结构中，就只剩下了`frm`文件，用来存储元数据信息，而数据和索引信息不在存储自己的独立表空间中，即自己的`idb`文件中了，而是存储到了共享表空间中了(`idbata`)。