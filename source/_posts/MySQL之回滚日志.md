---
title: MySQL之Undo Log
categories:
  - 数据库
tags: MySQL
abbrlink: ae3a4483
date: 2023-03-02 20:30:31
---
# MySQL之Undo Log

## 简介

`Undo Log`是`InnoDB`存储引擎生成的逻辑日志，实现了事务`ACID`特性中的**原子性**，主要作用于**事务回滚和`MVCC`**。

[详细分析MySQL事务日志(redo log和undo log) - 骏马金龙 - 博客园 (cnblogs.com)](https://www.cnblogs.com/f-ck-need-u/archive/2018/05/08/9010872.html)
[MySQL · 引擎特性 · 庖丁解InnoDB之UNDO LOG (taobao.org)](http://mysql.taobao.org/monthly/2021/10/01/)
[MySQL · 引擎特性 · InnoDB undo log 漫游 (taobao.org)](http://mysql.taobao.org/monthly/2015/04/01/)

## 基本概念

在数据修改的时候，不仅记录了`redo`，还记录了相对应的`undo`，如果因为某些原因导致事务失败或回滚了，可以借助该`undo`进行回滚。

`undo log`和`redo log`记录物理日志不一样，它是逻辑日志。**可以认为当`delete`一条记录时，`undo log`中会记录一条对应的`insert`记录，反之亦然，当`update`一条记录时，它记录一条对应相反的`update`记录。**

当执行`rollback`时，就可以从`undo log`中的逻辑记录读取到相应的内容并进行回滚。有时候应用到行版本控制的时候，也是通过`undo log`来实现的：当读取的某一行被其他事务锁定时，它可以从`undo log`中分析出该行记录以前的数据是什么，从而提供该行版本信息，让用户实现快照读。

**`undo log`是采用段(`segment`)的方式来记录的，每个`undo`操作在记录的时候占用一个`undo log segment`。**

另外，**`undo log`也会产生`redo log`，因为`undo log`也要实现持久性保护。**

> `ps`：`select`操作没有对数据行进行修改，不会生成`Undo Log`。



## 存储方式

**`undo log`是采用段(`segment`)的方式来记录的，每个`undo`操作在记录的时候占用一个`undo log segment`。**

一个`rollback segment`中有**1024**个`undo log segment`



> **版本 <  5.5**： 只支持1个`rollback segment`，这样就只能记录1024个`undo log segment`。
>
> **版本 >= MySQL 5.5**：可以支持128个`rollback segment`，即支持128*1024个`undo`操作，还可以通过变量` innodb_undo_logs `(5.6版本以前该变量是 `innodb_rollback_segments` )自定义多少个`rollback segment`，默认值为128。



存储位置：

> 默认存放在共享表空间中。
>
> 如果开启了 `innodb_file_per_table` ，将放在每个表的`.ibd`文件中。