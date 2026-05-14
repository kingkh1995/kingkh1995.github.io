## 第15章 RPC 框架进阶

### 15.1 消费端线程模型

消费端线程模型是理解 Dubbo 调用性能的关键。一次典型的同步 RPC 调用，涉及如下线程切换：

```
业务线程（Business Thread）
  │
  │ 调用 Proxy 接口方法
  │
  ▼
Cluster 层（路由 + 负载均衡）
  │
  ▼
Protocol 层（构建 RPC Request）
  │
  ▼
IO 线程（Netty EventLoop）
  │
  ├── 序列化请求对象
  ├── 写入 Socket
  ├── 等待响应（非阻塞）
  └── 读取响应，反序列化
  │
  ▼
业务线程（唤醒，获取结果）
```

**线程切换模型**：
- **同步调用**：业务线程发出请求后在 DefaultFuture 上等待（block）。IO 线程接收到响应后，唤醒等待的业务线程。一次调用涉及 2 次线程切换。
- **异步调用**：业务线程发出请求后立即返回 CompletableFuture。IO 线程接收到响应后，通过回调执行后续逻辑。无阻塞等待，线程利用率更高。

**设计要点**：
- 业务线程不宜过多，避免线程上下文切换开销
- IO 线程数通常设置为 CPU 核数（Netty 默认 2 × CPU）
- 避免在 IO 线程中执行耗时操作（如复杂的序列化、加锁等）

### 15.2 提供端线程模型

```
Provider Side:
┌──────────┐    ┌──────────┐    ┌───────────┐
│ IO Thread │ → │ Worker   │ → │ 业务线程池  │
│ (Netty)   │    │ (Handler)│    │ (ThreadPool)│
└──────────┘    └──────────┘    └───────────┘
      │              │               │
      │ 解码请求     │ 连接管理       │ 执行业务逻辑
      │ 编码响应     │ 心跳检测       │ Filter 链
      └──────────────┴───────────────┘
```

**线程池策略**：

| 策略 | 实现 | 特点 |
|------|------|------|
| fixed | FixedThreadPool | 固定大小，默认，适合常规场景 |
| cached | CachedThreadPool | 弹性伸缩，适合突发流量 |
| limited | LimitedThreadPool | 弹性伸缩但上限固定，适合资源受限 |
| eager | EagerThreadPool | 优先创建新线程而非入队列，适合高吞吐 |

**线程池参数调整关键点**：
- `threads`：核心线程数，影响服务并发处理能力
- `queues`：任务队列长度，影响请求的排队等待时间
- 线程池耗尽时，新请求将被拒绝并返回异常（`Thread pool is EXHAUSTED`）

### 15.3 Filter 拦截器机制

Filter 是 Dubbo 中最重要的拦截扩展点之一，作用类似于 Servlet Filter 或 Spring Interceptor。

**Filter 链机制**：
```
Consumer Side:  ContextFilter → ConsumerContextFilter → MonitorFilter → ...
                      ↓
Provider Side:  ExceptionFilter → TokenFilter → ClassLoaderFilter → AccessLogFilter → ...
```

每个 Filter 可以在 Invoker 执行前后插入自定义逻辑。多个 Filter 串联形成 Filter 链。

**内置 Filter 列表**：

| Filter | 作用 |
|--------|------|
| AccessLogFilter | 访问日志记录 |
| ExceptionFilter | 异常处理与包装 |
| TokenFilter | Token 鉴权 |
| ClassLoaderFilter | 线程上下文类加载器切换 |
| TimeoutFilter | 超时处理 |
| MonitorFilter | 监控统计 |
| ContextFilter | RpcContext 管理 |
| CacheFilter | 结果缓存 |
| ValidationFilter | 参数校验 |

**自定义 Filter**：通过实现 `Filter` 接口并注册为 SPI 扩展，可以实现自定义拦截逻辑，如日志增强、分布式链路注入、请求签名等。

### 15.4 异步调用

Dubbo 从 2.7.x 开始全面支持基于 `CompletableFuture` 的异步编程模型。

**同步 vs 异步**：

| 对比项 | 同步调用 | 异步调用 |
|--------|---------|---------|
| 线程模型 | 业务线程阻塞等待 | 业务线程立即返回 |
| 资源利用率 | 占用线程等待 | 线程可处理其他任务 |
| 编程复杂度 | 简单 | 需要回调处理 |
| 吞吐能力 | 受线程数限制 | 可支持更高并发 |

**异步调用模式**：
- `CompletableFuture<T>` 返回：立即返回 Future，IO 线程结束后自动完成 Future
- 支持链式编排：`.thenApply()`, `.thenCompose()`, `.whenComplete()`
- 异常处理：`.exceptionally()` 捕获异步异常

**使用建议**：
- IO 密集型的服务调用，推荐异步调用模式
- CPU 密集型的业务逻辑，同步调用也可以满足需求
- 异步调用提高了吞吐上限，但编程复杂度更高

### 15.5 版本与分组

**服务版本管理**：
- 通过 `version` 属性管理服务的不同版本
- Consumer 指定要调用的版本号，Provider 声明提供哪个版本的服务
- 版本升级策略：低版本 Provider 与高版本 Consumer 共存，逐步滚动升级

**服务分组**：
- 通过 `group` 属性对服务进行逻辑分组
- 同一接口的不同分组被视为完全独立的服务
- 典型应用：`group=dev` / `group=test` / `group=prod` 实现多环境隔离
- 与标签路由结合，可以实现更灵活的分组策略

### 15.6 隐式参数传递

**RpcContext**：Dubbo 使用 RpcContext 在调用链中传递隐式参数。

**使用场景**：
- 传递 Trace ID：分布式追踪中跨服务传递链路信息
- 传递用户身份：在服务间透传认证信息
- 传递业务上下文：如请求来源、租户 ID 等

**实现原理**：
- Consumer 端：`RpcContext.getContext().setAttachment(key, value)`
- Provider 端：`RpcContext.getContext().getAttachment(key)`
- 参数随 RPC 请求自动传递，无需在每个接口方法中显式定义参数

**注意事项**：
- 隐式参数仅在当前调用链路中传递
- 参数应保持小体积（序列化传输）
- 敏感信息应考虑加密传输

### 15.7 集群容错策略

当 Consumer 调用 Provider 失败时，集群容错策略决定了 Dubbo 的行为。

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| Failover | 失败自动切换，重试其他 Provider | 幂等操作，默认策略 |
| Failfast | 快速失败，立即抛出异常 | 非幂等操作，写操作 |
| Failsafe | 失败安全，忽略异常 | 日志记录等不重要调用 |
| Failback | 失败自动恢复，后台重试 | 消息通知等允许延迟场景 |
| Forking | 并行调用多个 Provider | 高实时性要求场景 |
| Broadcast | 广播调用所有 Provider | 状态更新需要同步到所有节点 |

**Failover 重试策略详解**：
- `retries=2` 表示失败后最多重试 2 次（默认值，不含首次）
- 每次重试通过 LoadBalance 重新选择 Provider
- 重试可能放大对 Provider 的压力（重试风暴）

**可用性 vs 一致性取舍**：
- Failover + 重试：高可用，但可能重复执行
- Failfast：一致性优先，放弃可用性
- Failsafe：可用性优先，放弃一致性

### 15.8 泛化调用

**使用场景**：
- API 网关：网关不需要依赖服务端的接口 JAR
- 测试平台：动态调用任意服务方法
- 服务治理：管理面需要探测服务健康状态

**工作方式**：
- Consumer 不持有服务接口的编译产物
- 通过 `GenericService` 接口发起调用，传入方法名、参数类型、参数值
- Provider 端将请求路由到实际的服务实现

**泛化实现**：在 Provider 端，不实际持有接口定义的情况下实现服务逻辑。通过 Spring Bean 的注入方式处理请求。

---

## 第16章 自定义扩展（SPI）

### 16.1 Dubbo SPI 机制原理

SPI（Service Provider Interface）是 Dubbo 可扩展性的基石。所有核心组件都是可替换的，用户可以通过自定义 SPI 实现来扩展 Dubbo 的功能。

**与 Java SPI 的区别**：

| 对比项 | Java SPI | Dubbo SPI |
|--------|---------|-----------|
| 加载方式 | 全量加载 | 按需懒加载 |
| 缓存机制 | 无 | 有（实例缓存） |
| 依赖注入 | 无 | 支持自适应注入 |
| 自适应扩展 | 无 | @Adaptive 支持 |
| 自动激活 | 无 | @Activate 支持 |
| 扩展点包装 | 无 | 支持 AOP 式包装 |

**@Adaptive（自适应扩展）**：
- 标注在接口方法上，表示该方法在运行时根据参数动态选择扩展实现
- 在编译时生成自适应类，运行期根据 URL 参数决定具体实现
- 示例：Protocol 接口的 `@Adaptive` 方法根据 `protocol` 参数选择 dubbo/triple/rest

**@Activate（自动激活扩展）**：
- 根据条件自动激活某些扩展
- 支持 `group`（消费端/提供端）、`value`（URL 参数匹配）、`order`（执行顺序）
- 典型应用：Filter 链中按条件动态编排 Filter 的执行

**扩展点包装**：
- 如果 SPI 实现有一个接收相同 SPI 接口类型的构造器，则该实现会被视为"包装类"
- 包装类会包裹在其他实现之外，形成链式调用
- 典型应用：ProtocolFilterWrapper 在 Protocol 实现外包装 Filter 链

### 16.2 主要 SPI 扩展点

| 分类 | 扩展点接口 | 说明 |
|------|-----------|------|
| 协议 | `Protocol` | RPC 协议实现 |
| 调用拦截 | `Filter` | 请求/响应拦截器 |
| 注册中心 | `RegistryFactory` + `Registry` | 服务注册与发现 |
| 路由 | `RouterFactory` + `Router` | 路由规则实现 |
| 负载均衡 | `LoadBalance` | 负载均衡策略 |
| 集群容错 | `Cluster` | 集群容错策略 |
| 序列化 | `Serialization` | 序列化/反序列化 |
| 网络传输 | `Transporter` | 网络传输层 |
| 代理工厂 | `ProxyFactory` | 动态代理生成 |
| 线程池 | `ThreadPool` | 线程池策略 |
| 配置中心 | `ConfigCenter` | 配置中心接入 |
| 元数据 | `MetadataReport` | 元数据上报 |

### 16.3 扩展点开发最佳实践

1. **单一职责原则**：每个 SPI 接口只负责一个功能维度，避免接口膨胀
2. **依赖注入**：SPI 实现可以通过 setter 方法注入其他 SPI 实现
3. **性能考虑**：扩展点加载过程增加了一层间接调用，关键路径上的扩展需要注意性能
4. **兼容性**：自定义扩展应保持与原生配置的兼容性

---

## 第17章 分布式事务

### 17.1 分布式事务场景

在微服务架构中，一个业务操作可能涉及多个独立服务的数据库操作，此时需要分布式事务来保证数据一致性。

### 17.2 Seata 集成

Seata 是阿里开源的分布式事务解决方案，Dubbo 提供了与 Seata 的原生集成。

**AT 模式**：
- 基于关系数据库的本地 ACID 事务
- 通过拦截 SQL 解析，自动生成反向 SQL 用于回滚
- **一阶段**：执行业务 SQL，同时记录 undo-log
- **二阶段**：全局提交时清除 undo-log，全局回滚时执行 undo-log 中记录的 SQL

**TCC 模式**：
- Try：预留业务资源
- Confirm：确认执行业务操作
- Cancel：取消执行业务操作
- 需要业务方实现三个阶段的逻辑

**Saga 模式**：
- 长事务拆分为多个本地事务
- 每个本地事务有对应的补偿操作
- 执行失败时，依次执行反向补偿操作

**XA 模式**：
- 基于 X/Open XA 规范
- 利用数据库对 XA 的原生支持
- 强一致性，但性能较低

### 17.3 分布式事务适用场景

| 模式 | 一致性 | 性能 | 代码侵入 | 适用场景 |
|------|--------|------|---------|---------|
| AT | 最终一致 | 高 | 低 | 大多数场景 |
| TCC | 最终一致 | 高 | 高 | 资源预留场景 |
| Saga | 最终一致 | 高 | 中 | 长事务 |
| XA | 强一致 | 低 | 低 | 对一致性要求极高 |

**性能权衡**：分布式事务不是银弹，应优先通过最终一致性和补偿机制解决问题，尽可能避免跨服务事务。

---

## 第18章 HTTP 网关接入

### 18.1 网关架构设计

在前端 HTTP 流量需要通过 Dubbo 后端服务处理时，引入 HTTP 网关层：

```
前端 (HTTP)
  │
  ▼
HTTP 网关 (Nginx / Spring Cloud Gateway / ShenYu / APISIX)
  │
  ├── HTTP → Dubbo 协议转换
  │
  ▼
Dubbo 后端服务集群
```

**核心问题**：网关需要将 HTTP 请求转换为 Dubbo RPC 调用，包括参数映射、协议转换、结果返回适配。

### 18.2 dubbo 协议网关接入

采用 dubbo 协议时，使用泛化调用模式通过 Gateway SDK 调用后端：

- 网关通过 `GenericService` 发起泛化调用
- 网关需要知道接口名、方法名、方法签名
- 调用结果序列化为 JSON 返回给前端

**适用场景**：后端统一使用 dubbo 协议，网关做协议转换。

### 18.3 triple 协议网关接入

triple 协议兼容 gRPC，天然支持 HTTP 协议：

- 网关可以直接代理 HTTP/2 流量到 triple Provider
- 无需额外的协议转换
- 支持 gRPC-Web 与标准 gRPC（HTTP/2）两种前端接入方式

**优势**：架构更简洁，无需在网关层进行协议转换（或只需轻量转换）。

### 18.4 网关选型建议

| 网关方案 | 协议支持 | 性能 | 生态 |
|---------|---------|------|------|
| ShenYu | dubbo + triple | 高 | Dubbo 生态原生 |
| APISIX | triple（HTTP/2） | 高 | 云原生 |
| Spring Cloud Gateway | 需自研 Dubbo 适配 | 中 | Spring 生态 |
| Nginx + Lua | 需自研 Dubbo 适配 | 高 | 传统方案 |
