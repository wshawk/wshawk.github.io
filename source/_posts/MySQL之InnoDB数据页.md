---
title: MySQL之InnoDB数据页
categories:
  - 数据库
tags: MySQL
date: 2022-11-14 17:27:12
---

# MySQL之InnoDB数据页

页是`InnoDB`管理存储空间的基本单位，一个页的大小一般是 `16KB` 。

`InnoDB` 为了不同的目的而设计了许多种不同类型的页 ，下文主要讲述存储数据行的页，官方称之为**索引页**。

页 一般由以下七个部分组成。

- File Header (38字节) ： 文件头，存放页的一些通用信息
- Page Header (56字节)：页面头，存放数据页专有的一些信息
- Infimum + Superemum (26字节)：两个虚拟的行记录，分别指向当前页的最小记录和最大记录
- User Records (大小不确定，数据行信息)： 实际存储的行记录内容
- Free Space (大小不确定)：空闲空间，页中尚未使用的空间
- Page Directory (大小不确定)：页面目录，页中某些记录的相对位置
- File Tailer (8字节)：文件尾部，校验页是否完整



## Infimum、Superemum 

 `Infimum`记录（也就是最小记录）的下一条记录就是 本页中主键值最小的用户记录，

本页中主键值最大的用户记录的下一条记录就是 `Supremum`记录（也就是最大记录）



## Page Directory 

记录在页中按照主键值由小到大顺序串联成一个单链表。

如果我们想根据主键值查找页中的某条记录，就只能从前往后遍历，直到遍历到对应的记录为止，明显这种处理方式效率很低。

每一行的记录都有记录头，信息如下：

| 名称           | 大小（bit） | 说明                                                         |
| -------------- | ----------- | ------------------------------------------------------------ |
| 预留位1        | 1           | 没有使用                                                     |
| 预留位2        | 1           | 没有使用                                                     |
| `deleted_flag` | 1           | 记录删除标记                                                 |
| `min_rec_flag` | 1           | B+树非叶子节点的最小目录项标记                               |
| `n_owned`      | 4           | 同一页内同一组里最大的记录会记录组里的记录数量，其余记录该值为0 |
| `heap_no`      | 13          | 当前记录在页面堆里的相对位置                                 |
| `record_type`  | 3           | 记录类型。0: 普通记录, 1: `B+`树非叶子节点目录项记录, 2: `Infimum`记录, 3: `Supremum`记录. |
| `next_record`  | 16          | 下一条记录的相对位置                                         |

优化处理如下:

1. **将所有正常的记录（包括最大和最小记录，不包括标记为已删除的记录）划分为几个组**。 
2.  **每个组的最后一条记录（也就是组内最大的那条记录）的头信息中的 `n_owned` 属性表示该记录拥有多少条记 录，也就是该组内共有几条记录。** 
3. **将每个组的最后一条记录的地址偏移量(`next record`)单独提取出来按顺序存储到靠近 页 的尾部的地方，这个地方就是所 谓的 `Page Directory` ，也就是 页目录 。页面目录中的这些地址 偏移量被称为 槽 （英文名： `Slot` ），所以这个页面目录就是由 槽 组成的。**



**对于最小记录所在的分组只能有 1 条记录， 最大记录所在的分组拥有的记录条数只能在 1~8 条之间，剩下的分组中记录的条数范围只能在是 4~8 条之间**



分组步骤如下：

- **初始情况下一个数据页里只有最小记录和最大记录两条记录，它们分属于两个分组。** 
- **之后每插入一条记录，都会从 页目录 中找到主键值比本记录的主键值大并且差值最小的槽，然后把该槽对 应的记录的 n_owned 值加1，表示本组内又添加了一条记录，直到该组中的记录数等于8个。** 
- **在一个组中的记录数等于8个后再插入一条记录时，会将组中的记录拆分成两个组，一个组中4条记录，另一 个5条记录。这个过程会在 页目录 中新增一个 槽 来记录这个新增分组中最大的那条记录的偏移量。**



优化之后的查询过程如下：

1. **通过二分法确定该记录所在的槽，并找到该槽中主键值最小的那条记录。**
2. **通过记录的 `next_record` 属性遍历该槽所在的组中的各个记录。**



## Page Header



以下部分来源：[MySQL系列（4）— InnoDB数据页结构 - 掘金 (juejin.cn)](https://juejin.cn/post/6974225353371975693)



记录数据页中存储的记录的状态信息，比如本页中已经存储了多少条记录，第 一条记录的地址是什么，页目录中存储了多少个槽等等，这个部分占用固定的 56 个字节。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/997d4cc05f05453cb4919db54cfcdb6c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

- `PAGE_N_DIR_SLOTS`

页中的记录会按主键顺序分为多个组，每个组会对应到一个槽（`Slot`），`PAGE_N_DIR_SLOTS` 就记录了 `Page Directory` 中槽的数量。

- `PAGE_HEAP_TOP`

`PAGE_HEAP_TOP` 记录了 `Free Space` 的地址，这样就可以快速从 `Free Space` 分配空间到 `User Records `了。

- `PAGE_N_HEAP`

本页中的记录的数量，包括最小记录（`Infimum`）和最大记录（`Supremum`）以及标记为删除（`delete_mask=1`）的记录。

- `PAGE_FREE`

已删除的记录会通过 `next_record`连成一个单链表，这个单链表中的记录空间可以被重新利用，`PAGE_FREE` 指向第一个标记为删除的记录地址，就是单链表的头节点。

- `PAGE_GARBAGE`

标记为已删除的记录占用的总字节数。

- `PAGE_N_RECS`

本页中记录的数量，不包括最小记录和最大记录以及被标记为删除的记录，注意和 `PAGE_N_HEAP` 的区别。



## File Header

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/afbeb375b5134bf58a3208c4456d692d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

- `FIL_PAGE_SPACE_OR_CHKSUM`

这个代表当前页面的校验和（checksum），每当一个页面在内存中修改了，在同步之前就要把它的校验和算出来。在一个页面被刷到磁盘的时候，首先被写入磁盘的就是这个 checksum。

- `FIL_PAGE_OFFSET`

每一个页都有一个单独的页号，InnoDB 通过页号来唯一定位一个页。

如某独立表空间 a.ibd 的大小为1GB，页的大小默认为16KB，那么总共有65536个页。FIL_PAGE_OFFSET 表示该页在所有页中的位置。若此表空间的ID为10，那么搜索页（10，1）就表示查找表a中的第二个页。

- `FIL_PAGE_PREV` 和 `FIL_PAGE_NEXT`

InnoDB 是以页为单位存放数据的，InnoDB 表是索引组织的表，数据是按主键顺序存放的。数据可能会分散到多个不连续的页中存储，这时就会通过 FIL_PAGE_PREV 和 FIL_PAGE_NEXT 将上一页和下一页连起来，就形成了一个双向链表。这样就通过一个双向链表把许许多多的页就都串联起来了，而无需这些页在物理上真正连着。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/30a5ee1b6f1543648ecad0b392fbfed6~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

- `FIL_PAGE_TYPE`

这个代表当前页的类型，InnoDB 为了不同的目的而设计了许多种不同类型的页。

InnoDB 有如下的一些页类型：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcb6dd1fe2b940d88d889e315a6e6562~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)



## File Tailer

将页写入磁盘时，最先写入的便是 File Header 中的 `FIL_PAGE_SPACE_OR_CHKSUM` 值，就是页面的校验和。在写入的过程中，数据库可能发生宕机，导致页没有完整的写入磁盘。

为了校验页是否完整写入磁盘，InnoDB 就设置了 `File Trailer` 部分。File Trailer 中只有一个`FIL_PAGE_END_LSN`，占用`8字节`。FIL_PAGE_END_LSN 又分为两个部分，前`4字节`代表页的校验和；后`4字节`代表页面被最后修改时对应的日志序列位置（LSN），与File Header中的`FIL_PAGE_LSN`相同。

默认情况下，InnoDB存储引擎每次从磁盘读取一个页就会检测该页的完整性，这时就会将 `File Trailer` 中的校验和、LSN 与 `File Header` 中的 `FIL_PAGE_SPACE_OR_CHKSUM`、`FIL_PAGE_LSN` 进行比较，以此来保证页的完整性。
