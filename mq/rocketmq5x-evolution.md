## 一、引言：为什么需要 RocketMQ 5.x

RocketMQ 4.x 在业界已得到广泛验证，但在云原生的大潮下，其架构层面的局限性逐渐显现：

- **高可用依赖 DLedger**：DLedgerCommitLog 虽然提供了 Raft 级别的数据一致性，但三副本的部署成本和性能开销较高，而且 Raft 日志复制与 CommitLog 耦合的设计增加了复杂度；
- **存储层绑定本地磁盘**：Topic 的数据与 Broker 强绑定，扩容容易缩容难——待下线节点上的历史数据无法被其他节点读取；
- **消费模型僵化**：队列必须独占分配给 consumer，rebalance 由客户端执行，在大规模场景下容易出现分配不均和震荡；
- **延迟消息仅支持固定级别**：18 个延迟级别无法满足业务对任意时间延迟的需求；
- **客户端协议私有不透明**：Remoting 协议是 RocketMQ 的私有 TCP 协议，多语言 SDK 维护成本高，功能对齐困难。

RocketMQ 5.x 正是在这样的背景下诞生的。它不是简单的版本号更新，而是对 RocketMQ 架构范式的一次全面重构。本文将从消费模型、高可用、通信架构、延迟消息、顺序消息、存储引擎六个维度，深入解析 5.x 的核心技术演进。

---

## 二、消费模型革新：POP 消费

### 2.1 传统 Pull 模式的痛点

在 4.x 的 Pull 模式下，客户端需要自己执行 Rebalance（基于一致性哈希或平均分配策略），每个队列在一个消费组内只能被一个 consumer 实例独占。这种"队列独占"模式存在两个核心问题：

1. **Rebalance 风暴**：当 consumer 数量变化时，所有 consumer 同时触发 rebalance，导致短暂的消费停止；
2. **队列与消费者强绑定**：如果某个 consumer 处理能力较弱，其分配的队列就会出现消费积压，而其他 consumer 无法帮忙分摊。

### 2.2 POP 消费的核心思想

POP 消费（RIP-19）的核心变化是**将 rebalance 和消费进度管理的责任从客户端转移到服务端（Broker）**。消费者不再需要关心队列分配的具体细节，只需要向 Broker 发送 pop 请求"索取"消息即可。

从源码层面来看，开启 POP 消费需要满足以下条件：

```java
// RebalancePushImpl#clientRebalance
// 开启 broker rebalance 需同时满足：
// 1. 使用 push api（非 lite pull 或 pull）
// 2. 关闭 clientRebalance
// 3. 非顺序消费（并发消费）
// 4. 非广播消费（集群消费）
consumer.setClientRebalance(false); // 关闭客户端 rebalance
```

Broker 端通过 `mqadmin setConsumeMode` 命令或全局配置来开启 POP：

```bash
# Topic + ConsumerGroup 纬度开启 POP
mqadmin setConsumeMode -b brokerAddr -t topic -g consumer_group -m POP -n 8

# 或 Broker 纬度全局配置
defaultMessageRequestMode=POP
```

### 2.3 Broker 端 Rebalance

Broker 端收到 consumer 的 `QUERY_ASSIGNMENT` 请求后，由 `QueryAssignmentProcessor` 执行分配。关键之处在于 POP 模式下的队列分配逻辑：

```java
// QueryAssignmentProcessor#allocate4Pop
// 默认情况下 popShareQueueNum = -1
// 所有 consumer 实例全量分配 queue
// queueId 设置为 -1，代表不关心具体队列
```

这意味着在 POP 模式下，每个 consumer 实例都会分配到所有 queue（queueId=-1 作为占位符），真正从哪个 queue 拉消息由 Broker 端决定。Broker 可以通过 pop 时"循环分配"的方式实现负载均衡，从而实现**队列共享**而非独占。

### 2.4 POP 消费进度管理

传统 Pull 模式由 consumer 管理 offset，因此队列共享会导致 offset 混乱。POP 模式采用了一套全新的机制：

- **CheckPoint**：每次 pop 消息时，Broker 会发送一条 checkpoint 消息（写入 `reviveTopic` 系统 Topic），用于记录消费进度；
- **Revive 机制**：consumer 在处理完消息后发送 ack，Broker 收到 ack 后更新消费进度。如果消息在 `invisibleTime` 内未被 ack，则该消息对其他 consumer 再次可见（类似 Kafka 的 offset 过期机制）。

```java
// PopProcessQueue：POP 模式下的 ProcessQueue
// 不再包含 offset 相关属性
// offset 由 Broker 端管理
```

这种设计使得一个队列可以被多个 consumer 实例同时消费，弹性更好，也避免了 rebalance 带来的 stop-the-world。

---

## 三、高可用架构升级

### 3.1 Controller 模式（RIP-67）

4.x 提供了两种 HA 模式：传统 Master-Slave 和 DLedgerCommitLog。前者无法自动选主，后者部署成本高。5.x 引入的 **Controller 模式**则是一个优雅的折中。

Controller 是一个独立的 Raft 集群（3 副本及以上），负责监控 Broker 心跳并执行自动选主。它有独立部署和嵌入 NameServer 两种方式。

从 5.2 版本开始，Controller 的 Raft 实现从 DLedger 迁移到了 **sofa-jraft**（RIP-67），与 Nacos 使用的 Raft 实现同源。

```java
// DLedgerController 核心成员
public class DLedgerController implements Controller {
    // DLedger 组件
    private final DLedgerServer dLedgerServer;         // Raft Server
    private final DLedgerControllerStateMachine statemachine; // Raft 状态机
    
    // 业务组件
    private final ReplicasInfoManager replicasInfoManager;     // 副本信息管理
    private final BrokerValidPredicate brokerAlivePredicate;   // Broker 判活
    private final ElectPolicy electPolicy;                     // 选主策略
}
```

**Controller 的工作流程**：

1. Broker 启动时与 Controller 协商分配 `brokerId`（不再是用户配置），持久化到 `brokerIdentity` 文件中；
2. Broker 定期向 Controller 发送心跳（包含 CommitLog 写进度）；
3. Controller 通过 `DefaultBrokerHeartbeatManager` 维护心跳信息：
   ```java
   public class DefaultBrokerHeartbeatManager implements BrokerHeartbeatManager {
       // cluster + brokerName + brokerControllerId → BrokerLiveInfo
       private final Map<BrokerIdentityInfo, BrokerLiveInfo> brokerLiveTable;
   }
   ```
4. 当 Master Broker 心跳超时，Controller 的 `scanInactiveMasterService` 定时任务触发判活，若确认下线则执行 `ElectPolicy` 从 SyncStateSet 中选出一个 Slave 提升为 Master。

**相比 DLedgerCommitLog 的优势**：
- 一个 Broker 组只需 2 个节点（1 Master + 1 Slave），而非 3 个；
- 数据复制仍使用传统的 Master-Slave HA 机制，不经过 Raft，延迟更低；
- 控制面（选主）和数据面（数据复制）解耦。

### 3.2 BrokerId 协商机制

5.x Controller 模式下，brokerId 由 Controller 统一分配，用户不再需要配置：

```java
// Controller 分配 BrokerId
// brokerId 从 1 开始递增
// Master 向 NameServer 注册使用 brokerId=0
// Slave 使用 Controller 分配的 brokerId
```

Broker 首次上线时与 Controller 交互，获取一个唯一的 brokerId。重启后从 `brokerIdentity` 文件恢复，保证 id 不变。这一机制为后续的弹性扩缩容奠定了基础。

### 3.3 SlaveActingMaster 模式

Controller 模式解决了自动选主的问题，但选主需要时间（秒级），而且如果集群中没有 Controller（4.x 传统 M-S 模式），Master 下线期间消费服务会受影响。

**SlaveActingMaster 模式**的核心思想是：当 Master 下线后，Slave 可以"代理"Master 的角色，提供更全面的消费能力，而不需要等待真正的选主完成。

```java
// NameServer 侧开启
supportActingMaster=true

// Broker 侧 Master 和 Slave 都需要开启
enableSlaveActingMaster=true
// Slave 还可选择开启二级消息远程逃逸
enableRemoteEscape=true
```

#### 解决消费进度回退

传统模式下，Master 重新上线后会发生两个事情，可能导致消费进度回退：

1. Slave 从 Master 同步 `consumerOffset.json`（Master 的消费进度覆盖 Slave 的）；
2. Consumer 发现 Master 上线后触发立即 Rebalance。

SlaveActingMaster 模式通过**预上线机制**和**消费进度隔离**来避免这个问题。Slave 在代理 Master 期间，消费进度的更新会单独记录，Master 恢复时不会直接覆盖。

#### 解决二级消息消费中断

4.x 中只有 Master Broker 可以调度延迟消息和事务消息回查：

```java
// 4.x 代码：只有 Master 才能处理
if (BrokerRole.SLAVE == messageStoreConfig.getBrokerRole()) {
    // SLAVE 不启动 ScheduleMessageService
    return;
}
```

SlaveActingMaster 模式下，Slave 可以通过**远程逃逸（RemoteEscape）** 机制，将需要在 Master 执行的定时任务转发到其他正常的 Master Broker 上执行。这保证了即使 Master 长时间下线，延迟消息和事务消息仍然可以被正常调度。

#### 队列全局锁的代理

4.x 中顺序消费的 Lock API 只能请求 Master：

```java
// 4.x consumer 锁续期
// ProcessQueue#isLockExpired
// 锁超过 30s 未向 broker 续期认为过期
// 锁请求只能发给 Master
```

SlaveActingMaster 模式下，NameServer 的路由查询逻辑被修改：当 Master 下线时，返回 Broker 组内最小 brokerId 的 Slave 作为**代理 Master 地址**。Consumer 对该地址发出的 Lock 请求会被 Slave 处理，从而保证顺序消费在 Master 下线期间不受影响。

```java
// RouteInfoManager#pickupTopicRouteData
// 满足三个条件时，将 brokerId=0 的地址替换为 Slave 地址：
// 1. NameServer 开启 supportActingMaster
// 2. 该 Broker 组 Master 下线
// 3. Broker 开启 enableSlaveActingMaster
```

---

## 四、新通信架构：gRPC + Proxy

### 4.1 架构变化

5.x 引入了 **RocketMQ-Proxy** 角色，作为客户端与 Broker 之间的中间层。Proxy 同时支持 gRPC（新客户端）和 Remoting（老客户端）两种协议，实现了协议转换和请求路由。

```
gRPC Client → Proxy (gRPC) → Proxy (Remoting) → Broker
Remoting Client → → → → → → → → → → → → → → → → Broker (直接)
```

这样的架构带来了几个关键变化：

1. **多语言客户端变得简单**：gRPC 协议有自动生成的跨语言 Stub，各语言 SDK 只需关注业务逻辑；
2. **NameServer 压力减轻**：原来要为每个客户端提供路由服务，现在只需为 Proxy 提供；
3. **安全性集中管控**：ACL、TLS 等安全策略可以在 Proxy 层统一处理。

### 4.2 Proxy 配置与启动

```json
{
  "namesrvAddr": "127.0.0.1:9876",
  "proxyMode": "cluster",
  "rocketMQClusterName": "DefaultCluster",
  "remotingListenPort": 8080,
  "grpcServerPort": 8081,
  "enableACL": true
}
```

### 4.3 新版客户端设计

新版 gRPC 客户端（`rocketmq-clients`）通过 SPI 机制加载：

```java
// 1. SPI 加载客户端实现
final ClientServiceProvider provider = ClientServiceProvider.loadService();

// 2. 设置 Proxy 端点
ClientConfiguration clientConfiguration = ClientConfiguration.newBuilder()
    .setEndpoints("127.0.0.1:8081")  // Proxy gRPC 地址
    .build();

// 3. 创建 Producer/Consumer
final Producer producer = provider.newProducerBuilder()
    .setClientConfiguration(clientConfiguration)
    .setTopics(topic)
    .build();
```

新版客户端的几个关键设计差异：

**路由独立缓存**：每个 Producer 和 Consumer 实例分别缓存各自关心的 Topic 路由数据（4.x 中路由和心跳是全局共用的）：

```java
// 4.x: 同一个 MQClientInstance 管理所有路由
MQClientInstance mqClient = MQClientManager.getInstance()
    .getOrCreateMQClientInstance(brokerConfig);

// 5.x: 每个 ClientImpl 独立管理路由
ClientImpl#startUp
    → fetchTopicRoute()  // 启动时主动加载
    → 每隔 30s 刷新路由缓存
```

**心跳变化**：新客户端每个实例**每 10s** 独立发送心跳，心跳包非常轻量：

```protobuf
message HeartbeatRequest {
  optional Resource group = 1;   // 消费组名
  ClientType client_type = 2;    // PRODUCER / PUSH_CONSUMER
}
```

而 4.x 的心跳包 `HeartbeatData` 包含完整的订阅关系等大量数据。Proxy 端通过 `ClientActivity#heartbeat` 处理心跳，将客户端的注册信息维护在 `CLIENT_SETTINGS_MAP` 缓存中。

### 4.4 Proxy 路由管理

Proxy 侧使用 Caffeine 管理本地的 Topic 路由缓存，20s 自动刷新：

```java
// TopicRouteService 路由缓存配置
cache = Caffeine.newBuilder()
    .refreshAfterWrite(20, TimeUnit.SECONDS)
    .build(topic -> loadTopicRoute(topic));
```

**路由转换的关键逻辑**——Proxy 会将路由中的 Broker 地址全部替换为自己的地址：

```java
// ClusterTopicRouteService#getTopicRouteForProxy
// 所有 broker 地址都替换为客户端传入的 endpoint
// 即 RocketMQ-Proxy 实例地址
```

这意味着客户端感知不到后端 Broker 的存在，整个集群对客户端来说只是一个 Proxy 地址列表。

### 4.5 负载均衡注意事项

新客户端 gRPC 的默认负载均衡策略是 **pick_first**（选择一个可用地址建立长连接）：

```java
// RpcClientImpl 构建 grpc channel
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("ipv4:127.0.0.1:8081,127.0.0.2:8081")
    .defaultLoadBalancePolicy("pick_first")  // 默认策略
    .build();
```

这意味着通过简单 IP 列表指定 Proxy，客户端不做真正的负载均衡——只会与其中一台 Proxy 通讯。生产环境建议通过域名（DNS）构造 Endpoints，在**服务端**（如 Nginx、K8s Service）实现负载均衡。

---

## 五、时序消息的进化：任意时间延迟消息

### 5.1 传统延迟消息的局限

4.x 的延迟消息通过 **延迟级别（Delay Level）** 实现，默认只有 18 个级别：

```properties
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

每条延迟消息会被写入系统 Topic `SCHEDULE_TOPIC_XXXX`，队列号 = delayLevel - 1。由于每个队列中的消息必定按照目标投递时间有序（因为延迟时间是固定的），`ScheduleMessageService` 可以轻松地按时间顺序投递。

但对于"任意时间延迟"的需求，固定级别的方式完全无法满足。

### 5.2 Timer 消息的时间轮设计

5.x 的 Timer 消息（RIP-43）采用**时间轮（TimerWheel）+ 时间轮日志（TimerLog）** 的架构来实现任意时间的延迟。

当 Producer 发送一条设置了 `deliverTimeMs` 的消息时：

```java
Message message = new Message(TOPIC, body);
message.setDeliverTimeMs(System.currentTimeMillis() + 10_000L);
SendResult result = producer.send(message);
```

Broker 侧的 `HookUtils#transformTimerMessage` 会对消息进行转换：

**Step 1 - 统一时间戳**：无论用户用哪种 API，都转换为目标投递时间戳：
```java
// 设置延迟时长：用 Broker 机器时间计算
// 设置目标投递时间：用 Producer 机器时间
// 两者有微妙的差别，取决于时钟是否同步
```

**Step 2 - 阈值检查**：最大延迟时间默认 `timerMaxDelaySec = 3天`，时间精度默认 `timerPrecisionMs = 1000ms`（取整到 1s）。

**Step 3 - 转换系统消息**：将原始 Topic 和 Queue 存入 properties，目标 Topic 替换为 `rmq_sys_wheel_timer`，queueId 替换为 0。

### 5.3 时间轮与 TimerLog

转换后的 Timer 消息被消费后，进入时间轮结构。时间轮分为两部分：

**TimerWheel**：按时间精度（1s）划分槽位，默认覆盖 2 天（`timerRollWindowSlot`）。每个 Slot 不存储实际数据，只存储指向 TimerLog 的指针。

**TimerLog**：顺序写入的日志文件（定长 100M），每条记录 52 字节，按照延迟时间形成一个单向链表。

```
TimerWheel (内存 + 文件)
  ├── Slot[0]: delayTime=0-1s    → 指向 TimerLog 位置
  ├── Slot[1]: delayTime=1-2s    → 指向 TimerLog 位置
  └── ...
      每个 Slot 记录 firstPos 和 lastPos

TimerLog (顺序写文件)
  ├── Record: slot.lastPos, offsetPy, sizePy, magic, delayedTime...
  └── 每个 Slot 的消息通过 lastPos 形成链表
```

```java
// TimerWheel#putSlot
// Slot 指向 TimerLog 的指针
// timeMs/precisionMs：期望延迟时间
// firstPos：Slot 中第一条消息的 TimerLog 物理偏移
// lastPos：Slot 中最后一条消息的 TimerLog 物理偏移
```

### 5.4 Enqueue 和 Dequeue

Timer 消息的流转由多组线程协作完成：

**Enqueue 阶段**（将消息放入时间轮）：

```
TimerEnqueueGetService (单线程消费 rmq_sys_wheel_timer)
  → 转换为 TimerRequest → enqueuePutQueue（内存队列）
  → TimerEnqueuePutService 消费
    → if (已到期): 直接入队 dequeuePutQueue
    → if (未到期): 写入 TimerLog + 更新 TimerWheel
```

**Dequeue 阶段**（将到期消息投递到目标 Topic）：

```
TimerDequeueGetService (单线程)
  → 根据当前时间找到 TimerWheel 对应 Slot
  → 通过 TimerLog 链表遍历消息
  → 重组 TimerRequest → dequeueGetQueue

TimerDequeueGetMessageService (多线程，默认 3 个)
  → 从 CommitLog 读取原始消息
  → 注入到 TimerRequest → dequeuePutQueue

TimerDequeuePutMessageService (多线程，默认 3 个)
  → 到期消息：投递到用户 Topic 和 Queue
  → 未到期滚动：重新投递到 rmq_sys_wheel_timer
```

```java
// TimerMessageStore#doEnqueue
// Step1: 如果延迟超过 timerRollWindowSlot（2天），需要滚动
// Step2: 顺序写入 TimerLog（记录 slot.lastPos, offsetPy...）
// Step3: 更新 TimerWheel 中 Slot 的 firstPos/lastPos

// 延迟时间与槽位关系：
// 小于 2天：直接找到对应槽位
// 2~2.66天：放在 1天的槽位上
// 2.66~3天：放在 2天的槽位上
```

### 5.5 HA 与重启恢复

Timer 消息涉及三份物理数据的进度记录：

```java
public class TimerCheckpoint {
    private volatile long lastReadTimeMs;         // 时间轮处理进度（时间戳）
    private volatile long lastTimerLogFlushPos;    // TimerLog 刷盘进度
    private volatile long lastTimerQueueOffset;    // 延迟 Topic 消费进度
}
```

重启恢复时使用 `TimerFlushService` 线程（每 1s 执行），从 checkpoint 记录的位置回放 TimerLog 重建 TimerWheel：

```java
// TimerMessageStore#recoverAndRevise
// 从 checkpoint 往前推 100M（一个 TimerLog 文件大小）
// 遍历 TimerLog 记录，校验完整性
// 修正 TimerWheel 中对应 Slot 的 lastPos 指向
// 返回实际的刷盘进度
```

---

## 六、新架构下的顺序消息

### 6.1 4.x 顺序消息回顾

4.x 的顺序消息依赖**客户端队列选择**和**Broker 全局锁**：

**Producer 侧**：通过 `MessageQueueSelector` 手动选择 Queue：
```java
producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Integer id = (Integer) arg;
        int index = id % mqs.size();
        return mqs.get(index);
    }
}, orderId);
```

**Broker 侧**：通过 `RebalanceLockManager` 维护 `mqLockTable`（Group-Queue-ClientId 映射），Consumer 通过 `LOCK_BATCH_MQ` 获取全局锁。

**Consumer 侧**：每个 Queue 同时只能由一个线程消费；锁每 20s 续期一次，超过 30s 未续期认为过期。

这套机制的问题在于：
1. Producer 必须感知队列拓扑（队列数变化时需要修改逻辑）；
2. Lock API 只能请求 Master，Master 下线时顺序消费中断；
3. 锁机制在 POP 模式下不适用。

### 6.2 5.x 新架构设计

5.x 的顺序消息基于 POP 消费重新设计，不再需要客户端 Lock API。

**关键配置**：
```bash
# Topic 必须设置为 FIFO 类型
mqadmin updateTopic -n 127.0.0.1:9876 -c DefaultCluster \
  -t MyOrderTopic -a +message.type=FIFO

# 消费组开启顺序消费能力
mqadmin updateSubGroup -n 127.0.0.1:9876 -c DefaultCluster \
  -g fifoConsumer1 -o true
```

**Producer 侧**：不再需要 `MessageQueueSelector`，只需设置 `messageGroup`：
```java
Message message = provider.newMessageBuilder()
    .setTopic("MyOrderTopic")
    .setMessageGroup(orderId + "")     // 代替队列选择
    .setBody(body)
    .build();
SendReceipt sendReceipt = producer.send(message);
```

Queue 选择由 Proxy 完成——通过 `hash(messageGroup)` 计算目标 queue。

**Consumer 侧**：API 与普通消费完全一致：

```java
PushConsumer pushConsumer = provider.newPushConsumerBuilder()
    .setConsumerGroup("fifoConsumer1")
    .setSubscriptionExpressions(subscription)
    .setMessageListener(messageView -> ConsumeResult.SUCCESS)
    .build();
```

### 6.3 OrderInfo 与队列独占

5.x 在 Broker 侧引入了 `ConsumerOrderInfoManager`，以 **OrderInfo** 数据结构来替代 Lock API：

```java
public static class OrderInfo {
    private long popTime;                          // Pop 请求时间戳
    private Long invisibleTime;                    // 不可见时间（默认 60s）
    private List<Long> offsetList;                 // 返回消息的 offset 列表
    private Map<Long, Long> offsetNextVisibleTime; // 每条消息的可见时间
    private Map<Long, Integer> offsetConsumedCount; // 消费次数
    private long commitOffsetBit;                  // ACK 位图
}
```

每次 Pop 请求时，通过 `checkBlock` 校验队列独占：

```java
// ConsumerOrderInfoManager#checkBlock
// 同时满足以下两个条件时，阻塞本次 Pop 请求：
// 1. 存在未 ACK 的消息
// 2. 未 ACK 消息仍处于不可见时间内
if (hasUnacked && !isInvisibleExpired) {
    return BLOCKED;  // 不返回任何消息
}
```

这种方式本质上是**将锁语义内化到了 Pop 请求中**。队列是否被"锁住"取决于是否有未 ACK 的消息，而不是一个独立的锁表。

Consumer 消费成功后发送 ACK：

```java
// Proxy 侧收到 ACK
AckMessageActivity#processAckMessage
  → 取消该消息的 Renew
  → 调用 Broker 执行 ACK

// Broker 侧
AckMessageProcessor#processRequest
  → 根据 reviveTopic.queueId=999 识别顺序消费
  → commitAndNext: 在 OrderInfo 中标记 offset 已 ACK
  → 更新内存消费进度
```

消费失败的重试机制：

```java
// ProcessQueueImpl#eraseFifoMessage
// 失败次数 < 16: 延迟重试（1s → 30m，逐级递增）
// 失败次数 ≥ 16: 投递到死信队列 DLQ 并 ACK（跳过该消息）
```

注意与 4.x 的重要区别：
- 4.x 顺序消息默认**无限本地重试**，不跳过消息；
- 5.x 默认**最多 16 次**，超过后进入 DLQ 并跳过，保证了消费的持续进行。

---

## 七、存储架构革命

存储层是 RocketMQ 5.x 变化最大的部分，涉及两个独立但可叠加的特性：**RocksDB 存储（RIP-66）** 和 **分级存储（RIP-57/65）**。

### 7.1 背景：存储层的两大瓶颈

在云原生的浪潮下，RocketMQ 存储层面临两个核心瓶颈：

1. **数据量膨胀快于单体硬件**：SSD 的容量和价格始终存在矛盾，无限扩容本地磁盘不现实；
2. **百万 Topic 场景下的性能退化**：传统的 JSON 全量持久化 + MMap ConsumeQueue 在 Topic 数量巨大时，Full GC 和文件句柄成为瓶颈。

这两个问题的解决方案分别对应分级存储和 RocksDB 存储。但需要强调的是——它们是两个**独立**的特性，解决不同层次的问题。

### 7.2 RocksDB 存储（RIP-66）——百万队列的基石

#### 7.2.1 问题的起点

在 4.x 中，Config 目录下的 `topics.json`、`subscriptionGroup.json` 等文件采用**全量持久化**策略：即使是变更一个 topic，也要将整个 topics.json（可能包含数万个 topic 配置）全量写一遍。

```java
// 4.x ConfigManager#persist
// 每次变更都全量序列化整个对象到 JSON 文件
// Topic 数量巨大时，内存和 IO 压力显著
```

当 Topic 数量达到百万级别时，这种方式的性能问题变得不可接受。

#### 7.2.2 架构设计

RocketMQ 的 RocksDB 存储通过 `StoreType` 枚举控制：

```java
public enum StoreType {
    DEFAULT("default"),           // 传统 MMap 文件存储
    DEFAULT_ROCKSDB("defaultRocksDB");  // RocksDB 存储
}
```

`storeType=defaultRocksDB` 时，Broker 使用 `RocksDBMessageStore`：

```java
// BrokerController#initializeMessageStore
if (this.messageStoreConfig.isEnableRocksDBStore()) {
    defaultMessageStore = new RocksDBMessageStore(
        messageStoreConfig, brokerStatsManager,
        messageArrivingListener, brokerConfig, topicConfigTable);
}
```

**需要特别注意的是，RocksDBMessageStore 并不替换 CommitLog 的存储**——CommitLog 仍然使用 MMap 顺序写来保证最高吞吐。RocksDB 替换的是以下两部分：

**1. ConsumeQueue 存储**

传统 MMap ConsumeQueue 在百万队列场景下会占用大量文件句柄。`RocksDBConsumeQueueStore` 使用两个 ColumnFamily 来存储 ConsumeQueue 数据：

```java
// RocksDBConsumeQueueStore 的两个 ColumnFamily
private final ConsumeQueueRocksDBStorage rocksDBStorage;

// RocksDBConsumeQueueTable：存储 CqUnit 数据
// Key = topic + queueId + cqOffset
// Value = CqUnit[phyOffset, msgSize, tagHashCode, msgStoreTime]
private final RocksDBConsumeQueueTable rocksDBConsumeQueueTable;

// RocksDBConsumeQueueOffsetTable：存储 offset 元数据
// 维护每个队列的 min/max offset 和对应的物理偏移
private final RocksDBConsumeQueueOffsetTable rocksDBConsumeQueueOffsetTable;
```

写入采用攒批模式，通过 `RocksGroupCommitService` 每批聚合最多 256 条 DispatchRequest 批量写入：

```java
// RocksGroupCommitService#run
// 攒批写入，每批最多 PREFERRED_DISPATCH_REQUEST_COUNT = 256 条
// 减少 RocksDB 写入放大
```

**2. Config 存储**

以下配置管理器替换为 RocksDB 实现：
- `RocksDBTopicConfigManager`
- `RocksDBSubscriptionGroupManager`
- `RocksDBConsumerOffsetManager`

这些管理器利用 RocksDB 的 Put API 实现增量持久化，不再需要全量写 JSON。

#### 7.2.3 双写模式：CombineConsumeQueueStore

为了支持从 MMap 到 RocksDB 的平滑迁移，5.x 提供了 `CombineConsumeQueueStore` 双写模式：

```java
public class CombineConsumeQueueStore implements ConsumeQueueStoreInterface {
    // 内部包含两个 ConsumeQueue 存储
    private final ConsumeQueueStore consumeQueueStore;           // MMap 传统
    private final RocksDBConsumeQueueStore rocksDBConsumeQueueStore; // RocksDB
    
    // assignOffsetStore：哪个负责分配 offset
    // currentReadStore：读优先走哪个
    private final AbstractConsumeQueueStore currentReadStore;
    private final AbstractConsumeQueueStore assignOffsetStore;
}
```

**迁移路径分三个阶段**：

```
阶段1: storeType=default
        → 纯 MMap 文件存储（传统方式）

阶段2: storeType=default + enableRocksDBStore=true
        → CombineConsumeQueueStore 双写
        → 新数据同时写入 MMap 和 RocksDB
        → 零停机，可灰度

阶段3: storeType=defaultRocksDB
        → 纯 RocksDB 存储
        → 移除旧的 MMap ConsumeQueue 文件
```

#### 7.2.4 MessageRocksDBStorage 统一引擎

RocketMQ 5.x 将 RocksDB 的底层操作封装在 `MessageRocksDBStorage` 和 `AbstractRocksDBStorage` 中，这个引擎不仅服务于 ConsumeQueue，也服务于：

- `TimerMessageRocksDBStore`：定时消息的索引存储
- `TransMessageRocksDBStore`：事务消息的存储
- `IndexRocksDBStore`：索引文件存储

这种统一设计避免了重复的 RocksDB 实例创建和管理开销。

#### 7.2.5 性能与适用场景

| 场景 | 推荐存储 | 原因 |
|------|----------|------|
| 普通规模（<1K Topic） | DEFAULT | 简单，运维成本低 |
| 大规模（>1K Topic） | DEFAULT_ROCKSDB | 避免大量 ConsumeQueue 文件句柄 |
| 百万队列（LMQ） | DEFAULT_ROCKSDB | RocksDB KV 存储可水平扩展 |
| 超低延迟敏感 | DEFAULT | RocksDB compaction 可能有毛刺 |

### 7.3 分级存储（RIP-57 / RIP-65）——冷热分层降本

#### 7.3.1 直写 vs 转写

分级存储面临的首要选择是**直写**还是**转写**：

- **直写**：用高可用分布式存储（如 OSS、HDFS）直接替代本地块存储，消息直接写到远端。优点是存储池化，缺点是延迟和依赖强；
- **转写**：热数据先写到本地磁盘（顺序写，高性能），满足条件后异步转储到廉价存储。优点是兼容现有架构，缺点是数据有两份。

```java
// 目前分级存储选择了转写模式
// 原因：成本、可移植性、延迟、可用性综合权衡
// 理想的终态可以是两者的结合
```

RocketMQ 选择先实现**转写**模式，主要有几个考虑：

- **成本**：将冷数据卸载到便宜存储，直接降低存储成本；
- **可移植性**：保留本地存储，只需实现 `TieredFileSegment` 接口即可切换后端，不绑定 IaaS；
- **可用性**：转写模式下系统弱依赖二级存储，二级存储不可用时不影响消息写入。

#### 7.3.2 技术架构

分级存储通过 **MessageStore 插件机制** 接入：

```xml
# Broker 配置
messageStorePlugIn=org.apache.rocketmq.tieredstore.TieredMessageStore
tieredBackendServiceProvider=org.apache.rocketmq.tieredstore.provider.posix.PosixFileSegment
tieredStoreFilePath=/data/rmq_data/broker-a-tieredstore
```

```java
// MessageStoreFactory 根据插件配置创建扩展
// TieredMessageStore 包装 DefaultMessageStore
// 内层（DefaultMessageStore）处理热数据
// 外层（TieredMessageStore）处理冷数据
```

**存储模型**采用了与本地存储一一对应的抽象：

| 本地存储 | 分级存储 | 说明 |
|----------|----------|------|
| MappedFile | TieredFileSegment | 单个文件句柄 |
| MappedFileQueue | TieredFlatFile | 文件列表管理 |
| CommitLog | TieredCommitLog | 按队列拆分存储 |
| ConsumeQueue | TieredConsumeQueue | 消费索引 |
| IndexFile | IndexStoreService | 索引文件（含重排） |

与传统本地存储最大的区别在于**CommitLog 的组织方式**：

- 本地存储：所有队列的消息全部混写在一个 CommitLog 中；
- 分级存储：每个队列的 CommitLog 独立存储。

这种设计是为了减少从二级存储读取时的 IOPS——如果所有队列混在一起，同一队列的消息可能分散在大量文件中，读取时需要多次 IO。

#### 7.3.3 写流程（Dispatcher）

TieredMessageStore 的写入不是由消息写入直接触发的，而是通过 **Dispatcher 机制**：

```java
// TieredDispatcher 实现了 CommitLogDispatcher
// 被注册到 DefaultMessageStore 的 Dispatcher 链中
// 在消息写入 CommitLog + 构建 ConsumeQueue 后被调用

// DefaultMessageStore#doDispatch
// reput 线程感知到 CommitLog 有新的数据
for (CommitLogDispatcher dispatcher : dispatcherList) {
    dispatcher.dispatch(request);  // 触发分层存储 Dispatcher
}
```

`MessageStoreDispatcherImpl` 是一个服务线程，每 20s 扫描一次所有队列，批量上传数据：

```
doScheduleDispatch (每 20s 遍历所有 FlatMessageFile):
  1. 计算本次上传的起始 offset 和结束 offset
     - 起始: ConsumeQueue 当前 maxOffset
     - 结束: min(maxOffset + groupCommitSize, commitOffset)
  2. 遍历 offset 范围，读取本地 ConsumeQueue 和 CommitLog
  3. append: 将数据放入上传缓冲区（同步）
  4. commit: 将缓冲区数据上传到二级存储（异步）
  5. 上传成功后更新 commit offset
```

触发上传的两个条件（满足其一即可）：
- **超时**：距离上次提交超过 30s（`tieredStoreGroupCommitInterval`）
- **数量**：待上传消息超过 4096 条（`tieredStoreGroupCommitCount`）

**Non-StopWrite 特性**：当后端分布式存储出现网络波动时，传统 Append 模型会因为顺序写入的阻塞影响后续数据上传。RocketMQ 采用混合模型——立刻切换文件并重传，不阻塞后续数据：

```
场景：commit offset = 150, max offset = 200
传统模型：等待 150-200 上传完成，后续数据等待
Non-StopWrite：立即切换文件，以 150 为文件名重传，同时 200+ 的数据正常写入新文件
```

#### 7.3.4 读流程（Fetcher）

分级存储的读取通过 `TieredMessageFetcher` 处理，核心决策是"从哪一层读"：

```java
// TieredMessageStore#getMessageAsync
// 1. 系统 Topic 消息走主存
// 2. 根据 tieredStorageLevel 决定是否走分级存储
// 3. 分级存储查询失败时降级走主存

// tieredStorageLevel 取值：
// DISABLE       - 禁用分级存储
// NOT_IN_DISK   - 默认，offset 不在本地磁盘时读分级存储
// NOT_IN_MEM    - offset 不在 PageCache 时读分级存储
// FORCE         - 强制走分级存储（测试用）
```

**预读缓存设计**：为了减少对二级存储的请求次数，引入了 Caffeine 缓存：

```java
// TieredMessageFetcher 缓存配置
Cache<MessageCacheKey, SelectBufferResult> cache = Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.SECONDS)  // 写后 10s 过期
    .maximumWeight(maxMemory * 0.3)           // 最大 JVM 内存的 30%
    .recordStats()
    .build();
```

读取流程采用类似 **Cache Aside** 的模式：

```
getMessageFromCacheAsync:
  Step1: 优先从 cache 逐条读取消息
  Step2: 如果全 miss，等预读线程完成
  Step3: 预读完成后再次尝试从 cache 读
  Step4: 如果缓存命中，组装结果返回，并触发预读
  Step5: 如果缓存未命中，以 2 倍 maxCount 从分级存储读取，放入缓存
```

#### 7.3.5 索引文件重排

索引文件（IndexFile）是为了支持按 Key 查询消息而设计的。传统格式使用 HashMap + 链地址法处理冲突，查询时需要多次随机读链表节点。这对于低 IOPS 的冷存储来说代价昂贵。

分级存储的索引文件对格式进行了优化——在文件写满后进行 **Compaction（重排）**，将同一个 hash 槽的索引项在物理位置上连续存储：

```
传统格式（随机 I/O 密集型）：
  Slot[0] → Item_A → Item_C → Item_D  (链表分散在文件各处)
  
重排后格式（顺序 I/O 友好）：
  Slot[0] → [Item_A, Item_C, Item_D]  (连续存储，一次读出)
```

具体的格式变化：

| 字段 | 传统格式 | 重排后格式 |
|------|----------|-------------|
| Hash 槽大小 | 4 字节（只存起始位置） | 8 字节（起始位置 + 总长度） |
| 索引项大小 | 32 字节（含链表指针） | 28 字节（去掉指针） |
| 查询方式 | 多次随机读 | 一次连续读 |

索引文件经历三个状态的流转：

```
UNSEALED（初始，类似本地格式，正在写入）
  → SEALED（写满，等待压缩）
    → 重排后写入新文件
    → UPLOAD（已上传到二级存储，删除本地临时文件）
```

```java
// IndexStoreService 服务线程
// 每 10s 扫描一次，找到最老的 SEALED 状态索引文件
// 执行 doCompactThenUploadFile:
//   1. 创建新文件，重排索引导入
//   2. 在分级存储中创建 FileSegment
//   3. 上传到二级存储
//   4. 替换索引文件表
//   5. 删除本地临时文件
```

### 7.4 RocksDB 与分级存储的关系

这里需要特别澄清一个常见的认知误区：**RocksDB 存储和分级存储是独立且可叠加的特性，而非谁基于谁实现**。

| 维度 | RocksDB 存储（RIP-66） | 分级存储（RIP-57/65） |
|------|------------------------|-----------------------|
| **解决的问题** | 百万队列下元数据和 CQ 的持久化效率 | 冷数据卸载到廉价存储以降本 |
| **替换什么** | ConsumeQueue + Config 存储 | CommitLog + CQ + Index 全部冷数据 |
| **不改什么** | CommitLog（始终保持 MMap 顺序写） | 热数据的写入路径 |
| **配置方式** | `storeType=defaultRocksDB` | `messageStorePlugIn=TieredMessageStore` |
| **元数据存储** | RocksDB 自身 | JSON 文件（`tieredStoreMetadata.json`） |
| **引入版本** | 5.2（RIP-66） | 5.1（RIP-57），5.2 重构（RIP-65） |

两者可以叠加使用——在 `RocksDBLiteLifecycleManager` 的实现中可以看到：

```
TieredMessageStore（外层：冷热分层）
  └── RocksDBMessageStore（内层：ConsumeQueue 用 RocksDB）
        └── MMap CommitLog（底层：保持顺序写）
```

**生产选型建议**：

| 场景 | 推荐配置 |
|------|----------|
| 普通场景 | `storeType=default` |
| 百万队列/LMQ | `storeType=defaultRocksDB` |
| 需要长期保留消息（>72h） | `+TieredMessageStore` 插件 |
| 超低延迟敏感场景 | 不分级，关闭 RocksDB compaction |

---

## 八、总结与展望

### 8.1 架构全景回顾

RocketMQ 5.x 的技术演进可以概括为"**一个中心，三条主线**"：

**中心：云原生化**
- 解耦控制面与数据面（Controller 独立部署）
- 存储与计算分离（分级存储）
- 协议标准化（gRPC）

**主线一：消费模型现代化**
- POP 消费解决 Rebalance 和队列独占问题
- 消费进度管理从客户端移到服务端
- 为后续的 Serverless 化奠定基础

**主线二：存储引擎分层化**
- RocksDB 存储解决了百万 Topic 场景下的元数据管理瓶颈
- 分级存储将冷数据卸载到廉价介质，显著降低存储成本
- 两者正交，可根据实际场景灵活组合

**主线三：通信协议标准化**
- gRPC + Proxy 架构统一了多语言 SDK
- 新客户端轻量设计，路由和心跳独立管理
- 安全性集中到 Proxy 层管控

### 8.2 版本选型建议

| 版本 | 核心特性 | 生产建议 |
|------|----------|----------|
| 5.0 | POP 消费、Controller 模式 | 特性验证，不建议生产 |
| 5.1 | Timer 消息、SlaveActingMaster | 可评估测试 |
| 5.2 | 分级存储 RIP-65、RocksDB RIP-66、sofa-jraft RIP-67 | **推荐生产起点** |
| 5.3+ | 双写模式、加速恢复、Pop 消费 RocksDB | 持续优化，按需升级 |

### 8.3 演进方向

RocketMQ 5.x 仍在快速演进中，值得关注的未来方向包括：

1. **直写模式完善**：在转写充分建设后，将热数据的 WAL 和索引构建移植过来，实现真正的直写；
2. **存储后端扩展**：目前仅有官方提供 POSIX 和内存实现，社区可开发 OSS、S3、HDFS 等多种实现；
3. **定时消息的 RocksDB 存储**（RIP-82）：进一步减少定时消息对本地磁盘的占用；
4. **更极致的 Serverless**：通过分级存储实现真正的共享存储架构，节点缩容时数据无缝迁移。

---

> **参考材料**：
> - [Apache RocketMQ GitHub](https://github.com/apache/rocketmq)
> - [RIP-57：Tiered storage for RocketMQ](https://github.com/apache/rocketmq/wiki/RIP-57-Tiered-storage-for-RocketMQ)
> - [RIP-65：Tiered Storage Optimization](https://github.com/apache/rocketmq/wiki/RIP-65-Tiered-Storage-Optimization)
> - [RIP-66：Support KV(RocksDB) Storage](https://github.com/apache/rocketmq/wiki/RIP-66-Support-KV(Rocksdb)-Storage)
> - [RIP-67：Replace DLedger with sofa-jraft](https://github.com/apache/rocketmq/issues/7300)
> - 程序猿阿越 RocketMQ 5.x 源码分析系列：[POP消费](https://juejin.cn/post/7280051662282244130)、[Controller模式](https://juejin.cn/post/7295036072755822604)、[SlaveActingMaster](https://juejin.cn/post/7301959466400251958)、[Timer消息](https://juejin.cn/post/7306102616312479798)、[新架构普通消息](https://juejin.cn/post/7311274792715108388)、[顺序消息](https://juejin.cn/post/7320169906222841856)、[分层存储](https://juejin.cn/post/7340603873605222435)
> - [Scarb：RocketMQ 分级存储原理详解 & 源码解析](https://hscarb.github.io/rocketmq/20240429-rocketmq-tieredstore.html)
> - [阿里云开发者社区：RocketMQ 5.0 分级存储背后一些有挑战的技术优化](https://developer.aliyun.com/article/1441642)
