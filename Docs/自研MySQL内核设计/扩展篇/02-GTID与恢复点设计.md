# GTID 与恢复点设计

## 模块目标

GTID 解决“事务身份”和“复制恢复点”问题，是复制体系的关键标识层。

## 当前仓库参照实现

- `sql/rpl_gtid*.cc`
- `sql/rpl_gtid.h`

## 如果你自己实现，应该怎么抽象

将 GTID 设计为独立事务标识，不与本地事务 ID 混用。事务 ID 面向本机并发控制，GTID 面向复制拓扑和恢复点。

## 核心数据结构

- `GTID`
- `GTIDSet`
- `ReplicationPosition`

## 关键流程

1. 事务提交前分配 GTID
2. GTID 写入 binlog
3. 副本消费时更新已执行 GTID 集合
4. 故障恢复基于 GTID 定位同步起点

## 工程实现建议

- GTID 集合操作要高效，否则拓扑大时成本很高
- 本地事务 ID 与 GTID 必须分层

## 最小可用版本

- 单源 GTID
- 已执行 GTID 集合
- GTID 基础恢复点

## 继续演进方向

- 多源复制
- GTID 压缩与存储优化

