## 3.4 XA 模式实战

### 3.4.1 为什么 AT 做不到强一致

AT 模式的本质缺陷：一阶段提交已经释放了本地锁，UNDO_LOG 是异步删除的。这决定了它的隔离级别上限是 READ COMMITTED（通过 `@GlobalLock`）。**对于银行转账、清结算这类场景，AT 不够**——需要 XA。

XA 模式的核心差异：一阶段**不提交本地事务，而是执行 XA PREPARE**。此时：
- 行锁**持续持有**直到二阶段 COMMIT/ROLLBACK
- 数据变更已持久化到 redo log，即使数据库崩溃也能恢复
- 直到 COMMIT 之前，其他事务看到的是 PREPARE 之前的数据

### 3.4.2 整体机制

Seata XA 模式将 XA 的 2PC 与 Seata 的事务框架结合：

```
XA 分支事务执行流程：
  1. 数据源代理创建 XAConnection
  2. XA start（使用 Seata 的 XID + BranchID 作为 XA Xid）
  3. 执行本地 SQL
  4. XA end
  5. XA prepare（此时已可回滚）
  6. 等待 TC 决策
  7. XA commit（全局提交） / XA rollback（全局回滚）
```

### 3.4.3 性能代价

| 维度 | AT 模式 | XA 模式 |
|------|---------|---------|
| 一阶段 | 提交本地事务 + 释放行锁 | XA PREPARE，**不释放行锁** |
| 二阶段提交 | 异步删 UNDO_LOG（~1ms） | XA COMMIT 同步执行（~5ms-20ms） |
| 行锁持有时间 | 仅一阶段本地事务期间 | 从一阶段到二阶段 COMMIT，通常 10ms-500ms |
| 并发吞吐 | 接近无事务场景（损失 <5%） | 热数据场景可下降 50%+ |

### 3.4.4 数据源配置

```java
// 使用 XA 数据源代理
import io.seata.rm.datasource.xa.DataSourceProxyXA;

DataSource ds = new DruidDataSource();
// ... 配置连接

DataSourceProxyXA dataSourceProxyXA = new DataSourceProxyXA(ds);
```

### 3.4.5 使用方式

与 AT 模式完全相同的使用方式，仅需配置不同：

```yaml
seata:
  data-source-proxy-mode: XA  # 可选：AT（默认）/ XA
```

业务代码同样只需 `@GlobalTransactional` 注解：

```java
@GlobalTransactional
public void purchase(String userId, String commodityCode, int orderCount) {
    storageService.deduct(commodityCode, orderCount);
    orderService.create(userId, commodityCode, orderCount);
}
```

> 不同之处：XA 模式下，底层使用的是 XA 协议而非 UNDO_LOG 回滚，由数据库自身保证事务的原子性和隔离性。

### 3.4.6 异步提交的陷阱

```yaml
seata:
  server:
    xa:
      async-commit: true  # ⚠️ 慎用
```

**故障场景**：TM 调用 TC commit → TC 立即返回成功 → 业务认为"操作完成" → 但 RM 的 XA COMMIT 尚未执行。后果：
1. 另一个事务的 `SELECT FOR UPDATE` 会阻塞（行锁未释放）
2. 普通 `SELECT` 读到 PREPARE 前的旧数据
3. 如果 TC 此时宕机，XA 分支永远留在 PREPARE 状态，需 DBA 手动 `XA RECOVER` + `XA COMMIT/ROLLBACK`

**结论**：除非业务明确接受"提交成功但数据暂时不可见"的窗口期，否则不要开启异步提交。

### 3.4.7 读写分离下的 XA

XA 模式与 MySQL 主从复制存在已知问题：一阶段 XA PREPARE 在主库执行，但从库可能在 COMMIT 同步前被查询到不一致数据。**解决方案**：XA 事务内的所有操作必须走主库，Seata 不自动做读写分离路由。

---

## 3.5 RocketMQ 集成

### 3.5.1 概述

使用 RocketMQ 作为 Seata 分布式事务的消息参与者，可以实现：**消息的发送与业务操作在同一个分布式事务中**，全局提交则消息可达，全局回滚则消息撤回。

### 3.5.2 引入依赖

```xml
<dependency>
    <groupId>org.apache.seata</groupId>
    <artifactId>seata-all</artifactId>
    <version>${seata.version}</version>
</dependency>
<!-- 或使用 Spring Boot Starter -->
<dependency>
    <groupId>org.apache.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
    <version>${seata.version}</version>
</dependency>
```

### 3.5.3 代码示例

```java
import io.seata.rm.mq.SeataMQProducerFactory;
import io.seata.rm.mq.SeataMQProducer;
import org.apache.rocketmq.common.message.Message;

public class BusinessServiceImpl implements BusinessService {

    private static final String NAME_SERVER = "127.0.0.1:9876";
    private static final String PRODUCER_GROUP = "test-group";
    private static final String TOPIC = "test-topic";

    private static SeataMQProducer producer =
        SeataMQProducerFactory.createSingle(NAME_SERVER, PRODUCER_GROUP);

    @GlobalTransactional
    public void purchase(String userId, String commodityCode, int orderCount) {
        // 发送消息——作为 Seata 事务的 RM 参与者
        producer.send(new Message(TOPIC,
            "testMessage".getBytes(StandardCharsets.UTF_8)));

        // 执行其他业务操作
        storageService.deduct(commodityCode, orderCount);
        orderService.create(userId, commodityCode, orderCount);
    }
}
```

### 3.5.4 工作原理

```
执行流程：
  1. @GlobalTransactional 开启全局事务，生成 XID
  2. SeataMQProducer 发送半消息（Half Message）
  3. 执行业务 SQL（storage + order）
  4. 全局提交 → 半消息转为正式消息，消费者可见
  5. 全局回滚 → 半消息被撤回，消费者不可见

退化行为：
  当前线程中无 XID 时，SeataMQProducer 退化为普通的 RocketMQ Producer（直接发送）
```

---

## 3.6 生产部署实践

### 3.6.1 事务分组——路由抽象的价值

事务分组（tx-service-group）不是简单的配置项，而是一层**逻辑隔离与流量调度**的抽象：

```
客户端: seata.tx-service-group = my_tx_group
           ↓（通过配置中心映射）
配置中心: service.vgroupMapping.my_tx_group = default
           ↓（拼接 TC 集群名）
注册中心: seata-server（cluster = default）
           ↓（拉取服务列表）
实际连接: 192.168.1.10:8091, 192.168.1.11:8091
```

**三层价值**：
1. **逻辑隔离**：不同微服务使用不同分组映射到不同 TC 集群，故障面缩到分组级别
2. **灰度发布**：预发分组映射到预发 TC，与生产物理隔离
3. **秒级容灾**：TC 集群故障时修改配置中心的映射关系即可切换，无需重启应用

### 3.6.2 TC 集群模式对比

| 维度 | DB 模式（经典） | Raft 模式（2.0+） |
|------|----------------|-------------------|
| 事务状态存储 | MySQL/Oracle | TC 本地 Raft Log |
| 全局锁存储 | 数据库 lock_table | Raft 状态机 |
| 高可用 | 依赖数据库 HA（MHA/MGR） | 自建 Raft 集群，无外部依赖 |
| 注册中心 | Nacos/Eureka/Consul 等 | 仅支持 file 模式 |
| 部署节点数 | 无上限（受限于 DB 连接数） | 3-5 节点（Raft 多数派） |

**选型建议**：已有成熟 DB HA 体系 → DB 模式；新项目或希望减少外部依赖 → Raft 模式。

### 3.6.3 预发与生产隔离

预发和生产共用同一数据库时：

```properties
# 生产 TC
store.db.globalTable = "global_table"
store.db.branchTable = "branch_table"
store.db.lockTable  = "lock_table"

# 预发 TC
store.db.globalTable = "global_table_pre"
store.db.branchTable = "branch_table_pre"
store.db.lockTable  = "lock_table"  # 锁表可共用，避免预发绕过全局锁
```

### 3.6.4 监控指标

| Prometheus 指标 | 含义 | 告警建议 |
|------|------|----------|
| `seata_transaction_total{status="rollbacked"}` | 回滚事务数 | 非零即告警 |
| `seata_transaction_total{status="timeout"}` | 超时事务数 | > 1/min |
| `seata_lock_wait_count` | 全局锁等待次数 | > 100/min |
| `undo_log_table_size_bytes` | UNDO_LOG 表大小 | > 1GB |

### 3.6.5 UNDO_LOG 维护

UNDO_LOG 在二阶段提交后标记删除（非物理删除），需要定期清理：

```sql
-- 每日清理（保留 7 天）
DELETE FROM undo_log 
WHERE log_created < DATE_SUB(NOW(), INTERVAL 7 DAY) 
  AND log_status = 0;
```

> 注意：`log_status=1` 的记录是回滚失败的 defense 数据，**禁止自动清理**，需人工确认后处理。

