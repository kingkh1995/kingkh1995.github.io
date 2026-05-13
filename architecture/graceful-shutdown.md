线上服务部署时直接 `kill -9` 是一种事故——正在处理的请求会丢失，已支付但未完成的订单可能悬空，连接池中的连接来不及归还。优雅停机（Graceful Shutdown）的目标是在关闭进程前，让服务体面地完成手头的工作：拒绝新请求、排空存量请求、释放资源、通知注册中心下线。

本文从 Linux 内核的信号机制出发，逐层深入 JVM、Spring、Dubbo 直到 Netty，完整展示一条 `kill -15` 命令背后发生的一切。内容侧重于底层原理和设计决策，而不是配置清单。

***

## Linux 信号基础

### 为什么是信号

操作系统通过**信号（Signal）**与进程通信。当运维执行 `kill -15 <pid>` 或容器编排平台（如 K8s）终止 Pod 时，内核会向目标进程发送 `SIGTERM` 信号。进程可以选择捕获这个信号，执行自定义的清理逻辑后再退出——这就是优雅停机的起点。

### 三个关键信号

| 信号 | 编号 | 默认行为 | 能否捕获/忽略 | 典型场景 |
|------|------|---------|-------------|---------|
| `SIGINT` | 2 | Term（终止进程） | 可以 | `Ctrl+C` 中断前台进程 |
| `SIGTERM` | 15 | Term（终止进程） | 可以 | `kill` 命令默认发送，K8s Pod 终止 |
| `SIGKILL` | 9 | Term（终止进程） | **不可以** | 强制杀死进程，最后的兜底手段 |

`SIGKILL` 无法被捕获或忽略——内核直接终止进程，不给你任何清理机会。这就是为什么 K8s 的 Pod 终止流程是：先发 `SIGTERM`，等待 `terminationGracePeriodSeconds`（默认 30 秒），超时后才发 `SIGKILL`。这段时间就是优雅停机的窗口期。

### 信号处理的三种方式

应用进程对每个信号有三种选择：

- **内核默认操作**：比如 `SIGTERM` 默认终止进程。如果不主动捕获，进程会被直接杀死。
- **捕获信号**：通过 `sigaction` 系统调用注册信号处理函数。收到信号时，内核中断当前执行流，转而执行你注册的回调——优雅停机的逻辑就写在这里。
- **忽略信号**：不做任何处理。但 `SIGKILL` 和 `SIGSTOP` 无法忽略。

### sigaction：注册信号处理函数的系统调用

```c
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);
```

- `signum`：要捕获的信号编号，比如 `SIGTERM` 即 15。
- `act`：`sigaction` 结构体，封装了信号处理函数指针 `sa_handler` 和行为控制标志 `sa_flags`。
- `oldact`：保存旧的信号处理配置，用于兼容性。

```c
struct sigaction {
    __sighandler_t sa_handler;  // 信号处理函数指针：优雅关闭的逻辑封装在这里
    unsigned long  sa_flags;    // 控制标志：SA_ONESHOT、SA_NOMASK 等
    sigset_t       sa_mask;     // 信号处理期间需要屏蔽的其他信号集
};
```

常见 `sa_flags`：

- `SA_ONESHOT`：信号处理函数仅生效一次，响应后恢复默认行为。
- `SA_NOMASK`：信号处理函数执行期间可以被其他信号中断。
- `SA_SIGINFO`：使用扩展的信号处理函数（携带更多上下文信息，如发送者 PID）。

***

## JVM 如何捕获信号

### Terminator：JVM 的信号入口

JVM 启动时，在 `System.initializeSystemClass()` 中会调用 `Terminator.setup()`，注册对三个关键信号的处理器：

```java
class Terminator {
    private static SignalHandler handler = null;

    static void setup() {
        if (handler != null) return;
        SignalHandler sh = new SignalHandler() {
            public void handle(Signal sig) {
                Shutdown.exit(sig.getNumber() + 0200);
            }
        };
        handler = sh;

        try { Signal.handle(new Signal("HUP"), sh);  } catch (IllegalArgumentException e) {}
        try { Signal.handle(new Signal("INT"), sh);  } catch (IllegalArgumentException e) {}
        try { Signal.handle(new Signal("TERM"), sh); } catch (IllegalArgumentException e) {}
    }
}
```

JVM 通过 `sun.misc.Signal.handle(Signal signal, SignalHandler handler)` 实现信号捕获，底层调用的是我们刚才讨论的 `sigaction` 系统调用。

三个被捕获的信号都执行同一个处理器：`Shutdown.exit(sig.getNumber() + 0200)`。`0200` 是八进制，即十进制的 128——JVM 用 `信号编号 + 128` 作为退出码，这是一个约定：信号编号 1~127 保留给 OS，加上 128 后形成 JVM 规范的退出码。

> JVM 只捕获 `SIGHUP`、`SIGINT`、`SIGTERM` 三种信号。如果 JVM 接收到其他信号（如 `SIGQUIT`），会执行内核默认操作直接终止进程，不会触发 ShutdownHook。

### Shutdown.exit：JVM 关闭的调度中心

```java
class Shutdown {
    static void exit(int status) {
        synchronized (Shutdown.class) {
            sequence();   // 1. 执行所有 ShutdownHook
            halt(status); // 2. 强制停止 JVM（调用 Runtime.getRuntime().halt()）
        }
    }

    private static void sequence() {
        synchronized (lock) {
            if (state != HOOKS) return;
        }
        runHooks();  // 遍历所有注册的 ShutdownHook，并发执行
        boolean rfoe;
        synchronized (lock) {
            state = FINALIZERS;
            rfoe = runFinalizersOnExit;
        }
        if (rfoe) runAllFinalizers();
    }
}
```

关键事实：

- `sequence()` 和 `halt()` 被包裹在 `synchronized (Shutdown.class)` 中——整个关闭流程是**串行化**的，防止多个信号同时触发关闭。
- `halt(status)` 调用的是 `Runtime.getRuntime().halt()`，而不是 `System.exit()`。`halt()` 不执行 ShutdownHook（ShutdownHook 已经在 `sequence()` 中执行过了），直接终止 JVM，避免死循环。
- `runFinalizersOnExit` 是一个已废弃的选项（`System.runFinalizersOnExit()`），默认 false，不会被执行。

***

## Runtime.addShutdownHook：用户侧的优雅停机入口

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("ShutdownHook running...");
    // 释放资源、关闭连接池、通知注册中心下线...
}));
```

### ShutdownHook 的执行特点

- **并发无序执行**：所有注册的 ShutdownHook 是 `Thread` 对象，由 JVM 并发启动，执行顺序无法保证。如果你的多个 Hook 之间有依赖关系，需要在 Hook 内部自行协调。
- **不能保证一定执行完**：如果 Hook 执行时间过长，JVM 可能在 Hook 完成前就被 `halt()` 强制终止。Hook 的执行没有超时保护——JVM 会一直等待所有 Hook 执行完毕，除非外部用 `kill -9` 强杀。
- **Hook 执行期间可以注册新的 Hook**：但新注册的 Hook 是否被执行取决于注册时机——如果注册时 `runHooks()` 已经遍历到末尾，新 Hook 可能被遗漏。
- **不要调用 `System.exit()`**：在 Hook 中调用 `System.exit()` 会导致 `Shutdown.exit()` 被递归调用，最终抛出 `IllegalStateException` 并死锁在 `synchronized (Shutdown.class)` 上。

### ShutdownHook 的局限性

ShutdownHook 机制本身很简单，但有几个容易踩的坑：

1. **进程被 `kill -9` 时无法执行**：`SIGKILL` 不能被捕获，ShutdownHook 不会被调用。
2. **线程池未正确关闭**：如果 Hook 中使用的线程池在 Hook 执行前已被 JVM 标记为关闭状态，提交任务会失败。
3. **日志系统可能已关闭**：JVM 关闭时日志框架的 ShutdownHook 可能先于你的 Hook 执行，导致你的清理日志无法输出。

> **最佳实践**：将 ShutdownHook 逻辑控制在秒级完成——释放连接、通知注册中心、flush 日志即可。不要在 Hook 中执行复杂的业务逻辑或依赖外部服务的长超时调用。

***

## Spring 如何利用 ShutdownHook

Spring 作为最广泛使用的 Java 框架，在 JVM ShutdownHook 之上构建了完整的容器优雅关闭机制。

### 注册 ShutdownHook

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
        implements ConfigurableApplicationContext, DisposableBean {

    @Override
    public void registerShutdownHook() {
        if (this.shutdownHook == null) {
            this.shutdownHook = new Thread() {
                @Override
                public void run() {
                    synchronized (startupShutdownMonitor) {
                        doClose();
                    }
                }
            };
            Runtime.getRuntime().addShutdownHook(this.shutdownHook);
        }
    }
}
```

在 SpringBoot 环境下，`SpringApplication` 在启动完成时会自动调用 `registerShutdownHook()`，用户无需手动注册。但在纯 Spring 环境下（非 SpringBoot），需要手动调用 `context.registerShutdownHook()`。

### Spring 容器关闭的五个回调阶段

当 JVM 触发 ShutdownHook → `doClose()` 被调用时，Spring 按照以下顺序依次执行回调：

1. **发布 `ContextClosedEvent` 事件**：容器中的 Bean 尚未开始销毁，可以在事件监听器中执行优雅关闭的前置操作（如拒绝新请求、通知网关拉出服务节点）。
2. **`DestructionAwareBeanPostProcessor.postProcessBeforeDestruction()`**：在每一个 Bean 销毁之前回调，可以执行 Bean 级别的清理。
3. **`@PreDestroy` 注解的方法**：最常用的资源释放入口，方法签名无限制，不需要实现特定接口。
4. **`DisposableBean.destroy()`**：实现该接口的 Bean 在销毁时被回调。
5. **自定义销毁方法**：通过 XML 的 `destroy-method` 属性或 `@Bean(destroyMethod = "...")` 指定的方法。

> Spring 的优雅关闭本质上是对 JVM ShutdownHook 的封装和扩展。它并没有改变 ShutdownHook 的并发无序特点——Spring 通过 `doClose()` 内部的同步控制，将原本并发的 Hook 执行转化为有序的容器销毁流程。

***

## Dubbo：在 Spring 基础上叠加服务治理

Dubbo 的优雅关闭在 Spring 容器关闭的基础上，增加了服务治理层面的操作。

### 关闭顺序

当 Spring `ContextClosedEvent` 被发布后，Dubbo 的 `DubboShutdownHookListener` 会触发以下流程：

```java
public void destroy() {
    // 1. 取消注册中心中本服务的注册（从服务列表中摘除）
    unexportServices();
    // 2. 取消对依赖服务的订阅
    unreferServices();
    // 3. 销毁注册中心客户端实例
    destroyRegistries();
    // 4. 关闭 Dubbo 协议（底层触发 Netty 关闭）
    DubboShutdownHook.destroyProtocols();
    // 5. 销毁服务发现客户端实例
    destroyServiceDiscoveries();
    // 6. 清理应用配置模型
    clear();
    // 7. 关闭线程池
    shutdown();
    // 8. 释放资源
    release();
}
```

关键点：`unexportServices()` 排在第一位——先通知注册中心下线，让流量不再路由到本节点，然后再处理本地残留请求和资源释放。这确保了**停止接收新请求 → 排空存量请求 → 释放资源**的正确顺序。

### SpringBoot 与纯 Spring 的兼容

Dubbo 通过 `SpringExtensionFactory` 自动注册 Spring 的 ShutdownHook，确保在 SpringBoot 和纯 Spring 环境下都能正常工作：

```java
public class SpringExtensionFactory implements ExtensionFactory {
    private static final Set<ApplicationContext> CONTEXTS = new ConcurrentHashSet<>();

    public static void addApplicationContext(ApplicationContext context) {
        CONTEXTS.add(context);
        if (context instanceof ConfigurableApplicationContext) {
            ((ConfigurableApplicationContext) context).registerShutdownHook();
        }
    }
}
```

`ServiceBean`（服务提供者）和 `ReferenceBean`（服务消费者）在初始化完成后会回调 `addApplicationContext`，保证无论在什么 Spring 环境下，ShutdownHook 都被正确注册。

***

## Netty：事件循环的优雅谢幕

当 Dubbo 调用 `server.close()` 关闭底层通信框架时，最终触发的是 Netty 的优雅关闭流程。这是整个优雅停机链路的最底层——所有网络 I/O 的清理工作都在这里完成。

### shutdownGracefully：两个核心参数

```java
EventLoopGroup group = new NioEventLoopGroup();
// 发起优雅关闭
group.shutdownGracefully(quietPeriod, timeout, unit);
```

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `quietPeriod` | 2 秒 | 静默期。Reactor 空闲后，如果在静默期内没有新任务提交，才正式关闭。每隔 100ms 检查一次。 |
| `timeout` | 15 秒 | 超时时间。优雅关闭的总时长不能超过该值，无论是否还有残留任务，超时后强制关闭。 |

### Reactor 的状态机流转

```
ST_NOT_STARTED → ST_STARTED → ST_SHUTTING_DOWN → ST_SHUTDOWN → ST_TERMINATED
```

| 状态 | 含义 | 能否提交新任务 |
|------|------|-------------|
| `ST_NOT_STARTED` | Reactor 创建但未启动 | 否 |
| `ST_STARTED` | 正常运行 | 是 |
| `ST_SHUTTING_DOWN` | 正在优雅关闭，仍在排空队列 | 是 |
| `ST_SHUTDOWN` | 已完成优雅关闭，不再接受新任务 | 否，提交会抛出 `RejectedExecutionException` |
| `ST_TERMINATED` | Reactor 线程已完全终止 | 否 |

### 关闭的核心防线：confirmShutdown()

Reactor 的工作循环在即将退出前，会调用 `confirmShutdown()` 做最后检查：

1. **执行 Netty 内部的 ShutdownHook**（注意：这是 Netty 自己的 ShutdownHook，是 `Runnable`，不是 JVM 的 `Thread` 类型 ShutdownHook。Netty 的 ShutdownHook 由 Reactor 线程**同步有序**执行）。
2. **处理遗留的异步任务队列**：`confirmShutdown()` 中会持续执行 `runAllTasks()`，直到任务队列为空。
3. **触发静默期**：队列清空后，等待 `quietPeriod`（默认 2 秒）。期间每隔 100ms 检查一次是否有新任务提交。
4. **超时兜底**：如果总耗时超过 `timeout`（默认 15 秒），无论是否还有任务，直接关闭。

```java
// Netty Reactor 工作循环的简化逻辑
protected void run() {
    for (;;) {
        try {
            // 1. 监听 Channel 上的 I/O 事件
            // 2. 处理 Channel 上的 I/O 事件
            // 3. 执行异步任务
        } finally {
            if (isShuttingDown()) {
                closeAll();               // 关闭所有注册的 Channel
                if (confirmShutdown()) {  // 确认可以安全退出
                    return;               // 退出工作循环
                }
            }
        }
    }
}
```

> **静默期的价值**：保证了"在最后 2 秒内确实没有再需要处理的工作了"，而不是"刚好队列在上一次轮询时空了"。这是防止提前关闭的关键设计。

### 完整链路回顾

从一条 `kill -15 <pid>` 到 Netty 完成关闭，完整调用链是：

```
kill -15 (SIGTERM)
  → Linux 内核中断进程，查找信号处理函数
  → sigaction() 调用注册的回调
  → JVM Signal.handle() → Shutdown.exit(143)
  → Shutdown.sequence() → runHooks()
  → Runtime.addShutdownHook 注册的所有 Thread 并发执行
    → Spring ShutdownHook → doClose()
      → ContextClosedEvent → Dubbo ShutdownHookListener
        → unexportServices()（取消注册中心注册）
        → destroyProtocols() → DubboProtocol.destroy()
          → server.close() → NettyServer.doClose()
            → bossGroup.shutdownGracefully()
            → workerGroup.shutdownGracefully()
              → Reactor.confirmShutdown()
                → closeAll() → runAllTasks() → 静默期检测 → 退出
```

整个过程体现了分层设计的思想：**Linux 内核负责信号传递，JVM 负责信号处理与 Hook 调度，Spring 负责容器生命周期管理，Dubbo 负责服务治理层面的摘除，Netty 负责网络 I/O 的安全关闭**。每一层只做自己该做的事，通过标准接口（系统调用、JVM API、Spring 事件、Dubbo SPI、Netty 方法调用）串联起来。

***

## 生产实践建议

### 必须避免的做法

- **直接 `kill -9`**：跳过所有优雅停机逻辑。仅在 `SIGTERM` 超时后作为最后兜底。
- **在 ShutdownHook 中调用 `System.exit()`**：会导致死锁。
- **在 ShutdownHook 中进行长耗时操作**：Hook 没有超时机制，会无限期阻塞 JVM 关闭。
- **依赖 Hook 的执行顺序**：JVM ShutdownHook 是并发无序的。如果需要顺序保证，在单个 Hook 内部串行执行。

### 推荐做法

- **优先使用框架的优雅停机机制**：SpringBoot 的 `server.shutdown=graceful`、Dubbo 的自动注册摘除，不要重复造轮子。
- **设置合理的超时**：K8s `terminationGracePeriodSeconds`、Netty `quietPeriod`/`timeout` 要配合设置。SpringBoot 的 `spring.lifecycle.timeout-per-shutdown-phase` 也要纳入考虑。
- **在 Hook 中只做轻量操作**：flush 日志、关闭连接池、通知注册中心。不要依赖外部 RPC 调用（被调方可能也已下线）。
- **日志记录 Hook 执行**：在 Hook 开头和结尾各打一行日志，方便排查"进程为什么关了很久"的问题。
