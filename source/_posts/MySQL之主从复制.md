---
title: MySQL之主从复制
categories:
  - 数据库
tags: MySQL
abbrlink: ae3a4474
date: 2023-03-12 20:10:31
---
# MySQL之主从复制

[小白都能懂的Mysql主从复制原理（原理+实操） (qq.com)](https://mp.weixin.qq.com/s/7dkPnF88o64w-u6c1MXctw)

[MySQL 主从复制原理不再难 - rickiyang - 博客园 (cnblogs.com)](https://www.cnblogs.com/rickiyang/p/13856388.html)

##  概念

MySQL 主从复制是指数据可以从一个MySQL数据库服务器主节点复制到一个或多个从节点。

MySQL 默认采用异步复制方式，这样从节点不用一直访问主服务器来更新自己的数据，数据的更新可以在远程连接上进行，从节点可以复制主数据库中的所有数据库或者特定的数据库，或者特定的表。



## 作用

主从复制是MySQL高可用集群的基石，可以实现实时灾备、读写分离、高可用。



## 原理

MySQL的主从复制是基于binlog来实现的，因为binlog记录了数据库中所有数据的改动。

根据binlog的记录的数据格式不同，复制的数据也有区别，目前主要有以下三种：

- 基于行的复制：binlog为ROW格式，把改变的内容复制过去，而不是把命令在从服务器上执行一遍. 从mysql5.0开始支持
- 基于语句的复制：binlog为STATEMENT格式，在主服务器执行SQL语句，在从服务器执行同样语句。
- 混合类型的复制：binlog为MIXED格式，默认采用基于语句的复制，一旦发现基于语句的无法精确的复制时，就会采用基于行的复制。





前面说到主从复制是基于binlog来实现的，下面简要说说是如何实现的。



![](https://mmbiz.qpic.cn/mmbiz_png/IJUXwBNpKljjrOU752F5ibPrkKk9WrZ9tOCRmhCuk6I0vnDS8ydDW7J5rVYibrPf0iasgfia7L7tenkClcKoiczkjlw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Mysql的主从复制中主要有三个线程：`master（binlog dump thread）、slave（I/O thread 、SQL thread）`，Master一条线程和Slave中的两条线程。

`master（binlog dump thread）`主要负责Master库中有数据更新的时候，会按照`binlog`格式，将更新的事件类型写入到主库的`binlog`文件中。

并且，Master会创建`log dump`线程通知Slave主库中存在数据更新，这就是为什么主库的binlog日志一定要开启的原因。

`I/O thread`线程在Slave中创建，该线程用于请求Master，Master会返回binlog的名称以及当前数据更新的位置、binlog文件位置的副本。

然后，将`binlog`保存在 **「relay log（中继日志）」** 中，中继日志也是记录数据更新的信息。

SQL线程也是在Slave中创建的，当Slave检测到中继日志有更新，就会将更新的内容同步到Slave数据库中，这样就保证了主从的数据的同步。



复制策略主要有以下几种

- 异步复制：Master不用等待Slave回应就可以提交。
- 半同步复制：Master至少会等待一个Slave回应后提交。
- 同步复制：Master会等待所有的Slave都回应后才会提交，这个主从的同步的性能会严重的影响。
- 延迟复制：Slave要落后于Master指定的时间。