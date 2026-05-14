## 3.1 AT 模式实战

### 3.1.1 整体机制

AT 模式是 Seata 独创的无侵入分布式事务解决方案，核心是在数据库层插入代理逻辑。

```
第一阶段：
  1. 解析 SQL，确定操作类型和影响行
  2. 查询前置镜像（Before Image）
  3. 执行业务 SQL
  4. 查询后置镜像（After Image）
  5. 构建回滚日志并插入 UNDO_LOG 表
  6. 向 TC 申请全局锁
  7. 提交本地事务（业务 + UNDO_LOG 在同一个本地事务中）

第二阶段 - 提交：
  收到提交请求 → 放入异步队列 → 立即返回成功 → 后台批量清理 UNDO_LOG

第二阶段 - 回滚：
  收到回滚请求 → 根据 XID + BranchID 查询 UNDO_LOG
  → 验证后置镜像与当前数据一致
  → 根据前置镜像生成补偿 SQL → 执行回滚
```

### 3.1.2 数据源代理

AT 模式要求使用 Seata 提供的 `DataSourceProxy` 包装实际数据源：

```java
// 原生数据源
DataSource ds = new DruidDataSource();
// ... 配置连接信息

// Seata AT 数据源代理
DataSourceProxy dataSourceProxy = new DataSourceProxy(ds);
```

如果使用 Spring Boot，可通过配置自动注入：

```yaml
seata:
  enabled: true
  application-id: applicationName
  tx-service-group: my_test_tx_group
  enable-auto-data-source-proxy: true  # 默认 true，自动代理数据源
```

### 3.1.3 DataSourceProxy 拦截链路

当 Spring 管理的 `DataSource` 被 `DataSourceProxy` 包装后，所有 JDBC 操作的执行路径变为：

```java
// 简化源码路径（io.seata.rm.datasource.DataSourceProxy）
PreparedStatementProxy.executeUpdate()
  → ExecuteTemplate.execute()
    → 1. StatementRecognizer.recognize(sql)     // SQL 识别：INSERT/UPDATE/DELETE
    → 2. BaseTransactionalExecutor.execute()
      → 2a. buildBeforeImage(sql)               // SELECT * FROM table WHERE pk IN (...)
      → 2b. statementCallback.execute(sql)      // 执行原始 SQL
      → 2c. buildAfterImage(pkValues)           // SELECT * FROM table WHERE pk IN (...)
      → 2d. buildUndoItem(beforeImage, afterImage)  // 序列化为 JSON
      → 2e. UndoLogManager.insertUndoLog(undoItem)
      → 2f. ConnectionProxy.commit()            // 提交本地事务（含 UNDO_LOG 写入）
```

**关键设计决策**：
- before image 和 after image 通过**主键**定位，而非原始 WHERE 条件。这避免了二次查询时因条件不精确导致的行集变化。
- UNDO_LOG 与业务 SQL 在**同一个本地事务**中提交——如果 UNDO_LOG 写入失败，整个本地事务回滚，不会产生不一致。
- `only-care-update-columns: true`（2.0+）只记录被更新列的前后值，大幅减少 UNDO_LOG 体积。

### 3.1.4 核心模型源码视角

理解 TC/TM/RM 之间的**通信协议**和**状态流转**，是排查一切生产问题的起点。

```mermaid
graph TB
    TC["TC — Transaction Coordinator<br/>独立进程，Netty Server 监听 8091<br/>维护 global_table / branch_table / lock_table"]
    TM["TM — Transaction Manager<br/>@GlobalTransactional 标注的方法<br/>向 TC 发起 begin / commit / rollback"]
    RM["RM — Resource Manager<br/>每个 DataSourceProxy 实例<br/>注册分支、上报状态、执行二阶段"]

    TM -->|begin / commit / rollback| TC
    RM -->|register / report| TC
    TC -->|branch commit / rollback| RM
```

- **TC**：独立部署的 Java 进程（`seata-server`），基于 Netty 接收 TM/RM 的 RPC 请求。内部维护三张核心表：`global_table`（全局事务）、`branch_table`（分支事务）、`lock_table`（全局锁）。TC 是**无状态**的（状态存储在 DB 或 Raft Log 中），因此可以水平扩展。
- **TM**：`GlobalTransactionalInterceptor` 拦截 `@GlobalTransactional` 注解的方法，向 TC 发起 begin/commit/rollback。每个全局事务生成一个全局唯一的 **XID**（格式：`ip:port:timestamp`）。
- **RM**：每个 `DataSourceProxy` 实例对应一个 RM。它劫持 JDBC 的 `PreparedStatement.executeUpdate()`，在 SQL 执行前后分别生成 before image 和 after image，并向 TC 注册分支事务。

**TC 通信细节**：TM/RM 与 TC 之间通过 Netty 建立长连接（默认端口 8091），协议采用自定义的 Protobuf 编码。关键设计点：
- **心跳检测**：默认开启，防止连接假死导致事务悬挂
- **Channel 复用**：同一个 JVM 进程中的所有 TM/RM 实例共享一个 Netty Channel 到 TC，减少连接开销
- **批量发送**：`transport.enableTcServerBatchSendResponse(true)` 可解决多个响应串行发送的线头阻塞问题（2.x 建议开启）

### 3.1.5 全局锁与写隔离

```
案例：两个事务 tx1 和 tx2 同时更新 m 字段（原值 1000）

tx1 先到达：
  1. 开始本地事务，获取本地锁
  2. 执行更新：m = 1000 - 100 = 900
  3. 获取全局锁（成功）
  4. 提交本地事务，释放本地锁

tx2 到达：
  1. 开始本地事务，获取本地锁
  2. 执行更新：m = 900 - 100 = 800
  3. 获取全局锁（被 tx1 持有，失败重试）
  4. 重试中...

tx1 执行全局提交，释放全局锁

tx2 获取全局锁成功，提交本地事务
```

AT 模式的写隔离依赖 TC 的 `lock_table`。每次分支注册时，RM 上报本次修改的**表名 + 主键值**作为 lock key。

```sql
-- TC 端的 lock_table 结构
CREATE TABLE `lock_table` (
  `row_key` VARCHAR(128) NOT NULL,      -- 格式: {resourceId}^^^{tableName}^^^{pk}
  `xid` VARCHAR(128) NOT NULL,
  `branch_id` BIGINT NOT NULL,
  `table_name` VARCHAR(32),
  `pk` VARCHAR(36),
  PRIMARY KEY (`row_key`)
);
```

**全局锁的生命周期**：

```mermaid
sequenceDiagram
    participant RM1 as RM（事务 A）
    participant TC as TC
    participant RM2 as RM（事务 B）

    Note over RM1,RM2: 两个全局事务同时更新 product.id=1

    RM1->>TC: registerBranch(xid_A, lockKey="product^^^1")
    TC->>TC: INSERT INTO lock_table
    TC-->>RM1: 注册成功，全局锁获取

    RM2->>TC: registerBranch(xid_B, lockKey="product^^^1")
    TC->>TC: INSERT INTO lock_table → DUPLICATE KEY!
    TC-->>RM2: LockConflictException

    Note over RM2: 策略：本地事务 rollback → sleep → 重新执行
    Note over RM2: （release local lock, then retry）

    RM1->>TC: commit(xid_A)
    TC->>TC: DELETE FROM lock_table WHERE xid='xid_A'
    TC-->>RM1: 全局锁释放

    RM2->>TC: 重试 registerBranch → 成功
```

**关键规则**：
- 提交本地事务**之前**必须先获取到全局锁
- 获取全局锁失败则不断重试（有超时限制）
- 超时未获取到全局锁 → 回滚本地事务，释放本地锁
- 回滚阶段若本地锁被其他事务持有 → 重试获取本地锁

**为什么一阶段要先释放本地锁再获取全局锁？** 如果 RM2 持有行锁的同时等待全局锁，而 RM1 回滚时需要行锁来执行反向 SQL，就会形成**跨层死锁**。Seata 的策略是：全局锁获取失败后，**立刻回滚本地事务、释放行锁**，然后轮询重试。

重试机制的核心参数：
- `client.rm.lock.retry-interval`：重试间隔（默认 10ms）
- `client.rm.lock.retry-times`：重试次数（默认 30 次）
- 总等待窗口：30 × 10ms = **300ms**，超过则抛 `LockConflictException`

**生产建议**：对于热点数据（如秒杀库存扣减），300ms 往往不够。但盲目增大重试次数会导致请求堆积。更好的解法是：业务层面拆分热点（如库存分片），或在 TM 层对全局事务做并发控制。

### 3.1.6 读隔离

AT 模式默认工作在 **READ UNCOMMITTED** 级别。原因：一阶段已提交本地事务，数据已写盘，但全局事务可能尚未最终提交（二阶段异步执行中），其他事务此时读到该数据即为"脏读"。

**两种隔离升级方案**：

| 方案 | 机制 | 成本 | 适用场景 |
|------|------|------|----------|
| `@GlobalLock` | 读前向 TC 检查全局锁，有锁则等待 | 仅一次 RPC，约 2ms 延迟 | 纯读路径，避免脏读 |
| `SELECT FOR UPDATE` | 先等全局锁释放，再获取本地行锁 | 行锁持有至事务结束 | 读后写路径，需要 REPEATABLE READ |

```java
// @GlobalLock：纯读接口，避免读到未提交的全局事务数据
@GlobalLock
@Transactional(readOnly = true)
public ProductDTO query(Long id) {
    return productMapper.selectById(id);
}

// SELECT FOR UPDATE：读后写路径，Seata 代理了 FOR UPDATE
@GlobalTransactional
public void deductStock(Long id, Integer qty) {
    Stock stock = stockMapper.selectForUpdate(id);  // 自动检查全局锁
    if (stock.getCount() < qty) throw new RuntimeException("库存不足");
    stockMapper.deduct(id, qty);
}
```

**@GlobalLock 的局限**：
1. 只能保证单行"当前已提交"，不能跨行保证一致性——A+B 总和的约束无法在 @GlobalLock 层面保障
2. 不能保证 REPEATABLE READ——两次 `@GlobalLock` 之间，数据可能被其他事务修改
3. 仅对本地 SQL 生效，无法影响远程服务的读取

> 出于性能考虑，Seata 只代理 `SELECT FOR UPDATE` 语句，普通 `SELECT` 不做代理。

### 3.1.7 AT 模式的使用场景与限制

**适用场景**：
- 单体应用平移到微服务架构，希望最低改造成本
- 使用关系型数据库（MySQL、PostgreSQL、Oracle 等）
- 业务逻辑通过 JDBC 访问数据库

**限制**：
- 仅支持关系型数据库
- 不适用于非 JDBC 数据源（如 Redis、MongoDB）
- 大事务下 UNDO_LOG 可能较大

### 3.1.8 脏写检测机制

Seata 在回滚时会校验数据是否被其他事务"脏写"，以决定能否安全回滚。

**脏写检测流程**：

```mermaid
flowchart TB
    A["从 undo_log 读取 before & after image"] --> B["SELECT 当前数据库数据"]
    B --> C{"当前数据 == after image？"}
    C -->|"一致"| D["用 before image 生成反向 SQL 并执行"]
    D --> E["DELETE FROM undo_log"]
    C -->|"不一致"| F["log_status = 1（defense 状态）"]
    F --> G["人工介入处理"]
```

**脏写的根本原因**：有代码绕过了 `DataSourceProxy` 直接操作数据库。典型场景：
- 定时任务使用独立的未代理 DataSource
- DBA 手动执行 UPDATE
- 批处理程序使用独立连接池

**防御策略**：
- TC 端配置 `server.undo.logSaveDays=7`，保留 defense 记录 7 天
- 监控 `undo_log WHERE log_status=1` 的数量并告警
- 审计所有数据库连接，确保都经过 DataSourceProxy

### 3.1.9 AT 模式的 SQL 限制

| 限制 | 说明 | 替代方案 |
|------|------|----------|
| 不支持存储过程/触发器 | Seata 无法解析内部 SQL | 改为应用层代码 |
| 不支持多表复杂 JOIN | 部分版本不支持 `UPDATE t1 JOIN t2` | 拆分为单表操作 |
| 不支持 SQL 嵌套 | 子查询中的 UPDATE 等 | 改写 SQL |
| 批量更新 | MySQL/MariaDB/PostgreSQL9.6+ 支持 `jdbcTemplate.batchUpdate` | 使用支持的 API |
| 跨库操作 | `DataSourceProxy` 只能代理一个 DataSource，跨库需多个 RM | 应用层聚合或改用 TCC/SAGA |

