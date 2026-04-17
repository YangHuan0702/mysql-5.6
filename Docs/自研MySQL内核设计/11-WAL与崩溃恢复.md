# WAL 与崩溃恢复

## 模块目标

该模块负责在系统崩溃、电源中断、进程异常退出后保证数据文件恢复到一致状态。没有这层，数据库无法接近生产可用。

它需要明确回答：

- 什么是必须先持久化的日志，什么是可以延后落盘的数据页
- checkpoint 到底记录什么，恢复从哪里开始
- 崩溃后如何知道哪些修改已经 durable，哪些只是内存中的幻影
- 如何在恢复阶段重做已提交修改并回滚未完成事务

## 当前仓库参照实现

关键参考：

- `storage/innobase/log/`
- `storage/innobase/trx/`
- `storage/innobase/buf/`
- `storage/innobase/srv/`
- `storage/innobase/page/`

从当前仓库看，InnoDB 风格恢复路径依赖 redo、checkpoint、脏页刷盘、页内 LSN、事务状态与 undo 回滚协作。真正的工程重点不在“恢复代码有多长”，而在这些元信息之间的一致约束。

## 如果你自己实现，应该怎么抽象

建议实现标准的 WAL 体系，并将其拆为 4 个子系统：

1. `LogManager`
负责 redo record 编码、log buffer、刷盘和 LSN 分配。

2. `CheckpointManager`
负责维护 recovery 起点，并协调脏页推进。

3. `FlushManager`
负责脏页刷盘、刷盘队列、后台 page cleaner。

4. `RecoveryManager`
负责启动恢复、日志扫描、redo replay 和未完成事务回滚。

如果把这四件事都塞进一个“日志模块”，后面很快就会失控。

## 核心数据结构

- `LogSequenceNumber`
  全局递增 LSN，表示 redo 流中的稳定位置。

- `RedoRecord`
  至少包含：
  - `lsn`
  - `page_id`
  - `record_type`
  - `payload`
  - `checksum`

- `LogBuffer`
  内存中的 redo 暂存区。

- `Checkpoint`
  至少包含：
  - `checkpoint_lsn`
  - `oldest_dirty_page_lsn`
  - `checkpoint_no`

- `DirtyPageEntry`
  记录页 ID、最早脏化 LSN、最近修改时间等。

- `RecoveryContext`
  恢复阶段状态，包括扫描位置、已提交事务集合、待回滚事务集合。

## 核心不变量

1. 数据页落盘前，对应 redo 必须已经 durable。
2. 页头记录的 `page_lsn` 必须大于等于该页已应用的最后一条 redo 的 LSN。
3. checkpoint 只能推进到“其之前所有必要 redo 均已 durable，且恢复可从此重新开始”的位置。
4. 恢复时，redo replay 必须幂等；重复回放同一条 redo 不应破坏页状态。

## WAL 设计要点

### 为什么必须先写日志

因为数据库需要允许“脏页延迟落盘”。如果每次事务提交都要求数据页同步写盘，吞吐会被随机 I/O 彻底打死。WAL 的核心价值是：

- 前台事务只需确保 redo durable
- 数据页可以由后台异步刷盘
- 崩溃后根据 redo 重放到一致状态

### Redo 记录粒度

第一版建议做“页内逻辑修改”或“物理修改片段”两种之一，但必须保证：

- 可回放
- 可校验
- 幂等

不要在第一版做过于抽象的高层逻辑 redo，否则恢复调试极其困难。

## 前台写路径

一次事务修改建议走这条路径：

1. 获得目标页并加页 latch
2. 修改页内容
3. 生成 redo record
4. 将 redo 追加到 log buffer，并分配连续 LSN
5. 更新页头 `page_lsn`
6. 标记页为 dirty
7. 事务提交时确保 redo flush 到 durable 介质

注意顺序：

- 先生成 redo，再释放对该页的保护
- 页头 `page_lsn` 必须和 redo 流中的位置建立对应关系

## Checkpoint 设计

checkpoint 的本质不是“写一个标记”，而是声明：

> 从某个 LSN 之前的历史开始，恢复已经不需要再追溯更早的 redo。

建议第一版采用保守 checkpoint 策略：

- 维护最老脏页对应的 LSN
- 周期性推动后台刷盘
- 当足够早的脏页都落盘后，推进 checkpoint

这比一开始就做复杂的 fuzzy checkpoint 更稳。

## 崩溃恢复流程

建议恢复流程拆成三个阶段：

### 1. Analysis

目标：

- 找到最近 checkpoint
- 确定需要扫描的 redo 范围
- 重建活动事务、脏页和恢复上下文

### 2. Redo

目标：

- 顺序扫描 redo
- 对每条记录检查目标页的 `page_lsn`
- 仅对尚未反映到页中的修改执行 replay

幂等判断通常类似：

- 如果 `page_lsn >= redo_lsn`，跳过
- 否则应用 redo，并推进 `page_lsn`

### 3. Undo

目标：

- 找出未提交事务
- 根据 undo 链逐个回滚未完成修改

恢复阶段的 undo 与运行时回滚共享大量语义，因此事务设计必须和恢复设计统一。

## Group Commit 与 durability 策略

第一版不要一开始就过度优化 group commit，但应该在设计上预留：

- 一个专门的 commit coordinator
- 多事务可共享一次 flush 的机制

推荐第一版先实现：

- 简单的 commit queue
- flush 完成后统一唤醒等待事务

这样后面升级到更高吞吐不需要推翻接口。

## 双写保护与页校验

如果目标是“接近生产可用”，仅 redo 还不够，还要考虑 torn page 和部分页写入问题。

建议文档里明确两层保护：

- 页校验和
- 双写缓冲或等价机制

第一版如果不做完整双写，也至少要把页校验做起来，否则恢复阶段很难判定页是否损坏。

## 工程实现建议

- 第一版就定义清楚 `page_lsn` 和 `log_lsn` 的关系。
- redo record 编码要稳定、可校验、可回放。
- redo replay 必须设计成幂等操作。
- 恢复上下文、活动事务表和 undo 结构要共享统一对象模型。
- 不要一开始就过度优化 group commit；先保证恢复逻辑绝对正确。

## 第一版应该做什么，不该做什么

### 应该做

- redo log
- log buffer
- checkpoint
- page lsn
- redo replay
- 未提交事务回滚
- 页校验

### 不该做

- 一开始就做极复杂 log format 压缩
- 一开始就做极激进 fuzzy checkpoint
- 一开始就做太多 durability 模式组合

## 最小可用版本

- redo log
- 周期性 checkpoint
- 崩溃后 redo replay
- 未提交事务回滚
- 基础页校验

## 继续演进方向

- 更高效的 group commit
- 更复杂的 fuzzy checkpoint
- 更强的双写保护
- 备份、恢复和增量恢复工具链对接

