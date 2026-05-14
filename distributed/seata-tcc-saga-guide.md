## 3.2 TCC 模式实战

### 3.2.1 整体机制

TCC（Try-Confirm-Cancel）将二阶段行为交由业务代码自定义：

| 阶段 | 操作 | 说明 |
|------|------|------|
| **Try** | 预留资源 | 检查业务状态，冻结/预留所需资源 |
| **Confirm** | 确认执行 | 真正执行业务，使用 Try 阶段预留的资源 |
| **Cancel** | 取消释放 | 释放 Try 阶段预留的资源 |

### 3.2.2 接口定义

使用 `@TwoPhaseBusinessAction` 注解标记 TCC 接口：

```java
import io.seata.rm.tcc.api.BusinessActionContext;
import io.seata.rm.tcc.api.TwoPhaseBusinessAction;

public interface TccStorageService {

    /**
     * Try 阶段：预留库存
     * 注解中 name 为 TCC bean 的唯一标识
     * commitMethod = Try 成功后执行的 Confirm 方法
     * rollbackMethod = Try 失败后执行的 Cancel 方法
     */
    @TwoPhaseBusinessAction(
        name = "tccStorageAction",
        commitMethod = "confirm",
        rollbackMethod = "cancel"
    )
    boolean prepare(BusinessActionContext actionContext,
                    @BusinessActionContextParameter(paramName = "commodityCode") String commodityCode,
                    @BusinessActionContextParameter(paramName = "count") int count);

    /**
     * Confirm 阶段：扣减库存
     */
    boolean confirm(BusinessActionContext actionContext);

    /**
     * Cancel 阶段：释放预留库存
     */
    boolean cancel(BusinessActionContext actionContext);
}
```

### 3.2.3 实现类

```java
public class TccStorageServiceImpl implements TccStorageService {

    /**
     * 冻结库存表
     */
    private Map<String, Integer> frozenMap = new ConcurrentHashMap<>();

    @Override
    public boolean prepare(BusinessActionContext actionContext,
                           String commodityCode, int count) {
        // 1. 检查库存是否充足
        int available = getAvailableStock(commodityCode);
        if (available < count) {
            throw new RuntimeException("库存不足");
        }

        // 2. 冻结库存（预留资源）
        frozenMap.put(commodityCode, count);

        // 3. 减少可用库存
        reduceAvailable(commodityCode, count);

        return true;
    }

    @Override
    public boolean confirm(BusinessActionContext actionContext) {
        // 真正扣减：清理冻结记录即可
        String commodityCode = (String) actionContext.getActionContext("commodityCode");
        frozenMap.remove(commodityCode);
        return true;
    }

    @Override
    public boolean cancel(BusinessActionContext actionContext) {
        // 释放预留：将冻结的库存加回可用库存
        String commodityCode = (String) actionContext.getActionContext("commodityCode");
        Integer count = (Integer) actionContext.getActionContext("count");
        Integer frozen = frozenMap.remove(commodityCode);
        if (frozen != null) {
            addAvailable(commodityCode, count);
        }
        return true;
    }
}
```

### 3.2.4 使用 @GlobalTransactional 开启分布式事务

```java
@Service
public class BusinessService {

    @Autowired
    private TccStorageService storageService;

    @Autowired
    private TccOrderService orderService;

    @GlobalTransactional
    public void purchase(String userId, String commodityCode, int count) {
        storageService.prepare(null, commodityCode, count);
        orderService.prepare(null, userId, commodityCode, count);
    }
}
```

### 3.2.5 本地 TCC 支持

如果 TCC 参与者是本地 Bean（而非远程 RPC 服务），需要在接口额外添加 `@LocalTCC` 注解：

```java
import io.seata.rm.tcc.api.LocalTCC;

@LocalTCC
public interface LocalTccService {
    @TwoPhaseBusinessAction(name = "localAction", commitMethod = "confirm", rollbackMethod = "cancel")
    boolean prepare(BusinessActionContext actionContext, String param);

    boolean confirm(BusinessActionContext actionContext);
    boolean cancel(BusinessActionContext actionContext);
}
```

### 3.2.6 TCCFence——防悬挂/幂等/空回滚的统一方案

从 Seata 1.5.1 开始，`useTCCFence = true` 借助一张日志表统一解决三大经典问题：

```sql
CREATE TABLE `tcc_fence_log` (
  `xid` VARCHAR(128) NOT NULL,
  `branch_id` BIGINT NOT NULL,
  `status` TINYINT NOT NULL COMMENT '1:TRIED, 2:COMMITTED, 3:ROLLBACKED, 4:SUSPENDED',
  `gmt_create` DATETIME(3) NOT NULL,
  `gmt_modified` DATETIME(3) NOT NULL,
  PRIMARY KEY (`xid`, `branch_id`)
);
```

三大异常的根因都是分布式系统中的**消息乱序和重试**：

| 异常 | 场景 | Fence 解决 |
|------|------|-----------|
| **空回滚** | Cancel 先于 Try 到达（Try 因网络超时未到，TM 触发 Cancel） | Cancel 查 fence → 无记录 → 插入 SUSPENDED，不执行 Cancel 业务 |
| **幂等** | Confirm/Cancel 因网络重试被多次调用 | 查 fence 状态，已 COMMITTED/ROLLBACKED → 幂等返回 |
| **防悬挂** | Try 在 Cancel 之后到达，应拒绝执行 | Try 先查 fence，有 SUSPENDED → 拒绝 Try |

**防悬挂的局限**：`useTCCFence` 要求 Try 操作是数据库事务。如果 Try 涉及外部 API 调用且失败了，后续重试的 Try 会被 fence 拦截。这类场景需要关闭 `useTCCFence` 并自行实现防悬挂逻辑。

### 3.2.7 TCC 注意事项

#### 幂等控制
Confirm 和 Cancel 方法可能被多次调用，需要保证幂等性。

#### 空回滚
Try 未执行但 Cancel 被执行的情况。例如：Try 请求超时，TM 触发回滚，Cancel 请求比原始的 Try 请求先到达。Cancel 中应允许**空补偿**（即未找到预留记录时也返回成功）。

```java
public boolean cancel(BusinessActionContext actionContext) {
    String commodityCode = (String) actionContext.getActionContext("commodityCode");
    Integer count = (Integer) actionContext.getActionContext("count");

    // 空回滚：如果没有预留记录，直接返回成功
    if (!frozenMap.containsKey(commodityCode)) {
        return true;
    }

    // 正常回滚
    frozenMap.remove(commodityCode);
    addAvailable(commodityCode, count);
    return true;
}
```

#### 防悬挂
Cancel 先于 Try 执行时，若 Try 后续到达则应该拒绝执行（因为事务已经判定为回滚）。

```rust
实现方案：记录 Cancel 的执行状态，Try 执行前检查该记录是否存在。
```

#### TCC 资源预留设计

核心原则：**预留而非直接操作**。

```sql
-- 错误设计：Try 直接扣减（Cancel 时无法复原）
UPDATE account SET balance = balance - 100 WHERE id = 1;

-- 正确设计：Try 冻结可用余额
UPDATE account SET frozen = frozen + 100 
WHERE id = 1 AND balance - frozen >= 100;

-- Confirm：正式扣减
UPDATE account SET balance = balance - 100, frozen = frozen - 100 
WHERE id = 1;

-- Cancel：释放冻结
UPDATE account SET frozen = frozen - 100 WHERE id = 1;
```

**设计检查清单**：
1. Try 只做检查和预留，不做真正的业务操作
2. Confirm 做真正的业务操作，**必须幂等**
3. Cancel 释放预留资源，**必须幂等**
4. 预留的资源量 = 总额 - 已冻结额（避免超卖）

#### TCC 的适用边界

**适合**：核心交易链路（扣款、下单），需要极致性能且团队有分布式事务经验。

**不适合**：
- 单服务内部的多表操作（用 AT 更简单）
- 无法优雅回滚的场景（发短信、发邮件 → 用 SAGA Forward）
- 团队缺乏分布式事务经验（测试和维护成本远高于 AT）

---

## 3.3 Saga 模式实战

### 3.3.1 整体机制

Saga 模式基于**状态机引擎**实现，通过 JSON 定义业务流程：

```
优势：
  ✓ 一阶段提交，无锁，高性能
  ✓ 事件驱动，异步执行，高吞吐
  ✓ 补偿服务实现简单

劣势：
  ✗ 不保证隔离性（需业务层处理）
```

### 3.3.2 定义服务接口

```java
public interface InventoryAction {
    /**
     * 正向操作：扣减库存
     */
    boolean reduce(String businessKey, BigDecimal amount, Map<String, Object> params);

    /**
     * 补偿操作：归还库存
     */
    boolean compensateReduce(String businessKey, Map<String, Object> params);
}

public interface BalanceAction {
    /**
     * 正向操作：扣减余额
     */
    boolean reduce(String businessKey, BigDecimal amount, Map<String, Object> params);

    /**
     * 补偿操作：归还余额
     */
    boolean compensateReduce(String businessKey, Map<String, Object> params);
}
```

### 3.3.3 定义状态机 JSON

```json
{
  "Name": "reduceInventoryAndBalance",
  "Comment": "reduce inventory then reduce balance in a transaction",
  "StartState": "ReduceInventory",
  "Version": "0.0.1",
  "States": {
    "ReduceInventory": {
      "Type": "ServiceTask",
      "ServiceName": "inventoryAction",
      "ServiceMethod": "reduce",
      "CompensateState": "CompensateReduceInventory",
      "Next": "ReduceBalance",
      "Input": [
        "$.[businessKey]",
        "$.[count]"
      ],
      "Output": {
        "reduceInventoryResult": "$.#root"
      },
      "Status": {
        "#root == true": "SU",
        "#root == false": "FA",
        "$Exception{java.lang.Throwable}": "UN"
      }
    },
    "ReduceBalance": {
      "Type": "ServiceTask",
      "ServiceName": "balanceAction",
      "ServiceMethod": "reduce",
      "CompensateState": "CompensateReduceBalance",
      "Input": [
        "$.[businessKey]",
        "$.[amount]"
      ],
      "Output": {
        "reduceBalanceResult": "$.#root"
      },
      "Status": {
        "#root == true": "SU",
        "#root == false": "FA",
        "$Exception{java.lang.Throwable}": "UN"
      },
      "Catch": [
        {
          "Exceptions": ["java.lang.Throwable"],
          "Next": "CompensationTrigger"
        }
      ],
      "Next": "Succeed"
    },
    "CompensateReduceInventory": {
      "Type": "ServiceTask",
      "ServiceName": "inventoryAction",
      "ServiceMethod": "compensateReduce",
      "Input": ["$.[businessKey]"]
    },
    "CompensateReduceBalance": {
      "Type": "ServiceTask",
      "ServiceName": "balanceAction",
      "ServiceMethod": "compensateReduce",
      "Input": ["$.[businessKey]"]
    },
    "CompensationTrigger": {
      "Type": "CompensationTrigger",
      "Next": "Fail"
    },
    "Succeed": {
      "Type": "Succeed"
    },
    "Fail": {
      "Type": "Fail",
      "ErrorCode": "PURCHASE_FAILED",
      "Message": "purchase failed"
    }
  }
}
```

### 3.3.4 状态节点类型

| 类型 | 说明 |
|------|------|
| `ServiceTask` | 执行服务调用 |
| `Choice` | 单条件选择路由 |
| `CompensationTrigger` | 触发补偿流程 |
| `Succeed` | 正常结束 |
| `Fail` | 异常结束 |
| `SubStateMachine` | 调用子状态机 |
| `CompensateSubMachine` | 补偿子状态机 |

### 3.3.5 状态判定三态模型

执行流程：从 `StartState` 开始，依次执行 ServiceTask。每个步骤根据返回值映射到状态（SU/FA/UN）。遇到失败时触发补偿或重试。

| status | compensateStatus | 最终语义 |
|--------|-----------------|----------|
| `SU` | null | 全部成功 |
| `FA` | null | 首个步骤失败，无事可补偿 |
| `FA`/`UN` | `SU` | 失败但补偿成功，最终一致 |
| `FA`/`UN` | `UN` | **补偿也失败，需人工介入** |

### 3.3.6 启动 Saga 事务

```java
@Autowired
private StateMachineEngine stateMachineEngine;

public void purchase() {
    Map<String, Object> params = new HashMap<>();
    params.put("businessKey", "BUS20240513001");
    params.put("count", 10);
    params.put("amount", new BigDecimal("100.00"));

    // 同步启动
    StateMachineInstance inst = stateMachineEngine.start(
        "reduceInventoryAndBalance",
        "default",
        params
    );

    // 异步启动
    stateMachineEngine.startAsync(
        "reduceInventoryAndBalance",
        "default",
        params,
        new AsyncCallback() {
            @Override
            public void onFinished(StateMachineInstance instance) {
                System.out.println("事务完成，状态：" + instance.getStatus());
            }
            @Override
            public void onError(StateMachineInstance instance, Exception e) {
                System.err.println("事务失败：" + e.getMessage());
            }
        }
    );

    // 失败后正向重试
    // stateMachineEngine.forward(inst.getId(), newParams);

    // 失败后补偿
    // stateMachineEngine.compensate(inst.getId(), null);
}
```

### 3.3.7 最佳实践

#### 允许空补偿
补偿服务执行时若未找到正向记录，也应返回成功（记录业务 key 即可）。

#### 防悬挂控制
检测当前业务 key 是否存在补偿记录，若存在则拒绝正向操作。

#### 幂等控制
正向和补偿服务都需要幂等，防止网络重试造成重复处理。

#### 处理隔离性问题
设计业务时遵循**"宁多扣、不少扣"**原则：
- 先扣款再发货（扣款成功但发货失败→可退款）
- 若已发货但扣款失败，无法将货收回

#### SAGA 的最大弱点：无隔离性

SAGA 不提供任何隔离保证。多个 SAGA 事务可以同时读写同一数据：
- A 转账给 B（冻结 A → 转账 → 解冻 A），期间 B 的余额被人消费，回滚时无法扣回
- 解决思路：正向操作使用**追加语义**（INSERT 日志记录），消除并发写入路径；不可避免的 UPDATE 使用 Forward 策略而非 Rollback

