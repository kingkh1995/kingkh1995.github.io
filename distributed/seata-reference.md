## 4.1 Seata Server 部署

### 4.1.1 启动命令

```shell
Usage: sh seata-server.sh (for linux/mac) or cmd seata-server.bat (for windows) [options]
  Options:
    --host, -h        注册中心暴露地址，默认 0.0.0.0
    --port, -p        监听端口，默认 8091
    --storeMode, -m   日志存储模式：file、db，默认 file
    --sessionMode, -n session 存储模式：file、db、redis，默认 file
    --lockMode, -k    锁存储模式：file、db、redis，默认 file
    --help            查看帮助
```

### 4.1.2 存储模式

Seata Server 使用**三层存储模型**：

| 存储层 | 可选模式 | 说明 |
|--------|----------|------|
| **Transaction Log** | file / db | 事务日志存储 |
| **Session** | file / db / redis | 会话状态存储 |
| **Lock** | file / db / redis | 全局锁存储 |

`file` 模式适合开发/测试环境，`db` 或 `redis` 模式适合生产环境。

### 4.1.3 DB 存储模式配置

修改 `conf/application.yml`：

```yaml
seata:
  store:
    mode: db
    db:
      datasource: druid
      db-type: mysql
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:3306/seata_server
      user: root
      password: your_password
      min-conn: 5
      max-conn: 30
      global-table: global_table
      branch-table: branch_table
      lock-table: lock_table
      distributed-lock-table: distributed_lock
      query-limit: 100
      max-wait: 5000
```

需要创建对应的数据库表，建表脚本在 Seata Server 的 `script/server/db/` 目录下。

### 4.1.4 集群部署

Seata Server 支持多节点集群部署，通过注册中心进行服务发现：

```
          ┌─────────────┐
          │   Nacos /   │
          │   Eureka    │
          └──────┬──────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───┴───┐  ┌───┴───┐  ┌───┴───┐
│TC-001 │  │TC-002 │  │TC-003 │
│:8091  │  │:8091  │  │:8091  │
└───────┘  └───────┘  └───────┘
```

客户端通过事务分组（`tx-service-group`）自动发现 TC 集群。

---

## 4.2 配置参考

### 4.2.1 registry.conf（注册中心 + 配置中心）

```yaml
registry:
  # file / nacos / eureka / redis / zk / consul / etcd3 / sofa
  type: nacos
  nacos:
    application: seata-server
    server-addr: 127.0.0.1:8848
    group: SEATA_GROUP
    namespace:
    cluster: default
    username:
    password:

config:
  # file / nacos / apollo / zk / consul / etcd3
  type: nacos
  nacos:
    server-addr: 127.0.0.1:8848
    group: SEATA_GROUP
    namespace:
    username:
    password:
    data-id: seataServer.properties
```

### 4.2.2 客户端核心配置（Spring Boot）

```yaml
seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: my_test_tx_group

  # 事务分组映射
  service:
    vgroup-mapping:
      my_test_tx_group: default  # 映射到 TC 集群名
    grouplist:
      default: 127.0.0.1:8091

  # AT / XA 数据源代理
  enable-auto-data-source-proxy: true
  data-source-proxy-mode: AT  # AT 或 XA

  # 客户端配置
  client:
    rm:
      report-success-enable: false  # 不上报分支成功状态（提升性能）
      async-commit-buffer-limit: 10000
      lock:
        retry-interval: 10       # 重试间隔（ms）
        retry-times: 30          # 全局锁重试次数
    tm:
      commit-retry-count: 5      # 全局提交重试次数
      rollback-retry-count: 5    # 全局回滚重试次数
```

### 4.2.3 核心配置项一览

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `seata.transport.type` | TCP | 通信类型：TCP、HTTP |
| `seata.transport.server` | NIO | 服务端通信模型：NIO、Netty |
| `seata.config.type` | file | 配置中心类型 |
| `seata.registry.type` | file | 注册中心类型 |
| `seata.store.mode` | file | 存储模式：file、db |
| `seata.service.vgroup-mapping` | | 事务分组到 TC 集群的映射 |
| `seata.service.grouplist` | | TC 地址列表 |
| `seata.client.rm.report-success-enable` | true | 是否上报分支成功状态 |
| `seata.client.rm.async-commit-buffer-limit` | 10000 | 异步提交缓冲区上限 |
| `seata.client.rm.lock.retry-interval` | 10ms | 全局锁重试间隔 |
| `seata.client.rm.lock.retry-times` | 30 | 全局锁重试次数 |
| `seata.client.tm.commit-retry-count` | 5 | 全局提交重试 |
| `seata.client.tm.rollback-retry-count` | 5 | 全局回滚重试 |
| `seata.enable-auto-data-source-proxy` | true | 是否自动代理数据源 |
| `seata.data-source-proxy-mode` | AT | AT 或 XA |

---

## 4.3 注册中心集成

### 4.3.1 Nacos

```yaml
registry:
  type: nacos
  nacos:
    server-addr: 127.0.0.1:8848
    namespace:
    cluster: default
    username:
    password:
```

### 4.3.2 Eureka

```yaml
registry:
  type: eureka
  eureka:
    service-url: http://localhost:8761/eureka
    application: seata-server
    weight: 1
```

### 4.3.3 Consul

```yaml
registry:
  type: consul
  consul:
    server-addr: 127.0.0.1:8500
    acl-token:
```

### 4.3.4 Zookeeper

```yaml
registry:
  type: zk
  zk:
    server-addr: 127.0.0.1:2181
    session-timeout: 6000
    connect-timeout: 2000
    username:
    password:
```

---

## 4.4 数据源与 ORM 支持

### 4.4.1 支持的数据源

| 数据源 | AT 模式 | XA 模式 |
|--------|---------|---------|
| MySQL (5.7+) | ✅ | ✅ |
| PostgreSQL | ✅ | ✅ |
| Oracle | ✅ | ✅ |
| MariaDB | ✅ | ✅ |
| SQL Server | ✅ | ✅ |
| DB2 | ✅ | ✅ |
| TiDB | ✅ | ❌ |
| H2（测试用）| ✅ | ✅ |

### 4.4.2 支持的 ORM 框架

| ORM 框架 | 支持情况 | 说明 |
|----------|----------|------|
| MyBatis / MyBatis-Plus | ✅ | 使用 DataSourceProxy 自动代理 |
| Hibernate / JPA | ✅ | 使用 DataSourceProxy 自动代理 |
| Spring JDBC Template | ✅ | 使用 DataSourceProxy 自动代理 |
| JOOQ | ✅ | 使用 DataSourceProxy 自动代理 |

### 4.4.3 手动数据源代理配置

如果关闭了自动代理，需要手动配置：

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource dataSource() {
        DruidDataSource druidDataSource = new DruidDataSource();
        // 设置数据库连接信息
        druidDataSource.setUrl("jdbc:mysql://localhost:3306/demo");
        druidDataSource.setUsername("root");
        druidDataSource.setPassword("password");
        druidDataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");

        // 使用 Seata AT 模式的数据源代理
        return new DataSourceProxy(druidDataSource);

        // 使用 Seata XA 模式的数据源代理（按需切换）
        // return new DataSourceProxyXA(druidDataSource);
    }
}
```

---

## 4.5 API 与注解参考

### 4.5.1 核心注解

| 注解 | 作用 | 使用位置 |
|------|------|----------|
| `@GlobalTransactional` | 开启全局分布式事务 | Service 方法 |
| `@GlobalLock` | 仅获取全局锁（不开启事务） | Service 方法 |
| `@TwoPhaseBusinessAction` | 标记 TCC 接口 | TCC 接口方法 |
| `@BusinessActionContextParameter` | TCC 参数传递 | TCC 方法参数 |
| `@LocalTCC` | 标记本地 TCC 实现 | TCC 接口类 |

### 4.5.2 @GlobalTransactional 参数

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface GlobalTransactional {

    /**
     * 全局事务超时时间（秒），默认 60s
     */
    int timeoutMills() default 60000;

    /**
     * 事务名称
     */
    String name() default "";

    /**
     * 重试次数
     */
    int retryCount() default 0;
}
```

### 4.5.3 TCC 核心 API

```java
// BusinessActionContext — 在 Try/Confirm/Cancel 之间传递上下文
public class BusinessActionContext implements Serializable {
    public String getXid();                          // 全局事务 XID
    public String getBranchId();                     // 分支事务 ID
    public String getActionName();                   // TCC 动作名称
    public Map<String, Object> getActionContext();   // 自定义参数
    public <T> T getActionContext(String key);       // 获取指定参数
}
```

### 4.5.4 Saga StateMachineEngine API

```java
public interface StateMachineEngine {
    // 同步启动
    StateMachineInstance start(String stateMachineName, String tenantId, Map<String, Object> startParams);

    // 带业务 key 启动
    StateMachineInstance startWithBusinessKey(String stateMachineName, String tenantId, String businessKey, Map<String, Object> startParams);

    // 异步启动
    StateMachineInstance startAsync(String stateMachineName, String tenantId, Map<String, Object> startParams, AsyncCallback callback);

    // 失败后正向重试
    StateMachineInstance forward(String stateMachineInstId, Map<String, Object> replaceParams);

    // 失败后补偿
    StateMachineInstance compensate(String stateMachineInstId, Map<String, Object> replaceParams);

    // 跳过失败节点继续执行
    StateMachineInstance skipAndForward(String stateMachineInstId);
}
```

### 4.5.5 事务分组映射

```java
// 配置示例
// application.yml

seata:
  tx-service-group: my_test_tx_group
  service:
    vgroup-mapping:
      my_test_tx_group: default    # 逻辑分组 → TC 集群
    grouplist:
      default: 127.0.0.1:8091     # TC 集群地址

// 客户端通过分组获取 TC 地址的流程：
// 1. 客户端获取 tx-service-group = "my_test_tx_group"
// 2. 通过 vgroup-mapping 找到 TC 集群名 = "default"
// 3. 通过 grouplist 获取 TC 地址列表
// 4. 负载均衡选择一个 TC 连接
```

---

## 附录

### A. 相关资源

| 资源 | 地址 |
|------|------|
| 官方文档 | https://seata.apache.org/ |
| GitHub | https://github.com/apache/incubator-seata |
| 示例仓库 | https://github.com/apache/incubator-seata-samples |
| 下载页面 | https://seata.apache.org/download/seata-server |
| 问题/讨论 | https://github.com/apache/incubator-seata/issues |

### B. 常见问题

**Q: AT 模式和 TCC 模式有什么区别？**

A: AT 模式通过 JDBC 代理**自动**生成回滚 SQL，对业务几乎无侵入；TCC 模式需要业务代码**手动**实现 Try/Confirm/Cancel 三阶段，侵入性强但更灵活。

**Q: 什么场景应该选择 Saga 而不是 AT？**

A: 长业务流程（如跨天审批）、参与者包含非数据库资源、需要高吞吐量场景优先选择 Saga。简单短事务优先选择 AT。

**Q: Seata 全局事务超时了怎么办？**

A: 通过 `@GlobalTransactional(timeoutMills = 60000)` 配置超时时间。超时后 TC 会主动发起回滚。超时重试次数由 `client.tm.commit-retry-count` 和 `client.tm.rollback-retry-count` 控制。

**Q: UNDO_LOG 表会无限增长吗？**

A: 第二阶段提交完成后，UNDO_LOG 会在异步任务中被批量清理。可以定期监控 UNDO_LOG 表大小，正常情况下不应有大量遗留数据。

### C. 常见生产故障排查

#### C.1 LockConflictException

```
io.seata.rm.datasource.exec.LockConflictException: get global lock fail
```

**排查路径**：
1. TC 端查锁持有者：`SELECT * FROM lock_table WHERE row_key LIKE '%{tableName}%'`
2. 查 `branch_table` 中对应 xid 的状态：若 status 长期为 `1`（begin），说明全局事务超时未提交
3. 检查该 xid 对应 TM 是否仍在运行：可能操作耗时过长（如调用外部 API）

**治理方案**：
- 缩短全局事务跨度，非核心操作移出边界
- 热点行拆分（如库存分片）
- 合理配置 `lock.retry-times` 和 `lock.retry-interval`

#### C.2 UNDO_LOG 膨胀

```sql
-- 诊断
SELECT log_status, COUNT(*) FROM undo_log GROUP BY log_status;

-- status=1 的记录：由脏写导致回滚失败，需逐条排查
SELECT xid, branch_id, log_created, log_status 
FROM undo_log WHERE log_status = 1 
ORDER BY log_created DESC LIMIT 20;
```

**根因**：status=1 → 脏写 → 排查是否有代码绕过 DataSourceProxy；status=0 但长期未清理 → TC 宕机或清理线程异常。

#### C.3 @GlobalTransactional 不生效

**排查 Checklist**：
1. Bean 是否被 Spring 代理？（`AopUtils.isAopProxy(bean)`）
2. `rollbackFor` 是否包含实际抛出的异常？（默认仅 `RuntimeException`）
3. 同类内部调用是否绕过代理？（`this.method()` 不走 AOP）
4. DataSource 是否被 `DataSourceProxy` 包装？
5. `seata.tx-service-group` 配置和 `vgroupMapping` 是否一致？

#### C.4 Cannot find global transaction

```
Could not found global transaction xid = xxx
```

**根因**：XID 传播丢失，分支事务带了错误的 xid 去注册。排查方法：在各服务打日志追踪 `RootContext.getXID()` 的传递链路。

---

> **文档信息**  
> 基于 Seata v2.6 官方文档编写  
> 官方文档：https://seata.apache.org/  
> 最后更新：2026 年 5 月
