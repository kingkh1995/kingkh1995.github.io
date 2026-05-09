![](assets/design-patterns.png)

### 分解模式 (Decomposition Patterns)
- **按业务能力分解** (Decompose by Business Capability)：根据业务领域内独立的价值创造能力划分子服务。
- **按子域分解** (Decompose by Subdomain)：依据领域驱动设计中的子域（核心域、支撑域、通用域）来拆分服务。
- **按事务分解** (Decompose by Transactions)：将业务事务边界作为服务划分的依据，确保事务完整性。
- **绞杀者模式** (Strangler Pattern)：逐步用新服务替换旧系统的部分功能，最终完全替代遗留系统。
- **隔板模式** (Bulkhead Pattern)：将系统资源隔离为独立分区，避免一个服务的故障拖垮整个系统。
- **边车模式** (Sidecar Pattern)：将辅助功能（如日志、监控）以独立进程部署在主服务旁边，共享同一资源环境。

### 集成模式 (Integration Patterns)
- **API网关模式** (API Gateway Pattern)：提供一个统一入口，对外暴露 API，负责路由、认证、限流等。
- **聚合器模式** (Aggregator Pattern)：一个服务调用多个下游服务，然后将结果合并返回给客户端。
- **代理模式** (Proxy Pattern)：一个服务作为另一个服务的替身，控制访问或进行额外处理。
- **网关路由模式** (Gateway Routing Pattern)：根据请求的不同将流量路由到对应的微服务。
- **链式微服务模式** (Chained Microservice Pattern)：服务之间通过同步调用链完成一个业务请求。
- **分支模式** (Branch Pattern)：一个服务根据请求内容，并发调用多个服务并组合它们的响应。
- **客户端UI组合模式** (Client-Side UI Composition Pattern)：在前端由各个微前端或 UI 组件自行组合页面区域。

### 数据库模式 (Database Patterns)
- **每服务数据库** (Database per Service)：每个微服务拥有自己独立的数据库，保证服务自治和数据封装。
- **每服务共享数据库** (Shared Database per Service)：多个服务共享同一个数据库，通过划分表或模式来区分，减少维护成本但耦合度较高。
- **CQRS** (Command Query Responsibility Segregation)：命令查询职责分离，将读操作和写操作分成不同的模型或服务，以优化性能和扩展性。
- **事件溯源** (Event Sourcing)：不直接存储当前状态，而是存储所有状态变更的事件序列，通过重放事件还原状态。
- **Saga模式** (Saga Pattern)：通过一系列本地事务和补偿操作，实现跨服务的长期事务一致性。

### 可观测性模式 (Observability Patterns)
- **日志聚合** (Log Aggregation)：集中收集各服务的日志，支持统一查询和分析。
- **性能指标** (Performance Metrics)：采集和监控服务运行数据（如延迟、吞吐量），用于性能分析和告警。
- **分布式追踪** (Distributed Tracing)：跟踪一个请求在多个微服务间的完整调用链路，便于定位瓶颈和故障。
- **健康检查** (Health Check)：让服务暴露一个可被负载均衡或容器编排系统查询的健康状态端点。

### 横切关注点模式 (Cross-Cutting Concern Patterns)
- **外部配置** (External Configuration)：将配置信息从服务内部剥离，通过外部配置中心（如配置服务器）统一管理。
- **服务发现模式** (Service Discovery Pattern)：服务实例能够动态注册和发现彼此，通常借助服务注册表实现。
- **断路器模式** (Circuit Breaker Pattern)：当某个服务调用失败率达到阈值时，短路请求并快速失败，防止级联故障。
- **蓝绿部署模式** (Blue-Green Deployment Pattern)：同时维护两套完整的生产环境（蓝和绿），通过切换路由实现零停机部署和回滚。
