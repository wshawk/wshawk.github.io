---
title: MySQL之高可用集群方案
categories:
  - 数据库
tags: MySQL
abbrlink: ae3a4477
date: 2023-03-20 21:22:31
---

# MySQL之高可用集群方案

[20 InnoDB Cluster：改变历史的新产品.md (lianglianglee.com)](https://learn.lianglianglee.com/专栏/MySQL实战宝典/20  InnoDB Cluster：改变历史的新产品.md)

## MHA（Master High Availability）

MHA(Master High Availability)是一款开源的 MySQL 高可用程序，它为 MySQL 数据库主从复制架构提供了 automating master failover 的功能。

MHA 是由业界大名鼎鼎的 Facebook 工程师 Yoshinorim 开发，开源地址为：https://github.com/yoshinorim/mha4mysql-manager它由两大组件所组成，MHA Manger 和 MHA Node。

MHA Manager 通常部署在一台服务器上，用来判断多个 MySQL 高可用组是否可用。当发现有主服务器发生宕机，就发起 failover 操作。MHA Manger 可以看作是 failover 的总控服务器。

而 MHA Node 部署在每台 MySQL 服务器上，MHA Manager 通过执行 Node 节点的脚本完成failover 切换操作。

MHA Manager 和 MHA Node 的通信是采用 ssh 的方式，也就是需要在生产环境中打通 MHA Manager 到所有 MySQL 节点的 ssh 策略，那么这里就存在潜在的安全风险。

另外，ssh 通信，效率也不是特别高。

所以，**MHA 比较适合用于规模不是特别大的公司，所有MySQL 数据库的服务器数量不超过 20 台**。

![](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%98%E5%AE%9D%E5%85%B8/assets/Cgp9HWDVmMOAevY4AAEiqr35GSw848.png)



## Orchestrator

Orchestrator 是另一款开源的 MySQL 高可用套件，除了支持 failover 的切换，还可通过Orchestrator 完成 MySQL 数据库的一些简单的复制管理操作。Orchestrator 的开源地址为：https://github.com/openark/orchestrator

你可以把 Orchestrator 当成 MHA 的升级版，而且提供了 HTTP 接口来进行相关数据库的操作，比起 MHA 需要每次登录 MHA Manager 服务器来说，方便很多。

下图显示了 Orchestrator 的高可用设计架构：

![Drawing 5.png](https://learn.lianglianglee.com/专栏/MySQL实战宝典/assets/CioPOWDVmLaAd1fTAAOTUVey1gw096.png)

其基本实现原理与 MHA 是一样的，只是把元数据信息存储在了元数据库中，并且提供了HTTP 接口和命令的访问方式，使用上更为友好。

但是由于管控节点到下面的 MySQL 数据库的管理依然是 ssh 的方式，依然存在 MHA 一样的短板问题，总的来说，关于 Orchestrator 我想提醒你，依然只建议使用在较小规模的数据库集群。

## InnoDB Cluster(MGR)

### MGR技术

MGR 是官方在 MySQL 5.7 版本推出的一种基于状态机的数据同步机制。与半同步插件类似，MGR 是通过插件的方式启用或禁用此功能。

![图片2.png](https://learn.lianglianglee.com/专栏/MySQL实战宝典/assets/Cgp9HWDdkMWAYcbMAAELdJr4eqw754.png)

MGR 复制结构图

注意，我们谈及 MGR，不要简单认为它是一种新的数据同步技术，而是应该把它理解为高可用解决方案，而且特别适合应用于对于数据一致性要求极高的金融级业务场景。

首先，MGR 之间的数据同步并没有采用复制技术，而是采用 GCS（Group Communication System）协议的日志同步技术。

GSC 本身是一种类似 Paxos 算法的协议，要求组中的大部分节点都接收到日志，事务才能提交。所以，MRG 是严格要求数据一致的，特别适合用于金融级的环境。由于是类 Paxos 算法，集群的节点要求数量是奇数个，这样才能满足大多数的要求。

有的同学可能会问了：之前介绍的无损半同步也能保证数据强一致的要求吗？

是的，虽然通过无损半同步复制也能保证主从数据的一致性，但通过 GCS 进行数据同步有着更好的性能：当启用 MGR 插件时，MySQL 会新开启一个端口用于数据的同步，而不是如复制一样使用MySQL 服务端口，这样会大大提升复制的效率。

其次，MGR 有两种模式：

- 单主（Single Primary）模式；
- 多主（Multi Primary）模式。

单主模式只有 1 个节点可以写入，多主模式能让每个节点都可以写入。而多个节点之间写入，如果存在变更同一行的冲突，MySQL 会自动回滚其中一个事务，自动保证数据在多个节点之间的完整性和一致性。

最后，在单主模式下，MGR 可以自动进行 Failover 切换，不用依赖外部的各种高可用套件，所有的事情都由数据库自己完成，比如最复杂的选主（Primary Election）逻辑，都是由 MGR 自己完成，用户不用部署额外的 Agent 等组件。

**说了这么多 MGR 的优势，那么它有没有缺点或限制呢？** 当然有，主要是这样几点：

1. 仅支持 InnoDB 表，并且每张表一定要有一个主键；
2. 目前一个 MGR 集群，最多只支持 9 个节点；
3. 有一个节点网络出现抖动或不稳定，会影响集群的性能。

第 1、2 点问题不大，因为目前用 MySQL 主流的就是使用 InnoDB 存储引擎，9 个节点也足够用了。

而第 3 点我想提醒你注意，和复制不一样的是，由于 MGR 使用的是 Paxos 协议，对于网络极其敏感，如果其中一个节点网络变慢，则会影响整个集群性能。而半同步复制，比如 ACK 为1，则 1 个节点网络出现问题，不影响整个集群的性能。所以，在决定使用 MGR 后，切记一定要严格保障网络的质量。

而多主模式是一种全新的数据同步模式，接下来我们看一看在使用多主模式时，该做哪些架构上的调整，从而充分发挥 MGR 多主的优势。

### 多主模式的注意事项

#### 冲突检测

MGR 多主模式是近几年数据库领域最大的一种创新，而且目前来看，仅 MySQL 支持这种多写的 Share Nothing 架构。

多主模式要求每个事务在本节点提交时，还要去验证其他节点是否有同样的记录也正在被修改。如果有的话，其中一个事务要被回滚。

比如两个节点同时执行下面的 SQL 语句：

```sql
-- 节点1

UPDATE User set money = money - 100 WHERE id = 1;

-- 节点2

UPDATE User set money = money + 300 WHERE id = 1;
```

如果一开始用户的余额为 200，当节点 1 执行 SQL 后，用户余额变为 100，当节点 2 执行SQL，用户余额变味了 500，这样就导致了节点数据的不同。所以 MGR 多主模式会在事务提交时，进行行记录冲突检测，发现冲突，就会对事务进行回滚。

在上面的例子中，若节点 2 上的事务先提交，则节点 1 提交时会失败，事务会进行回滚。

所以，如果要发挥多主模式的优势，就要避免写入时有冲突。**最好的做法是：每个节点写各自的数据库，比如节点 1 写 DB1，节点 2 写 DB2，节点 3 写 DB3，这样集群的写入性能就能线性提升了。**

不过这要求我们在架构设计时，就做好这样的考虑，否则多主不一定能带来预期中的性能提升。

#### 自增处理

在多主模式下，自增的逻辑发生了很大的变化。简单来说，自增不再连续自增。

因为，如果连续自增，这要求每次写入时要等待自增值在多个节点中的分配，这样性能会大幅下降，所以 MGR 多主模式下，我们可以通过设置自增起始值和步长来解决自增的性能问题。看下面的参数：

```ini
group_replication_auto_increment_increment = 7
```

参数 group_replication_auto_increment_increment 默认为 7，自增起始值就是 server-id。

假设 MGR 有 3 个节点 Node1、Node2、Node3，对应的 server-id 分别是 1、2、3, 如果这时多主插入自增的顺序为 Node1、Node1、Node2、Node3、Node1，则自增值产生的结果为：

![图片3.png](https://learn.lianglianglee.com/专栏/MySQL实战宝典/assets/Cgp9HWDdkPSAG1iMAAC--XyGLbc889.png)

可以看到，由于是多主模式，允许多个节点并发的产生自增值。所以自增的产生结果为1、8、16、17、22，自增值不一定是严格连续的，而仅仅是单调递增的，这与单实例 MySQL 有着很大的不同。

在 05 讲表结构设计中，我也强调过：尽量不要使用自增值做主键，在 MGR 存在问题，在后续分布式架构中也一样存在类似的自增问题。**所以，对于核心业务表，还是使用有序 UUID 的方式更为可靠，性能也会更好。**

总之，使用 MGR 技术后，所有高可用事情都由数据库自动完成。那么，业务该如何利用 MGR的能力，是否还需要 VIP、DNS 等机制保证业务的透明性呢？接下来，我们就来看一下，**业务如何利用 MGR 的特性构建高可用解决方案。**

### InnoDB Cluster

MGR 是基于 Paxos 算法的数据同步机制，将数据库状态和日志通过 Paxos 算法同步到各个节点，但如果要实现一个完整的数据库高可用解决方案，就需要更高一层级的 InnoDB Cluster 完成。

一个 InnoDB Cluster 由三个组件组成：MGR 集群、MySQL Shell、MySQL Router。具体如下图所示：

![图片4.png](https://learn.lianglianglee.com/专栏/MySQL实战宝典/assets/CioPOWDdkV2AVrxFAAIKlxjsK_I547.png)

其中，MySQL Shell 用来管理 MGR 集群的创建、变更等操作。以后我们最好不要手动去管理 MGR 集群，而是通过 MySQL Shell 封装的各种接口完成 MGR 的各种操作。如：

```javascript
mysql-js> cluster.status()

{

    "clusterName": "myCluster", 

    "defaultReplicaSet": {

        "name": "default", 

        "primary": "ic-2:3306", 

        "ssl": "REQUIRED", 

        "status": "OK", 

        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 

        "topology": {

            "ic-1:3306": {

                "address": "ic-1:3306", 

                "mode": "R/O", 

                "readReplicas": {}, 

                "role": "HA", 

                "status": "ONLINE"

            }, 

            "ic-2:3306": {

                "address": "ic-2:3306", 

                "mode": "R/W", 

                "readReplicas": {}, 

                "role": "HA", 

                "status": "ONLINE"

            }, 

            "ic-3:3306": {

                "address": "ic-3:3306", 

                "mode": "R/O", 

                "readReplicas": {}, 

                "role": "HA", 

                "status": "ONLINE"

            }

        }

    }, 

    "groupInformationSourceMember": "mysql://root@localhost:6446"

}
```

MySQL Router 是一个轻量级的代理，用于业务访问 MGR 集群中的数据，当 MGR 发生切换时（这里指 Single Primary 模式），自动路由到新的 MGR 主节点，这样业务就不用感知下层MGR 数据的切换。

为了减少引入 MySQL Router 带来的性能影响，官方建议 MySQL Router 与客户端程序部署在一起，以一种类似 sidecar 的方式进行物理部署。这样能减少额外一次额外的网络开销，基本消除引入 MySQL Router 带来的影响。

所以，这里 MySQL Router 的定位是一种轻量级的路由转发，而不是一个数据库中间件，主要解决数据库切换后，做到对业务无感知。