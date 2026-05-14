## 第5章 环境搭建与快速开始

### 5.1 开发环境要求

Dubbo 3.x 的开发环境要求如下：

| 组件 | 版本要求 |
|------|---------|
| JDK | 8 以上（推荐 JDK 17+ 以获得最佳性能） |
| Maven | 3.6+ 或 Gradle 7+ |
| Spring Boot | 2.3+（使用 Starter 模式） |
| 构建工具 | Maven / Gradle |

Dubbo 3.x 与 JDK 8/11/17 均保持兼容，但从长期演进角度来看，建议迁移到 JDK 17+。

### 5.2 Spring Boot Starter 模式

Dubbo Spring Boot Starter 是官方推荐的开发方式，它通过 Spring Boot 的自动装配机制将 Dubbo 无缝集成到 Spring Boot 应用中。

**自动装配原理**：
- `@EnableDubbo` 注解触发 Dubbo 配置的自动装配
- `DubboAutoConfiguration` 负责注册核心组件到 Spring 容器
- `DubboServiceAutoConfiguration` 负责扫描 `@DubboService` 注解的服务并暴露为远程服务
- `DubboReferenceAutoConfiguration` 负责处理 `@DubboReference` 注解的服务引用注入

### 5.3 API 编程模式

Dubbo 的 API 编程模式不依赖 Spring，允许在纯 Java 环境中使用 Dubbo：

- **ServiceConfig**：用于暴露服务的 API 封装，手动配置协议、注册中心、服务接口和实现
- **ReferenceConfig**：用于引用远程服务的 API 封装，手动配置协议、注册中心、服务接口
- 适合非 Spring 应用、测试环境、或者对 Spring 依赖敏感的场景

### 5.4 应用初始化流程

从应用启动到服务暴露/引用的完整流程：

```
Provider 启动流程：
  1. 加载配置（application.yml / dubbo.properties / API 代码）
  2. 解析配置对象（ServiceConfig → ServiceMetadata）
  3. 初始化注册中心客户端（RegistryProtocol）
  4. 构建 Invoker（ProxyFactory.getInvoker）
  5. 暴露服务（Protocol.export → 开启网络监听）
  6. 注册服务地址到注册中心（Registry.register）
  7. 启动完成，等待 Consumer 调用

Consumer 启动流程：
  1. 加载配置
  2. 解析配置对象（ReferenceConfig → ReferenceMetadata）
  3. 初始化注册中心客户端
  4. 订阅服务（Registry.subscribe）
  5. 获取 Provider 地址列表并建立连接
  6. 生成代理对象（ProxyFactory.getProxy）
  7. 注入代理对象到业务代码
  8. 启动完成，等待调用
```

### 5.5 三种开发模式对比

| 维度 | Spring Boot Starter | XML 配置 | API 编程 |
|------|-------------------|---------|---------|
| 配置方式 | properties/yaml + 注解 | XML 文件 | Java 代码 |
| 自动装配 | 是 | 需手动引入 XML | 需手动构建 |
| 灵活性 | 中等 | 高 | 最高 |
| 推荐度 | 强烈推荐 | 维护期项目 | 非 Spring 项目 |
| 适配 Spring Boot | 原生支持 | 需额外适配 | 不支持 |

**选型原则**：新项目优先选择 Spring Boot Starter；维护旧项目可选 XML；无 Spring 依赖的场景使用 API 编程。

---

## 第6章 通信协议选择与配置

### 6.1 协议体系概览

Dubbo 3.x 支持多种 RPC 通信协议，按设计思想可分为三大类：

- **HTTP 协议类**：triple（推荐）、REST
- **TCP 私有协议类**：dubbo
- **兼容协议类**：gRPC、Thrift、Hessian、WebService、Redis、Memcached 等

### 6.2 triple 协议（推荐）

triple 是 Dubbo 3.x 主推的 RPC 协议，它兼容 gRPC 标准并扩展了 Dubbo 的服务治理能力。

**技术特性**：

- **HTTP/1.1 + HTTP/2 双重支持**：统一端口支持两种协议，HTTP/2 提供多路复用、头部压缩、Server Push 等能力；HTTP/1.1 提供对传统 HTTP 客户端的兼容
- **与 gRPC 完全兼容**：triple 协议的 wire format 与 gRPC 一致，Dubbo 服务可以与标准 gRPC 客户端（Go、Python、Node.js 等）互通
- **流式通信模型**：支持四种 gRPC 标准的通信模式

**Streaming 通信模型**：

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| Unary | 一次请求 + 一次响应 | 标准 RPC 调用 |
| Server Stream | 一次请求 + 流式响应 | 大结果集返回、推送 |
| Client Stream | 流式请求 + 一次响应 | 大文件上传、批量数据提交 |
| Bidirectional Stream | 双向流式通信 | 实时聊天、实时协同 |

- **Protobuf 序列化**：triple 协议的默认序列化方案，提供强类型定义和跨语言互通能力
- **背压机制（3.3+）**：在流式通信中支持背压（Backpressure），通过 HTTP/2 的流控机制实现流量感知和自适应调整

**适用场景**：跨语言通信、需要与 gRPC 生态互通、流式通信、云原生环境。**推荐作为新项目的默认协议**。

### 6.3 dubbo 协议（TCP）

dubbo 协议是 Dubbo 经典的 TCP 私有 RPC 协议。

**技术特性**：
- 基于 TCP 的长连接复用
- 低延迟（因省去 HTTP 协议的开销）
- 内置连接池和心跳保活

**字段格式**：
- Header（定长 16 字节）：Magic + Flags + Status + Request ID + Body Length
- Body（变长）：序列化后的请求/响应数据

**适用场景**：
- **推荐使用**：高并发内网调用、延迟敏感型场景
- **不推荐**：需要跨语言互通、需要通过 HTTP 网关的场景

**优缺点分析**：

| 优点 | 缺点 |
|------|------|
| 性能极高，吞吐量大 | 非标准协议，无法与 HTTP 工具互通 |
| 长连接复用，减少连接开销 | 跨语言支持有限 |
| 连接池管理成熟 | 需要通过额外网关接 HTTP 流量 |

### 6.4 REST 协议

Dubbo 通过 REST 协议支持发布 RESTful 风格的 HTTP 服务。

**技术特性**：
- 基于 JAX-RS 标准注解（`@GET`、`@POST`、`@Path` 等）
- 直接支持 HTTP 工具（curl、Postman）和浏览器调试
- 便于异构系统集成

**适用场景**：对外开放 API、移动端后端服务、异构系统（非 Java）调用。

### 6.5 多协议配置

Dubbo 3.x 支持**单端口多协议**（Single Port Multi-Protocol）。

**实现原理**：在同一个端口上监听多种协议请求，通过请求的首字节识别不同的协议。引入 Protocol Detector 机制，将入站请求分发到对应的协议处理器。

**优势**：
- 减少端口占用
- 简化运维配置
- 统一网络层管理

### 6.6 协议选型建议

```
服务通信场景
├── 内部服务间调用
│   ├── 高并发低延迟需求 → dubbo 协议 (TCP)
│   ├── 多语言互通需求 → triple 协议 (gRPC 兼容)
│   └── 混合推荐 → triple 协议（兼顾性能与兼容性）
├── 外部 API 开放
│   ├── RESTful 风格 → REST 协议
│   └── 高性能 API → triple 协议 + HTTP/2
├── 流式通信场景
│   └── 推荐 → triple 协议（原生支持 Streaming）
└── 服务网格环境
    └── 推荐 → triple 协议（Proxyless 支持）
```

---

## 第7章 注册中心集成

### 7.1 注册中心的抽象架构

Dubbo 通过 Registry SPI 实现了注册中心的抽象，使得用户可以选择不同的注册中心实现而无需修改业务代码。核心架构如下：

```
RegistryFactory        [工厂接口，创建 Registry 实例]
  └── Registry         [核心接口，定义注册/订阅/注销/取消订阅]
        ├── ZookeeperRegistry
        ├── NacosRegistry
        ├── KubernetesRegistry
        ├── ConsulRegistry
        ├── EtcdRegistry
        └── RedisRegistry
```

### 7.2 Zookeeper 作为注册中心

Zookeeper 是 Dubbo 最经典的注册中心选择，也是生产环境中最广泛使用的方案。

**数据模型与存储结构**：
```
/dubbo
  /com.example.UserService
    /providers
      /dubbo://192.168.1.1:20880/com.example.UserService?...
    /consumers
      /consumer://192.168.1.2:20880/com.example.UserService?...
    /configurators
      /override://0.0.0.0/com.example.UserService?...
    /routers
      /condition://0.0.0.0/com.example.UserService?...
```

**工作原理**：
- Provider 在 `/dubbo/{interface}/providers` 下创建临时节点
- Consumer 监听 `/dubbo/{interface}/providers` 的子节点变化
- 临时节点在 Provider 断开连接时自动移除
- 通过 ZK Watcher 机制实现变更推送

**优缺点**：

| 优点 | 缺点 |
|------|------|
| 成熟稳定，生产验证充分 | 在大规模节点下，Watch 风暴可能压垮 ZK |
| 临时节点机制天然支持健康检测 | 数据量受限于 ZNode 的大小限制 |
| CP 模型，数据一致性强 | 需要独立部署并维护 ZK 集群 |

### 7.3 Nacos 作为注册中心

Nacos 是阿里开源的服务发现和配置管理平台，Dubbo 对其提供了全面的支持。

**工作原理**：
- Provider 通过 HTTP 或 gRPC 注册服务到 Nacos Server
- Consumer 通过订阅接口从 Nacos 获取服务列表
- Nacos 通过 UDP 或 gRPC 推送变更

**优缺点**：

| 优点 | 缺点 |
|------|------|
| 内置配置中心，AP+CP 模式 | 在大规模服务推送场景下偶有延迟 |
| 自动健康检查，支持权重设置 | Nacos 2.x 基于 gRPC，运维复杂度略高 |
| 轻量管理面（Web Console） | |

### 7.4 Kubernetes 作为注册中心

Dubbo 3.x 支持基于 Kubernetes Service 的服务发现，无需独立部署注册中心。

**工作原理**：
- Dubbo SDK 通过 K8s API Server 的 Watch 机制获取 EndpointSlice
- Pod IP + Service Port 构成 Provider 地址
- Readiness Probe 决定 Pod 是否被纳入服务端点

**优缺点**：

| 优点 | 缺点 |
|------|------|
| 无需额外部署注册中心 | 强依赖 K8s 环境 |
| 原生 K8s 生命周期管理 | 非 K8s 环境不适用 |
| 与 Service Mesh 无缝集成 | |

### 7.5 多注册中心配置

Dubbo 支持同时注册到多个注册中心，实现跨注册中心的服务互通：

- 同一个 Provider 可以同时注册到 Zookeeper + Nacos
- Consumer 可以从多个注册中心获取地址列表
- 常用于多机房部署或多环境迁移过渡期

### 7.6 注册中心选型建议

| 场景 | 推荐方案 |
|------|---------|
| 传统 VM 部署，内部调用 | Zookeeper 或 Nacos |
| K8s 部署，云原生 | K8s Service Discovery |
| K8s 部署 + 服务网格 | K8s + Istio / Dubbo Mesh |
| 需要配置中心 + 注册中心 | Nacos（一站式） |
| 大规模集群（数千+节点） | Nacos 或 K8s（避免 ZK Watch 风暴） |

---

## 第8章 应用部署

### 8.1 传统注册中心部署模式（VM 环境）

在传统虚拟机上部署 Dubbo 应用是最经典的部署方式。

**架构拓扑**：
```
                    ┌─────────────┐
                    │   Registry   │
                    │ (ZK / Nacos) │
                    └──────┬──────┘
              ┌────────────┼────────────┐
              │            │            │
         ┌────▼────┐ ┌────▼────┐ ┌────▼────┐
         │Provider1 │ │Provider2 │ │ProviderN │
         │UserSrv   │ │UserSrv   │ │UserSrv   │
         └────┬────┘ └────┬────┘ └────┬────┘
              │            │            │
         ┌────▼────────────▼────────────▼────┐
         │           Consumer               │
         │        (Gateway / App)           │
         └──────────────────────────────────┘
```

**部署建议**：
- Provider 部署在独立的服务器或容器中，避免资源共享冲突
- 注册中心集群至少部署 3 节点保证高可用
- 使用 Nginx / HAProxy 作为网关层统一入口

**高可用部署方案**：
1. 注册中心集群（3+ 节点）
2. 多副本 Provider 部署（至少 2 副本）
3. Consumer 设置多注册中心（跨机房容灾）

### 8.2 Kubernetes 部署

Dubbo 3.x 对 Kubernetes 提供了深度适配。

**部署架构**：
```
           ┌─────────────────────────────────────┐
           │           Kubernetes Cluster        │
           │                                     │
           │  ┌─────────────────────────────┐    │
           │  │  Consumer (Deployment)       │    │
           │  └──────────┬──────────────────┘    │
           │             │ dubbo://              │
           │  ┌──────────▼──────────────────┐    │
           │  │  Service (ClusterIP)         │    │
           │  │  - user-service              │    │
           │  └──────────┬──────────────────┘    │
           │             │                        │
           │  ┌──────────▼──────────────────┐    │
           │  │  Provider (StatefulSet)      │    │
           │  │  Pod-1 │ Pod-2 │ Pod-N       │    │
           │  └─────────────────────────────┘    │
           │                                     │
           │  ┌─────────────────────────────┐    │
           │  │  API Server (Service Discovery) │
           │  └─────────────────────────────┘    │
           └─────────────────────────────────────┘
```

**K8s 服务发现适配**：
- Dubbo 消费端通过 K8s API Server 获取 EndpointSlice
- Provider Pod 就绪后自动加入 Service 端点
- Pod 被删除或故障时自动移除

**生命周期集成**：
- **Readiness Probe**：Dubbo 服务完全就绪后返回成功
- **Liveness Probe**：Dubbo 框架状态正常时返回成功
- **PreStop Hook**：Provider 在 Pod 停止前通知注册中心取消注册，避免流量中断

### 8.3 服务网格（Service Mesh）部署

Dubbo 3.x 支持两种方式接入服务网格：

**Dubbo Mesh（Proxyless 模式）**：
- Dubbo SDK 直接与 Istiod 控制面通信
- 不通过 Sidecar 代理转发数据流
- 延迟比 Sidecar 模式更低

**Istio 集成**：
- Dubbo 通过 xDS 协议从 Istiod 获取服务发现与路由配置
- 支持 VirtualService、DestinationRule 等 Istio CRD
- 流量管控规则由 Istio 控制面统一管理

**部署模式对比**：

| 模式 | 数据面 | 控制面 | 延迟 | 运维复杂度 |
|------|--------|--------|------|-----------|
| 传统部署 | Dubbo SDK | 注册中心 | 低 | 低 |
| Sidecar 模式 | Dubbo SDK + Envoy | Istio | 中（+Sidecar 延迟） | 高 |
| Proxyless 模式 | Dubbo SDK（xDS） | Istio | 低 | 中 |
