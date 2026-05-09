## JVM 通用参数

### 内存

| 参数 | 说明 | JDK 8 | JDK 17 | JDK 25 |
|:--|:--|:--|:--|:--|
| `-Xms` | 初始堆大小 | ✅ | ✅ | ✅ |
| `-Xmx` | 最大堆大小，建议与 `-Xms` 一致 | ✅ | ✅ | ✅ |
| `-Xmn` | 新生代（Young）大小 | ✅ | ✅ | ✅（G1/ZGC 下不设） |
| `-Xss` | 线程栈大小，默认 1M（64 位） | ✅ | ✅ | ✅；虚拟线程（JDK 21+）不受此限制 |
| `-XX:NewRatio` | 老年代/新生代比值（默认 2，即 Old:Young=2:1） | ✅ | ✅ | ✅（Parallel 下有效） |
| `-XX:SurvivorRatio` | Eden/Survivor 比值（默认 8） | ✅ | ✅ | ✅（Parallel 下有效） |
| `-XX:MaxTenuringThreshold` | 晋升老年代的对象年龄阈值（默认 15，CMS 默认 6） | ✅ | ✅ | ✅（G1 上限 15） |
| `-XX:PretenureSizeThreshold` | 超过该大小的对象直接进入老年代（仅 CMS/Parallel） | ✅ | ✅ | ✅（G1/ZGC 不适用） |
| `-XX:MetaspaceSize` | 首次 Full GC 触发阈值（非初始分配），GC 后 JVM 可能动态调整 | ✅ | ✅ | ✅ |
| `-XX:MaxMetaspaceSize` | 元空间上限，默认无限制，建议与 `-XX:MetaspaceSize` 一致，避免启动期频繁 GC | ✅ | ✅ | ✅ |
| `-XX:MaxDirectMemorySize` | 直接内存上限，默认等于 `-Xmx` | ✅ | ✅ | ✅ |
| `-XX:SoftMaxHeapSize` | 软堆上限（ZGC 尽量不超，也会触发 G1 提前回收） | ❌ | ✅ JDK 13+ | ✅ |
| `-XX:+UseCompactObjectHeaders` | 紧凑对象头，减 10–20% 堆占用 | ❌ | ❌ | ✅ JDK 25 正式 |
| `-XX:MinHeapFreeRatio` | GC 后堆空闲比例下限，低于则扩容（默认 40） | ✅ | ✅ | ✅ |
| `-XX:MaxHeapFreeRatio` | GC 后堆空闲比例上限，高于则缩容（默认 70） | ✅ | ✅ | ✅ |
| `-XX:+AlwaysPreTouch` | 启动时提交并清零所有堆内存，避免运行时缺页 | ✅ | ✅ | ✅（生产建议开启） |
| `-XX:+UseTransparentHugePages` | 透明大页，提升 TLB 命中率 | ✅ | ✅ | ✅（JDK 25 G1 存在 THP 回归） |
| `-XX:+UseLargePages` | 显式大页（需 OS 预先配置） | ✅ | ✅ | ✅ |
| `-XX:LargePageSizeInBytes` | 大页尺寸 | ✅ | ✅ | ✅ |
| `-XX:+UseTLAB` | 启用线程本地分配缓冲（默认开启） | ✅ | ✅ | ✅ |
| `-XX:TLABSize` | TLAB 大小 | ✅ | ✅ | ✅ |
| `-XX:+UseCompressedOops` | 压缩普通对象指针（堆 ≤ 32G 默认开启） | ✅ | ✅ | ✅ |
| `-XX:+UseCompressedClassPointers` | 压缩类指针（堆 ≤ 32G 默认开启） | ✅ | ✅ | ✅（JDK 25 紧凑对象头下可达 64G） |
| `-XX:+UseNUMA` | 启用 NUMA 感知内存分配（多 CPU 节点） | ✅ | ✅ | ✅（Parallel/G1/ZGC 均支持） |

### GC 通用

| 参数 | 说明 | JDK 8 | JDK 17 | JDK 25 |
|:--|:--|:--|:--|:--|
| `-XX:ParallelGCThreads` | STW 阶段并行线程数，默认 CPU 核数 | ✅ | ✅ | ✅ |
| `-XX:ConcGCThreads` | 并发阶段线程数 | ✅ | ✅ | ✅ |
| `-XX:MaxGCPauseMillis` | 期望最大暂停时间（ms），软目标 | ✅ | ✅ | ✅ |
| `-XX:GCTimeRatio` | 吞吐量 = N/(1+N)，默认 99 即 GC 占比 ≤1% | ✅ | ✅ | ✅ |
| `-XX:+ScavengeBeforeFullGC` | Full GC 前先做一次 YGC（默认开启） | ✅ | ✅ | ✅ |
| `-XX:+DisableExplicitGC` | 忽略 `System.gc()` 调用 | ✅ | ✅ | ✅ |
| `-XX:+ExplicitGCInvokesConcurrent` | `System.gc()` 触发并发 GC 而非 Full GC | ✅ | ✅（CMS/G1） | ✅（G1） |
| `-XX:+UseStringDeduplication` | 字符串去重 | ✅（G1） | ✅（G1/ZGC/Shenandoah） | ✅（G1/ZGC/Shenandoah） |
| `-XX:StringDeduplicationAgeThreshold` | 字符串去重的对象年龄阈值（默认 3） | ✅ | ✅ | ✅ |

### 偏向锁

| 参数 | JDK 8 | JDK 17 | JDK 25 |
|:--|:--|:--|:--|
| `-XX:+UseBiasedLocking` | ✅ 默认开启 | ❌ JDK 15 弃用 → JDK 21 移除 | ❌ |

### 诊断 & 调试

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
| `-XX:+UnlockExperimentalVMOptions` | 解锁实验性参数 | ✅ | ✅ | ✅ |

### CDS（类数据共享）

| 参数 | 说明 | JDK 8 | JDK 17 | JDK 25 |
|:--|:--|:--|:--|:--|
| `-XX:+UseAppCDS` | 应用类数据共享 | 商用 | ✅ JDK 10+ 开源 | ✅ |
| `-Xshare:dump` | 生成 CDS 归档 | ✅ | ✅ | ✅ |
| `-Xshare:on/off/auto` | CDS 模式 | ✅ | ✅ | ✅ |

---

## 垃圾收集器一览

| 收集器 | 分代 | 算法 | 目标 | JDK 8 | JDK 17 | JDK 25 |
|:--|:--|:--|:--|:--|:--|:--|
| **Serial** | 分代 | 标记-复制 / 标记-整理 | 低开销，单线程 | ✅ | ✅ | ✅ |
| **Parallel** | 分代 | 标记-复制 / 标记-整理 | 高吞吐 | **默认** | ✅ | ✅ |
| **CMS** | 分代 | ParNew + 并发标记-清除 + Serial Old | 低延迟 | ✅ | ❌ JDK 14 移除 | ❌ |
| **G1** | 分代 | 标记-复制（Region 化） | 均衡 | ✅ | **默认**（JDK 9+） | **默认** |
| **ZGC** | 分代 | 并发标记-复制（染色指针） | 亚毫秒暂停 | ❌ | ✅ 非分代 | ✅ 纯分代 |
| **Shenandoah** | 分代 | 并发标记-复制（转发指针） | 亚毫秒暂停 | ❌ | ✅ 非分代 | ✅ 分代正式 |

### LTS 关键差异

| 维度 | JDK 8 | JDK 17 | JDK 25 |
|:--|:--|:--|:--|
| 默认 GC | Parallel | G1 | G1 |
| GC 日志 | `-XX:+PrintGCDetails` 等 | `-Xlog:gc*` 统一日志 | 同 JDK 17 |
| CMS | ✅ 可用 | ❌ JDK 14 移除 | ❌ |
| ZGC | ❌ | ✅ 非分代（JDK 11 实验 → 15 生产） | ✅ 纯分代（非分代已移除） |
| Shenandoah | ❌ | ✅ 非分代（JDK 12+） | ✅ 分代正式 |
| 虚拟线程 | ❌ | ❌（JDK 21 正式） | ✅ |
| 偏向锁 | ✅ 默认开启 | ❌ JDK 21 移除 | ❌ |
| 紧凑对象头 | ❌ | ❌ | ✅ 减 10–20% 堆 |
| AppCDS | 商用特性 | ✅ 开源（JDK 10+） | ✅ |
| JFR | 商用特性 | ✅ 开源（JDK 11+） | ✅ |
| jvisualvm | ✅ 内置 | ❌ JDK 14 移除 | ❌（可单独下载） |
| `-XX:+UseCompressedOops` | ≤ 32G 堆 | ≤ 32G 堆 | ≤ 32G 堆 |
| `-XX:+UseCompressedClassPointers` | ≤ 32G 堆 | ≤ 32G 堆 | ≤ 32G 堆（紧凑对象头下可至 64G） |

---

## GC 参数速查

### CMS（JDK 8–13，JDK 14 已移除）

| 参数 | 说明 |
|:--|:--|
| `-XX:+UseConcMarkSweepGC` | 启用 ParNew + CMS + Serial Old |
| `-XX:+UseParNewGC` | 显式启用 ParNew 作为新生代收集器 |
| `-XX:CMSInitiatingOccupancyFraction=68` | 老年代占用率达 68% 触发 CMS 并发周期 |
| `-XX:+UseCMSInitiatingOccupancyOnly` | 仅用固定阈值，关闭 JVM 动态调整 |
| `-XX:+CMSParallelRemarkEnabled` | 并行重新标记 |
| `-XX:+CMSScavengeBeforeRemark` | 重新标记前先做一次 YGC，减少跨代引用 |
| `-XX:+CMSClassUnloadingEnabled` | 并发标记时卸载无用类 |
| `-XX:+UseCMSCompactAtFullCollection` | Full GC 后压缩老年代 |
| `-XX:CMSFullGCsBeforeCompaction=0` | 每 N 次 Full GC 后压缩，0 = 每次都压缩 |
| `-XX:+CMSParallelInitialMarkEnabled` | 并行初始标记 |
| `-XX:CMSWaitDuration` | CMS 并发周期完成后等待时间（ms） |
| `-XX:+ExplicitGCInvokesConcurrent` | `System.gc()` 触发 CMS 并发周期而非 Full GC |
| `-XX:ParallelCMSThreads` | CMS 并发线程数 |

**升级建议：** CMS → G1（堆 ≤ 32G，均衡延迟）或 ZGC（低延迟，堆 > 16G）。

### G1（JDK 9+ 默认）

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
| `-XX:+G1UseAdaptiveConcRefinement` | 自适应并发引用更新线程 | 默认开启 |
| `-XX:+ReduceInitialCardMarks` | 减少初始卡标记 | 默认开启 |
| `-XX:+ClassUnloadingWithConcurrentMark` | 并发标记时卸载类 | 默认开启 |
| `-XX:+G1SummarizeRSetStats` | 打印 Remembered Set 统计（GC 日志中） | 调试用 |
| `-XX:G1SummarizeRSetStatsPeriod=0` | RS 统计打印间隔（GC 次数），0 = 仅结束 | 调试用 |

**JDK 25 G1 关键改进：**
- 混合 GC 阶段更智能的 Region 选择，减少尾部暂停尖刺（JDK-8351405）
- 多 Region 共享 `G1CardSet`，降低 remembered set 内存开销（JDK-8343782）
- ⚠️ 已知回归：THP + `madvise` 模式下可能分配不到大页，后续更新修复

### ZGC（JDK 11+，JDK 25 纯分代）

| 参数 | 说明 | 备注 |
|:--|:--|:--|
| `-XX:+UseZGC` | 启用 ZGC | JDK 25 下始终为分代模式 |
| `-XX:ZAllocationSpikeTolerance` | 分配尖刺容忍度 | 默认 2.0 |
| `-XX:ZCollectionInterval` | 两次 GC 最短间隔（秒） | 默认 0（不禁限制） |
| `-XX:ZFragmentationLimit` | 碎片上限百分比 | JDK 25 重构后碎片更少 |
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
| JDK 8 | ❌ 不存在 |
| JDK 17 | `-XX:+UseZGC`（非分代） |
| JDK 25 | `-XX:+UseZGC`（纯分代，默认；非分代已移除） |

### Parallel（吞吐优先）

| 参数 | 说明 |
|:--|:--|
| `-XX:+UseParallelGC` | 启用 Parallel Scavenge + Parallel Old |
| `-XX:ParallelGCThreads` | GC 线程数 |
| `-XX:MaxGCPauseMillis` | 最大暂停目标 |
| `-XX:GCTimeRatio=99` | 吞吐量目标 = 99/(1+99) ≈ 99% |
| `-XX:+UseAdaptiveSizePolicy` | 自适应调整分代大小（默认开启） |
| `-XX:YoungGenerationSizeIncrement` | 新生代动态调整步长（自适应策略下） |
| `-XX:TenuredGenerationSizeIncrement` | 老年代动态调整步长 |

### Shenandoah（JDK 12+）

| 参数 | 说明 | 备注 |
|:--|:--|:--|
| `-XX:+UseShenandoahGC` | 启用 Shenandoah | |
| `-XX:ShenandoahGCMode=generational` | 分代模式 | JDK 25 正式，无需 `UnlockExperimentalVMOptions` |
| `-XX:ShenandoahGCHeuristics=adaptive` | 启发式策略 | adaptive / compact / static / aggressive |
| `-XX:ShenandoahPauseMax=200` | 期望最大暂停（ms） | 软目标 |
| `-XX:ShenandoahMinFreeThreshold=10` | 最小空闲堆百分比 | 低于触发 GC |
| `-XX:ShenandoahAllocationThreshold` | 分配阈值 | 触发 GC 的分配量 |
| `-XX:+ShenandoahPacing` | 分配速率控制 | 减缓分配以跟上 GC |
| `-XX:ShenandoahParallelSafepointTimeout` | Safepoint 超时并行清理 | 减少 Safepoint 耗时 |

---

## JDK 命令行工具

### jps — 虚拟机进程状态

```
jps [-q] [-m] [-l] [-v] [hostid]
```

| 选项 | 作用 |
|:--|:--|
| `-q` | 只输出 LVMID |
| `-m` | 输出 main() 参数 |
| `-l` | 输出主类全名 / JAR 路径 |
| `-v` | 输出 JVM 启动参数 |

### jstat — 运行时状态监控

```
jstat -<option> <vmid> [<interval> [<count>]]
```

| 选项 | 监控内容 |
|:--|:--|
| `-class` | 类加载/卸载、耗时 |
| `-compiler` | JIT 编译信息 |
| `-printcompilation` | 已被即时编译的方法 |
| `-gc` | 各区域容量、使用量、GC 时间 |
| `-gccapacity` | 各区域最大/最小空间 |
| `-gcutil` | 各区域使用百分比 |
| `-gccause` | 使用百分比 + 最近一次 GC 原因 |
| `-gcnew` | 新生代 GC 情况 |
| `-gcnewcapacity` | 新生代最大/最小空间 |
| `-gcold` | 老年代 GC 情况 |
| `-gcoldcapacity` | 老年代最大/最小空间 |
| `-gcmetacapacity` | 元空间最大/最小 |

**输出示例：**

`jstat -gcutil <pid> 1s 3` — 各区域使用百分比，每秒一次共 3 次：

```
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00   4.20  55.30  23.10  95.20  92.80   123    1.520     0    0.000    1.520
  0.00   4.20  58.10  23.15  95.20  92.81   124    1.531     0    0.000    1.531
  0.00   4.20  60.45  23.20  95.21  92.82   125    1.542     0    0.000    1.542
```

列说明：S0/S1=Survivor 使用率，E=Eden，O=Old，M=Metaspace，CCS=压缩类空间，YGC=Young GC 次数，YGCT=YGC 耗时，FGC=Full GC 次数，FGCT=FGC 耗时，GCT=总 GC 耗时。

`jstat -gc <pid> 1s 2` — 各区域容量与使用量（KB）：

```
 S0C    S1C    S0U    S1U      EC       EU       OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
2048.0 2048.0  0.0   86.0  16384.0  9065.0   40960.0    9466.4   48640.0 46301.0 5120.0 4751.0   123    1.520     0    0.000    1.520
```

`jstat -gccause <pid> 1s` — 使用百分比 + GC 原因：

```
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT     LGCC                 GCC
  0.00   4.20  55.30  23.10  95.20  92.80   123    1.520     0    0.000    1.520  G1 Evacuation Pause   No GC
```

### jinfo — 查看/调整 JVM 参数

```
jinfo [option] <pid>
```

| 选项 | 作用 |
|:--|:--|
| `-flag <name>` | 查看指定参数值 |
| `-flag [+/-]<name>` | 开启/关闭布尔参数 |
| `-flag <name>=<value>` | 设置可写参数 |
| `-flags` | 打印所有 JVM 参数 |
| `-sysprops` | 打印系统属性 |

### jmap — 堆转储与统计

```
jmap [option] <vmid>
```

| 选项 | 作用 |
|:--|:--|
| `-heap` | 堆详细信息（收集器、配置、分代状态） |
| `-histo[:live]` | 对象统计（类、实例数、合计大小） |
| `-clstats` | 类加载器统计 |
| `-dump:[live,]format=b,file=<file>` | 生成 heap dump |
| `-F` | 强制生成 dump（`live` 参数下不支持） |
| `-finalizerinfo` | 等待 finalization 的对象 |

> ⚠️ `jmap -dump` 加 `live` 会触发 Full GC。生产建议用 `jcmd <pid> GC.heap_dump` 替代。

### jstack — 线程堆栈

```
jstack [-l] [-F] [-m] <vmid>
```

| 选项 | 作用 |
|:--|:--|
| `-l` | 输出锁的额外信息（拥有的锁、等待的锁） |
| `-F` | 强制输出堆栈（进程无响应时） |
| `-m` | 混合模式（含 native C/C++ 堆栈） |

**常用组合：**
```bash
# CPU 高排查四步
top -Hp <pid>                          # 1. 找到高 CPU 线程
printf "%x\n" <tid>                    # 2. TID 转 16 进制
jstack <pid> | grep -A20 <hex_tid>     # 3. 查看线程堆栈

# 线程状态分布统计
jstack <pid> | grep "java.lang.Thread.State" | sort | uniq -c | sort -rn

# 查找死锁
jstack <pid> | grep -A10 "deadlock"
```

### jcmd — 集成式工具箱（JDK 7+，推荐）

```
jcmd                     # 列出所有 Java 进程（等效 jps -lm）
jcmd <pid> help          # 列出该进程所有可用命令
```

| jcmd 命令 | 传统等效 | 说明 |
|:--|:--|:--|
| `jcmd <pid> VM.version` | — | JVM 版本信息 |
| `jcmd <pid> VM.uptime` | — | JVM 运行时长 |
| `jcmd <pid> VM.flags` | `jinfo -flags <pid>` | 所有 JVM 参数 |
| `jcmd <pid> VM.system_properties` | `jinfo -sysprops <pid>` | 系统属性 |
| `jcmd <pid> VM.command_line` | — | 启动命令行 |
| `jcmd <pid> GC.run` | — | 触发 Full GC |
| `jcmd <pid> GC.run_finalization` | — | 触发 finalization |
| `jcmd <pid> GC.heap_dump <file>` | `jmap -dump <pid>` | 生成 heap dump（推荐，不触发 Full GC） |
| `jcmd <pid> GC.class_histogram` | `jmap -histo <pid>` | 类实例统计 |
| `jcmd <pid> GC.class_stats` | — | 类元数据统计 |
| `jcmd <pid> Thread.print` | `jstack <pid>` | 线程堆栈 |
| `jcmd <pid> Thread.print -l` | `jstack -l <pid>` | 线程堆栈 + 锁信息 |
| `jcmd <pid> VM.native_memory summary` | — | NMT 概要（需开启 `-XX:NativeMemoryTracking`） |
| `jcmd <pid> VM.native_memory detail` | — | NMT 详情 |
| `jcmd <pid> VM.native_memory baseline` | — | NMT 基线 |
| `jcmd <pid> VM.native_memory summary.diff` | — | 与基线差异 |
| `jcmd <pid> JFR.start name=<n> duration=<t> filename=<f>` | — | 启动 JFR 录制 |
| `jcmd <pid> JFR.dump name=<n> filename=<f>` | — | 导出 JFR 数据 |
| `jcmd <pid> JFR.stop name=<n>` | — | 停止 JFR 录制 |
| `jcmd <pid> JFR.check` | — | 查看 JFR 录制状态 |
| `jcmd <pid> VM.info` | — | 系统与 JVM 信息 |
| `jcmd <pid> VM.classloader_stats` | — | 类加载器统计 |
| `jcmd <pid> VM.class_hierarchy` | — | 打印类层级 |
| `jcmd <pid> PerfCounter.print` | — | 性能计数器 |

**NMT 使用流程：**
```bash
# 启动时开启（detail 有 5–10% 性能损失）
-XX:NativeMemoryTracking=detail

# 查询
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory detail scale=MB

# 建基线 → 压测/运行 → 看差异定位泄漏
jcmd <pid> VM.native_memory baseline
jcmd <pid> VM.native_memory summary.diff
```

---

## GC 调优流程

### 1. 选收集器

| 场景 | 推荐 | 说明 |
|:--|:--|:--|
| 堆 < 2G，单核 | Serial | 极简无额外开销 |
| 批处理 / 离线，高吞吐优先 | **Parallel** | 可接受秒级暂停 |
| 通用服务端，2–32G 堆 | **G1** | JDK 9+ 默认，开箱即用 |
| 低延迟，> 16G 堆，暂停 < 10ms | **ZGC** | JDK 25 纯分代，全并发 |
| 低延迟，需多平台支持 | **Shenandoah** | JDK 25 分代正式 |
| 容器化微服务，内存敏感 | G1 / ZGC | ZGC 更快归还内存给 OS |

### 2. 各版本建议 JVM 配置

**JDK 8（CMS 或 G1）：**
```
# CMS（低延迟场景）
-Xms4g -Xmx4g -Xmn2g -Xss256k
-XX:+UseConcMarkSweepGC -XX:+UseParNewGC
-XX:CMSInitiatingOccupancyFraction=70 -XX:+UseCMSInitiatingOccupancyOnly
-XX:+CMSParallelRemarkEnabled -XX:+CMSScavengeBeforeRemark
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/data/logs/gc-%t.log
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M
-XX:+AlwaysPreTouch

# G1（JDK 8 中使用 G1 需显式开启）
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

**JDK 25（ZGC，低延迟场景）：**
```
-Xms16g -Xmx16g -Xss256k
-XX:+UseZGC -XX:SoftMaxHeapSize=12g
-XX:+UseCompactObjectHeaders
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/
-XX:ErrorFile=/data/logs/hs_err_%p.log
-Xlog:gc*=info:file=/data/logs/gc-%t.log:time,uptime,level,tags:filecount=5,filesize=20M
-XX:+AlwaysPreTouch
```

### 3. G1 调优路径

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

**常见 G1 日志诊断：**

| 日志关键词 | 含义 | 建议 |
|:--|:--|:--|
| `To-space exhausted` | 晋升失败，老年代不足 | 增大 `-Xmx` 或 `G1ReservePercent` |
| `Humongous allocation` | 大对象分配（≥ Region 一半） | 排查大数组/大 Buffer |
| `Full GC` | 并发周期回收不够 | 增大 IHOP 阈值或 `-Xmx` |
| `Concurrent Mark abort` | 并发标记中断（YGC 过多） | 适当降低 `MaxGCPauseMillis` |

### 4. ZGC 调优要点

- **核心只有堆大小**：留 20–30% 余量供并发回收期间的分配
- 容器环境用 `-XX:SoftMaxHeapSize`，低于 `-Xmx`
- 不需要设新生代大小、SurvivorRatio 等分代参数
- 目标暂停 < 1ms

### 5. Full GC 定位思路

```
出现 Full GC
  → 看 GC 日志 cause 确认是元空间还是堆不足
    → 元空间：增大 -XX:MaxMetaspaceSize，或排查类加载泄漏
    → 堆不足：
      → jcmd <pid> GC.class_histogram 看 top 对象
      → 大对象频繁分配 → 优化数据结构
      → 堆不够 → 增大 -Xmx
```

---

## 现网诊断调优实战

### 案例一：反射膨胀导致频繁 Full GC

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

### 案例二：大文件定时加载导致暂停飙升

**现象：** 定时任务加载大文件时 YGC 耗时从 20ms 飙升至 500ms+。

**原因：** 大文件对象直接入老年代，新生代中存活大对象导致复制算法耗时暴增。

**JDK 8 + CMS 方案：** `-XX:SurvivorRatio=65536` 去掉 Survivor，大对象一次 YGC 后直入老年代。

**JDK 17/25 + G1 方案：** 升级 G1，大对象（≥ Region 一半）直接分配 Humongous Region，不参与新生代复制。

**JDK 25 + ZGC 方案：** 全并发回收，不受大对象数量和大小影响。

---

### 案例三：Safepoint 导致长时间停顿

**现象：** GC 日志中 GC 耗时（real）正常，但 `user + sys` 远大于 real——大量时间花在等线程进安全点。

**排查：**
```bash
# JDK 8
-XX:+PrintGCApplicationStoppedTime -XX:+PrintGCApplicationConcurrentTime
-XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1

# 定位超时线程
-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=2000
```

**原因：** 可数循环（索引 `int` 或更小类型）不在循环头插入安全点检查，单次循环耗时长会拖慢全部线程。

**方案：** 循环索引从 `int` 改为 `long`，编译器在循环头插入安全点检查。

---

### 案例四：CPU 飙升排查

**常见原因：** 死循环、频繁 GC、大量线程调度、外部进程调用。

```bash
top -c                               # 1. 找到高 CPU 进程
top -Hp <pid>                        # 2. 找到高 CPU 线程
printf "%x\n" <tid>                  # 3. TID 转 16 进制
# 4. 连续采样确认
for i in 1 2 3 4 5; do jstack <pid> | grep -A20 <hex_tid>; sleep 1; done
# 5. 线程状态分布
jstack <pid> | grep "java.lang.Thread.State" | sort | uniq -c | sort -rn
```

**快速判断：**
- GC 线程 CPU 高 → 堆太小或内存泄漏，看 GC 日志
- 业务线程 CPU 高 → 死循环/算法问题，看 jstack
- C2 Compiler 线程高 → JIT 压力大，正常但持续过久需排查

---

### 案例五：大内存硬件 CMS 退化

**现象：** 内存 16G → 64G 升级后，Full GC 停顿反而成倍增加。

**原因（JDK 8 + CMS）：** 大对象直入老年代，CMS 无法并发回收，退化为单线程 Serial Old Full GC，堆越大暂停越久。

**方案：**
- JDK 8：拆分为多个小 JVM 实例，每个单独用 CMS
- JDK 17/25：**直接升 ZGC**，全并发回收，暂停与堆大小无关

---

## 参考

- [JDK 25 G1/Parallel/Serial GC Changes — Thomas Schatzl](https://tschatzl.github.io/2025/08/12/jdk25-g1-serial-parallel-gc-changes.html)
- [Oracle JDK 25 Release Notes](https://www.oracle.com/java/technologies/javase/25-relnote-issues.html)
- [Java 25 vs Java 21 Upgrade Guide](https://www.javacodegeeks.com/2026/04/java-25-vs-java-21-the-upgrade-guide-nobodyhas-written-yet.html)
- [JVM Garbage Collectors Guide](https://marciniak.cloud/posts/jvm-garbage-collectors-guide/)
- [HotSpot GC Tuning Guide (JDK 25)](https://docs.oracle.com/en/java/javase/25/gctuning/)
