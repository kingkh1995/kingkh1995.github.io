## 第25章 配置项手册

### 25.1 配置体系概述

Dubbo 的配置体系以 URL 为核心，遵循"统一配置模型"的设计：

- 所有配置最终解析为标准 URL 格式
- 支持多层配置覆盖机制
- 配置来源包括：系统属性、配置文件、环境变量、配置中心

### 25.2 配置加载流程

```
系统属性 / JVM 参数
    │
    ▼
环境变量
    │
    ▼
application.yml / application.properties (Spring Boot)
    │
    ▼
dubbo.properties
    │
    ▼
配置中心（远程）
    │
    ▼
最终配置 URL
```

上层覆盖下层：系统属性具有最高优先级，配置中心的动态配置可以在运行时覆盖静态配置。

### 25.3 核心配置项

#### 25.3.1 应用配置

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `dubbo.application.name` | string | 必填 | 应用名称 |
| `dubbo.application.version` | string | - | 应用版本 |
| `dubbo.application.owner` | string | - | 应用负责人 |
| `dubbo.application.organization` | string | - | 所属组织 |
| `dubbo.application.compiler` | string | javassist | 代理编译器 |
| `dubbo.application.logger` | string | slf4j | 日志框架 |

#### 25.3.2 服务配置

**Provider / Consumer 通用**：

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `timeout` | int | 1000 | 调用超时时间（ms） |
| `retries` | int | 2（不含首次） | 重试次数 |
| `loadbalance` | string | random | 负载均衡策略 |
| `cluster` | string | failover | 集群容错策略 |
| `version` | string | - | 服务版本 |
| `group` | string | - | 服务分组 |

#### 25.3.3 协议配置

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `dubbo.protocol.name` | string | dubbo | 协议名称（dubbo/triple/rest） |
| `dubbo.protocol.port` | int | -1（自动分配） | 服务端口 |
| `dubbo.protocol.host` | string | - | 绑定主机地址 |
| `dubbo.protocol.threads` | int | 200 | 业务线程池大小 |
| `dubbo.protocol.iothreads` | int | CPU+1 | IO 线程数 |
| `dubbo.protocol.heartbeat` | int | 60000 | 心跳间隔（ms） |
| `dubbo.protocol.charset` | string | UTF-8 | 字符编码 |

#### 25.3.4 注册中心配置

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `dubbo.registry.address` | string | - | 注册中心地址 |
| `dubbo.registry.protocol` | string | zookeeper | 注册中心协议 |
| `dubbo.registry.timeout` | int | 10000 | 注册中心连接超时（ms） |
| `dubbo.registry.register` | boolean | true | 是否注册到注册中心 |
| `dubbo.registry.subscribe` | boolean | true | 是否订阅注册中心 |
| `dubbo.registry.group` | string | dubbo | 注册中心分组 |
| `dubbo.registry.check` | boolean | true | 启动时是否检查注册中心 |

#### 25.3.5 QOS 配置

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `dubbo.application.qos-port` | int | 22222 | QOS 端口 |
| `dubbo.application.qos-enable` | boolean | true | 是否启用 QOS |
| `dubbo.application.qos-accept-foreign-ip` | boolean | false | 是否接受外网连接 |

### 25.4 三种配置方式对照

| 配置内容 | Spring Boot (application.yml) | XML (dubbo.xml) | API (Java代码) |
|---------|-----------------------------|-----------------|---------------|
| 应用名 | `dubbo.application.name=xxx` | `<dubbo:application name="xxx"/>` | `ApplicationConfig app = new ApplicationConfig("xxx")` |
| 协议 | `dubbo.protocol.name=xxx` | `<dubbo:protocol name="xxx"/>` | `ProtocolConfig protocol = new ProtocolConfig()` |
| 注册中心 | `dubbo.registry.address=xxx` | `<dubbo:registry address="xxx"/>` | `RegistryConfig registry = new RegistryConfig()` |
| 服务暴露 | `@DubboService` | `<dubbo:service interface="" ref=""/>` | `ServiceConfig<Xxx> service = new ServiceConfig<>()` |
| 服务引用 | `@DubboReference` | `<dubbo:reference interface="" id=""/>` | `ReferenceConfig<Xxx> ref = new ReferenceConfig<>()` |

---

## 第26章 SPI 扩展点清单

### 26.1 SPI 概述回顾

Dubbo SPI 是框架可扩展性的基础。所有核心组件均通过 SPI 定义接口，通过配置选择具体实现。扩展点配置文件位于 `META-INF/dubbo/internal/`。

### 26.2 完整 SPI 扩展点

| 分类 | 扩展点接口 | 默认实现 | 自适应策略 |
|------|-----------|---------|-----------|
| **协议** | | | |
| | `org.apache.dubbo.rpc.Protocol` | dubbo | @Adaptive("protocol") |
| | `org.apache.dubbo.remoting.Transporter` | netty | @Adaptive("server") |
| | `org.apache.dubbo.remoting.Exchanger` | header | @Adaptive("exchanger") |
| **调用** | | | |
| | `org.apache.dubbo.rpc.Filter` | - | @Activate 条件激活 |
| | `org.apache.dubbo.rpc.cluster.Cluster` | failover | @Adaptive("cluster") |
| | `org.apache.dubbo.rpc.cluster.RouterFactory` | condition | @Adaptive("router") |
| | `org.apache.dubbo.rpc.cluster.LoadBalance` | random | @Adaptive("loadbalance") |
| **注册** | | | |
| | `org.apache.dubbo.registry.RegistryFactory` | zookeeper | @Adaptive("protocol") |
| | `org.apache.dubbo.configcenter.ConfigCenterFactory` | zookeeper | @Adaptive("protocol") |
| | `org.apache.dubbo.metadata.report.MetadataReportFactory` | zookeeper | @Adaptive("protocol") |
| **代理** | | | |
| | `org.apache.dubbo.rpc.ProxyFactory` | javassist | @Adaptive("proxy") |
| | `org.apache.dubbo.common.compiler.Compiler` | javassist | - |
| **序列化** | | | |
| | `org.apache.dubbo.common.serialize.Serialization` | hessian2 | @Adaptive("serialization") |
| **线程池** | | | |
| | `org.apache.dubbo.common.threadpool.ThreadPool` | fixed | @Adaptive("threadpool") |
| | `org.apache.dubbo.remoting.Dispatcher` | all | @Adaptive("dispatcher") |
| **监控** | | | |
| | `org.apache.dubbo.monitor.MonitorFactory` | - | - |
| | `org.apache.dubbo.qos.probe.LivenessProbe` | - | - |
| | `org.apache.dubbo.qos.probe.ReadinessProbe` | - | - |
| **其他** | | | |
| | `org.apache.dubbo.cache.CacheFactory` | lru | @Adaptive("cache") |
| | `org.apache.dubbo.validation.Validation` | - | @Adaptive("validation") |
| | `org.apache.dubbo.rpc.telnet.TelnetHandler` | - | @Activate |

---

## 第27章 错误码 FAQ

### 27.1 错误码体系结构

Dubbo 的错误码编码格式为：**{层级}-{序号}**

| 层级 | 范围 | 说明 |
|------|------|------|
| 0 | 0-x | Common 层（公共异常） |
| 1 | 1-x | Registry 层（注册中心异常） |
| 2 | 2-x | Router 层（路由异常） |
| 3 | 3-x | Protocol 层（协议异常） |
| 4 | 4-x | Config 层（配置异常） |

### 27.2 Common 层错误码（0-x）

| 码号 | 错误原因 | 解决方向 |
|------|---------|---------|
| 0-1 | 线程池资源枯竭 | 增加线程池大小或优化业务逻辑处理速度 |
| 0-2 | 非法属性值 | 检查配置项的值是否符合要求 |
| 0-3 | 无法访问缓存路径 | 检查文件系统权限，确认缓存目录可写 |
| 0-7 | 未找到反射类 | 检查类路径（classpath）和相关依赖 |
| 0-15 | 加载扩展类时发生异常 | 检查 SPI 配置文件和扩展实现类 |
| 0-21 | 构建的实例过多 | 检查是否存在循环引用或过度创建对象 |
| 0-23 | 序列化数据转换异常 | 检查序列化对象的兼容性 |
| 0-28 | 危险的行为操作 | 确认操作的安全风险 |

### 27.3 Registry 层错误码（1-x）

| 码号 | 错误原因 | 解决方向 |
|------|---------|---------|
| 1-1 | 地址非法 | 检查注册中心地址格式 |
| 1-4 | 空地址 | 检查 Provider 是否成功注册 |
| 1-17 | Metadata Server 失效 | 检查元数据中心的状态 |
| 1-35 | ZK 异常 | 检查 Zookeeper 集群状态 |
| 1-37 | Nacos 异常 | 检查 Nacos Server 状态 |
| 1-38 | Socket 连接异常 | 检查网络连通性和防火墙策略 |
| 1-39 | 获取元数据失败 | 检查元数据中心配置和连接 |

### 27.4 Router 层错误码（2-x）

| 码号 | 错误原因 | 解决方向 |
|------|---------|---------|
| 2-1 | 路由选址执行失败 | 检查路由规则配置是否正确 |
| 2-2 | 没有可用的 Provider | 确认 Provider 已注册并在线，否则检查路由规则是否误过滤 |
| 2-3 | 路由关闭失败 | 等待当前请求处理完成再关闭 |
| 2-10 | 调用服务提供方失败 | 检查 Provider 的状态和日志 |

### 27.5 Protocol 层错误码（3-x）

（常见协议层异常以具体的 RpcException 形式抛出，详细的协议层错误码请参阅 Dubbo 官方文档。）

### 27.6 Config 层错误码（4-x）

（常见配置层异常包括配置项校验失败、配置冲突等，详见 Dubbo 官方文档。）

---

## 第28章 源码架构

### 28.1 代码模块划分

Dubbo 的源码按功能分为以下模块：

| 模块 | 路径 | 职责 |
|------|------|------|
| dubbo-common | `dubbo-common/` | 公共工具类、SPI 框架、配置解析 |
| dubbo-remoting | `dubbo-remoting/` | 网络通信抽象与实现（Netty） |
| dubbo-rpc | `dubbo-rpc/` | RPC 协议实现（dubbo/triple/rest） |
| dubbo-cluster | `dubbo-cluster/` | 集群容错、路由、负载均衡 |
| dubbo-registry | `dubbo-registry/` | 注册中心实现 |
| dubbo-config | `dubbo-config/` | 配置解析（Spring Boot / API / XML） |
| dubbo-metadata | `dubbo-metadata/` | 元数据管理 |
| dubbo-monitor | `dubbo-monitor/` | 监控统计 |
| dubbo-filter | `dubbo-filter/` | 内置 Filter 实现 |
| dubbo-plugin | `dubbo-plugin/` | 插件模块 |
| dubbo-native | `dubbo-native/` | GraalVM Native Image 支持 |

### 28.2 模块依赖关系

```
dubbo-common（基础）
    │
    ├── dubbo-remoting（网络层）
    │
    ├── dubbo-rpc（协议层）
    │
    ├── dubbo-cluster（集群层）
    │     │
    ├── dubbo-registry（注册层）←→ dubbo-metadata（元数据层）
    │     │
    └── dubbo-config（配置层）
          │
      dubbo-monitor（监控层）
```

### 28.3 服务调用源码路径

以 Consumer 发起到 Provider 的一次三荷调用为例，源码追踪路径如下：

```
org.apache.dubbo.rpc.proxy.InvokerInvocationHandler
  → org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker
    → org.apache.dubbo.rpc.cluster.directory.RegistryDirectory
      → org.apache.dubbo.rpc.cluster.router.RouterChain
        → org.apache.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
          → org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker
            → org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeClient
              → org.apache.dubbo.remoting.transport.netty4.NettyClient
                → org.apache.dubbo.remoting.transport.netty4.NettyCodecAdapter
```

### 28.4 扩展点开发指南

Dubbo 的 SPI 加载机制位于 `org.apache.dubbo.common.extension.ExtensionLoader` 中。

**核心流程**：
1. `ExtensionLoader.getExtensionLoader(SPI接口.class)` 获取加载器
2. 扫描 `META-INF/services/`、`META-INF/dubbo/`、`META-INF/dubbo/internal/` 目录
3. 按需加载指定的扩展实现（懒加载）
4. 处理 @Adaptive 注解生成自适应实现
5. 处理 @Activate 注解按条件激活扩展
6. 处理包装类（Wrapper）实现扩展链

**自定义扩展步骤**：
1. 实现目标 SPI 接口
2. 在 `META-INF/dubbo/` 下创建以 SPI 接口全限定名命名的文件
3. 文件中写入扩展名与实现类的映射关系
4. 通过配置或注解引用自定义扩展
