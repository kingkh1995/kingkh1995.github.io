## JVM 诊断与调优参数速查

### JVM 通用参数

#### 内存

| 参数 | 说明 | JDK 8 | JDK 17 | JDK 21 | JDK 25 |
|:--|:--|:--|:--|:--|:--|
| `-Xms` | 初始堆大小 | ✅ | ✅ | ✅ | ✅ |
| `-Xmx` | 最大堆大小，建议与 `-Xms` 一致 | ✅ | ✅ | ✅ | ✅ |
| `-Xmn` | 新生代（Young）大小 | ✅ | ✅（G1 下不设） | ✅（G1/ZGC 下不设） | ✅（Parallel 下有效） |
| `-Xss` | 线程栈大小，默认 1M（64 位） | ✅ | ✅ | ✅；虚拟线程不受此限制 | ✅；虚拟线程使用动态分配栈 |
| `-XX:NewRatio` | 老年代/新生代比值（默认 2） | ✅ | ✅（Parallel 下有效） | ✅（Parallel 下有效） | ✅（Parallel 下有效） |
| `-XX:SurvivorRatio` | Eden/Survivor 比值（默认 8） | ✅ | ✅（Parallel 下有效） | ✅（Parallel 下有效） | ✅（Parallel 下有效） |
| `-XX:MaxTenuringThreshold` | 晋升老年代的对象年龄上限（默认 15，G1/Parallel 均适用） | ✅（Parallel 15，CMS 6） | ✅（G1 上限 15，CMS 已移除） | ✅ | ✅ |
| `-XX:PretenureSizeThreshold` | 超过该大小的对象直接进入老年代（仅 Parallel） | ✅ | ✅（G1/ZGC 不适用） | ✅（G1/ZGC 不适用） | ✅（G1/ZGC 不适用） |
| `-XX:MetaspaceSize` | 首次 Full GC 触发阈值（非初始分配），GC 后 JVM 可能动态调整 | ✅ | ✅ | ✅ | ✅ |
| `-XX:MaxMetaspaceSize` | 元空间上限，默认无限制，建议与 `-XX:MetaspaceSize` 一致 | ✅ | ✅ | ✅ | ✅ |
| `-XX:MaxDirectMemorySize` | 直接内存上限，默认等于 `-Xmx` | ✅ | ✅ | ✅ | ✅ |
| `-XX:SoftMaxHeapSize` | 软堆上限（ZGC 尽量不超，也会触发 G1 提前回收） | ❌ | ✅ JDK 13+ | ✅ | ✅ |
| `-XX:+UseCompactObjectHeaders` | 紧凑对象头，减 10–20% 堆占用（JDK 24 实验性 → JDK 25 产品特性） | ❌ | ❌ | ❌ | ✅（无需 UnlockExperimentalVMOptions） |
| `-XX:MinHeapFreeRatio` | GC 后堆空闲比例下限，低于则扩容（默认 40） | ✅ | ✅ | ✅ | ✅ |
| `-XX:MaxHeapFreeRatio` | GC 后堆空闲比例上限，高于则缩容（默认 70） | ✅ | ✅ | ✅ | ✅ |
| `-XX:+AlwaysPreTouch` | 启动时提交并清零所有堆内存，避免运行时缺页 | ✅ | ✅ | ✅ | ✅（生产建议开启） |
| `-XX:+UseTransparentHugePages` | 透明大页，提升 TLB 命中率 | ✅ | ✅ | ✅ | ✅（JDK 25 G1 存在 THP 回归） |
| `-XX:+UseLargePages` | 显式大页（需 OS 预先配置） | ✅ | ✅ | ✅ | ✅ |
| `-XX:LargePageSizeInBytes` | 大页尺寸 | ✅ | ✅ | ✅ | ✅ |
| `-XX:+UseTLAB` | 启用线程本地分配缓冲（默认开启） | ✅ | ✅ | ✅ | ✅ |
| `-XX:TLABSize` | TLAB 大小 | ✅ | ✅ | ✅ | ✅ |
| `-XX:+UseCompressedOops` | 压缩普通对象指针（堆 ≤ 32G 默认开启） | ✅ | ✅ | ✅ | ✅ |
| `-XX:+UseCompressedClassPointers` | 压缩类指针（堆 ≤ 32G 默认开启） | ✅ | ✅ | ✅ | ✅（JDK 25 紧凑对象头下可达 64G） |
| `-XX:+UseNUMA` | 启用 NUMA 感知内存分配 | ✅ | ✅ | ✅ | ✅（Parallel/G1/ZGC 均支持） |

> **虚拟线程与 `-Xss`：** JDK 21 引入的虚拟线程（Virtual Threads）使用由 JVM 动态分配的栈空间，**不受 `-Xss` 限制**。但平台线程（Platform Threads）仍受 `-Xss` 约束。混合使用场景下，`-Xss` 只需覆盖平台线程即可，不必为虚拟线程预留大额栈空间。

#### GC 通用

| 参数 | 说明 | JDK 8 | JDK 17 | JDK 21 | JDK 25 |
|:--|:--|:--|:--|:--|:--|
| `-XX:ParallelGCThreads` | STW 阶段并行线程数，默认 CPU 核数 | ✅ | ✅ | ✅ | ✅ |
| `-XX:ConcGCThreads` | 并发阶段线程数 | ✅ | ✅ | ✅ | ✅ |
| `-XX:MaxGCPauseMillis` | 期望最大暂停时间（ms），软目标 | ✅ | ✅ | ✅ | ✅ |
| `-XX:GCTimeRatio` | 吞吐量 = N/(1+N)，默认 99 即 GC 占比 ≤ 1% | ✅ | ✅ | ✅ | ✅ |
| `-XX:+ScavengeBeforeFullGC` | Full GC 前先做一次 YGC（默认开启） | ✅ | ✅ | ✅ | ✅ |
| `-XX:+DisableExplicitGC` | 忽略 `System.gc()` 调用 | ✅ | ✅ | ✅ | ✅ |
| `-XX:+ExplicitGCInvokesConcurrent` | `System.gc()` 触发并发 GC 而非 Full GC | ✅（CMS） | ✅（G1/ZGC/Shenandoah） | ✅ | ✅（G1/ZGC/Shenandoah） |
| `-XX:+UseStringDeduplication` | 字符串去重 | ✅（G1） | ✅（G1/ZGC/Shenandoah） | ✅ | ✅（G1/ZGC/Shenandoah） |
| `-XX:StringDeduplicationAgeThreshold` | 字符串去重的对象年龄阈值（默认 3） | ✅ | ✅ | ✅ | ✅ |

#### 已移除/弃用参数

| 参数 | JDK 8 | JDK 17 | JDK 21 | JDK 25 | 说明 |
|:--|:--|:--|:--|:--|:--|
| `-XX:+UseBiasedLocking` | ✅ 默认开启 | ⚠️ 弃用，默认关闭（JEP 374） | ❌ 已移除（JEP 415） | ❌ | 偏向锁在 JDK 15 默认关闭并弃用，JDK 21 彻底移除。迁移方案：使用 `java.util.concurrent` 锁或无锁数据结构 |

#### 诊断 & 调试

| 参数 | 说明 | JDK 8 | JDK 17 | JDK 25 |
|:--|:--|:--|:--|:--|
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM 时生成堆转储 | ✅ | ✅ | ✅ |
| `-XX:HeapDumpPath=<path>` | 堆转储文件路径 | ✅ | ✅ | ✅ |
| `-XX:ErrorFile=<path>` | JVM 崩溃日志（hs_err）路径 | ✅ | ✅ | ✅ |
| `-XX:OnOutOfMemoryError=<cmd>` | OOM 时执行指定命令 | ✅ | ✅ | ✅ |
| `-XX:NativeMemoryTracking=summary\|detail` | 开启 NMT（detail 有 5–10% 性能损失） | ✅ | ✅ | ✅ |
| `-XX:+PrintCommandLineFlags` | 启动时打印 JVM 参数 | ✅ | ✅ | ✅ |
| `-XX:+PrintFlagsFinal` | 打印所有 JVM 参数最终值 | ✅ | ✅ | ✅ |
| `-XX:+PrintFlagsRanges` | 打印参数取值范围 | ❌ | ✅ JDK 9+ | ✅ |
| `-XX:+UnlockDiagnosticVMOptions` | 解锁诊断参数 | ✅ | ✅ | ✅ |
| `-XX:+UnlockExperimentalVMOptions` | 解锁实验性参数 | ✅ | ✅ | ✅（JDK 24 前开启紧凑对象头需此参数） |

#### CDS（类数据共享）

| 参数 | 说明 | JDK 8 | JDK 17 | JDK 25 |
|:--|:--|:--|:--|:--|
| `-XX:+UseAppCDS` | 应用类数据共享 | 商用 | ✅ JDK 10+ 开源 | ✅ |
| `-Xshare:dump` | 生成 CDS 归档 | ✅ | ✅ | ✅ |
| `-Xshare:on/off/auto` | CDS 模式 | ✅ | ✅ | ✅ |

***

### 垃圾收集器一览

| 收集器 | 分代 | 算法 | 目标 | JDK 8 | JDK 17 | JDK 21 | JDK 25 |
|:--|:--|:--|:--|:--|:--|:--|:--|
| **Serial** | 分代 | 标记-复制 / 标记-整理 | 低开销，单线程 | ✅ | ✅ | ✅ | ✅ |
| **Parallel** | 分代 | 标记-复制 / 标记-整理 | 高吞吐 | **默认** | ✅ | ✅ | ✅ |
| **CMS** | 分代 | ParNew + 并发标记-清除 + Serial Old | 低延迟 | ✅ | ❌ JDK 14 移除 | ❌ | ❌ |
| **G1** | 分代 | 标记-复制（Region 化） | 均衡 | ✅ | **默认**（JDK 9+） | **默认** | **默认** |
| **ZGC** | 非分代 → 分代 | 并发标记-复制（染色指针） | 亚毫秒暂停 | ❌ | ✅ 非分代 | ✅ 分代（需 `-XX:+ZGenerational`） | ✅ 纯分代（默认） |
| **Shenandoah** | 非分代 → 分代 | 并发标记-复制（转发指针） | 亚毫秒暂停 | ❌ | ✅ 非分代 | ✅ 非分代 | ✅ 分代（无需 UnlockExperimentalVMOptions） |

#### LTS 关键差异

| 维度 | JDK 8 | JDK 17 | JDK 21 | JDK 25 |
|:--|:--|:--|:--|:--|
| 默认 GC | Parallel | G1 | G1 | G1 |
| GC 日志 | `-XX:+PrintGCDetails` 等 | `-Xlog:gc*` 统一日志 | 同 JDK 17 | 同 JDK 17 |
| CMS | ✅ 可用 | ❌ 已移除 | ❌ | ❌ |
| ZGC | ❌ | ✅ 非分代（JDK 15 生产） | ✅ 分代可选（JEP 439） | ✅ 纯分代（JEP 474/490） |
| Shenandoah | ❌ | ✅ 非分代（JDK 17 生产） | ✅ 非分代 | ✅ 分代（JEP 521） |
| 虚拟线程 | ❌ | ❌ | ✅（JEP 444） | ✅ |
| 偏向锁 | ✅ 默认开启 | ⚠️ 弃用 | ❌ 已移除 | ❌ |
| 紧凑对象头 | ❌ | ❌ | ❌ | ✅ 产品特性（JEP 519） |
| AppCDS | 商用特性 | ✅ 开源（JDK 10+） | ✅ | ✅ |
| JFR | 商用特性 | ✅ 开源（JDK 11+） | ✅ | ✅ |

> **ZGC 分代演进：** JDK 17 只有非分代 ZGC。JDK 21 引入分代 ZGC（JEP 439），需显式加 `-XX:+ZGenerational`。JDK 25 中分代成为唯一模式（JEP 474/490），非分代已移除，`-XX:+ZGenerational` 不再需要。

> **Shenandoah 分代演进：** JDK 17 中 Shenandoah 为生产就绪的非分代收集器。分代模式是 JDK 24 实验性引入（JEP 404），JDK 25 中不再需要 `UnlockExperimentalVMOptions`（JEP 521），但尚未成为默认。

***

### GC 参数速查

#### G1（JDK 9+ 默认）

| 参数 | 说明 | 备注 |
|:--|:--|:--|
| `-XX:+UseG1GC` | 启用 G1 | JDK 9+ 默认，可省略 |
| `-XX:MaxGCPauseMillis=200` | 期望最大暂停时间（ms） | **软目标**；过小会增加 GC 频率 |
| `-XX:G1HeapRegionSize` | Region 大小（1M–32M，JDK 18 起可至 512M） | 默认 = 堆 / 2048，取 2 的幂 |
| `-XX:InitiatingHeapOccupancyPercent=45` | 堆占用率阈值，触发并发标记 | 默认 45 |
| `-XX:G1MixedGCCountTarget=8` | 混合 GC 的期望次数 | 调大可减少单次混合 GC 量 |
| `-XX:G1MixedGCLiveThresholdPercent=85` | Region 中存活对象占比超此值则不回收 | 默认 85 |
| `-XX:G1NewSizePercent=5` | 新生代占堆最小百分比 | 默认 5 |
| `-XX:G1MaxNewSizePercent=60` | 新生代占堆最大百分比 | 默认 60 |
| `-XX:G1ReservePercent=10` | 预留空间百分比，防止晋升失败 | 默认 10 |
| `-XX:G1HeapWastePercent=5` | 可容忍的堆浪费百分比 | 低于此值不触发混合 GC |
| `-XX:G1OldCSetRegionThresholdPercent=10` | 每次混合 GC 最多回收的老年代 Region 数上限 | 默认 10 |
| `-XX:G1RSetUpdatingPauseTimePercent=10` | 更新 Remembered Set 占暂停时间的比例 | 默认 10 |
| `-XX:ParallelGCThreads` | STW 阶段并行线程数 | 默认 CPU 核数 |
| `-XX:ConcGCThreads` | 并发阶段线程数 | 默认 ParallelGCThreads / 4 |
| `-XX:+G1UseAdaptiveIHOP` | 自适应调整 IHOP 阈值 | JDK 9+ 默认开启 |
| `-XX:+ClassUnloadingWithConcurrentMark` | 并发标记时卸载类 | 默认开启 |
| `-XX:+G1SummarizeRSetStats` | 打印 Remembered Set 统计（GC 日志中） | 调试用 |

**JDK 25 G1 关键改进：**
- 混合 GC 阶段更智能的 Region 选择，减少尾部暂停尖刺（JDK-8351405）
- 多 Region 共享 `G1CardSet`，降低 remembered set 内存开销（JDK-8343782）
- ⚠️ 已知回归：THP + `madvise` 模式下可能分配不到大页，后续更新修复

#### ZGC（JDK 11+ 实验，JDK 15+ 生产，JDK 25 纯分代）

| 参数 | 说明 | 备注 |
|:--|:--|:--|
| `-XX:+UseZGC` | 启用 ZGC | JDK 21 需加 `-XX:+ZGenerational` 才为分代；JDK 25 始终分代 |
| `-XX:+ZGenerational` | 启用分代模式 | JDK 21 专用；JDK 25 已移除 |
| `-XX:ZAllocationSpikeTolerance` | 分配尖刺容忍度 | 默认 2.0 |
| `-XX:ZCollectionInterval` | 两次 GC 最短间隔（秒） | 默认 0（不限制） |
| `-XX:+ZUncommit` | 允许归还内存给 OS | 默认开启 |
| `-XX:ZUncommitDelay=300` | 归还前等待秒数 | 默认 300 |
| `-XX:+ZProactive` | 主动触发 GC（默认开启） | 即使堆未满也周期性 GC |
| `-XX:SoftMaxHeapSize` | 软堆上限，ZGC 尽量不超 | JDK 13+，容器场景推荐 |
| `-XX:+UseLargePages` | 启用大页 | 提升 TLB 效率 |
| `-XX:ParallelGCThreads` | 并行阶段线程数 | 同 G1 |
| `-XX:ConcGCThreads` | 并发阶段线程数 | 建议 ≤ ParallelGCThreads / 2 |

**版本差异：**

| 版本 | 启用方式 |
|:--|:--|
| JDK 17 | `-XX:+UseZGC`（非分代） |
| JDK 21 | `-XX:+UseZGC -XX:+ZGenerational`（分代） |
| JDK 25 | `-XX:+UseZGC`（纯分代，默认；非分代已移除） |

#### Shenandoah（JDK 12+）

| 参数 | 说明 | 备注 |
|:--|:--|:--|
| `-XX:+UseShenandoahGC` | 启用 Shenandoah | |
| `-XX:ShenandoahGCMode=generational` | 分代模式 | JDK 25 无需 `UnlockExperimentalVMOptions` |
| `-XX:ShenandoahGCHeuristics=adaptive` | 启发式策略 | adaptive / compact / static / aggressive |
| `-XX:ShenandoahPauseMax=200` | 期望最大暂停（ms） | 软目标 |
| `-XX:ShenandoahMinFreeThreshold=10` | 最小空闲堆百分比 | 低于触发 GC |
| `-XX:ShenandoahAllocationThreshold` | 分配阈值 | 触发 GC 的分配量 |
| `-XX:+ShenandoahPacing` | 分配速率控制 | 减缓分配以跟上 GC |

#### Parallel（吞吐优先）

| 参数 | 说明 |
|:--|:--|
| `-XX:+UseParallelGC` | 启用 Parallel Scavenge + Parallel Old |
| `-XX:ParallelGCThreads` | GC 线程数 |
| `-XX:MaxGCPauseMillis` | 最大暂停目标 |
| `-XX:GCTimeRatio=99` | 吞吐量目标 = 99/(1+99) ≈ 99% |
| `-XX:+UseAdaptiveSizePolicy` | 自适应调整分代大小（默认开启） |

#### CMS（JDK 8–13，JDK 14 已移除）

| 参数 | 说明 |
|:--|:--|
| `-XX:+UseConcMarkSweepGC` | 启用 ParNew + CMS + Serial Old |
| `-XX:CMSInitiatingOccupancyFraction=68` | 老年代占用率达 68% 触发 CMS 并发周期 |
| `-XX:+UseCMSInitiatingOccupancyOnly` | 仅用固定阈值，关闭 JVM 动态调整 |
| `-XX:+CMSScavengeBeforeRemark` | 重新标记前先做一次 YGC，减少跨代引用 |
| `-XX:+ExplicitGCInvokesConcurrent` | `System.gc()` 触发 CMS 并发周期而非 Full GC |

**升级建议：** CMS → G1（堆 ≤ 32G，均衡延迟）或 ZGC（低延迟，堆 > 16G）。

***

### GC 收集器选型决策树

选型不是"哪个最新选哪个"，而是根据**延迟要求、吞吐量要求和堆大小**做权衡：

```
堆 < 2G 且 CPU 核数 ≤ 2？
  → 是 → Serial（极简，无额外开销）
  → 否 → 延迟要求 < 10ms 且堆 > 16G？
      → 是 → ZGC（亚毫秒暂停，堆大小无关）
      → 否 → 延迟要求 < 10ms 且堆 ≤ 16G？
          → 是 → Shenandoah（亚毫秒，平台支持更广）
          → 否 → 吞吐优先，批处理/离线？
              → 是 → Parallel（高吞吐，可接受秒级暂停）
              → 否 → G1（均衡，JDK 9+ 默认，开箱即用）
```

**快速对照：**

| 场景 | 推荐收集器 | 理由 |
|:--|:--|:--|
| 堆 < 2G，单核 | Serial | 极简无额外开销 |
| 批处理 / 离线，高吞吐优先 | **Parallel** | 可接受秒级暂停 |
| 通用服务端，2–32G 堆 | **G1** | JDK 9+ 默认，开箱即用 |
| 低延迟，> 16G 堆，暂停 < 10ms | **ZGC** | JDK 25 纯分代，全并发 |
| 低延迟，需多平台支持 | **Shenandoah** | JDK 25 分代可用 |
| 容器化微服务，内存敏感 | G1 / ZGC | ZGC 更快归还内存给 OS |
| 金融交易，亚毫秒级停顿 | **ZGC（JDK 17+）** | 实测暂停 < 1ms |
| 电商大促，高并发 + 大堆 | **G1（JDK 17+）** | 可控暂停 + 高吞吐平衡 |

***

### GC 日志分析与 JFR

#### JDK 9+ 统一日志格式 `-Xlog`

JDK 9 引入统一日志框架（JEP 158/271），取代 JDK 8 中繁杂的 `-XX:+PrintGCDetails`、`-XX:+PrintGCDateStamps` 等参数。

**基础格式：**

```
-Xlog:<tag-set>*=<level>:<output>:<decorators>:<output-options>
```

**常用配置：**

| 配置 | 说明 |
|:--|:--|
| `-Xlog:gc*=info:stdout:time,uptime,level,tags` | GC 全部信息输出到 stdout |
| `-Xlog:gc*=info:file=/path/gc-%t.log:time,uptime,level,tags:filecount=5,filesize=20M` | 输出到文件，轮转 5 个，每个 20M |
| `-Xlog:gc+heap=debug` | 堆详情 debug 级别 |
| `-Xlog:gc+phases=debug` | GC 各阶段 debug 级别 |
| `-Xlog:safepoint=info` | Safepoint 信息 |

> JDK 8 迁移到 JDK 17+ 时，将 `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:file` 替换为 `-Xlog:gc*=info:file=...:time,uptime,level,tags` 即可。

#### 常见 GC 日志模式解读

**G1 日志关键模式：**

| 日志关键词 | 含义 | 建议 |
|:--|:--|:--|
| `To-space exhausted` | 晋升失败，老年代不足 | 增大 `-Xmx` 或 `G1ReservePercent` |
| `Humongous allocation` | 大对象分配（≥ Region 一半） | 排查大数组/大 Buffer，考虑拆分 |
| `Full GC` | 并发周期回收不够 | 增大 IHOP 阈值或 `-Xmx` |
| `Concurrent Mark abort` | 并发标记中断（YGC 过多） | 适当降低 `MaxGCPauseMillis` |
| `Evacuation failure` | 存活对象超出目标区域容量 | 增大堆或调大 `G1ReservePercent` |

**ZGC 日志关键模式：**

| 日志关键词 | 含义 | 建议 |
|:--|:--|:--|
| `Out Of Memory` | 堆耗尽且 GC 无法回收 | 增大 `-Xmx` 或排查内存泄漏 |
| `Allocation Stall` | 分配线程等待 GC 完成 | 增大堆或调高 `ZAllocationSpikeTolerance` |
| `Relocation Stall` | 对象迁移等待 | 通常短暂，持续出现需增大堆 |

**ZGC 分代日志（JDK 21+）：**

| 日志关键词 | 含义 |
|:--|:--|
| `ZGC Minor Collection` | 新生代回收，频率高、暂停极低 |
| `ZGC Major Collection` | 老年代回收，频率低、资源消耗大 |
| `Young Generation` / `Old Generation` | 分代 ZGC 的内存池名称 |

#### JFR 在调优中的使用

JFR（Java Flight Recorder，JEP 167/328）是 JDK 内置的低开销事件记录框架，对生产环境性能影响通常 < 1%。

**启动时持续记录（适合长期监控）：**

```bash
-XX:StartFlightRecording=name=prod,disk=true,maxsize=500m,dumponexit=true,filename=/data/jfr/prod.jfr
```

**运行时按需记录（适合问题排查）：**

```bash
jcmd <pid> JFR.start name=debug duration=5m filename=/data/jfr/debug.jfr
jcmd <pid> JFR.dump name=debug filename=/data/jfr/debug.jfr
jcmd <pid> JFR.stop name=debug
```

**JFR 分析关注的事件：**

| 事件 | 调优价值 |
|:--|:--|
| `jdk.GCPhasePause` | 精确到子阶段的 GC 暂停时间 |
| `jdk.SafepointBegin` / `jdk.SafepointEnd` | Safepoint 进入耗时，定位"假 GC" |
| `jdk.ObjectAllocationInNewTLAB` | 大对象分配追踪 |
| `jdk.JavaMonitorEnter` / `jdk.JavaMonitorWait` | 锁竞争热点 |
| `jdk.ThreadPark` / `jdk.ThreadSleep` | 线程阻塞模式 |
| `jdk.CPULoad` | CPU 负载与 GC 线程关联 |

> JFR 数据可用 JDK Mission Control（JMC）可视化分析，也可编程解析。对于线上偶发 GC 尖刺，**持续开启 JFR 并事后截取**是最有效的定位手段。

***

### JDK 命令行工具

#### jps — 虚拟机进程状态

```
jps [-q] [-m] [-l] [-v] [hostid]
```

| 选项 | 作用 |
|:--|:--|
| `-q` | 只输出 LVMID |
| `-m` | 输出 main() 参数 |
| `-l` | 输出主类全名 / JAR 路径 |
| `-v` | 输出 JVM 启动参数 |

#### jstat — 运行时状态监控

```
jstat -<option> <vmid> [<interval> [<count>]]
```

| 选项 | 监控内容 |
|:--|:--|
| `-gc` | 各区域容量、使用量、GC 时间 |
| `-gcutil` | 各区域使用百分比 |
| `-gccause` | 使用百分比 + 最近一次 GC 原因 |
| `-gcnew` | 新生代 GC 情况 |
| `-gcold` | 老年代 GC 情况 |

**输出示例：**

`jstat -gcutil <pid> 1s 3` — 各区域使用百分比，每秒一次共 3 次：

```
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00   4.20  55.30  23.10  95.20  92.80   123    1.520     0    0.000    1.520
  0.00   4.20  58.10  23.15  95.20  92.81   124    1.531     0    0.000    1.531
  0.00   4.20  60.45  23.20  95.21  92.82   125    1.542     0    0.000    1.542
```

列说明：S0/S1=Survivor 使用率，E=Eden，O=Old，M=Metaspace，CCS=压缩类空间，YGC=Young GC 次数，YGCT=YGC 耗时，FGC=Full GC 次数，FGCT=FGC 耗时，GCT=总 GC 耗时。

#### jinfo — 查看/调整 JVM 参数

```
jinfo [option] <pid>
```

| 选项 | 作用 |
|:--|:--|
| `-flag <name>` | 查看指定参数值 |
| `-flag [+/-]<name>` | 开启/关闭布尔参数 |
| `-flags` | 打印所有 JVM 参数 |

#### jmap — 堆转储与统计

```
jmap [option] <vmid>
```

| 选项 | 作用 |
|:--|:--|
| `-heap` | 堆详细信息（收集器、配置、分代状态） |
| `-histo[:live]` | 对象统计（类、实例数、合计大小） |
| `-dump:[live,]format=b,file=<file>` | 生成 heap dump |

> ⚠️ `jmap -dump` 加 `live` 会触发 Full GC。生产建议用 `jcmd <pid> GC.heap_dump` 替代。

#### jstack — 线程堆栈

```
jstack [-l] [-F] [-m] <vmid>
```

| 选项 | 作用 |
|:--|:--|
| `-l` | 输出锁的额外信息 |
| `-F` | 强制输出堆栈（进程无响应时） |
| `-m` | 混合模式（含 native C/C++ 堆栈） |

**常用组合：**

```bash
## CPU 高排查四步
top -Hp <pid>                          # 1. 找到高 CPU 线程
printf "%x\n" <tid>                    # 2. TID 转 16 进制
jstack <pid> | grep -A20 <hex_tid>     # 3. 查看线程堆栈

## 线程状态分布统计
jstack <pid> | grep "java.lang.Thread.State" | sort | uniq -c | sort -rn

## 查找死锁
jstack <pid> | grep -A10 "deadlock"
```

#### jcmd — 集成式工具箱（JDK 7+，推荐）

```
jcmd                     # 列出所有 Java 进程（等效 jps -lm）
jcmd <pid> help          # 列出该进程所有可用命令
```

| jcmd 命令 | 说明 |
|:--|:--|
| `jcmd <pid> VM.version` | JVM 版本信息 |
| `jcmd <pid> VM.flags` | 所有 JVM 参数 |
| `jcmd <pid> GC.run` | 触发 Full GC |
| `jcmd <pid> GC.heap_dump <file>` | 生成 heap dump（推荐，不触发 Full GC） |
| `jcmd <pid> GC.class_histogram` | 类实例统计 |
| `jcmd <pid> Thread.print` | 线程堆栈 |
| `jcmd <pid> VM.native_memory summary` | NMT 概要（需开启 `-XX:NativeMemoryTracking`） |
| `jcmd <pid> JFR.start name=<n> duration=<t> filename=<f>` | 启动 JFR 录制 |
| `jcmd <pid> JFR.dump name=<n> filename=<f>` | 导出 JFR 数据 |

**NMT 使用流程：**

```bash
## 启动时开启（detail 有 5–10% 性能损失）
-XX:NativeMemoryTracking=detail

## 查询
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory detail scale=MB

## 建基线 → 压测/运行 → 看差异定位泄漏
jcmd <pid> VM.native_memory baseline
jcmd <pid> VM.native_memory summary.diff
```

***

### GC 调优流程

#### 1. 选收集器

参考上文**选型决策树**。核心原则：
- **不要优化不存在的问题**：G1 在大多数服务端场景已足够，除非有明确的延迟 SLA 才上 ZGC
- **先调堆大小，再调收集器参数**：`-Xmx` 和 `-Xms` 一致是第一条铁律
- **不要同时调多个参数**：一次只改一个，观察效果

#### 2. 各版本建议 JVM 配置

**JDK 8（CMS 或 G1）：**

```
## CMS（低延迟场景）
-Xms4g -Xmx4g -Xmn2g -Xss256k
-XX:+UseConcMarkSweepGC -XX:+UseParNewGC
-XX:CMSInitiatingOccupancyFraction=70 -XX:+UseCMSInitiatingOccupancyOnly
-XX:+CMSParallelRemarkEnabled -XX:+CMSScavengeBeforeRemark
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/data/logs/gc-%t.log
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M
-XX:+AlwaysPreTouch

## G1（JDK 8 中使用 G1 需显式开启）
-Xms4g -Xmx4g -Xss256k
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/data/logs/gc-%t.log
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M
-XX:+AlwaysPreTouch
```

**JDK 17（G1 默认）：**

```
-Xms4g -Xmx4g -Xss256k
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/
-XX:ErrorFile=/data/logs/hs_err_%p.log
-Xlog:gc*=info:file=/data/logs/gc-%t.log:time,uptime,level,tags:filecount=5,filesize=20M
-XX:+AlwaysPreTouch
```

**JDK 17（ZGC，低延迟场景）：**

```
-Xms16g -Xmx16g -Xss256k
-XX:+UseZGC
-XX:SoftMaxHeapSize=12g
-XX:+ZUncommit -XX:ZUncommitDelay=300
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/
-XX:ErrorFile=/data/logs/hs_err_%p.log
-Xlog:gc*=info:file=/data/logs/gc-%t.log:time,uptime,level,tags:filecount=5,filesize=20M
-XX:+AlwaysPreTouch
```

**JDK 21（ZGC 分代）：**

```
-Xms16g -Xmx16g -Xss256k
-XX:+UseZGC -XX:+ZGenerational
-XX:SoftMaxHeapSize=12g
-XX:+ZUncommit -XX:ZUncommitDelay=300
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/
-XX:ErrorFile=/data/logs/hs_err_%p.log
-Xlog:gc*=info:file=/data/logs/gc-%t.log:time,uptime,level,tags:filecount=5,filesize=20M
-XX:+AlwaysPreTouch
```

> JDK 21 使用 ZGC 时建议显式开启分代模式（`-XX:+ZGenerational`），相比非分代模式吞吐量可提升 10–40%。

**JDK 25（G1 默认，推荐开启新特性）：**

```
-Xms4g -Xmx4g -Xss256k
-XX:MaxGCPauseMillis=200
-XX:+UseCompactObjectHeaders
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/
-XX:ErrorFile=/data/logs/hs_err_%p.log
-Xlog:gc*=info:file=/data/logs/gc-%t.log:time,uptime,level,tags:filecount=5,filesize=20M
-XX:+AlwaysPreTouch
```

> JDK 25 中 ZGC 始终为分代模式，无需 `-XX:+ZGenerational`。紧凑对象头（`-XX:+UseCompactObjectHeaders`）已成为产品特性，无需 `UnlockExperimentalVMOptions`，建议测试后开启。

#### 3. G1 调优路径

```
调大 -Xmx 解决容量问题
  → 调整 -XX:MaxGCPauseMillis（默认 200ms 通常已够）
    → 若 Mixed GC 耗时长，增大 -XX:G1MixedGCCountTarget
      → 若晋升失败（to-space exhausted），调大 -XX:G1ReservePercent
        → 若停顿波动大，保持自适应 IHOP 开启
```

**核心指标：**

| 指标 | 健康值 |
|:--|:--|
| YGC 耗时 | < 50ms |
| Mixed GC 耗时 | < 200ms |
| Full GC 次数 | 24h 内 ≤ 1 |
| Full GC 耗时 | < 1s |

#### 4. ZGC 调优要点

- **核心只有堆大小**：留 20–30% 余量供并发回收期间的分配
- 容器环境用 `-XX:SoftMaxHeapSize`，低于 `-Xmx`
- JDK 17 非分代模式下不设新生代大小、SurvivorRatio 等分代参数
- JDK 21+ 分代模式下 ZGC 自动管理分代比例，通常无需手动干预
- 目标暂停 < 1ms
- 持续观察 GC 日志中的 `Allocation Stall`，若频繁出现需增大堆

#### 5. Full GC 定位思路

```
出现 Full GC
  → 看 GC 日志 cause 确认是元空间还是堆不足
    → 元空间：增大 -XX:MaxMetaspaceSize，或排查类加载泄漏
    → 堆不足：
      → jcmd <pid> GC.class_histogram 看 top 对象
      → 大对象频繁分配 → 优化数据结构
      → 堆不够 → 增大 -Xmx
      → 对象持续增长 → 内存泄漏排查（见下）
```

**内存泄漏排查流程（结合 jvm-monitoring-tools.md）：**

```
1. 确认泄漏：jstat -gcutil <pid> 1s，观察 O 区（Old）是否只增不减
2. 获取 dump：jcmd <pid> GC.heap_dump /data/dump/leak.hprof
3. MAT 分析：Dominator Tree 找 retained heap 最大的对象
4. 定位路径：Path to GC Roots → exclude weak/soft references
5. 代码审查：检查 static 集合、未关闭的资源、缓存未设上限
6. 验证修复：修复后重新压测，观察 jstat O 区是否稳定
```

***

### Project Leyden AOT 缓存调优（JDK 24~25）

Project Leyden 是 OpenJDK 改善 Java 启动速度和预热时间的长期项目，核心思路是通过**训练运行（Training Run）**捕获应用的类加载、链接和方法执行画像，生成 AOT 缓存，后续生产运行直接从缓存恢复，无需重新执行耗时的类加载和 JIT 预热过程。

**关键 JEP：**

| JEP | 版本 | 内容 |
|:--|:--|:--|
| JEP 483 | JDK 24 | Ahead-of-Time Class Loading & Linking：将训练运行中的类加载和链接状态存入 AOT 缓存 |
| JEP 514 | JDK 25 | Ahead-of-Time Command-Line Ergonomics：简化 AOT 缓存命令行使用 |
| JEP 515 | JDK 25 | Ahead-of-Time Method Profiling：将方法执行画像存入缓存，JIT 启动即生成优化代码 |

**AOT 缓存使用流程：**

```bash
## 1. 训练运行（生成 AOT 缓存）
java -XX:AOTCache=app.aot -cp app.jar com.example.Main

## 2. 生产运行（使用 AOT 缓存）
java -XX:AOTCache=app.aot -cp app.jar com.example.Main
```

> **注意：** JEP 514 简化了命令行体验，训练运行和生产运行可以使用**完全相同的命令行**（只需带上 `-XX:AOTCache`）。如果缓存文件不存在，JVM 会自动创建；如果存在，则自动加载。

**典型收益与限制：**

- **启动时间**：常见提升 2–4 倍，类加载密集型应用收益最大
- **预热时间**：由于方法画像已存在于缓存中，JIT 编译器可在启动后立即生成优化后的本地代码，无需等待热点代码收集
- **与 Native Image 的区别**：Leyden **不牺牲 Java 的动态性**——反射、动态类加载、JVMCI 等能力完全保留，无需配置可达性元数据
- **限制**：仅支持内置类加载器（Bootstrap / Platform / App ClassLoader），用户自定义类加载器加载的类无法被缓存

**适用场景：**

- Serverless / FaaS 场景（冷启动敏感）
- 容器化微服务（频繁扩缩容）
- CI/CD 流水线中的短期任务（如 Gradle/Maven 守护进程）

> **生产建议：** JDK 25 中 Leyden 的 AOT 缓存流程已经成熟，建议对启动时间敏感的应用进行评估。训练运行应尽可能模拟生产负载，以确保缓存中的方法画像具有代表性。

***

### 现网诊断调优实战

#### 案例一：反射膨胀导致频繁 Full GC

**现象：** 系统运行一段时间后 Full GC 频繁（每分钟多次），CPU 飙升，每次 GC 后内存恢复，很快又升满。

**原因：** 大量 `BeanUtils.copyProperties` 触发反射膨胀机制。JVM 默认用 JNI 存取器执行反射，同类方法调用超 15 次后切换为字节码存取器，每属性生成 `GeneratedMethodAccessorXXX` 代理类，由 `DelegatingClassLoader` 加载且不回收，持续累积撑爆元空间。

**排查：**

```bash
jstat -gc <pid> 1s          # 元空间持续增长 → FGC
jcmd <pid> GC.class_histogram | grep GeneratedMethodAccessor
jcmd <pid> VM.classloader_stats
```

**方案：**
- 短期：`-Dsun.reflect.inflationThreshold=2147483647` 关闭反射膨胀
- 长期：改用 MapStruct 等编译期代码生成

---

#### 案例二：大文件定时加载导致暂停飙升

**现象：** 定时任务加载大文件时 YGC 耗时从 20ms 飙升至 500ms+。

**原因：** 大文件对象直接入老年代，新生代中存活大对象导致复制算法耗时暴增。

**JDK 8 + CMS 方案：** `-XX:SurvivorRatio=65536` 去掉 Survivor，大对象一次 YGC 后直入老年代。

**JDK 17/25 + G1 方案：** 升级 G1，大对象（≥ Region 一半）直接分配 Humongous Region，不参与新生代复制。

**JDK 25 + ZGC 方案：** 全并发回收，不受大对象数量和大小影响。

---

#### 案例三：电商大促 G1 调优（JDK 17）

**场景：** 某电商平台，JDK 17，堆 24G，大促期间 QPS 从 2k 飙升至 15k，GC 暂停从 30ms 恶化到 400ms+，频繁触发 Full GC。

**诊断过程：**

1. **GC 日志分析：** `-Xlog:gc*=info` 输出显示大量 `To-space exhausted` 和 `Humongous allocation`，Mixed GC 单次耗时 300ms+
2. **jstat 观察：** 老年代（O 区）使用率在大促开始后持续攀升，YGC 频率从 5s 一次变为 1s 一次
3. **JFR 录制：** `jdk.GCPhasePause` 事件显示 Evacuation 阶段占暂停 80% 以上

**根因：**
- 高并发下短期对象暴增，新生代自动扩到 60%（G1MaxNewSizePercent 默认上限），导致老年代空间紧张
- 订单详情缓存对象（平均 512KB）大量进入 Humongous Region，触发频繁 Mixed GC
- `G1MixedGCLiveThresholdPercent=85` 过高，导致大量高存活率 Region 被纳入混合回收，复制耗时剧增

**调优方案：**

```
-XX:G1MaxNewSizePercent=35          # 限制新生代最大 35%，保老年代空间
-XX:G1MixedGCLiveThresholdPercent=70 # 降低存活阈值，减少高存活 Region 回收
-XX:G1MixedGCCountTarget=16         # 分散 Mixed GC 压力
-XX:G1ReservePercent=15             # 增大预留，应对突发分配
-XX:InitiatingHeapOccupancyPercent=35 # 提前触发并发标记
```

**效果：** 大促期间 GC 最大暂停降至 80ms，Full GC 归零，吞吐量提升 22%。

> **方法论：** 大堆 + 高并发 + 突发流量场景，G1 的核心矛盾是"新生代扩张挤占老年代"。通过限制 `G1MaxNewSizePercent` 并提前触发并发标记，可避免老年代空间耗尽导致的 Full GC。

---

#### 案例四：金融交易 ZGC 调优（JDK 17）

**场景：** 某证券核心交易系统，JDK 17，堆 32G，要求 GC 暂停 < 2ms（P99），原使用 G1 时大促期间暂停偶发 50ms+，触发风控熔断。

**迁移至 ZGC：**

```
-Xms32g -Xmx32g
-XX:+UseZGC
-XX:SoftMaxHeapSize=28g
-XX:ZCollectionInterval=5
-XX:+ZProactive
-XX:+AlwaysPreTouch
-Xlog:gc*=info:file=/data/logs/gc-%t.log:time,uptime,level,tags:filecount=10,filesize=50M
```

**调优要点：**
- **`-XX:SoftMaxHeapSize=28g`**：容器环境限制内存使用，避免被 cgroup OOM Killer 命中
- **`-XX:ZCollectionInterval=5`**：金融交易有固定收盘结算窗口，强制 5 秒一次 GC 防止堆在峰值时耗尽
- **`-XX:+ZProactive`**：周期性主动 GC，避免堆使用率达到临界值才触发
- **不手动设置 `ConcGCThreads`**：JDK 17 ZGC 支持动态 GC 线程数（`UseDynamicNumberOfGCThreads` 默认开启），手动干预反而可能降低自适应效果

**监控验证：**

```bash
## 实时查看 ZGC 暂停
jcmd <pid> VM.gc_info

## JFR 精确暂停分析
jcmd <pid> JFR.start name=zgc_check duration=1m filename=/data/jfr/zgc.jfr
```

**效果：** P99 GC 暂停从 G1 的 45ms 降至 0.8ms，P999 < 2ms，无 Allocation Stall。吞吐量下降约 8%，在可接受范围内。

> **方法论：** ZGC 的调优哲学是"少即是多"——堆大小留足余量后，绝大多数场景无需手动调整参数。金融交易等极端延迟敏感场景，核心工作是**验证暂停 SLA** 而非调参数。

---

#### 案例五：Safepoint 导致长时间停顿

**现象：** GC 日志中 GC 耗时（real）正常，但 `user + sys` 远大于 real——大量时间花在等线程进安全点。

**排查：**

```bash
## JDK 8
-XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1

## 定位超时线程
-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=2000
```

**原因：** 可数循环（索引 `int` 或更小类型）不在循环头插入安全点检查，单次循环耗时长会拖慢全部线程。

**方案：** 循环索引从 `int` 改为 `long`，编译器在循环头插入安全点检查。

---

#### 案例六：CPU 飙升排查

**常见原因：** 死循环、频繁 GC、大量线程调度、外部进程调用。

```bash
top -c                               # 1. 找到高 CPU 进程
top -Hp <pid>                        # 2. 找到高 CPU 线程
printf "%x\n" <tid>                  # 3. TID 转 16 进制
## 4. 连续采样确认
for i in 1 2 3 4 5; do jstack <pid> | grep -A20 <hex_tid>; sleep 1; done
## 5. 线程状态分布
jstack <pid> | grep "java.lang.Thread.State" | sort | uniq -c | sort -rn
```

**快速判断：**
- GC 线程 CPU 高 → 堆太小或内存泄漏，看 GC 日志
- 业务线程 CPU 高 → 死循环/算法问题，看 jstack
- C2 Compiler 线程高 → JIT 压力大，正常但持续过久需排查

---

#### 案例七：大内存硬件 CMS 退化

**现象：** 内存 16G → 64G 升级后，Full GC 停顿反而成倍增加。

**原因（JDK 8 + CMS）：** 大对象直入老年代，CMS 无法并发回收，退化为单线程 Serial Old Full GC，堆越大暂停越久。

**方案：**
- JDK 8：拆分为多个小 JVM 实例，每个单独用 CMS
- JDK 17/25：**直接升 ZGC**，全并发回收，暂停与堆大小无关

***

### 参考

- [JDK 25 G1/Parallel/Serial GC Changes — Thomas Schatzl](https://tschatzl.github.io/2025/08/12/jdk25-g1-serial-parallel-gc-changes.html)
- [Oracle JDK 25 Release Notes](https://www.oracle.com/java/technologies/javase/25-relnote-issues.html)
- [JEP 439: Generational ZGC](https://openjdk.org/jeps/439)
- [JEP 474: ZGC: Generational Mode by Default](https://openjdk.org/jeps/474)
- [JEP 490: ZGC: Remove the Non-Generational Mode](https://openjdk.org/jeps/490)
- [JEP 519: Compact Object Headers](https://openjdk.org/jeps/519)
- [JEP 521: Generational Shenandoah](https://openjdk.org/jeps/521)
- [JEP 374: Deprecate and Disable Biased Locking](https://openjdk.org/jeps/374)
- [HotSpot GC Tuning Guide (JDK 25)](https://docs.oracle.com/en/java/javase/25/gctuning/)
- [深入理解 Java 虚拟机（第 3 版）— 周志明](https://book.douban.com/subject/34907497/)
