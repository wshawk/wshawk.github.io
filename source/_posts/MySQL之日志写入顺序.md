---
title: MySQL之日志写入顺序
categories:
  - 数据库
tags: MySQL
abbrlink: ae3a4486
date: 2022-11-13 22:30:31
---

# MySQL之日志写入顺序

当MySQL执行一条`UPDATE`语句时，InnoDB存储引擎和Server层会协同工作，按照严格的顺序写入`undo log`、`redo log`和`binlog`，并通过**两阶段提交**保证数据一致性和崩溃恢复能力。以下是详细的执行流程和日志写入顺序：



## 1. 更新语句执行前的准备
- 客户端发送`UPDATE`语句，MySQL Server通过连接器、分析器、优化器后，调用InnoDB的接口执行更新。
- 如果更新的数据页不在内存的**缓冲池**中，InnoDB会先将该页从磁盘读入缓冲池。



## 2. 记录 `undo log`（回滚日志）
- **顺序：第1步写入**  
  在修改数据之前，InnoDB会先将修改前的旧值写入**`undo log`**，记录到`undo log buffer`中。  
  - 例如：将“修改前”的整行数据或主键信息保存下来，用于事务回滚或MVCC。  
  - `undo log`本身也是物理修改（因为`undo`页也需要持久化），因此对`undo`页的修改也会在稍后被`redo log`保护。



## 3. 修改缓冲池中的数据页
- **顺序：第2步（内存操作）**  
  在内存中更新缓冲池里的数据页，将新值写入，此时该页变为“脏页”（内存与磁盘不一致）。  
  - 注意：这一步**不会立即写回磁盘**，而是等待后续刷盘。



## 4. 记录 `redo log`（重做日志）到缓冲区
- **顺序：第3步写入**  
  在数据页修改的同时，InnoDB会生成对应的**物理日志**（记录“在哪个页的哪个偏移量改为什么值”），并写入`redo log buffer`。  
  - 这一步遵循**WAL（Write-Ahead Logging）**原则：日志先于数据持久化。  
  - `undo`页的修改也会同时被记录到`redo log`中。

此时，`UPDATE`语句的执行阶段完成，但事务尚未提交。



## 5. 事务提交：两阶段提交（关键步骤）
当客户端发起`COMMIT`时，MySQL会进入**两阶段提交**流程，确保`redo log`和`binlog`的一致性。

### 第一阶段：`redo log` 进入 `prepare` 状态并刷盘
- **顺序：第4步**  
  InnoDB将当前事务的`redo log buffer`刷入磁盘的`redo log file`，并记录事务状态为`PREPARE`。  
  - 如果`innodb_flush_log_at_trx_commit=1`，此时会强制`fsync`落盘，保证`redo log`已持久化。

### 第二阶段：写入 `binlog`（二进制日志）并刷盘
- **顺序：第5步**  
  MySQL Server将事务的`binlog`（逻辑日志，记录SQL语句或行变更）写入`binlog file`，并执行`fsync`刷盘。  
  - 至此，`binlog`已持久化。

### 第三阶段：`redo log` 进入 `commit` 状态
- **顺序：第6步**  
  InnoDB在`redo log`中写入一个`COMMIT`标记，表示事务正式提交。此时事务才算真正完成。  
  - 这一步通常不需要再次刷盘，只需在内存中标记即可（但`redo log`中已有完整的修改记录）。



## 6. 返回成功
- **顺序：最后一步**  
  提交完成后，MySQL向客户端返回“更新成功”。



## 日志写入顺序总结（时间线）

| 步骤 | 操作内容 | 日志类型 | 说明 |
|------|----------|----------|------|
| 1 | 记录旧值到`undo log buffer` | `undo log` | 内存中记录，为回滚做准备 |
| 2 | 修改缓冲池中的数据页 | 无 | 内存中的脏页生成 |
| 3 | 生成`redo log`并写入`redo log buffer` | `redo log` | 记录物理修改 |
| 4 | **两阶段提交第1阶段**：`redo log`刷盘并标记`PREPARE` | `redo log` | 持久化到`redo log file` |
| 5 | **两阶段提交第2阶段**：`binlog`刷盘 | `binlog` | 持久化到`binlog file` |
| 6 | **两阶段提交完成**：`redo log`标记`COMMIT` | `redo log` | 在`redo log`中写入提交标记 |
| 7 | 返回客户端成功 | - | - |



## 为什么要这么设计？两阶段提交的作用

- **保证数据一致性**：如果崩溃发生在步骤4之后、步骤5之前，恢复时会发现`redo log`处于`PREPARE`状态且无对应`binlog`，则事务回滚；如果崩溃在步骤5之后，则`redo log`和`binlog`都已持久化，事务可以提交。这样就避免了主从数据不一致或数据丢失。
- **`undo log`先于修改**：确保任何修改都可以被撤销，即使系统在修改后立即崩溃，重启后也可以通过`undo log`回滚未提交的事务。



## 补充说明

- **`undo log`的持久化**：虽然`undo log`是逻辑日志，但其所在的数据页修改也会被`redo log`记录，因此`undo log`的持久化依赖于`redo log`的WAL机制。
- **刷盘策略的影响**：如果`innodb_flush_log_at_trx_commit`或`sync_binlog`设置为非1的值，刷盘时机可能延迟，但日志写入的顺序逻辑不变。
- **自动提交**：如果`autocommit=1`，则每条`UPDATE`语句执行完后会自动提交，即上述提交阶段会立即执行。

通过这种严格的顺序和两阶段提交，MySQL在保证高性能的同时，实现了事务的ACID特性和主从复制的数据一致性。
