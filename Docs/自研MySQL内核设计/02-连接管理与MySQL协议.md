# 连接管理与 MySQL 协议

## 模块目标

该模块负责处理客户端接入、握手、认证、命令分发和连接生命周期管理。生产可用的单机内核不需要一开始就实现所有 MySQL 协议细节，但必须具备：

- 稳定的握手和认证流程
- 清晰的会话对象模型
- 请求与响应的协议编解码
- 连接级资源隔离和超时控制

## 当前仓库参照实现

主要参考：

- `sql/mysqld.cc`: 建立监听与连接处理线程
- `sql/sql_parse.cc`: `dispatch_command()`
- `sql/sql_connect.cc`: 连接处理相关逻辑
- `sql/protocol.cc`: 协议编码与结果集输出
- `sql/net_serv.cc`, `sql-common/client.c`, `libmysql/`: 网络和客户端协议实现
- `plugin/auth/`: 认证插件能力

## 如果你自己实现，应该怎么抽象

建议把协议层拆成 4 个对象：

- `Connection`: 纯网络连接和读写缓冲
- `Session`: 用户、库名、系统变量、事务、当前语句状态
- `CommandDispatcher`: 按协议命令类型分发，如 `COM_QUERY`, `COM_INIT_DB`
- `ProtocolEncoder`: 错误包、OK 包、列定义、行数据等编码

不要把网络套接字、会话状态和 SQL 执行逻辑塞在一个类里。

## 核心数据结构

- `HandshakePacket`
- `AuthContext`
- `Connection`
- `Session`
- `Command`
- `ProtocolWriter`
- `ResultSetMetadata`

## 关键流程

1. accept 新连接
2. 发送 handshake packet
3. 接收认证信息并完成认证
4. 创建 `Session`
5. 进入命令循环
6. 对每个命令执行分发、执行、响应输出
7. 连接关闭时释放会话资源、事务资源和引擎句柄

## 工程实现建议

- 协议层一定要做收发包大小控制，避免大包直接打垮前台线程。
- `Session` 必须是贯穿 SQL 层、事务层和存储层的主线对象。
- 第一版尽量先用“一连接一线程”或受控线程池，避免过早引入复杂事件驱动框架。
- 认证逻辑和权限检查要分层：认证解决“你是谁”，权限解决“你能做什么”。

## 最小可用版本

- TCP 监听
- 基础 MySQL 握手
- 用户认证
- `COM_QUERY`
- `COM_INIT_DB`
- `COM_PING`
- 基础错误返回与结果集输出

## 继续演进方向

- 认证插件扩展
- 连接池友好的协议优化
- 压缩协议
- 连接级限流、配额和 admission control

