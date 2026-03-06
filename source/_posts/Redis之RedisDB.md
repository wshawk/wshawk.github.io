---
title: Redis之redisDb
categories:
  - 后端
tags: 
  - Redis
  - redisDb
abbrlink: 209c3ceq
date: 2026-03-04 17:38:45
---
# Redis之redisDb

`redisDb` 是 Redis 单机数据库中每个逻辑数据库的物理实现，它不仅是键值对的容器，更是一个高度集成的数据管理中枢。理解 `redisDb` 的结构与运作机制，是掌握 Redis 内存管理、过期策略、阻塞操作、事务、持久化、复制及集群模式的基石。本文基于 Redis 源码（以 7.0 版本为基准），全面剖析 `redisDb` 的设计思想与实现细节，并融入最新版本的演进特征。



### 1. `redisDb` 数据结构全景

每个 Redis 服务器拥有多个数据库（默认 16 个），存储在 `redisServer.db` 数组中。客户端通过 `SELECT` 切换当前数据库，实际就是修改 `redisClient.db` 指针指向对应的 `redisDb` 实例。`redisDb` 结构定义在 `server.h` 中，核心字段如下：

```c
typedef struct redisDb {
    dict *dict;                 // 键空间，存储所有键值对
    dict *expires;              // 过期字典，存储键的过期时间
    dict *blocking_keys;        // 因阻塞命令（如 BLPOP）等待的键和客户端
    dict *ready_keys;           // 准备就绪的键，用于延迟唤醒阻塞客户端
    dict *watched_keys;         // 事务 WATCH 监视的键及其客户端
    int id;                      // 数据库编号（0,1,2...）
    long long avg_ttl;           // 平均生存时间（采样估算）
    list *defrag_later;          // 待内存碎片整理的键列表
    // 集群模式下特有
    clusterSlotToKeyMapping *slots_to_keys; // 哈希槽到键的映射（基数树）
    // ...
} redisDb;
```

#### 1.1 键空间 `dict` —— 数据的本体
- **键**：`sds`（简单动态字符串）。
- **值**：`redisObject`，包含类型（`type`）、编码（`encoding`）、引用计数、LRU/LFU 信息以及指向实际数据的 `ptr` 指针。
- 所有 Redis 命令最终都是对 `dict` 的增删改查。

#### 1.2 过期字典 `expires` —— 时间的守护者
- **键**：指针，直接指向 `dict` 中键的 `sds` 内存地址，**避免重复存储键名**，是 Redis 内存优化的重要手段。
- **值**：`long long` 类型，存储过期时间戳（毫秒精度）。
- **内存陷阱**：当 `dict` 发生 rehash 时，`sds` 本身不会移动（内联在 `dictEntry` 中），但键的指针保持有效。删除键时，`dbSyncDelete()` / `dbAsyncDelete()` 会确保同时从 `dict` 和 `expires` 中移除对应条目，杜绝悬空引用。
- **内存开销**：即使 `expires` 为空，作为字典仍有桶数组开销；但对于永不过期的键，该字典几乎不占用额外内存。

#### 1.3 辅助索引

- **`blocking_keys`**：用于阻塞式列表弹出（如 `BLPOP`）。在 Redis 3.0 后，其值从 `list *` 优化为 `dict`，键为客户端，值为该客户端阻塞的键列表（支持多键阻塞）。当列表有新数据时，通过此字典快速定位待唤醒客户端。
- **`ready_keys`**：**延迟处理队列**，存放已就绪的键。当键被修改（如 `LPUSH`）时，不会立即处理阻塞客户端，而是将键加入 `ready_keys`，在 `beforeSleep()` 中统一唤醒，避免递归调用和重入问题。  
  **假阳性细节**：若多个客户端阻塞在同一键上，第一个被唤醒的客户端可能消耗所有数据，后续客户端被唤醒时会发现列表仍为空，此时它们会**重新进入阻塞状态**，而非报错。这种设计保证了正确性，代价是少量的无效唤醒。
- **`watched_keys`**：实现事务乐观锁。键被 `WATCH` 时，记录监视该键的客户端。当键被修改，`touchWatchedKey()` 标记客户端为“脏”，后续 `EXEC` 将失败。**历史漏洞**：早期版本存在客户端异常断开时未清理 `watched_keys` 的内存泄漏，现代版本通过 `unwatchAllKeys()` 在 `freeClient()` 或 `EXEC`/`DISCARD` 时确保清理。
- **`defrag_later`**：主动碎片整理模块使用。当键的底层数据结构（如 `quicklist`、`skiplist`）占用内存超过 `DEFRAG_LATER_THRESHOLD`（默认 100MB），会被加入该列表。主线程在 `beforeSleep()` 中分配时间片（`active-defrag-cycle`，默认 1ms）逐步处理这些键，避免阻塞。阈值判断的是**对象本身的大小**，不包括 `redisObject` 开销。
- **`slots_to_keys`（集群）**：基数树（rax）实现，用于快速定位本节点上键所属的哈希槽，支持 `CLUSTER GETKEYSINSLOT` 和槽迁移。每次写入/删除键都需要更新该索引，带来 O(log n) 额外开销，因此仅在集群模式启用。`slots_to_keys` 仅存储键名指针（指向 `dict` 中的 `sds`），不存储实际数据。



### 2. 关键操作机制

#### 2.1 键的访问：`lookupKey` 系列函数
所有键操作都通过 `lookupKeyRead()` / `lookupKeyWrite()` 进行，它们封装了以下逻辑：
- **过期检查**：调用 `expireIfNeeded()`，若键已过期则删除并返回 `NULL`。
- **LRU/LFU 更新**：读操作会更新 `redisObject` 的 `lru` 字段（24 位）。LFU 模式下，该字段存储访问频次的对数计数器；LRU 模式下存储最近访问的毫秒级时间戳。这是内存淘汰策略的数据基础。
- **读写分离**：`lookupKeyWrite` 总是执行过期检查；`lookupKeyRead` 在从库场景下可能跳过检查（见下文）。

#### 2.2 过期键清理策略
- **惰性删除**：访问键时触发，实时删除过期键。
- **定期删除**：由 `serverCron` 调用 `activeExpireCycle()`，随机抽查 20 个键，若过期比例超 25% 则重复，总耗时不超过 25ms。同时会**采样估算 `avg_ttl`**：计算被抽查键的剩余 TTL 平均值，直接覆盖 `db->avg_ttl`（非滑动平均），因此 `INFO` 中的 `avg_ttl` 仅反映最近一次采样结果，波动较大，不能用于精确决策。
- **从库特殊处理**：从库的 `expireIfNeeded` 返回 1（表示键已过期），但**不执行删除**，调用者（如 `lookupKeyRead`）会将该键视为不存在。这导致从库内存中可能保留已过期的键，直到主库发送 `DEL` 命令才真正释放。若从库与主库断开连接，过期键会累积，直到 `maxmemory` 触发淘汰或连接恢复。

#### 2.3 阻塞操作实现
以 `BLPOP` 为例：
1. 客户端尝试弹出空列表，被加入 `blocking_keys` 中对应键的等待队列。
2. 当有元素推入该列表（如 `LPUSH`），`signalListAsReady()` 将该键加入 `ready_keys`（而不是立即唤醒）。
3. 命令执行结束后，在 `beforeSleep()` 中调用 `handleClientsBlockedOnKeys()`，遍历 `ready_keys`，找到对应键的阻塞客户端并唤醒，同时将元素返回。
4. **延迟处理原因**：避免在命令执行过程中唤醒客户端导致的递归调用，确保事件驱动的稳定性。

#### 2.4 事务乐观锁
- 客户端执行 `WATCH key` 时，`key` 和该客户端被记录到 `watched_keys`。
- 当任何命令修改了 `key`（如 `SET`），`touchWatchedKey()` 将该客户端标记为 `REDIS_DIRTY_CAS`。
- 执行 `EXEC` 时，若客户端被标记为脏，事务失败；否则执行队列中的命令。

#### 2.5 内存碎片整理
- 通过 `ACTIVE_DEFRAG` 配置开启，`defrag_later` 列表存放待处理的大键。
- 主线程在 `beforeSleep()` 中调用 `activeDefragCycle()`，每次处理时间不超过 `active-defrag-cycle` 毫秒（默认 1ms）。
- 判定大键的标准：底层数据结构（如 `quicklist`、`skiplist`）占用内存超过 `DEFRAG_LATER_THRESHOLD`（100MB），**仅计算对象本身大小**，不含 `redisObject` 开销。



### 3. 集群模式下的 `redisDb`

#### 3.1 为什么只有 DB0？
- **分片模型冲突**：Redis Cluster 将键空间划分为 16384 个哈希槽，每个键通过 `CRC16(key) % 16384` 确定槽位。若允许多 DB，路由需增加 DB 维度，导致元数据膨胀、管理复杂，且破坏线性扩展能力。
- **设计取舍**：强制使用单一数据库，业务隔离由**键前缀**实现（如 `user:1000`），多键操作依赖**哈希标签**（`{...}`）保证键落入同一槽。
- **`SELECT` 命令报错时机**：集群模式下，`SELECT 0` 被允许（但无意义），而 `SELECT n`（n≠0）会立即报错 `"SELECT is not allowed in cluster mode"`，而非 `"invalid DB index"`。

#### 3.2 `slots_to_keys` 的实现细节
```c
typedef struct clusterSlotToKeyMapping {
    rax *rax;  // 基数树，键为槽号，值为该槽下的键列表
} clusterSlotToKeyMapping;
```
- **作用**：快速获取指定槽内的所有键，用于 `CLUSTER GETKEYSINSLOT` 和槽迁移（`MIGRATE`）。
- **更新时机**：每次写入或删除键时，同步更新该索引，带来 O(log n) 开销。
- **与 `dict` 的关系**：仅为辅助索引，不存储实际数据，键名指针指向 `dict` 中的键对象（内存复用）。



### 4. 与持久化/复制的交互

#### 4.1 RDB 持久化
- **生成**：`rdbSave()` 遍历 `dict`，写入所有未过期的键（跳过 `expires` 中已过期的键）。`fork()` 子进程利用写时复制（Copy-on-Write）保证快照一致性。
- **加载**：直接重建 `dict` 和 `expires`，过期键在加载后通过惰性删除逐步清理。

#### 4.2 AOF 持久化
- **重写**：`rewriteAppendOnlyFile()` 遍历当前 `redisDb` 的 `dict`，将每个键转换为最小写命令集合（如 `SET`、`RPUSH`）写入新 AOF 文件。若存在多个数据库，会在每个数据库的命令前插入 `SELECT` 命令。
- **历史包袱**：多 DB 导致 AOF 文件包含大量 `SELECT`，增加回放时间。混合持久化（RDB + AOF）中，RDB 部分不保存 DB 编号（假定为 DB 0），进一步弱化多 DB 的实用性。

#### 4.3 复制
- 主库的每个写命令都会传播给从库。
- 从库的 `expires` 仅用于读操作时判断键是否过期（`expireIfNeeded()` 在从库中**不执行删除**），实际删除依赖主库的 `DEL` 命令。



### 5. 新版本演进（Redis 6.0/7.0）

| 特性           | 版本 | 对 `redisDb` 的影响                                          |
| -------------- | ---- | ------------------------------------------------------------ |
| **I/O 多线程** | 6.0  | 读命令解析和写回复可并行，但 `redisDb` 操作仍由主线程串行执行，无锁竞争。 |
| **ACL 日志**   | 6.0  | 每个 `redisDb` 的操作会记录到 ACL 日志，影响 `lookupKey` 路径。 |
| **Function**   | 7.0  | 类似 Module，但 Function 可以跨 DB 访问，需额外注意 `SELECT` 切换时的上下文。 |



### 6. 统计与监控

- **`avg_ttl`**：由 `activeExpireCycle()` 采样估算，通过 `INFO` 命令可查看。
- **`keyspace_hits` / `keyspace_misses`**：虽存储在全局 `redisServer`，但在每个 `redisDb` 的 `lookupKey` 操作中更新，反映数据库命中率。
- **`INFO keyspace`**：输出每个数据库的键总数、过期键数、平均 TTL。



### 7. 总结

`redisDb` 是 Redis 数据管理的核心枢纽，它通过 `dict` 存储数据，`expires` 管理生命周期，辅助索引支撑阻塞、事务、碎片整理和集群功能。其设计体现了 **“简单结构，复杂协同”** 的思想：每个字段职责单一，但通过事件驱动、持久化、复制等模块的紧密配合，构成了一个高性能、可扩展的数据库引擎。理解 `redisDb` 不仅有助于日常运维和问题排查，更能为阅读源码、开发模块或定制 Redis 打下坚实基础。