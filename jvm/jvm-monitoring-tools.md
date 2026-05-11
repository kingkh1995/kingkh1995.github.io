## JVM 监控与故障诊断工具

### 线上诊断的思路框架

线上环境出问题时，首先要回答三个问题：**CPU 还是内存？GC 还是线程？是泄漏还是参数不当？** 诊断流程可以概括为：

1. **确认进程：** `jps` 或 `ps` 找到目标 JVM 进程 ID。
2. **快速扫描：** `jstat -gcutil` 看 GC 趋势，`top -H` 看线程 CPU 分布。
3. **定向深挖：** 根据初步判断选择 `jstack`（线程）、`jmap`（内存）或 `jcmd`（综合）。
4. **必要时生成转储：** Heap Dump、Thread Dump，配合 MAT 或 JMC 离线分析。

> **核心原则：** 线上优先用低开销的命令行工具，避免在生产环境直接挂载重型分析器。JFR（JDK Flight Recorder）是目前唯一可以在生产环境持续运行的内置方案。

***

### 命令行工具：生产环境的第一响应

JDK 自带的命令行工具是线上排查的基石。以下只保留生产环境最常用、最高频的参数组合，不展开完整手册。

#### jps — 快速定位 JVM 进程

```
jps -lvm
```

| 参数 | 作用 |
|:--|:--|
| `-l` | 输出主类全名或 JAR 路径 |
| `-v` | 输出 JVM 启动参数 |
| `-m` | 输出 main() 传入的参数 |

`jps` 本质上是读取 `/tmp/hsperfdata_<user>` 下的性能计数器文件，比 `ps` 更精准地过滤出 Java 进程。如果该目录被清理或权限变更，`jps` 会失效，此时回退到 `ps -ef | grep java`。

> 在容器环境中，`jps` 只能看到同一容器内的进程。如果宿主机上运行了多个容器，需要在目标容器内部执行 `jps`。

#### jstat — GC 与运行时状态监控

`jstat` 是线上最频繁使用的工具，**不需要暂停应用**，开销极低。

```
## 各区域使用率 + GC 次数/耗时，每秒采样，共 5 次
jstat -gcutil <pid> 1s 5

## 容量详情（KB），用于判断代际空间是否充足
jstat -gc <pid> 1s 3

## 类加载情况，排查 Metaspace 增长
jstat -class <pid> 1s 5

## GC 原因（G1/CMS 下非常有用）
jstat -gccause <pid> 1s 5
```

**`-gcutil` 输出关键列：**

| 列 | 含义 | 诊断价值 |
|:--|:--|:--|
| `S0`、`S1` | Survivor 0/1 使用率 | 观察对象在新生代的存活周期 |
| `E` | Eden 使用率 | 判断 YGC 频率是否合理 |
| `O` | Old 使用率 | **持续上升 → 疑似泄漏或晋升过快** |
| `M` | Metaspace 使用率 | 动态类加载场景的关注点 |
| `YGC` / `YGCT` | Young GC 次数 / 总耗时 | 计算单次 YGC 平均耗时 |
| `FGC` / `FGCT` | Full GC 次数 / 总耗时 | **频繁增长 → 必须深入** |
| `GCT` | GC 总耗时 | 评估 GC 对吞吐的影响 |

**`-gccause` 比 `-gcutil` 多一列 `LGCC` 和 `GCC`：**

| 列 | 含义 |
|:--|:--|
| `LGCC` | 上一次 GC 原因 |
| `GCC` | 当前 GC 原因 |

常见原因包括 `Allocation Failure`（Eden 满）、`G1 Evacuation Pause`（G1 转移暂停）、`Metadata GC Threshold`（Metaspace 阈值）、`System.gc()`（显式调用）。

> **怎么判断 Full GC 是否频繁？** 观察 `FGC` 列是否在持续增加。如果 `O`（老年代）在 Full GC 后没有明显下降，说明要么存在内存泄漏，要么老年代空间本身不足。

#### jinfo — 查看与动态修改 JVM 参数

```
## 查看所有参数（含默认值）
jinfo -flags <pid>

## 查看特定参数当前值
jinfo -flag MaxHeapSize <pid>

## 动态开启/关闭部分参数（仅限 Manageable 参数）
jinfo -flag +PrintGCDetails <pid>
jinfo -flag -PrintGCDetails <pid>
```

`jinfo` 的核心价值在于**查看启动后 JVM 实际生效的参数值**，尤其是那些被 JVM 动态调整的值（如 G1 的自适应 IHOP）。注意：只有标记为 `manageable` 的参数才能运行时修改，可用 `java -XX:+PrintFlagsFinal | grep manageable` 查看列表。

> 一个常见场景是：应用启动时设置了 `-Xmx4g`，但容器内存限制为 3g。`jinfo -flag MaxHeapSize` 可以直接看到 JVM 实际读取到的值，确认是否被容器感知的 cgroup 限制所影响。

#### jmap — 内存快照与直方图

```
## 堆中对象的统计直方图（按类聚合）
jmap -histo:live <pid> | head -n 20

## 生成堆转储文件
jmap -dump:format=b,file=heap.hprof <pid>

## 查看类加载器统计（排查类泄漏）
jmap -clstats <pid>
```

| 参数 | 场景 | 注意 |
|:--|:--|:--|
| `-histo:live` | 快速定位大对象或异常对象数量 | 会触发一次 Full GC，只统计存活对象 |
| `-dump` | 深度分析内存泄漏 | **生成期间会 STW**，大堆可能耗时数十秒 |
| `-clstats` | 排查 Metaspace 溢出 | 显示每个类加载器加载的类数量 |

**`-histo:live` 输出示例：**

```
 num     #instances         #bytes  class name
----------------------------------------------
   1:        123456       98764800  [B
   2:         54321       43456800  java.lang.String
   3:         12345       34567890  com.example.UserSession
```

如果某个业务类（如 `UserSession`）的实例数远超预期，且 `bytes` 占比很大，这就是重点怀疑对象。

> **生产环境慎用 `jmap -dump`：** 对数 GB 的堆执行 dump 会造成明显停顿。推荐在 JVM 启动参数中添加 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dumps`，让 JVM 在 OOM 时自动 dump，避免人工触发。

#### jstack — 线程状态与死锁分析

```
## 打印所有线程的栈轨迹
jstack -l <pid> > thread_dump.txt

## 只打印锁的附加信息（锁地址、拥有者）
jstack -l <pid>
```

`-l` 参数会输出额外的锁信息，包括：

- **持有的锁：** `locked <0x...>`
- **等待的锁：** `waiting to lock <0x...>`
- **死锁检测：** 如果存在死锁，`jstack` 会在输出末尾自动列出**死锁链**。

**线程状态速查：**

| 状态 | 含义 | 排查方向 |
|:--|:--|:--|
| `RUNNABLE` | 正在运行或等待 CPU | CPU 飙高时重点看 |
| `BLOCKED` | 等待监视器锁（synchronized）| 锁竞争、死锁 |
| `WAITING` | 无限期等待（`Object.wait()` 等）| 等待条件不满足 |
| `TIMED_WAITING` | 限期等待（`sleep`、`join(timeout)`）| 正常或超时逻辑问题 |
| `NEW` / `TERMINATED` | 新建 / 已终止 | 通常无异常 |

**`jstack` 输出中的线程名很重要。** 现代框架（Spring、Netty、Tomcat）通常会给线程起有意义的名字，例如 `http-nio-8080-exec-5` 或 `lettuce-eventExecutorLoop-1-2`。通过线程名可以快速定位到具体的线程池或组件。

#### jcmd — 一体化诊断命令

JDK 7+ 引入的综合性工具，可以替代多数上述工具的功能。

```
## 列出该用户的所有 Java 进程
jcmd -l

## 打印 GC 概况
jcmd <pid> GC.heap_info

## 生成线程 dump
jcmd <pid> Thread.print

## 生成堆 dump
jcmd <pid> GC.heap_dump /data/dumps/heap.hprof

## 打印 VM 参数
jcmd <pid> VM.flags

## 打印类加载器统计
jcmd <pid> VM.classloader_stats

## 执行最终化（Finalizer）分析
jcmd <pid> GC.finalizer_info
```

`jcmd` 的优势在于**单一入口、统一输出格式**，尤其适合脚本化巡检。对于不熟悉工具链的新成员，`jcmd` 比记忆多个命令更容易上手。

> `jcmd <pid> VM.native_memory summary` 可以查看 NMT（Native Memory Tracking）数据，排查堆外内存泄漏（Direct Buffer、JNI 分配等）。需要在启动参数中开启 `-XX:NativeMemoryTracking=summary`。

***

### GUI 工具：从临时查看到持续记录

#### JConsole — JDK 自带的简易监控

JConsole 是 JDK 自带的 JMX 客户端，可以查看堆内存、线程、类加载、CPU 使用率的实时曲线。

**适用场景：** 本地开发或测试环境快速验证内存/线程趋势。

**局限性：**

- 无法分析历史数据，只能看实时。
- 线程分析功能极弱，不支持死锁自动检测。
- 堆直方图功能缺失。

> 生产环境几乎不用 JConsole，但它的 MBean 面板在查看自定义 JMX 指标时仍有价值。

#### VisualVM — 功能全面但已不再内置

VisualVM 在 JDK 6~8 内置，JDK 9 起移除，需单独下载。

**核心能力：**

- CPU 和内存采样（Sampler）
- Heap Dump 查看（集成 Visual GC 插件）
- 线程状态时间线

**为什么不推荐生产环境使用 VisualVM？**

1. **需要 JMX 连接或 attach 到目标进程**，配置复杂且有安全风险。
2. **CPU/Memory Sampler 是采样型分析器**，虽然开销低于 instrument 型，但对高吞吐服务仍有 5–10% 的性能损耗。
3. **无法持续记录**，只能实时观察，错过的问题无法回溯。

#### JMC（JDK Mission Control）与 JFR — 生产环境的最佳实践

**JFR（JDK Flight Recorder）** 是 JDK 内置的**低开销事件记录框架**，**JDK 11+ 开源**。它在 JVM 内部以循环缓冲的方式记录事件，默认对应用性能的影响**低于 1%**。

**JMC** 是 JFR 数据的可视化分析工具，可以离线分析 `.jfr` 文件。

**JFR 的核心优势：**

| 维度 | JFR | VisualVM / JProfiler |
|:--|:--|:--|
| 性能开销 | < 1% | 5–20% |
| 持续记录 | ✅ 循环缓冲，默认保留最近 1 小时 | ❌ 只能实时 |
| 事后分析 | ✅ 导出 `.jfr` 离线分析 | ❌ |
| 生产环境 | ✅ 推荐 | ⚠️ 谨慎使用 |
| GC 详情 | 完整事件（阶段、暂停、原因）| 有限 |
| 锁竞争 | 监视器等待、线程公园（park）事件 | 基本无 |

**启用 JFR 的方式：**

```
## 启动参数开启持续记录（默认写入循环缓冲）
-XX:+UnlockDiagnosticVMOptions
-XX:+DebugNonSafepoints
-XX:StartFlightRecording=disk=true,dumponexit=true,filename=/data/dumps/app.jfr,maxage=1h

## 运行时手动触发（JDK 11+）
jcmd <pid> JFR.start name=MyRecording duration=60s filename=/data/dumps/recording.jfr
```

**JMC 分析的关键视图：**

- **GC 详情：** 精确到每个 GC 事件的暂停时间、阶段耗时、GC 原因。G1 的 `Evacuation Pause` 可以细分为 `Ext Root Scanning`、`Update RS`、`Scan RS`、`Code Root Scanning`、`Object Copy` 等子阶段，定位哪一步拖长了暂停。
- **内存：** 对象分配速率、按类分配直方图（无需 Heap Dump）。**分配热点**视图可以直接看出哪些方法在疯狂分配对象。
- **线程：** 线程状态分布、锁竞争热点、热点方法。
- **I/O：** Socket 和 File I/O 事件，定位阻塞调用。
- **方法分析：** 基于 JFR 采样的热点方法，配合调用栈可以定位 CPU 瓶颈。

> **实战建议：** 在生产环境统一开启 JFR 循环记录（maxage=1h 或 6h）。当告警触发时，立即 `jcmd <pid> JFR.dump` 导出最近一段时间的 `.jfr` 文件，配合 JMC 离线定位根因。这比临时 attach 分析器可靠得多。

#### JFR Cooperative Sampling（JEP 518，JDK 25）

JFR 之前的异步栈采样需要在安全点（safepoint）外挂起目标线程来获取调用栈，但栈元数据只有在安全点才保证有效，因此这种方式可能导致崩溃或死锁等稳定性问题。JEP 518 引入了 Cooperative Sampling，**仅在安全点处遍历调用栈**，同时通过优化采样策略尽量降低安全点偏差（safepoint bias）。这一改进显著提升了 JFR 采样的稳定性，使其在生产环境中更加可靠。该行为变更对用户完全透明，**无需新增任何命令行参数**。

#### JFR CPU-Time Profiling（JEP 509，JDK 25，实验性）

JDK 25 引入了基于 CPU 时间（CPU-time）而非墙钟时间（wall-clock time）的 JFR 事件类型。CPU-time profiling 能够帮助开发者区分方法是真正消耗了 CPU 周期，还是只是在等待 I/O 完成。这对于定位 CPU 密集型瓶颈和识别被 I/O 阻塞的慢方法尤为有用。可以通过 `-XX:StartFlightRecording` 启动参数或 `jcmd` 的 profiling 配置来开启。需要注意的是，该特性在 JDK 25 中仍为**实验性（experimental）**，未来版本可能会转为正式产品特性。

#### JFR Method Timing & Tracing（JEP 520，JDK 25）

JEP 520 新增了方法级别的 JFR 事件，包括方法进入（method entry）、方法退出（method exit）以及精确的执行时长。借助这些事件，开发者可以在不依赖采样的情况下精确定位热点方法。由于需要为每个方法调用生成事件，其开销明显高于采样方式，因此更适合在针对性的 profiling 会话中使用，而不建议作为持续生产录制的默认配置。将这些事件与现有的 JFR 事件结合使用，可以获得更全面的性能分析视角。

从 JDK 25 开始，JFR 在采样稳定性、CPU 时间分析和方法级追踪三个方面都得到了显著增强，进一步巩固了其作为生产环境首选诊断工具的地位。

***

### 实战诊断案例

#### CPU 飙高：`top -H` → `jstack` 定位热点线程

**现象：** 监控告警某台机器 CPU 使用率飙升至 90%+。

**排查流程：**

1. `top -H -p <pid>` 找出占用 CPU 最高的线程，记录其 **十进制线程 ID**（如 `1523`）。
2. 将线程 ID 转为十六进制：`printf "%x\n" 1523` → `5f3`。
3. `jstack -l <pid> | grep -A 20 "nid=0x5f3"` 定位到具体线程栈。
4. 分析线程状态：
   - 如果是 `RUNNABLE`，查看栈顶方法，通常是无穷循环、正则回溯或密集计算。
   - 如果是 `BLOCKED`，记录它等待的锁地址，在 dump 中搜索该锁的持有者。

**怎么看出问题？** `top -H` 中某个线程的 `%CPU` 长期占单个核的 100%。

**怎么确认根因？** `jstack` 中该线程处于 `RUNNABLE`，栈顶停留在业务代码的某个循环或计算逻辑中。结合代码审查确认是否为预期行为。

> **注意：** 如果是 `GC 线程`（如 `G1 Conc#0`）CPU 高，说明 GC 压力大，应转向 GC 分析而非业务代码。`jstat -gcutil` 可以快速确认这一点。

#### 内存泄漏：`jmap -histo:live` → Heap Dump → MAT 分析

**现象：** 老年代使用率（`jstat -gcutil` 中的 `O` 列）持续上升，Full GC 后下降幅度很小，最终触发 OOM。

**排查流程：**

1. `jstat -gcutil <pid> 5s` 观察 `O` 列趋势，确认持续增长。
2. `jmap -histo:live <pid> | head -n 20` 快速查看哪些类的实例数异常偏多。
3. 生成 Heap Dump：`jcmd <pid> GC.heap_dump /data/dumps/heap.hprof`（或等 OOM 自动 dump）。
4. 用 **MAT（Eclipse Memory Analyzer）** 打开：
   - 运行 **Leak Suspects Report**，自动列出最可疑的泄漏点。
   - 对怀疑对象使用 **Path to GC Roots**，排除软弱引用，找到阻止对象被回收的引用链。
   - 使用 **Dominator Tree** 查看对象的保留集（Retained Heap），定位真正占用内存的大对象。

**怎么看出问题？** `jmap -histo:live` 中某非基础类的实例数远超预期，或 MAT 的 Dominator Tree 中某个业务对象的 Retained Heap 占堆的 30% 以上。

**怎么确认根因？** Path to GC Roots 指向一个静态 Map、缓存或监听器列表，该集合本应清理条目却未清理。修复方向通常是添加过期策略、改用 WeakReference 或在生命周期结束时手动注销。

> **MAT 的支配树 vs 直方图：** 直方图只看单个对象的浅堆（Shallow Heap），而支配树计算的是保留堆（Retained Heap），即对象被回收后能释放的总内存。泄漏分析应以 Retained Heap 为准。

#### 死锁：`jstack -l` 检测死锁链

**现象：** 部分请求超时，线程池耗尽，但 CPU 使用率不高。

**排查流程：**

1. `jstack -l <pid> > thread_dump.txt`
2. 拉到文件末尾，查看 **"Found one Java-level deadlock"** 段落。
3. 死锁信息会明确列出：
   - 死锁涉及的线程名
   - 每个线程持有的锁（`locked`）
   - 每个线程等待的锁（`waiting to lock`）
   - 锁对应的 Java 对象地址

**输出示例：**

```
Found one Java-level deadlock:
=============================
"Thread-A":
  waiting to lock monitor 0x00007f1234 (object 0x1234, a java.lang.Object),
  which is held by "Thread-B"
"Thread-B":
  waiting to lock monitor 0x00007f5678 (object 0x5678, a java.lang.Object),
  which is held by "Thread-A"
```

**怎么看出问题？** `jstack` 直接输出了死锁链，无需额外分析。

**怎么确认根因？** 查看两个线程栈中 `locked` 和 `waiting to lock` 对应的代码位置。死锁通常由以下模式导致：

- **嵌套锁顺序不一致：** 线程 A 先锁 X 再锁 Y，线程 B 先锁 Y 再锁 X。
- **回调中的隐式锁：** 在 `synchronized` 方法内调用外部接口，接口回调又尝试获取同一把锁。

**修复方向：** 统一全局加锁顺序、使用 `tryLock()` 带超时、或将嵌套锁改为单锁或并发容器（`ConcurrentHashMap` 替代 `Collections.synchronizedMap`）。

#### 频繁 Full GC：`jstat -gcutil` 分析老年代趋势

**现象：** 应用响应时间抖动，`jstat` 中 `FGC` 列增长很快。

**排查流程：**

1. `jstat -gcutil <pid> 5s 20` 持续观察，记录 `O`、`FGC`、`FGCT` 的变化。
2. 区分两种模式：
   - **模式 A：** `O` 缓慢增长 → Full GC → `O` 下降不多 → 继续增长。这是**内存泄漏**特征，按上一节处理。
   - **模式 B：** `O` 在较低水平（如 40%）就频繁触发 Full GC。这是**GC 参数或晋升策略不当**。
3. 模式 B 的进一步判断：
   - 查看 `E`（Eden）：如果 Eden 很快填满，说明新生代太小，对象过早晋升到老年代。
   - 查看 `S0` / `S1`：如果 Survivor 使用率一直很高，说明存活对象多，晋升阈值或 Survivor 区不足。
   - `jstat -gccause` 查看最近一次 GC 原因（如 `G1 Evacuation Pause`、`Allocation Failure`、`Metadata GC Threshold`）。

**怎么确认根因？**

- 如果是 **G1** 且 GC 原因是 `G1 Evacuation Pause` 伴随老年代高占用：调整 `-XX:InitiatingHeapOccupancyPercent` 或检查是否有大对象直接进 Humongous Region。
- 如果是 **CMS**：检查 `-XX:CMSInitiatingOccupancyFraction` 是否设置过低，导致并发周期过于频繁。
- 如果是 **Metaspace 原因**：`M` 列接近 100% 且 GC 原因是 `Metadata GC Threshold`，按下一节处理。

> **一个常见误区：** 看到 Full GC 就增加堆内存。如果根本原因是对象晋升过快（Eden/Survivor 设置不当），单纯增大堆只会让 GC 间隔变长，但无法减少 GC 次数。应先通过 `jstat` 确认代际比例是否合理。

#### Metaspace 溢出：`jstat -class` + `jmap -clstats`

**现象：** `java.lang.OutOfMemoryError: Metaspace`，或 `jstat` 中 `M` 列持续接近 100%。

**排查流程：**

1. `jstat -class <pid> 5s` 观察 `Loaded`（已加载类数）是否持续增长。
2. `jmap -clstats <pid>` 查看每个类加载器加载的类数量：
   - 如果某个自定义类加载器（如 `AppClassLoader` 的子类、Tomcat 的 `WebappClassLoader`）的类数量异常高且只增不减，说明存在**类泄漏**。
3. 常见根因：
   - **动态代理/字节码生成未清理：** CGLIB、ByteBuddy、Groovy 脚本引擎反复生成新类。
   - **OSGi / 热部署场景：** 旧类加载器未被回收，导致其加载的所有类都留在 Metaspace。
   - **反射缓存膨胀：** `Method` / `Constructor` 对象持有类引用，间接阻止类卸载。

**怎么确认根因？** `jmap -clstats` 中某个类加载器的类数量远超预期，且在对应组件卸载后没有下降。结合代码确认该类加载器是否被静态集合、线程局部变量或 JNI 全局引用持有。

**修复方向：**

- 限制动态生成类的缓存大小，使用 LRU 淘汰。
- 热部署场景确保旧类加载器的所有实例、线程、资源都被清理。
- 适当提高 `-XX:MaxMetaspaceSize`（治标不治本，类泄漏严重时仍会溢出）。

> **Metaspace 与 PermGen 的区别：** JDK 8 之前 PermGen 是固定大小的 JVM 内存区域，OOM 时错误信息是 `OutOfMemoryError: PermGen space`。JDK 8 起 Metaspace 使用本地内存，默认无上限，但类泄漏同样会导致本地内存耗尽。设置 `-XX:MaxMetaspaceSize` 可以限制上限，同时暴露类泄漏问题。

***

### JDK 版本演进中的工具变化

#### JDK 14 ~ 17：JFR 开源与事件流

| 特性 | 版本 | 说明 |
|:--|:--|:--|
| JFR 开源 | JDK 11（正式）| 此前是 Oracle JDK 商业特性 |
| JFR 事件流（JEP 349）| JDK 14 | 支持通过 `RecordingStream` API **实时消费 JFR 事件**，无需等待 recording 结束 |
| JVMCI 诊断增强 | JDK 15+ | Graal 编译器的诊断事件纳入 JFR，可分析 JIT 编译瓶颈 |
| JMC 独立发布 | JDK 11+ | 不再绑定 JDK 发行版，从官网独立下载 |

**JFR 事件流（JEP 349）实战意义：**

以前分析 JFR 必须等 recording 结束并导出文件。JDK 14 起可以写一段 Java 代码实时订阅事件：

```java
try (var rs = new RecordingStream()) {
    rs.enable("jdk.CPULoad").withPeriod(Duration.ofSeconds(1));
    rs.onEvent("jdk.CPULoad", event -> {
        if (event.getFloat("jvmUser") > 0.8f) {
            // 触发告警或自动 dump
        }
    });
    rs.start();
}
```

这使得**基于 JFR 的自动化监控和告警**成为可能，不再依赖人工导出文件。

> 事件流 API 适合集成到应用的自监控模块中，例如当检测到 GC 暂停超过阈值时，自动触发一次线程 dump 和 JFR dump。

#### JDK 21 ~ 25：虚拟线程与 JFR 增强

| 特性 | 版本 | 对诊断的影响 |
|:--|:--|:--|
| 虚拟线程（Virtual Threads）| JDK 21 | `jstack` 和 Thread Dump 中虚拟线程以新格式呈现，载体线程（carrier thread）与虚拟线程关系需额外关注 |
| JFR 虚拟线程事件 | JDK 21+ | 新增 `jdk.VirtualThreadStart`、`jdk.VirtualThreadEnd`、`jdk.VirtualThreadPinned` 事件，可监控 pinning 问题 |
| JFR 持续分析增强 | JDK 22+ | 默认 recording 模板优化，磁盘 I/O 和 Socket 事件精度提升 |
| ZGC 分代正式 | JDK 23+ | ZGC 的 JFR GC 事件格式变化，分析时需注意 `ZGCYoungPhase`、`ZGCOldPhase` 等新事件 |

**虚拟线程的线程 dump 变化：**

传统 `jstack` 打印的是平台线程（OS 线程）。虚拟线程由 JVM 调度，可能挂载在不同的平台线程上。JDK 21 的线程 dump 会显示：

```
## 虚拟线程
"VirtualThread-123" #... virtual  waiting on java.util.concurrent.locks...
   java.base/java.lang.VirtualThread$$Lambda/0x... run(VirtualThread.java:...)

## 载体线程（可能同时携带虚拟线程栈）
"ForkJoinPool-1-worker-3" #... platform  carrying VirtualThread-123
```

**诊断注意：** 如果虚拟线程出现 `pinned`（被 `synchronized` 或 `native` 方法钉在载体线程上），会丧失虚拟线程的可扩展性优势。JFR 的 `jdk.VirtualThreadPinned` 事件可以量化这一问题。

> JDK 21+ 的生产环境建议：开启 JFR 并关注 `jdk.VirtualThreadPinned` 事件计数。如果 pinned 频率高，优先将 `synchronized` 块替换为 `ReentrantLock`。

***

### 工具选型速查表

| 问题类型 | 首选工具 | 辅助工具 | 避免 |
|:--|:--|:--|:--|
| 进程定位 | `jps -lvm` | `ps`, `jcmd -l` | |
| GC 趋势 | `jstat -gcutil` | `jstat -gccause` | |
| GC 原因 | `jstat -gccause` | JFR GC 事件 | |
| CPU 飙高 | `top -H` → `jstack` | JFR CPU 事件 | VisualVM CPU Sampler（线上） |
| 内存泄漏 | `jmap -histo:live` → Heap Dump → MAT | JFR 内存分配事件 | `jmap -dump`（大堆 STW 长） |
| 死锁 | `jstack -l` | JMC 线程视图 | |
| 类/Metaspace 泄漏 | `jmap -clstats`, `jstat -class` | JFR Class Load 事件 | |
| 综合诊断 | `jcmd` | | |
| 持续监控 | JFR + JMC | | Arthas（非 JDK 工具，需评估） |

> **最后提醒：** 命令行工具解决的是**"现在发生了什么"**，JFR 解决的是**"过去一段时间发生了什么"**。生产环境的最佳实践是**JFR 持续记录 + 命令行工具快速验证 + Heap/Thread Dump 深度分析**的三层体系。
