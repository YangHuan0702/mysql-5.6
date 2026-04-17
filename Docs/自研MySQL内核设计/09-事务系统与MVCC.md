# 事务系统与 MVCC

## 模块目标

事务系统负责提供原子性、一致性和隔离性。MVCC 负责在并发读写下提供一致性读语义。对于接近生产可用的单机 MySQL 内核，这部分不是“某个子模块”，而是把 SQL 层、锁管理、undo、redo、恢复、purge 串起来的主干。

这一层必须回答清楚以下问题：

- 一个事务从开始到提交，内部状态如何流转
- 读请求如何看到一致快照，写请求如何与并发事务隔离
- 更新前镜像和提交后持久化日志如何分工
- 崩溃后如何恢复已提交事务，回滚未完成事务
- 长事务、历史版本堆积和 purge 如何治理

## 当前仓库参照实现

当前仓库中最关键的参照点是：

- `sql/handler.cc`: SQL 层事务协调，如 `ha_commit_trans()`
- `storage/innobase/trx/`: 事务对象、提交、回滚、事务系统
- `storage/innobase/read/`: 一致性读与 read view
- `storage/innobase/row/`: 行版本访问、更新路径
- `storage/innobase/lock/`: 事务锁与可见性协作
- `storage/innobase/log/`: redo 体系

从这个参照实现可以看到，MySQL / InnoDB 风格事务系统有几个强约束：

- SQL 层只负责编排事务边界，不直接决定物理版本可见性
- MVCC 依赖 undo 链，而不是靠“复制整行快照”解决
- redo 负责持久化已发生的物理或逻辑修改，undo 负责逻辑回滚和一致性读
- 锁和 MVCC 不是替代关系，而是组合关系

## 如果你自己实现，应该怎么抽象

建议采用 InnoDB 风格事务模型，但在实现上做更清晰的分层：

1. `TransactionManager`
负责事务 ID 分配、活动事务表、全局提交序列和 read view 构造。

2. `Transaction`
负责单事务状态、锁集合、undo 链、redo 上下文、隔离级别和生命周期。

3. `MVCCManager`
负责版本可见性判断、read view 管理、历史版本遍历。

4. `UndoManager`
负责 undo segment 分配、undo record 组织、回滚和 purge 输入。

5. `CommitManager`
负责提交协议、log flush 协调、提交顺序与 durability 语义。

这种分法比“把一切都塞进 `Transaction`”更稳，因为事务系统真正要做的是协调多个子系统。

## 核心数据结构

- `TxnId`
  全局递增事务 ID。用于版本可见性和 undo 归属，不直接等价于提交序号。

- `CommitSeqNo`
  可选的全局提交序列。用于稳定地判断提交先后顺序，便于可见性推导和后续复制扩展。

- `Transaction`
  包含：
  - `txn_id`
  - `state`
  - `isolation_level`
  - `read_view`
  - `undo_head`
  - `held_locks`
  - `redo_context`
  - `last_error`

- `ReadView`
  包含：
  - `low_limit_id`
  - `up_limit_id`
  - `creator_txn_id`
  - `active_txn_set`

- `UndoRecord`
  至少包含：
  - 记录所属事务 ID
  - 被修改记录的逻辑定位信息
  - 修改前旧值或必要字段
  - 上一版本指针
  - 回滚类型

- `VersionPointer`
  记录当前行版本与历史 undo 链的入口。

## 关键不变量

这一层最重要的不是 API，而是不变量。建议明确写入设计文档并做断言检查：

1. 每个活动事务在事务管理器里只有一个权威状态。
2. 每条已修改记录都能追溯到其最新版本的创建事务。
3. 一致性读绝不阻塞在普通写锁上，而是通过可见性判断选择当前版本或历史版本。
4. 已提交事务的 redo 必须先于对应脏页落盘。
5. 未提交事务的修改在恢复后必须可回滚。

## 事务状态机

建议把事务状态显式建模为：

- `ACTIVE`
- `PREPARED_TO_COMMIT`
- `COMMITTED_IN_MEMORY`
- `COMMITTED_DURABLE`
- `ROLLING_BACK`
- `ROLLED_BACK`

如果第一版不做 XA 或两阶段提交，可以不暴露 `PREPARED` 给外部，但内部最好保留“进入提交协议”的状态节点，否则后面很难引入更严格的 durability 编排。

## Read View 设计

### Read Committed

- 每条语句开始时创建新的 `ReadView`
- 语句结束即失效
- 优点是实现简单
- 缺点是同一事务内两次一致性读可能看到不同结果

### Repeatable Read

- 事务第一次一致性读时创建 `ReadView`
- 事务结束前复用同一个视图
- 适合作为 MySQL 风格默认隔离级别

建议第一版优先实现：

- 当前读
- Repeatable Read 下一致性读

如果资源有限，可以先做 Repeatable Read 的核心语义，再补 Read Committed 的“每语句新视图”优化。

## 可见性判断规则

给定记录版本的创建事务 ID `row_txn_id` 和读视图 `ReadView`，可见性逻辑建议按以下顺序判断：

1. 如果 `row_txn_id == creator_txn_id`，则本事务自己的修改可见。
2. 如果 `row_txn_id < low_limit_id`，说明对应事务在视图前已提交，可见。
3. 如果 `row_txn_id >= up_limit_id`，说明对应事务在视图创建后启动，不可见。
4. 如果 `row_txn_id` 在 `active_txn_set` 中，说明创建视图时该事务仍活跃，不可见。
5. 否则可见。

若当前版本不可见，则沿 undo 链向旧版本回溯，直到找到可见版本或判定记录不存在。

## 写路径设计

一次更新操作建议拆成以下步骤：

1. 通过当前读定位目标记录，并申请必要事务锁。
2. 生成 undo record，保存回滚所需旧值。
3. 在内存页内修改记录，更新版本指针和 `row_txn_id`。
4. 生成 redo record，记录页内修改或逻辑修改信息。
5. 将本次修改挂入事务私有的修改集。

这里最容易犯的错误是把 undo 当作 redo 或把 redo 当作 undo。职责必须强区分：

- undo 服务于回滚和快照读
- redo 服务于崩溃恢复

## 提交路径设计

推荐的第一版提交路径：

1. 将事务状态从 `ACTIVE` 切换到 `PREPARED_TO_COMMIT`
2. 将该事务的 redo 追加到 log buffer
3. 根据提交策略决定是否立即 flush
4. flush 成功后标记 `COMMITTED_DURABLE`
5. 从 active transaction table 删除该事务
6. 唤醒等待此事务锁释放的其他事务

如果要追求更接近生产可用，建议至少支持两种策略：

- `flush_at_commit = 1`
  每次提交强刷，最安全
- `flush_at_commit = 2` 或批量刷
  吞吐更高，但可能丢失最近极短窗口事务

第一版默认值应该偏保守。

## 回滚路径设计

回滚逻辑建议明确区分：

- 语句失败导致的 statement rollback
- 用户显式 `ROLLBACK`
- 崩溃恢复阶段回滚未提交事务

逻辑步骤：

1. 取事务 undo 链尾部
2. 逆序遍历 undo record
3. 恢复旧值或删除新插入版本
4. 释放事务锁
5. 状态转为 `ROLLED_BACK`

不要把“事务 abort 后直接丢弃内存修改”作为实现思路；生产可用引擎必须依赖 undo 逐项回退。

## Purge 与历史版本治理

MVCC 一旦能跑，很快就会遇到第二个问题：历史版本永远不清理，系统会膨胀。

建议设计独立 `PurgeWorker`：

- 追踪最老活跃 read view 的边界
- 清理比该边界更早、且不再可见的 undo 记录
- 在必要时回收被 delete-mark 的旧记录版本

第一版不必追求多线程 purge，但一定要有 purge 机制，否则长跑压测后系统不可持续。

## 工程实现建议

- 第一版就把“当前读”和“一致性读”区别清楚。
- 第一版就决定事务 ID、提交序号和 read view 的关系，不要后补。
- 事务提交路径必须显式记录状态变化，便于恢复和调试。
- 回滚、purge、恢复共享大量 undo 语义，数据结构设计时要提前统一。
- 对长事务要有可观测性：持续时间、持锁数、保留历史版本数。

## 第一版应该做什么，不该做什么

### 应该做

- 完整的事务 begin / commit / rollback
- 一种完整隔离级别，推荐 Repeatable Read
- undo 链与 read view
- 活动事务表
- 提交顺序与 redo 协调
- 单线程 purge

### 不该做

- 一上来就做所有 ANSI 隔离级别细节
- 一上来就做 XA 或分布式事务
- 一上来就做过度复杂的提交并行化

## 最小可用版本

- `BEGIN`, `COMMIT`, `ROLLBACK`
- Repeatable Read 的基础语义
- 当前读与一致性读区分
- undo 链和可见性判断
- 崩溃后未提交事务可回滚

## 继续演进方向

- 更高效的 undo 回收与 purge 并行化
- 长事务治理与快照观测
- 更细粒度隔离级别优化
- 提交分组和更复杂的 durability 策略

