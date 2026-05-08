## 计算机基础

- 操作系统
    - 进程 & 线程
    - 内存管理
    - 文件系统
    - CPU 缓存 & 伪共享
    - 锁 & 同步机制
- Linux
    - 性能分析（perf, flame graph）
    - 内核参数调优（sysctl）
    - cgroups & namespaces
    - eBPF
- 网络
    - IP / IPv6
    - TCP
    - HTTP
    - HTTPS / SSL / TLS（含 TLS 1.3, mTLS）
    - HTTP/2, HTTP/3（QUIC）
    - WebSocket
    - DNS
    - ARP
    - NAT
    - BGP（基础概念）
- IO 模型
- 零拷贝
- 算法
    - 时间复杂度 & 空间复杂度
    - 排序（快排, 归并, 堆排）
    - 查找（二分, 哈希, BFS/DFS）
    - 字符串（KMP, Trie）
    - 图（最短路径, 拓扑排序）
    - 分布式一致性（Paxos, Raft — 另见"分布式"）
    - 负载均衡算法（轮询, 加权, 最少连接, 一致性哈希）
- 数据结构
    - 哈希
    - 红黑树
    - B+ 树
    - LSM 树
    - Bitmap
    - 布隆过滤器
    - 字典树
    - 跳跃表
    - 图

## Java 核心

- 集合
- 泛型
- 注解 & 注解处理器
- 反射
- 代理（JDK 动态代理, CGLIB）
- 异常处理
- 序列化
- SPI
- Stream
- Lambda
- 方法句柄（MethodHandle）
- 密封类 & 记录类（Record）
- 模式匹配
- IO & NIO
- 网络通信（BIO/NIO/AIO）
- 线程
    - 线程模型
    - 线程池
    - ThreadLocal
    - 虚拟线程（Virtual Threads）
- 并发
    - 机制（synchronized, volatile, CAS）
    - 容器（ConcurrentHashMap, CopyOnWriteArrayList 等）
    - AQS
    - CompletableFuture
    - 原子类
    - VarHandle
    - Structured Concurrency
    - Scoped Values
- 正则
- SQL
- 国际化
- 编码加密
- 模块化（JPMS）

## JVM

- 内存模型（JMM）
- 内存结构（堆, 栈, 元空间, 直接内存）
- 类加载
- 字节码
- 垃圾回收器（Serial, Parallel, CMS, G1, ZGC, Shenandoah）
- 参数调优
- CDS & AOT（AppCDS, GraalVM Native Image）
- 命令行工具
    - jps, jstat, jstack, jmap, jhat, jinfo
    - top -H, vmstat, iostat, perf
    - Arthas
- GUI 工具
    - JConsole（JDK 自带，仅查看）
    - MAT（dump 分析）
    - JVisualVM（JDK 已不再自带）
    - JProfiler（付费）
- 其他工具
    - JFR（JDK 内置，记录事件）
    - JMC（分析 .jfr 文件）
    - JMX
    - JMH
- Java Agent

## 设计模式 & 编程范式

- GoF 23
    - 创建型：Factory, Abstract Factory, Builder, Prototype, Singleton
    - 结构型：Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy
    - 行为型：Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor
- DDD（领域驱动设计）
- CQRS
- Event Sourcing
- Clean Architecture / 六边形架构
- Reactive Programming（Reactor, RxJava, Reactive Streams）
- Functional Programming（Java Stream, Vavr）

## 数据库

- 关系型
    - MySQL
    - PostgreSQL
- 文档型
    - MongoDB
- KV
    - Redis
- 列式
    - ClickHouse（OLAP）
    - DuckDB（嵌入式 OLAP）
- 图
    - Neo4j
- 时序
    - TimescaleDB
    - InfluxDB
- 嵌入式
    - RocksDB
- NewSQL / 分布式
    - TiDB
- 客户端
    - Druid
    - HikariCP
    - Jedis
    - Lettuce
    - Redisson
- 工具
    - Flyway（数据库迁移）
    - Liquibase（数据库迁移）

## ORM

- MyBatis
- Hibernate
- PageHelper

## 搜索

- Elasticsearch
- OpenSearch
- Apache Lucene

## Spring 核心

- Bean（生命周期, 作用域）
- IoC & DI
- AOP
- JDBC
- 事务（声明式, 编程式, 传播行为）
- 自动装配
- 设计模式（模板, 策略, 工厂 等）
- Cache
- Async
- Scheduler
- Retry
- Validator
- Event
- Environment
- SPI
- JMX
- Logger
- Mail
- EL 表达式
- 国际化
- 可观测性

## Spring 框架 & 微服务生态

- Spring MVC
- Spring WebFlux
- Spring Boot
- Spring Data JPA
- Spring Statemachine（状态机）
- Spring Batch（批处理）
- Spring Modulith（模块化）
- Spring Cloud Alibaba
    - Nacos（配置中心 & 注册中心）
    - Seata（分布式事务管理）
    - Sentinel（限流 & 断路器）
    - Sidecar
    - Dubbo
- Spring Cloud
    - Spring Cloud Bus（消息总线 & 动态配置）
    - Spring Cloud Commons（服务注册发现）
    - Spring Cloud Load Balancer（负载均衡）
    - Spring Cloud Circuit Breaker（断路器）
    - Spring Cloud Config（配置中心）
    - Spring Cloud Consul（配置中心 & 注册中心）
    - Spring Cloud Gateway（网关）
    - Spring Cloud OpenFeign（RPC）
    - Micrometer Tracing（链路追踪）
    - Spring Cloud Stream（消息驱动框架）
    - Spring Cloud Data Flow（数据流处理框架）
    - Spring Cloud Task

## 微服务基础设施

- 配置中心
    - Apollo
    - Nacos
    - Spring Cloud Config
- 注册中心
    - Nacos
    - Consul
    - Eureka
- 网关
    - Spring Cloud Gateway
    - ShenYu
- 负载均衡
    - Nginx
    - LVS
    - HAProxy
- RPC 框架
    - Dubbo
    - gRPC
    - OpenFeign
- HTTP 服务器
    - Tomcat
    - Netty
    - Vert.x
- 通信客户端
    - Netty
    - Vert.x
    - OkHttp
    - Java HttpClient
    - AsyncHttpClient
- 消息队列
    - RocketMQ
    - Kafka
    - RabbitMQ
- 流处理
    - Flink
    - Spring Cloud Data Flow

## 安全

- 认证 & 授权
    - OAuth 2.0
    - OpenID Connect（OIDC）
    - SAML
    - JWT
    - RBAC & ABAC
- 框架
    - Spring Security
    - Spring Authorization Server
    - Spring Vault
    - Keycloak
    - Shiro
- 加解密
    - 对称加密（AES, DES）
    - 非对称加密（RSA, ECC）
    - 哈希（SHA, bcrypt, Argon2）
- 数据脱敏
- 请求幂等
- HTTPS / mTLS

## 分布式系统

- 理论
    - CAP
    - BASE
    - 最终一致性
    - ZAB
    - Raft
    - Gossip
    - 一致性哈希
- 分布式锁
- 分布式事务
    - TCC
    - Saga
    - WAL
- 分布式唯一 ID
- 分布式缓存
- 分布式限流
- 分布式容错
    - 多活
    - 优雅关机
    - 容灾补偿
    - 熔断 & 降级
- 协调服务
    - Zookeeper
    - Etcd
    - JRaft
    - Dledger
- 数据库方案
    - 读写分离
    - 分库分表（ShardingSphere, Mycat）
    - 冷热分离
    - 写合并
    - 深度分页
    - 索引优化
    - 数据同步
- 多租户

## 缓存

- 本地缓存
    - Caffeine
    - Guava Cache
    - Ehcache
- 分布式缓存
    - Redis
- 缓存模式
    - Cache-Aside
    - Read/Write Through
    - Write Behind

## 数据集成与同步

- Canal
- Debezium
- Flink

## 任务调度 & 工作流

- 任务调度
    - ElasticJob
    - XXL-Job
    - Quartz
    - ShedLock
- 工作流引擎
    - Activiti
    - Flowable

## 并发框架

- JCTools
- Disruptor
- Netty
- Vert.x

## 容器 & 云原生

- 容器
    - Docker
    - Podman
    - Docker Compose
- 编排
    - Kubernetes（K8s）
    - Helm
    - HPA / VPA
- 工具
    - k9s
    - kubectx / kubens
- Service Mesh
    - Istio
    - Linkerd
- Event Mesh
- Serverless（基础概念）
- CDN
- 部署策略
    - 滚动发布
    - 蓝绿部署
    - 金丝雀发布
- IaC
    - Terraform
    - Pulumi
    - Ansible
- GraalVM Native Image（云原生编译）

## CI/CD

- Jenkins
- GitHub Actions
- GitLab CI
- ArgoCD（GitOps）

## 可观测性

- 规范 & 采集
    - OpenTelemetry
    - Micrometer
- 指标 & 可视化
    - Prometheus + Grafana
- 链路追踪
    - SkyWalking
    - Jaeger
- 日志
    - ELK（Elasticsearch + Logstash + Kibana）
    - Loki
- 日志框架
    - SLF4J
    - Log4j2
    - Logback
- 监控
    - CAT
    - JMX
    - JFR + JMC

## 测试 & 代码质量

- 测试框架
    - JUnit 5
    - Mockito
    - Testcontainers
    - ArchUnit
- 性能测试
    - JMeter
    - JMH
- 代码质量
    - SonarQube
    - SpotBugs
    - Checkstyle / PMD

## 序列化 & 数据处理

- JSON
    - Jackson
    - Gson
    - Fastjson2
- 二进制
    - Protobuf
    - Thrift
    - Hessian
    - Avro
    - MessagePack
- 文本处理
    - Apache POI
    - EasyExcel
    - OpenCSV
    - FreeMarker
    - SnakeYAML / Jackson YAML
- 图片处理
    - ZXing
    - Thumbnails

## API & 文档

- API 设计
    - RESTful
    - GraphQL
    - gRPC-web
    - RSocket
    - API 版本管理
- 文档工具
    - SpringDoc（OpenAPI 3）
    - Spring REST Docs
- 调试工具
    - Postman / Apifox

## 开发工具 & 效率

- 构建工具
    - Maven
    - Gradle
- 版本控制
    - Git
- 环境管理
    - SDKMAN
    - asdf
- IDE
    - IntelliJ IDEA
- Java 工具库
    - Lombok
    - MapStruct
    - Guava
    - JWT
    - JGraphT
    - Resilience4j
- 脚本
    - Bash / Shell
    - Python

## AI & 新兴技术

- Spring AI
- LangChain4j
- MCP（Model Context Protocol）
- RAG
