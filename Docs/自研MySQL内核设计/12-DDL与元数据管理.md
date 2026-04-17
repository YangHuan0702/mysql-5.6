# DDL 与元数据管理

## 模块目标

这个模块负责定义数据库对象的结构和生命周期，包括：

- 表、索引、列定义
- 元数据缓存
- DDL 执行
- DDL 与事务、锁、恢复之间的关系

如果事务、索引和恢复是数据面的内核，那么 DDL 与元数据管理就是控制面的内核。接近生产可用的 MySQL 内核，不能把 DDL 理解成“改一张系统表”，而必须把它看作一条跨越 SQL 层、元数据层、缓存层、存储层和恢复层的复合操作链路。

## 当前仓库参照实现

关键参考：

- `sql/sql_table.cc`
- `sql/datadict.cc`
- `sql/table.cc`
- `sql/table_cache.cc`
- `sql/mdl.cc`
- `scripts/mysql_system_tables.sql`
- `sql/partition_element.h` 等 DDL log 相关结构

从当前仓库可以看到几个关键事实：

- DDL 一定和 MDL 强绑定
- 打开表路径与元数据缓存强绑定
- 存储引擎并不只接收“新定义”，还要参与物理变更
- 恢复阶段要考虑未完成 DDL 的清理或继续执行

## 如果你自己实现，应该怎么抽象

建议把 DDL 和元数据体系拆成四层：

1. `CatalogStore`
持久化数据字典。存储 schema、table、column、index 定义及版本信息。

2. `CatalogCache`
内存中的元数据对象缓存，向 SQL 层和执行器提供快速访问。

3. `DDLExecutor`
DDL 编排器。负责把用户语句变成一串可执行的元数据和物理变更步骤。

4. `DDLRecoveryManager`
负责未完成 DDL 的恢复、回滚或清理。

这种拆分的关键意义在于：DDL 的“逻辑定义变更”和“物理结构变更”必须能既协同又解耦。否则后面在线 DDL、崩溃恢复和缓存一致性会全面失控。

## 核心数据结构

- `SchemaDef`
  - `schema_id`
  - `name`
  - `version`

- `TableDef`
  - `table_id`
  - `schema_id`
  - `name`
  - `columns`
  - `indexes`
  - `engine_name`
  - `table_version`

- `IndexDef`
  - `index_id`
  - `table_id`
  - `index_type`
  - `key_parts`
  - `is_unique`

- `CatalogVersion`
  用于标识元数据对象版本与 cache invalidation 边界。

- `DDLOperation`
  - `ddl_id`
  - `ddl_type`
  - `target_object`
  - `state`
  - `old_version`
  - `new_version`
  - `physical_stage`

- `DDLObjectHandle`
  SQL 层打开表后持有的稳定元数据句柄，避免执行中对象被直接替换。

## 元数据存储设计

第一版建议采用“系统表持久化 + 内存缓存”的经典设计：

- 系统表存放权威定义
- 启动时加载必要元数据
- 打开表时按需装载到 cache

不要一开始把元数据直接散落在文件头或目录树里。原因很简单：

- schema object 会越来越多
- DDL 版本迁移需要统一存储
- 权限、视图、触发器、函数等对象迟早会进入同一套体系

## 元数据缓存设计

`CatalogCache` 至少需要解决 4 个问题：

1. 打开表时快速命中
2. 并发 DDL 时版本一致性
3. DDL 后缓存失效
4. 执行中对象句柄稳定性

建议采用：

- cache entry 带版本号
- 打开表时获取 `DDLObjectHandle`
- DDL 提交后发布新版本并失效旧版本
- 已经在执行中的语句继续持有旧 handle，直到执行结束

这样可以避免大量“执行中对象被热替换”的灾难性问题。

## DDL 状态机

DDL 必须显式建模状态机。建议第一版至少有：

- `INIT`
- `VALIDATED`
- `METADATA_UPDATED`
- `PHYSICAL_CHANGE_STARTED`
- `PHYSICAL_CHANGE_DONE`
- `CACHE_PUBLISHED`
- `COMMITTED`
- `FAILED`
- `ROLLED_BACK`

为什么需要这么细：

- 方便恢复
- 方便故障排查
- 方便后面扩展成 online DDL

如果只保留“开始 / 结束”两个状态，崩溃恢复时根本无法判断 DDL 走到哪一步。

## DDL 执行主路径

推荐将 DDL 执行拆成以下阶段：

1. 语义校验
   - 对象是否存在
   - 名称是否合法
   - 引擎是否支持该变更

2. 获取元数据锁
   - 通常是目标对象的排他 MDL
   - 必要时还要获取上层 schema 锁

3. 创建 DDL operation 记录
   - 分配 `ddl_id`
   - 初始化状态

4. 更新持久化元数据
   - 写入新版本定义
   - 记录旧版本到新版本的映射

5. 执行物理变更
   - 例如建表文件、建索引、重建表、变更页格式等

6. 发布缓存新版本
   - 失效旧 cache entry
   - 让新打开表路径命中新定义

7. 提交并清理 DDL operation

## DDL 与 MDL 的关系

DDL 的第一原则：没有稳定的元数据锁，就没有稳定的 DDL。

建议至少保证：

- DML 打开表时持有共享型 MDL
- DDL 执行时申请排他型 MDL
- DDL 在真正修改元数据前，必须等待所有冲突型 DML 释放句柄

这一点不能被“先做简单版”削弱，否则表定义在执行中被替换，会直接破坏执行器和缓存一致性。

## DDL 与存储引擎的关系

DDLExecutor 不应该直接操纵物理文件细节。建议通过存储引擎接口下推物理变更：

- `create_table(def)`
- `drop_table(def)`
- `build_index(def, index_def)`
- `alter_table(old_def, new_def, plan)`

这样才能保持：

- SQL 层只负责编排
- 引擎层只负责物理实现
- 不同引擎未来具备扩展空间

## DDL 日志与恢复

如果目标是接近生产可用，DDL 必须具备恢复语义。

建议引入 `DDLLog` 或等价机制，记录：

- DDL 唯一标识
- 当前状态
- 目标对象
- 已完成的物理步骤
- 回滚所需信息

恢复时建议按两类处理：

1. 可回滚 DDL
   恢复阶段清理中间产物并回退元数据。

2. 应继续完成的 DDL
   恢复阶段继续推进到可提交状态。

第一版不一定要支持“继续执行”，但至少要支持“识别残留中间态并清理”。

## DDL 分类建议

为了控制复杂度，建议把 DDL 分成三类：

### A 类：纯元数据 DDL

例如：

- 重命名注释
- 某些不涉及存储布局的属性修改

### B 类：轻量物理变更 DDL

例如：

- 创建新表
- 删除表
- 新增二级索引（如果采用离线构建）

### C 类：重建式 DDL

例如：

- 改列类型
- 调整主键
- 变更行格式

第一版建议把绝大多数复杂 `ALTER TABLE` 落到 C 类，统一采用“重建表”语义，而不是过早追求复杂 in-place DDL。

## 打开表路径与缓存一致性

`open_table` 路径是元数据体系里最容易被低估的关键点。

建议保证：

- 打开表总是从 `CatalogCache` 获取稳定句柄
- 若 cache miss，则从 `CatalogStore` 装载
- 若发生 DDL，则新语句看到新版本，旧语句继续持有旧 handle

不要让执行中语句在中途“重新取最新定义”，这会导致极难诊断的一致性 bug。

## 工程实现建议

- 第一版可以先做较保守的“阻塞式 DDL”，不要强行上在线 DDL。
- 元数据缓存一定要有版本概念，否则并发 DDL / DML 很难保证一致性。
- DDL 路径必须可恢复，至少要有中间状态清理策略。
- `CatalogStore` 与 `CatalogCache` 一开始就要解耦，不要把 cache 当成权威存储。
- 复杂 `ALTER TABLE` 第一版优先走“重建表”路径。

## 第一版应该做什么，不该做什么

### 应该做

- 系统表形式的数据字典
- 元数据缓存与版本号
- 阻塞式 DDL
- DDL operation / DDL log
- DDL 崩溃后中间态清理
- 打开表句柄稳定性

### 不该做

- 一开始就做完整 online DDL
- 一开始就做所有对象类型的事务化数据字典
- 一开始就做高度自动化的后台 schema 演进

## 最小可用版本

- `CREATE TABLE`
- `DROP TABLE`
- `CREATE INDEX`
- 受限 `ALTER TABLE`
- 表定义缓存
- DDL 日志与基础恢复

## 继续演进方向

- Online DDL
- 原子 DDL
- 更丰富的 schema object
- 更完善的数据字典事务化
- 跨对象依赖关系管理

