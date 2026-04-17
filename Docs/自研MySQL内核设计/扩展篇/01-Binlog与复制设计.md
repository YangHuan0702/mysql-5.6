# Binlog 与复制设计

## 模块目标

该模块不属于单机第一阶段必做能力，但它是后续复制与外部 CDC 的基础。核心问题是：

- 哪些事务需要进入 binlog
- binlog 与事务提交顺序如何保持一致
- row / statement / mixed 模式如何支持

## 当前仓库参照实现

- `sql/binlog.cc`
- `sql/log_event.cc`
- `sql/sql_binlog.cc`
- `sql/rpl_*`

## 如果你自己实现，应该怎么抽象

建议把 binlog 设计成独立日志子系统，而不是 redo 的一个附属字段。它服务的是复制与外部消费，不是页恢复。

## 核心数据结构

- `BinlogEvent`
- `BinlogTxnContext`
- `CommitSequence`

## 关键流程

1. 事务执行期间收集变更
2. 提交阶段构造 binlog event
3. 与事务提交顺序对齐
4. 持久化并对外暴露位点

## 工程实现建议

- 第一版优先 row-based
- 明确 binlog 和 redo 的职责边界
- 提交顺序一致性必须早设计

## 最小可用版本

- row-based binlog
- 提交顺序一致
- 位点推进

## 继续演进方向

- mixed 模式
- 外部 CDC 订阅
- 更高效 group commit 对接

