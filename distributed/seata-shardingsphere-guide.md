## Seata AT 模式与 ShardingSphere 分库分表集成实战

### 背景：为什么需要整合？

在企业级微服务架构中，**分库分表**与**分布式事务**是两个独立但常常同时出现的需求：

- **分库分表**（ShardingSphere）：解决单库数据量过大后的水平扩展问题，将数据分散到多个数据库实例
- **分布式事务**（Seata AT）：解决跨库/跨服务的数据一致性问题

当二者共存时，核心矛盾在于：Seata 的 `DataSourceProxy` 需要拦截实际的数据源来完成 undo_log 和全局锁的生成，而 ShardingSphere 的 `ShardingSphereDataSource` 本身也是一个虚拟数据源，它对物理数据源进行聚合和路由。**数据源的嵌套代理顺序和 SQL 解析链路的兼容性**，是整合的关键。

```
┌────────────────────────────────────────────────────┐
│                  业务层（@GlobalTransactional）       │
├────────────────────────────────────────────────────┤
│           ShardingSphereDataSource（逻辑数据源）      │
│           ┌──────────────────────────────────┐      │
│           │  SQL 解析 → 路由 → 改写 → 执行     │      │
│           └──────────────────────────────────┘      │
├────────────────────────────────────────────────────┤
│      Seata DataSourceProxy（AT 模式代理层）          │
│      ┌────────────────────────────────────┐         │
│      │  BeforeImage → 执行业务SQL → AfterImage     │
│      │  → UNDO_LOG → 注册分支 → 全局锁             │
│      └────────────────────────────────────┘         │
├────────────────────────────────────────────────────┤
│        物理数据源 ds0 / ds1（实际数据库实例）          │
└────────────────────────────────────────────────────┘
```

### 架构原理：Seata AT 如何融入 ShardingSphere 事务 SPI

ShardingSphere 提供了 `io.apache.shardingsphere.transaction.spi.ShardingSphereTransactionManager` SPI，用于接入第三方分布式事务实现。Seata AT 通过 `SeataATShardingSphereTransactionManager` 实现该 SPI 接入：

```java
// ShardingSphere 事务 SPI 接口（简化）
public interface ShardingSphereTransactionManager extends AutoCloseable {

    // 初始化事务管理器（绑定 DataSource）
    void init(DatabaseType databaseType, Collection<Resource> resources);

    // 获取事务类型：LOCAL / XA / BASE
    TransactionType getTransactionType();

    // 判断当前是否在分布式事务中
    boolean isInTransaction();

    // 开启事务（绑定 XID）
    void begin(Connection connection, @Nullable Resource resource);

    // 提交/回滚
    void commit(Connection connection, @Nullable Resource resource);
    void rollback(Connection connection, @Nullable Resource resource);
}
```

**整合关键点**：

1. **数据源感知**：ShardingSphere 将用户配置的物理 DataSource 包装为 `ShardingSphereDataSource`。Seata 的 `DataSourceProxy` 需要包装的是每个物理 DataSource，而非包装整个 `ShardingSphereDataSource`
2. **XID 传播**：ShardingSphere 的分片执行引擎是多线程的（多个分片并行执行），Seata 的 XID 存储在 `RootContext`（基于 `ThreadLocal`），需要在主线程与子线程间显式传递
3. **事务类型切换**：在 `@GlobalTransactional` 方法内，需要将 ShardingSphere 的 Connection 事务类型从 LOCAL 切换为 BASE（Seata AT），通过 `ShardingSphereConnection.setAutoCommit(false)` 触发

#### 两种集成路径

| 集成方式 | 说明 | 适用场景 |
|---------|------|---------|
| **ShardingSphere-JDBC + Seata 原生** | 业务代码手动将物理 DataSource 包装为 Seata 的 DataSourceProxy，再交给 ShardingSphere 聚合 | 精细化控制，需自行处理数据源编排 |
| **ShardingSphere-JDBC + Seata SPI** | 利用 ShardingSphere 提供的 Seata 事务 SPI（`shardingsphere-transaction-base-seata-at`），自动集成 | 配置简单，官方维护的集成方案 |

**推荐使用第二种方案**（ShardingSphere 官方 SPI），除非你有特殊的定制需求。

### 实战案例：订单分库分表 + 分布式事务

#### 场景描述

假设有一个电商订单系统，`t_order` 表数据量极大，按 `user_id` 分库分表：
- 2 个数据库实例：`ds0`、`ds1`
- 每个库 2 张表：`t_order_0`、`t_order_1`
- 分片键：`user_id`（库：`user_id % 2`，表：`(user_id / 2) % 2`）

下单时需要在分布式事务中同时操作：
1. `t_order` 插入订单（分片到不同库）
2. `t_account` 扣减余额（在 account 库）
3. `t_stock` 扣减库存（在 stock 库）

这涉及 3 个数据库实例 + 分片，必须依赖 Seata AT 保证跨库事务一致性。

#### 项目结构

```
order-app
├── pom.xml
├── src/main/resources
│   ├── shardingsphere.yaml          # ShardingSphere 分片配置
│   ├── seata.conf                   # Seata 客户端配置
│   ├── application.yaml
│   └── undo_log.sql                 # 每个分片库都需要执行
└── src/main/java/com/example/order
    ├── OrderApplication.java
    ├── service/OrderService.java    # 核心业务（含 @GlobalTransactional）
    ├── entity/Order.java
    ├── mapper/OrderMapper.java
    └── config/SeataConfig.java      # 数据源编排（关键！）
```

#### 依赖配置（pom.xml）

```xml
<!-- ShardingSphere JDBC 核心 -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core</artifactId>
    <version>5.4.1</version>
</dependency>

<!-- ShardingSphere Seata AT 集成 SPI（关键！） -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-transaction-base-seata-at</artifactId>
    <version>5.4.1</version>
</dependency>

<!-- Seata 客户端 -->
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-all</artifactId>
    <version>1.8.0</version>
</dependency>
```

**版本兼容矩阵**（重要）：

| ShardingSphere 版本 | Seata 版本 | 说明 |
|--------------------|-----------|------|
| 5.3.x | 1.6.x ~ 1.8.x | 兼容性较好 |
| 5.4.x | 1.7.x ~ 2.x | 推荐组合 |
| 5.5.x | 2.x | 最新版本，建议升级 |
| 4.x | 1.4.x ~ 1.5.x | 旧版，需自行处理数据源代理 |

#### ShardingSphere 配置

```yaml
# shardingsphere.yaml
dataSources:
  ds0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/order_ds0
    username: root
    password: root
  ds1:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/order_ds1
    username: root
    password: root

rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds${0..1}.t_order_${0..1}
        tableStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: t_order_table_inline
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: t_order_db_inline
        keyGenerateStrategy:
          column: id
          keyGeneratorName: snowflake

    shardingAlgorithms:
      t_order_db_inline:
        type: INLINE
        props:
          algorithm-expression: ds${user_id % 2}
      t_order_table_inline:
        type: INLINE
        props:
          algorithm-expression: t_order_${(user_id / 2) % 2}

    keyGenerators:
      snowflake:
        type: SNOWFLAKE

props:
  sql-show: true
```

#### Seata 配置

```yaml
# seata.conf / application.yaml
seata:
  enabled: true
  application-id: order-app
  tx-service-group: default_tx_group
  enable-auto-data-source-proxy: false   # 关键！必须关闭自动代理
  service:
    vgroup-mapping:
      default_tx_group: default
    grouplist:
      default: 127.0.0.1:8091

  client:
    rm:
      lock:
        retry-interval: 10
        retry-times: 30
      table-meta-checker-enable: false    # 分片场景下建议关闭
    tm:
      commit-retry-count: 5
      rollback-retry-count: 5

  transport:
    type: TCP
    enable-tc-server-batch-send-response: true
```

**`enable-auto-data-source-proxy: false`** 是整合 ShardingSphere 时的关键配置。如果开启，Seata 会自动包装 Spring 容器中所有的 DataSource Bean，包括 ShardingSphereDataSource 本身，导致**多层代理**问题。正确做法是手动编排数据源代理。

#### 数据源编排（核心）

这是 Seata + ShardingSphere 整合中**最重要、最容易出错**的环节。正确的代理顺序：

```
正确：物理DataSource → Seata DataSourceProxy → ShardingSphereDataSource
错误：物理DataSource → ShardingSphereDataSource → Seata DataSourceProxy ❌
```

```java
@Configuration
public class SeataConfig {

    /**
     * 正确的数据源编排：
     * 1. 先创建物理数据源（HikariCP/Druid）
     * 2. 将每个物理数据源包装为 Seata DataSourceProxy
     * 3. 将 DataSourceProxy 列表交给 ShardingSphere 创建逻辑数据源
     */
    @Bean
    @Primary
    public DataSource shardingSphereDataSource() throws Exception {
        // 1. 创建物理数据源
        HikariDataSource ds0 = new HikariDataSource();
        ds0.setJdbcUrl("jdbc:mysql://localhost:3306/order_ds0");
        ds0.setUsername("root");
        ds0.setPassword("root");

        HikariDataSource ds1 = new HikariDataSource();
        ds1.setJdbcUrl("jdbc:mysql://localhost:3306/order_ds1");
        ds1.setUsername("root");
        ds1.setPassword("root");

        // 2. ★关键步骤：将物理数据源包装为 Seata DataSourceProxy
        DataSourceProxy seataDs0 = new DataSourceProxy(ds0);
        DataSourceProxy seataDs1 = new DataSourceProxy(ds1);

        // 3. 将 Seata 代理后的数据源放入 ShardingSphere 配置
        Map<String, DataSource> dataSourceMap = new HashMap<>();
        dataSourceMap.put("ds0", seataDs0);
        dataSourceMap.put("ds1", seataDs1);

        // 4. 创建 ShardingSphereDataSource
        ShardingSphereDataSource dataSource = ShardingSphereDataSourceFactory
            .createDataSource(dataSourceMap, getShardingConfig(), new Properties());

        return dataSource;
    }

    private YamlShardingRuleConfiguration getShardingConfig() {
        // 读取 shardingsphere.yaml 中的分片规则
        // 或直接使用 Java API 构建
        return ...;
    }
}
```

#### 核心业务代码

```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private AccountClient accountClient;

    @Autowired
    private StockClient stockClient;

    /**
     * 下单：跨库分布式事务 + 分片写入
     * user_id 用于路由到正确的分片
     */
    @GlobalTransactional(name = "order-create", timeoutMills = 30000)
    public Order createOrder(OrderDTO dto) {
        // 1. 插入订单（通过 ShardingSphere 路由到对应分片）
        Order order = new Order();
        order.setUserId(dto.getUserId());
        order.setProductId(dto.getProductId());
        order.setAmount(dto.getAmount());
        order.setStatus(0);
        orderMapper.insert(order);  // 自动路由到 ds{user_id%2}.t_order_{(user_id/2)%2}

        // 2. 扣减账户余额（远程服务，在 account 库）
        accountClient.deduct(dto.getUserId(), dto.getAmount());

        // 3. 扣减库存（远程服务，在 stock 库）
        stockClient.deduct(dto.getProductId(), dto.getQuantity());

        return order;
    }
}
```

### 核心原理：数据源代理链深度解析

#### Seata 的代理机制

Seata AT 模式通过 `DataSourceProxy` → `ConnectionProxy` → `PreparedStatementProxy` 三层代理，拦截 JDBC 操作：

```java
public class DataSourceProxy extends AbstractDataSourceProxy {

    private final DataSource targetDataSource;  // 实际被代理的数据源

    @Override
    public ConnectionProxy getConnection() throws SQLException {
        // 返回 ConnectionProxy，而非原生 Connection
        Connection targetConnection = targetDataSource.getConnection();
        return new ConnectionProxy(this, targetConnection);
    }
}
```

`ConnectionProxy` 在 `commit()` 时，会插入 undo_log 和注册分支事务：

```java
public class ConnectionProxy extends AbstractConnectionProxy {

    @Override
    public void commit() throws SQLException {
        if (context.isGlobalTransaction()) {
            // 全局事务中：生成 undo_log + 注册分支 + 获取全局锁
            processGlobalTransactionCommit();
        } else {
            // 本地事务：直接提交
            targetConnection.commit();
        }
    }
}
```

#### ShardingSphere 的 SQL 执行流程

当 Seata 代理的数据源被 ShardingSphere 聚合后，SQL 执行流程变为：

```java
// 1. 业务层调用 orderMapper.insert(order)
// 2. ShardingSphere 解析 SQL，根据分片键 user_id 路由到 ds0.t_order_0
// 3. ShardingSphere 通过 Seata 代理后的 DataSourceProxy 获取 ConnectionProxy
// 4. ConnectionProxy 感知到全局事务（RootContext.getXID() != null）
// 5. 执行 SQL：PreparedStatementProxy.executeUpdate()
// 6. Seata 拦截：生成 BeforeImage → 执行 SQL → 生成 AfterImage → 写入 UNDO_LOG
// 7. ConnectionProxy.commit() → 注册分支事务到 TC → 提交本地事务
```

#### 为什么物理 DataSource 必须在 Seata 代理之后交给 ShardingSphere？

Seata 的 `DataSourceProxy` 需要拦截 `getConnection()` 返回 `ConnectionProxy`，而 ShardingSphere 在路由 SQL 后，会调用实际数据源的 `getConnection()` 获取物理连接来执行 SQL。如果 Seata 代理在 ShardingSphere 外层：

```
错误顺序：
DataSourceProxy(ShardingSphereDataSource(物理ds0, 物理ds1))
    → getConnection() 返回 ConnectionProxy，但 ShardingSphereConnection 不是真正可执行的连接
    → Seata 无法拦截分片后到各物理库的 SQL
    → undo_log 和分支注册全部失效
```

**Seata 的 `DataSourceProxy` 必须直接包装真正的 `java.sql.DataSource`，而不是包装逻辑数据源。**

#### ShardingSphere 官方 SPI 的原理

`shardingsphere-transaction-base-seata-at` 包实现了 `ShardingSphereTransactionManager` SPI：

```java
public class SeataATShardingSphereTransactionManager implements ShardingSphereTransactionManager {

    private final SeataTransactionHolder holder = new SeataTransactionHolder();

    @Override
    public void begin(Connection connection, @Nullable Resource resource) {
        // 从 ShardingSphereConnection 中获取 XID（通过 Seata 的上下文）
        String xid = RootContext.getXID();
        if (xid == null) {
            throw new TransactionException("XID not found, check @GlobalTransactional");
        }
        // 将 Connection 的事务类型标记为 BASE
        // 后续 ShardingSphere 执行分片 SQL 时，会通过 Seata 代理的 DataSource 提交
    }

    @Override
    public void commit(Connection connection, @Nullable Resource resource) {
        // Seata AT 的 commit 由 TC 协调，这里只需同步本地事务
        connection.commit();
    }
}
```

当 ShardingSphere 路由 SQL 时，如果检测到当前 Connection 的事务类型为 BASE（Seata AT），就会调用 `SeataATShardingSphereTransactionManager` 来管理事务生命周期。

### 实战问题与解决方案

#### 问题 1：数据源代理顺序错误导致 undo_log 未生成

**现象**：全局事务正常开启，业务 SQL 正常执行，但 `undo_log` 表中没有数据，回滚时无法补偿。

**根因**：Seata 的 `enable-auto-data-source-proxy: true` 自动包装了 `ShardingSphereDataSource`，导致 Seata 代理的是逻辑数据源而非物理数据源。分片后的 SQL 绕过了 Seata 的拦截。

**解决方案**：

```yaml
seata:
  enable-auto-data-source-proxy: false   # 关闭自动代理
```

手动编排数据源（见上文"数据源编排"章节），确保 `DataSourceProxy` 包装的是物理 DataSource。

**验证方法**：在任意分片上执行 SQL 检查 UNDO_LOG 是否生成：

```sql
-- 每个分片库上执行
SELECT COUNT(*) FROM undo_log;
```

如果全局事务执行后 `COUNT(*) = 0`，说明 Seata 未正确代理数据源。

#### 问题 2：undo_log 表的分片策略

**现象**：运行时报错 `Table 'undo_log' does not exist` 或 `Cannot find table meta of undo_log`。

**根因**：Seata AT 要求每个分片库中都有 `undo_log` 表。如果 ShardingSphere 将 `undo_log` 当作分片表进行了路由，会报错。

**解决方案**：将 `undo_log` 配置为**广播表**（broadcast table），确保每条 SQL 在每个分片库中都执行：

```yaml
rules:
  - !SHARDING
    # ... 分片表配置

  - !BROADCAST
    tables:
      - undo_log          # undo_log 在所有分片库中全量存在
```

**或者**：如果你使用的是单表模式（非广播表），可以设置 `undo_log` 为默认数据源的单表：

```yaml
rules:
  - !SINGLE
    defaultDataSource: ds0   # undo_log 只存在于 ds0
```

但**强烈建议使用广播表方案**，因为 Seata 的二阶段提交可能在任意分片上执行，`undo_log` 需要与业务数据在同一个库中。

**每个分片库的 undo_log DDL**：

```sql
CREATE TABLE IF NOT EXISTS `undo_log` (
  `branch_id` BIGINT NOT NULL COMMENT 'branch transaction id',
  `xid` VARCHAR(128) NOT NULL COMMENT 'global transaction id',
  `context` VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
  `rollback_info` LONGBLOB NOT NULL COMMENT 'rollback info',
  `log_status` INT(11) NOT NULL COMMENT '0:normal status,1:defense status',
  `log_created` DATETIME(6) NOT NULL COMMENT 'create datetime',
  `log_modified` DATETIME(6) NOT NULL COMMENT 'modify datetime',
  UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COMMENT = 'AT transaction mode undo table';
```

#### 问题 3：表元数据获取失败（Cannot find table meta）

**现象**：

```
io.seata.common.exception.ShouldNeverHappenException: [xid:xxx] failed to fetch table meta of `t_order`
```

**根因**：Seata 在生成 before/after image 时需要获取表结构元数据（主键、列名、列类型）。在分片场景下，ShardingSphere 将逻辑表名 `t_order` 改写为物理表名 `t_order_0`，Seata 的 SQL 解析器尝试根据改写后的物理表名获取元数据时，可能因表名变更而失败。

**解决方案**：

**方案 A**：关闭 Seata 的表元数据缓存检查（推荐）:

```yaml
seata:
  client:
    rm:
      table-meta-checker-enable: false
```

这会让 Seata 在每次需要时都从数据库中获取元数据，而非依赖缓存。

**方案 B**：手动注册表元数据（Seata 2.0+）:

```yaml
seata:
  client:
    rm:
      table-meta:
        t_order_0:           # 物理表名
          columns:
            id: BIGINT
            user_id: BIGINT
            product_id: BIGINT
            amount: DECIMAL(10,2)
            status: INT
          pk: id
        t_order_1:
          # ... 同理
```

**方案 C**：使用 `@ShardingTableMeta` 注解（Seata 社区插件），标注分片表信息。

#### 问题 4：全局锁与分片键

**现象**：LockKey 格式异常，全局锁冲突频繁。

**根因**：Seata 的全局锁 LockKey 格式为 `resourceId^^^tableName^^^pkValue`，其中 `resourceId` 是 JDBC URL。在分片场景下，不同分片库有不同的 JDBC URL，但 `tableName` 是逻辑表名还是物理表名需要明确。

**关键点**：

- Seata 根据执行 SQL 时的实际连接来确定 `resourceId`，也就是说不同分片库上是各自独立的 LockKey namespace
- `tableName` 默认使用逻辑表名（SQL 解析前的表名），但 ShardingSphere 改写后可能变化
- 全局锁在**单行粒度的主键**上工作，只要不同分片上的数据主键不冲突，锁就是独立且正确的

**最佳实践**：

1. 使用 Snowflake 等分布式 ID 确保全局唯一主键，避免跨分片的主键冲突
2. 分片键最好也是业务主键的一部分，这样 LockKey 自然有了分片感知能力
3. 避免跨分片的热点行竞争（同一分片键上的同一行）

#### 问题 5：XID 在多分片执行时的传递

**现象**：分支事务注册时 `Could not found global transaction xid`。

**根因**：ShardingSphere 的分片执行引擎使用线程池并行执行多个分片的 SQL。Seata 的 `RootContext` 基于 `ThreadLocal`，子线程无法继承主线程的 XID。

**解决方案**：

ShardingSphere 官方 SPI 包 `shardingsphere-transaction-base-seata-at` 已经解决了这个问题——它在创建分片执行线程时，会将 XID 从主线程传递到子线程。核心逻辑：

```java
// SeataATShardingSphereTransactionManager 内部
public void begin(Connection connection, @Nullable Resource resource) {
    String xid = RootContext.getXID();
    // ShardingSphere 的路由引擎在执行前将 xid 存入 ExecutionContext
    // 每个分片执行线程从 ExecutionContext 获取 xid
    // ...
}
```

如果你是手动整合（不使用官方 SPI），需要在业务代码中显式传递：

```java
@GlobalTransactional
public void createOrder(OrderDTO dto) {
    String xid = RootContext.getXID();

    // 如果使用自定义线程池执行分片操作
    executor.execute(() -> {
        RootContext.bind(xid);   // 传递 XID
        try {
            // 分片 SQL 执行...
        } finally {
            RootContext.unbind();
        }
    });
}
```

#### 问题 6：ShardingSphere 版本兼容明细

不同 ShardingSphere 版本的集成方式有细微差异，总结如下：

| 版本 | 集成方式 | 额外依赖 | 注意事项 |
|------|---------|---------|---------|
| 5.0.0 - 5.1.x | DataSourceProxy 手动包装物理 DS | `seata-all` | 不支持自动 SPI 集成，需手动编码 |
| 5.2.x - 5.3.x | SPI + 自动/手动 | `shardingsphere-transaction-base-seata-at` | SPI 集成可用，少量 SPI 类名变更 |
| 5.4.x | SPI + 自动/手动（推荐） | `shardingsphere-transaction-base-seata-at` | 稳定的 SPI 集成，推荐生产使用 |
| 5.5.x | SPI + 自动/手动 | `shardingsphere-transaction-base-seata-at` | 注意 YAML 配置格式变更 |

#### 问题 7：批量操作兼容性

**现象**：`batchUpdate()` 执行时，Seata 无法正确生成 UNDO_LOG。

**根因**：Seata AT 的 batch 支持有一定限制——`jdbcTemplate.batchUpdate()` 仅在 MySQL/MariaDB/PostgreSQL 9.6+ 上受支持，且需要在同一个 PreparedStatement 上连续执行 `addBatch()`。

```java
// 支持的写法
PreparedStatement ps = connection.prepareStatement(sql);
for (Order order : orders) {
    ps.setLong(1, order.getUserId());
    ps.setBigDecimal(2, order.getAmount());
    ps.addBatch();
}
ps.executeBatch();   // Seata 拦截到 executeBatch，整体生成 undo_log
connection.commit();

// 不支持的写法
for (Order order : orders) {
    jdbcTemplate.update(sql, order.getUserId(), order.getAmount());
    // 每执行一次，Seata 生成一次 undo_log
    // 如果循环内没有 commit，undo_log 内容可能被覆盖
}
```

**最佳实践**：在分片场景中，batch 操作会按分片拆分为多个子 batch 分别执行。Seata 会在每个分片连接上分别处理各自的 batch 操作。

#### 问题 8：@GlobalLock 在分片场景的使用

**现象**：使用 `@GlobalLock` 读取分片数据时，全局锁检查失败。

**根因**：`@GlobalLock` 默认只在当前连接上检查全局锁。分片场景下，SQL 被路由到多个分片，每个分片都需要独立检查全局锁。

**解决方案**：ShardingSphere 的 Seata SPI 集成中，`@GlobalLock` 作用于逻辑 SQL 上，SPI 层会自动将锁检查下推到所有涉及的分片连接。但需要注意：

```java
// 正确用法：@GlobalLock 标注在 Service 方法上
@GlobalLock
@Transactional(readOnly = true)
public Order getOrder(Long userId, Long orderId) {
    // 此 SQL 会被路由到具体分片
    // ShardingSphere + Seata 确保在涉及的分片上检查全局锁
    return orderMapper.selectById(orderId, userId);
}
```

不传分片键的查询（全路由扫描）会触发所有分片上的全局锁检查，性能开销较大，应尽量避免。

#### 问题 9：事务超时与分片数量关系

**现象**：分片数量多时，全局事务经常超时回滚。

**根因**：分片数量越多，分支事务数量也越多。每个分片都需要完成 before image 生成、SQL 执行、after image 生成、UNDO_LOG 写入、分支注册这一完整链路。分片多时，总耗时线性增长。

**优化建议**：

| 措施 | 原理 | 效果 |
|------|------|------|
| 减少无用分片 | 不必要的分片增加分支数量 | 显著 |
| 适当调大 timeoutMills | 为分片执行预留充分时间 | 中 |
| 异步化非核心分片 | 将部分分片操作移到事务外 | 显著 |
| 调整锁重试参数 | 减少锁冲突带来的额外等待 | 中 |

```java
// 为分片场景设置更充裕的超时时间
@GlobalTransactional(name = "order-create", timeoutMills = 60000)
public Order createOrder(OrderDTO dto) {
    // ...
}
```

### 性能调优与最佳实践

#### 1. 分片数量与事务性能的平衡

一般建议：
- 单个全局事务涉及的分片数尽量控制在 8 个以内
- 每个分片的 SQL 条数尽量控制在 3 条以内
- 如果分片数超过 16 个，考虑将业务拆分为多个小事务

#### 2. 关闭不必要的特性

```yaml
seata:
  client:
    rm:
      report-success-enable: false   # 不上报分支成功，减少 RPC
      table-meta-checker-enable: false  # 分片场景必关
      only-care-update-columns: true   # 2.0+ 特性，减少 undo_log 体积
```

#### 3. Seata TC 集群部署建议

分片场景下，由于分片数量增多导致分支事务数量大，建议：

```
存储模式：DB（事务日志）+ Redis（全局锁）
TC 集群：至少 2 节点，通过 Nacos 注册发现
```

#### 4. 数据源连接池配置

每个分片库对应一个连接池，需要为每个物理数据源独立配置：

```yaml
dataSources:
  ds0:
    # ...
    maximumPoolSize: 20         # 每个分片库的连接池
    minimumIdle: 5
    connectionTimeout: 30000
  ds1:
    # ...
    maximumPoolSize: 20
```

总连接数 = 分片数 × `maximumPoolSize`，需要根据数据库端最大连接数合理规划。

#### 5. 禁止使用 AutoDataSourceProxy

再次强调：**`enable-auto-data-source-proxy: false`** 是整合 Seata + ShardingSphere 时最关键的配置。不可遗漏。

#### 6. 测试验证 checklist

整合完成后，建议按以下 checklist 验证：

| 验证项 | 方法 | 预期结果 |
|--------|------|---------|
| UNDO_LOG 生成 | 开启全局事务执行一条 INSERT，检查分片库的 undo_log | 每个受影响的分片都有对应的 undo_log 记录 |
| 正常提交 | 执行完整的全局事务 | 数据正确写入，undo_log 被清理（log_status=0） |
| 异常回滚 | 在全局事务中抛异常 | 所有分片上的数据被回滚到事务前状态 |
| XID 传播 | 查看 Seata TC 日志 | 全局事务含所有分支的注册记录 |
| 全局锁 | 并发执行同一行的更新 | 后者等待前者完成后才执行 |
| 多分片回滚 | 跨 2 个分片插入数据后回滚 | 两个分片上的数据都回滚 |
