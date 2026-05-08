## 计算机基础

- 操作系统
- Linux
- 网络
    - IP/IPv6
    - TCP
    - HTTP
    - HTTPS
    - SSL/TLS
    - WebSocket
    - DNS
    - ARP
    - NAT
- IO模型
- 零拷贝
- 数据结构
    - 哈希
    - 红黑树
    - B+树
    - LSM树
    - Bitmap
    - 布隆过滤器
    - 字典树
    - 跳跃表
    - 图

## 抓包工具
- tcpdump
- Wireshark

## Java

- 集合
- 泛型
- 注解
- 反射
- 代理
- SPI
- Stream
- Lambda
- IO
- NIO
- 网络通信
- 线程
    - 线程模型
    - 线程池
    - ThreadLocal
    - 虚拟线程
- 并发
    - 机制
    - 容器
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
- 模块化

## JVM

- 内存模型
- 内存结构
- 类加载
- 字节码
- 垃圾回收器
- 参数调优
- 命令行工具
    - jps, jstat, jstack, jmap, jhat, jinfo
    - top -H, vmstat, iostat, perf
    - Arthas
- GUI工具
    - JConsole （JDK自带，仅查看）
    - MAT （dump分析）
    - JVisualVM （JDK已不再自带）
    - JProfiler （付费）
- 其他工具
    - JFR （JDK内置，记录事件）
    - JMC （分析.jfr文件）
    - JMX
    - JMH
    - JMeter
- Java Agent
- GraalVM
    - Native Image

## 数据库

- MySQL
- PostgreSQL
- Redis
- TiDB
- MongoDB
- RocksDB
- ClickHouse OLAP

## 数据库客户端

- Druid
- HikariCP
- Jedis
- Lettuce
- Redisson

## 数据库工具

- Flyway 数据库迁移
- Liquibase 数据库迁移

## ORM

- MyBatis
- Hibernate
- PageHelper

## 搜索

- Elasticsearch
- OpenSearch
- Apache Lucene

## Spring

- Bean
- IoC
- AOP
- JDBC
- 事务
- 自动装配
- 设计模式
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
- EL表达式
- 国际化
- 可观测性

## Spring框架

- Spring MVC
- Spring WebFlux
- Spring Boot
- Spring Data JPA
- Spring Statemachine 状态机
- Spring Batch 批处理
- Spring Modulith 模块化
- Spring AI

## Spring Cloud Alibaba

- Nacos 配置中心 & 注册中心
- Seata 分布式事务管理
- Sentinel 限流 & 断路器
- Spring Cloud Alibaba Sidecar
- Dubbo

## Spring Cloud

- Spring Cloud Bus 消息总线 & 动态配置
- Spring Cloud Commons 服务注册发现
- Spring Cloud Load Balancer 负载均衡
- Spring Cloud Circuit Breaker 断路器
- Spring Cloud Config 配置中心
- Spring Cloud Consul 配置中心 & 注册中心
- Spring Cloud Gateway 网关
- Spring Cloud OpenFeign RPC
- Micrometer Tracing 链路追踪
- Spring Cloud Stream 消息驱动框架

## 流处理 & 事件驱动

- Spring Cloud Stream 消息驱动框架
- Spring Cloud Stream Applications
- Spring Cloud Data Flow 数据流处理框架
- Spring Cloud Task

## 安全框架

- Spring Security
- Spring Authorization Server
- Spring Vault
- Keycloak
- Shiro

## 开发工具

- Maven
- Gradle
- Git

## 容器 & 编排

- Docker
- Podman
- Docker Compose
- Kubernetes (K8s)

## CI/CD

- Jenkins
- GitHub Actions
- GitLab CI

## HTTP服务器

- Tomcat
- Netty
- Vert.x

## MQ

- RocketMQ
- Kafka
- RabbitMQ

## 监控 & 链路追踪 & 可观测

- OpenTelemetry
- Micrometer
- Prometheus + Grafana
- SkyWalking
- Jaeger
- CAT
- ELK
- Loki

## 配置中心

- Apollo
- Nacos
- Spring Cloud Config

## 注册中心

- Nacos
- Spring Cloud Consul
- Eureka

## 网关

- Spring Cloud Gateway
- ShenYu

## 分布式协调

- Zookeeper
- Etcd
- JRaft
- Dledger

## RPC框架

- Dubbo
- gRPC
- OpenFeign

## 任务调度

- ElasticJob
- XXL-Job
- Quartz

## 分库分表

- ShardingSphere
- Mycat

## 通信客户端

- Netty
- Vert.x
- OkHttp
- Java HttpClient
- AsyncHttpClient

## 缓存框架

- Caffeine
- Guava Cache
- Ehcache

## 数据同步与集成

- Canal
- Flink
- Debezium

## 工作流引擎

- Activiti
- Flowable

## 并发框架

- JCTools
- Disruptor
- Netty
- Vert.x

## 测试框架

- JUnit 5
- Mockito
- Testcontainers
- JMeter

## 分布式

- 分布式系统
    - CAP
    - BASE
    - 最终一致性
    - ZAB
    - Raft
    - Gossip
    - 一致性哈希算法
- 分布式锁
- 分布式事务
    - TCC
    - Saga
    - WAL
- 分布式唯一ID
- 分布式缓存
- 分布式容错
    - 多活
    - 优雅关机
    - 容灾补偿
- 分布式限流

## 数据库方案与优化

- 读写分离
- 分库分表
- 冷热分离
- 写合并
- 深度分页
- 索引优化
- 数据同步

## 安全

- 加解密算法
- 数据脱敏
- 请求幂等

## Java工具库

- Lombok
- MapStruct
- Guava
- JWT
- JGraphT
- JavaMail
- Resilience4j

## 日志框架

- SLF4J
- Log4j2
- Logback

## 图片处理

- ZXing
- Thumbnails

## 文本处理

- Apache POI
- EasyExcel
- OpenCSV
- FreeMarker
- Jackson Dataformat YAML
- SnakeYAML

## JSON序列化

- Jackson
- Gson
- Fastjson2

## 二进制序列化

- Protobuf
- Thrift
- Hessian
- Avro

## REST文档

- SpringDoc
- Spring REST Docs

## 负载均衡

- CDN
- Nginx
- LVS
- HAProxy

## 微服务框架

- Guice 依赖注入
- Quarkus 云原生
- Micronaut 云原生
- COLA DDD

## 微服务治理

- ServiceMesh
    - Istio
    - Linkerd
- EventMesh

## 架构模式

- DDD
- CQRS
- Event Sourcing

## 存储

- MinIO 对象存储

## AI

- Spring AI
- LangChain4j
