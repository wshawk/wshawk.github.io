---
title: MySQL之InnoDB Buffer Pool
categories:
  - 数据库
tags: MySQL
date: 2022-11-14 22:45:31
---

# MySQL之InnoDB Buffer Pool

## 简介

在`MySQL` 服务器启动的时候向操作系统申请了**一片连续的内存**，这块内存就叫做 `Buffer Pool `。

默认情况下 `Buffer Pool` 只有 `128M` 大小。

可以在启动服务器的时候配置 `innodb_buffer_pool_size` 参数的值，来设置`Buffer Pool`的大小，单位是**字节。**需要注意的是， `Buffer Pool` 也不能太小，最小值为 `5M` (当小于该值时会自动设置成 `5M `)。

本文内容均来自《MySQL是怎样运行的：从根儿上理解MySQL》

## 内部组成

`Buffer Pool `中默认的缓存页大小和在磁盘上默认的页大小是一样的，都是 `16KB` 。

为了更好的管理这些在 `Buffer Pool` 中的缓存页，衍生了控制信息，主要包括该页所属的表空间编号、页号、缓存页在 `Buffer Pool` 中的地址、链表节点信息、一些锁信息以及 `LSN` 信息。

每个页对应的控制信息占用的一块内存可以称为一个**控制块**，控制块和缓存页是一 一对应的。

每个缓存页的控制信息的长度是固定大小的，在`MySQL5.7.21`这个版本中，每个控制块占用的大小是`808`字节。

设置的`innodb_buffer_pool_size`并不包含这部分控制块占用的内存空间大小，也就是说**实际上`Buffer Pool`的大小是超过预设的值的**，一般是超过`5%`左右。



## free链表

`free`链表负责记录`Buffer Pool`中，每一个节点都代表一个空闲的缓存页，在将磁盘中的页加载到 `Buffer Pool `时，会从 `free`链表中寻找空闲的缓存页。

**这里的链表都会有一个基节点，记录着链表中的节点数量，头节点的指针，尾节点的指针等信息。**链表的基节点占用的内存空间并不包含在为 `Buffer Pool` 申请的一大片连续内存空间之内，而是单独申请的一块内存空间。

每当需要从磁盘中加载一个页到`Buffer Pool` 中时，就从 free链表 中 取一个空闲的缓存页，并且把该缓存页对应的 控制块 的信息填上（就是该页所在的表空间、页号之类的信 息），然后把该缓存页对应的` free`链表 节点从链表中移除，表示该缓存页已经被使用了。



## 缓存页的哈希处理

> 如何判断数据页有没有被缓存到`Buffer Pool`?

以用 **表空间号 + 页号** 作为 `key` ， **缓存页** 作为 `value` 创建一个哈希表，在需要访问某个页的数据 时，先从哈希表中根据 表空间号 + 页号 看看有没有对应的缓存页，如果有，直接使用该缓存页就好，如果没 有，那就从 free链表 中选一个空闲的缓存页，然后把磁盘中对应的页加载到该缓存页的位置。

## flush链表

修改了 `Buffer Pool`中某个缓存页的数据，那它就和磁盘上的页不一致了，这样的缓存页也被称为 **脏 页**。

修改过的缓存页对应的控制块都会作为一个节点加入到一个链表中，因为这个**链表节点对应的缓存页都是需要被刷新到磁盘上的**， 所以也叫 `flush`链表 。



## lru链表

当需要缓存的页占用的内存大小超过了 `Buffer Pool` 大小，也就 是` free`链表 中已经没有多余的空闲缓存页的时候，就需要把某些旧的 缓存页从 `Buffer Pool` 中移除，然后再把新的页放进来。

可以再创建一个链表，由于这个链表是为了按照**最近最少使用的原则**去淘汰缓存页 的，所以这个链表可以被称为 `lru`链表 （英文全称：Least Recently Used）。



### 简单的lru链表

简单的`lru`链表逻辑如下：

- 如果该页不在 `Buffer Pool` 中，在把该页从磁盘加载到 `Buffer Pool `中的缓存页时，就把该缓存页对应的 控制块 作为节点塞到链表的头部。

- 如果该页已经缓存在 `Buffer Pool` 中，则直接把该页对应的 控制块 移动到 `lru`链表 的头部。



### 划分区域的lru链表

`InnoDB` 提供了**预读**服务 ，就是 `InnoDB` 认为执行当前的请求可能之后会读取某些页面，就预先把它们加载到 `Buffer Pool` 中。

预读本来是个好事儿，如果预读到 `Buffer Pool` 中的页成功的被使用到，那就可以极大的提高语句执 行的效率。可是如果用不到呢？这些预读的页都会放到 `lru` 链表的头部，但是如果此时 `Buffer Pool` 的 容量不太大而且很多预读的页面都没有用到的话，这就会导致处在 `lru` 链表 尾部的一些缓存页会很快的 被淘汰掉，也就是所谓的 劣币驱逐良币 ，会**大大降低缓存命中率**。

全表扫描，当遇到全表扫描时，会访问表中所有的数据页，会将所有数据页加载到 `Buffer Pool `中。

总结：

- 加载到 `Buffer Pool` 中的页不一定被用到。 

- 如果非常多的使用频率偏低的页被同时加载到 `Buffer Pool` 时，可能会把那些使用频率非常高的页从 `Buffer Pool` 中淘汰掉。



**为了提高缓存的命中率，`InnoDB`将`lru` 链表按照一定比例分成两截。**

- 一部分存储使用频率非常高的缓存页，所以这一部分链表也叫做 热数据 ，或者称 `young`区域 。

-  另一部分存储使用频率不是很高的缓存页，所以这一部分链表也叫做 冷数据 ，或者称 `old`区域 。



可以查看系统变量 `innodb_old_blocks_pct` 的值来确定**` old` 区域 在`lru`链表 中所占的比例**。

```sql
SHOW VARIABLES LIKE 'innodb_old_blocks_pct';
```



- 启动时修改` innodb_old_blocks_pct `参数
- 运行时修改`SET GLOBAL innodb_old_blocks_pct = xx`;

> ps：这个系统变量属于全局变量 ，一经修改，会对所有客户端生效



优化后：

当磁盘上的某个页面在初次加载到`Buffer Pool`中的某个缓存页时，该缓存页对应 的控制块会被放到`old`区域的头部。

在对某个处在 `old `区域的缓存页进行第一次访问时就在它对应的控制块中 记录下来这个访问时间，如果后续的访问时间与第一次访问的时间在某个时间间隔内，那么该页面就不会被 从`old`区域移动到`young`区域的头部，否则将它移动到`young`区域的头部。

间隔时间是由系统变量 `innodb_old_blocks_time `控制，默认值是 `1000 `，单位是`毫秒`。

查看：

```sql
SHOW VARIABLES LIKE 'innodb_old_blocks_time';
```



### 更进一步优化

对于 `young` 区域的缓存页来说，我们每次访问一个缓存页就要把它移动到 `lru`链表 的头部，开销也比较大。

在`young` 区域的缓存页都是热点数据，也就是可能被经常访 问的，这样频繁的对  `lru`链表进行节点移动操作也不太好。

优化策略如下：

只有被访问的缓存页位于 `young` 区域的 `1/4` 的后边，才会被移动到 `lru`链表 头部，这样就 可以降低调整  `lru`链表 的频率，从而提升性能（也就是说如果某个缓存页对应的节点在 `young` 区域的前`1/4` 中， 再次访问该缓存页时也不会将其移动到 `lru`链表头部）

优化的最终目标：**尽量高效的提高 Buffer Pool 的缓存命中率**。



## 其他的一些链表

- `unzip lru`链表： 用于管理解压页，
- `zip clean`链表：用于管理没有被解压的压缩页
- ...



## 刷新脏页到磁盘

后台有专门的线程每隔一段时间负责把脏页刷新到磁盘，这样可以不影响用户线程处理正常的请求。

主要有两种 刷新路径： 

- 从  `lru`链表 的冷数据中刷新一部分页面到磁盘。 后台线程会定时从 `lru`链表 尾部开始扫描一些页面，扫描的页面数量可以通过系统变量 `innodb_lru_scan_depth` 来指定，如果从里边儿发现脏页，会把它们刷新到磁盘。这种刷新页面的方式被称之为 `BUF_FLUSH_LRU `。 
- 从 `flush`链表 中刷新一部分页面到磁盘。 后台线程也会定时从 flush链表 中刷新一部分页面到磁盘，刷新的速率取决于当时系统是不是很繁忙。这种 刷新页面的方式被称之为 `BUF_FLUSH_LIST` 。 

有时候后台线程刷新脏页的进度比较慢，导致用户线程在准备加载一个磁盘页到 `Buffer Pool` 时没有可用的缓存 页，这时就会尝试看看 `lru`链表 尾部有没有可以直接释放掉的未修改页面，如果没有的话会不得不将 `lru`链表尾部的一个脏页同步刷新到磁盘（和磁盘交互是很慢的，这会降低处理用户请求的速度）。

这种刷新单个页面到磁 盘中的刷新方式被称之为 `BUF_FLUSH_SINGLE_PAGE `。 当然，有时候系统特别繁忙时，也可能出现用户线程批量的从 `flush`链表中刷新脏页的情况，很显然在处理用户请求过程中去刷新脏页是一种严重降低处理速度的行为（毕竟磁盘的速度满的要死），这属于一种迫不得已的情 况，不过这得放在后边唠叨 `redo` 日志的 `checkpoint` 时说了。

## 多个Buffer Pool实例

 `Buffer Pool` 本质是 `InnoDB` 向操作系统申请的一块连续的内存空间，在多线程环境下，访问 `Buffer Pool` 中的各种链表都需要加锁处理啥的，在 `Buffer Pool `特别大而且多线程并发访问特别高的情况下， 单一的 `Buffer Pool` 可能会影响请求的处理速度。所以在 `Buffer Pool` 特别大的时候，我们可以把它们拆分成若 干个小的 `Buffer Pool `，每个 `Buffer Pool `都称为一个 实例 ，它们都是独立的，在多线程并发访问时不会互相影响，从而提高并发能力。

可以在服务器启动的时候通过设置 `innodb_buffer_pool_instances` 的值来修改 `Buffer Pool` 实例的个数。

```sql
[server]
innodb_buffer_pool_instances = 2
```



 `Buffer Pool` 实例并不是创建的越多越好，分别管理各个 `Buffer Pool` 也是需要性能开销的，`InnoDB`规定：**当`innodb_buffer_pool_size`的值小于`1G`的时候设置多个实例是无效的，`InnoDB`会默认把 `innodb_buffer_pool_instances` 的值修改为`1`。**





## innodb_buffer_pool_chunk_size

在 `MySQL 5.7.5` 之前， `Buffer Pool `的大小只能在服务器启动时通过配置` innodb_buffer_pool_size `启动参数 来调整大小，在服务器运行过程中是不允许调整该值的。

在 `5.7.5 `以及之后的版本中支持 了在服务器运行过程中调整 `Buffer Pool `大小的功能，但是有一个问题，就是每次当我们要重新调整 `Buffer Pool` 大小时，都需要重新向操作系统申请一块连续的内存空间，然后将旧的 `Buffer Pool` 中的内容复制到这一 块新空间，这是极其耗时的。

后面的`MySQL`就不再一次性为某个 `Buffer Pool `实例向操作系统申请 一大片连续的内存空间，而是以一个所谓的 `chunk `为单位向操作系统申请空间。

也就是说一个 `Buffer Pool `实例 其实是由若干个 `chunk `组成的，一个 `chunk` 就代表一片连续的内存空间，里边儿包含了若干缓存页与其对应的控制块。

`chunk` 的大小是我们在启动操作 `MySQL` 服务器时通过 `innodb_buffer_pool_chunk_size` 启动参数指定的，它的默认值是 `134217728` ，也就是 `128M` 。不过需要注意的是，**`innodb_buffer_pool_chunk_size`的值只能在服务器启动时指 定，在服务器运行过程中是不可以修改的。**

`innodb_buffer_pool_chunk_size`的值并不包含缓存页对应的控制块的内存空间大小，所以实际上`InnoDB`向操作系统申请连续内存空间时，每个`chunk`的大小要比`innodb_buffer_pool_chunk_size `的值大一些，约`5%`。



## 配置Buffer Pool时的注意事项

- `innodb_buffer_pool_size` 必须是 `innodb_buffer_pool_chunk_size` * `innodb_buffer_pool_instances` 的倍数（这主要是想保证每一个 `Buffer Pool` 实例中包含的 `chunk`数量相同）。

`innodb_buffer_pool_size` 的值必须是 `2G` 或者 `2G` 的整数倍，如果指定的 `innodb_buffer_pool_size `大于 `2G `并且不是 `2G` 的整数倍，那么服务器会自动的把 `innodb_buffer_pool_size`的值调整为` 2G` 的整数倍。



- 如果在服务器启动时，`innodb_buffer_pool_chunk_size` * `innodb_buffer_pool_instances` 的值已经大 于 `innodb_buffer_pool_size` 的值，那么 `innodb_buffer_pool_chunk_size` 的值会被服务器自动设置为` innodb_buffer_pool_size` / `innodb_buffer_pool_instances` 的值。



## 查看Buffer Pool的状态信息

通过以下命令，可以查看关于 `InnoDB` 存储引擎运行过程 中的一些状态信息，其中就包括 `Buffer Pool `的一些信息。

```sql
 SHOW ENGINE INNODB STATUS;
 
 (...省略前边的许多状态)
----------------------
BUFFER POOL AND MEMORY
----------------------
Total memory allocated 13218349056;
Dictionary memory allocated 4014231
Buffer pool size 786432
Free buffers 8174
Database pages 710576
Old database pages 262143
Modified db pages 124941
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 6195930012, not young 78247510485
108.18 youngs/s, 226.15 non-youngs/s
Pages read 2748866728, created 29217873, written 4845680877
160.77 reads/s, 3.80 creates/s, 190.16 writes/s
Buffer pool hit rate 956 / 1000, young-making rate 30 / 1000 not 605 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 710576, unzip_LRU len: 118
I/O sum[134264]:cur[144], unzip sum[16]:cur[0]
--------------
(...省略后边的许多状态)
```

`Total memory allocated` ：代表 `Buffer Pool` 向操作系统申请的连续内存空间大小，包括全部控制块、缓 存页、以及碎片的大小。 

`Dictionary memory allocated` ：为数据字典信息分配的内存空间大小，注意这个内存空间和 Buffer Pool 没啥关系，不包括在 `Total memory allocated` 中。 

`Buffer pool size` ：代表该 `Buffer Pool` 可以容纳多少缓存 页 ，注意，**单位是页** ！ 

`Free buffers` ：代表当前 `Buffer Pool `还有多少空闲缓存页，也就是 `free`链表 中还有多少个节点。 

`Database pages` ：代表 `LRU `链表中的页的数量，包含 `young` 和 `old` 两个区域的节点数量。 

`Old database pages` ：代表` LRU` 链表 `old` 区域的节点数量。 

`Modified db pages` ：代表脏页数量，也就是 `flush`链表 中节点的数量。 

`Pending reads `：正在等待从磁盘上加载到 `Buffer Pool `中的页面数量。 当准备从磁盘中加载某个页面时，会先为这个页面在 `Buffer Pool `中分配一个缓存页以及它对应的控制块， 然后把这个控制块添加到 `LRU `的 `old` 区域的头部，但是这个时候真正的磁盘页并没有被加载进来， `Pending reads` 的值会跟着加1。 `Pending writes LRU` ：即将从 `LRU` 链表中刷新到磁盘中的页面数量。

 `Pending writes flush list` ：即将从 `flush` 链表中刷新到磁盘中的页面数量。

` Pending writes single page` ：即将以单个页面的形式刷新到磁盘中的页面数量。 

`Pages made young` ：代表` LRU` 链表中曾经从` old `区域移动到 `young `区域头部的节点数量。 这里需要注意，一个节点每次只有从 `old` 区域移动到` young` 区域头部时才会将 `Pages made young` 的值加` 1`，也就是说如果该节点本来就在` young` 区域，由于它符合在 `young` 区域`1/4`后边的要求，下一次访问这个页 面时也会将它移动到 `young `区域头部，但这个过程并不会导致` Pages made young `的值加1。

` Page made not young` ：在将` innodb_old_blocks_time` 设置的值大于`0`时，首次访问或者后续访问某个处 在 `old `区域的节点时由于不符合时间间隔的限制而不能将其移动到 `young` 区域头部时， `Page made not young` 的值会加`1`。 这里需要注意，对于处在 `young` 区域的节点，如果由于它在 `young `区域的`1/4`处而导致它没有被移动到 `young` 区域头部，这样的访问并不会将` Page made not young `的值加1。

 `youngs/s `：代表每秒从 `old` 区域被移动到 `young` 区域头部的节点数量。 

`non-youngs/s` ：代表每秒由于不满足时间限制而不能从 `old` 区域移动到 `young` 区域头部的节点数量。 

`Pages read 、 created 、 written `：代表读取，创建，写入了多少页。后边跟着读取、创建、写入的速 率。 

`Buffer pool hit rate` ：表示在过去某段时间，平均访问`1000`次页面，有多少次该页面已经被缓存到 `Buffer Pool` 了。 

`young-making rate` ：表示在过去某段时间，平均访问`1000`次页面，有多少次访问使页面移动到 `young` 区 域的头部了。 需要大家注意的一点是，这里统计的将页面移动到 `young` 区域的头部次数不仅仅包含从 `old` 区域移动到 `young` 区域头部的次数，还包括从 `young` 区域移动到 `young` 区域头部的次数（访问某个 `young` 区域的节 点，只要该节点在 `young` 区域的`1/4`处往后，就会把它移动到 `young` 区域的头部）。 

`not (young-making rate)` ：表示在过去某段时间，平均访问`1000`次页面，有多少次访问没有使页面移动 到 `young `区域的头部。 需要大家注意的一点是，这里统计的没有将页面移动到 `young` 区域的头部次数不仅仅包含因为设置了 `innodb_old_blocks_time` 系统变量而导致访问了 `old` 区域中的节点但没把它们移动到 `young` 区域的次数， 还包含因为该节点在 `young` 区域的前`1/4`处而没有被移动到 `young` 区域头部的次数。 

`LRU len` ：代表 `LRU`链表 中节点的数量。 

`unzip_LRU` ：代表 `unzip_LRU`链表 中节点的数量。

 `I/O sum` ：最近`50s`读取磁盘页的总数。 

`I/O cur `：现在正在读取的磁盘页数量。 

`I/O unzip sum` ：最近`50s`解压的页面数量。 

`I/O unzip cur` ：正在解压的页面数量。