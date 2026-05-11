## JVM 垃圾回收：算法与收集器演进

垃圾收集（Garbage Collection，GC）是 JVM 最核心的自动内存管理机制。对于资深后端工程师来说，理解 GC 不是"背参数"，而是理解它在你系统里的行为模式——什么时候 STW、什么时候并发、为什么线上突然 Full GC、选择 G1 还是 ZGC。

本文从对象存活判断出发，逐层深入 GC 算法、HotSpot 实现细节、七种收集器的设计与适用场景，最终落到生产环境的故障排查和选型决策。全文聚焦 HotSpot VM，不讨论 OpenJ9 等非主流实现。

阅读前提：本文假设读者已了解 JVM 运行时数据区的基本划分（堆的分代结构、方法区/元空间、虚拟机栈等），可参考同系列《JVM 运行时数据区与对象内存布局》。

***

## 对象存活判断

GC 的第一步是找出"哪些对象还活着"。垃圾收集器不会无缘无故清理对象——它需要一个精确的存活判定机制。

### 引用计数算法

引用计数（Reference Counting）的思路非常直观：在每个对象上维护一个计数器，记录有多少个引用指向它。计数器归零时，对象即判定为死亡。

**优点**：实现简单，判定效率高，不需要暂停用户线程。

**缺点**：致命缺陷——无法解决循环引用问题。两个对象互相引用对方，即使外部没有任何引用指向它们，计数器也永远不会归零，导致内存泄漏。这是 Java 虚拟机不采用引用计数算法的根本原因。

```java
Obj a = new Obj();  // a 的计数器 = 1
Obj b = new Obj();  // b 的计数器 = 1
a.ref = b;          // b 的计数器 = 2
b.ref = a;          // a 的计数器 = 2
a = null;           // a 的计数器 = 1
b = null;           // b 的计数器 = 1
// a 和 b 永远无法被回收——这就是循环引用问题
```

### 可达性分析算法

JVM 主流实现全部采用**可达性分析**（Reachability Analysis）：从一组称为 **GC Roots** 的根对象出发，沿着引用链向下搜索，不可达的对象即判定为可回收。

Java 中可作为 GC Roots 的对象包括：

| 类别 | 说明 | 实战关注点 |
|------|------|-----------|
| 虚拟机栈引用 | 栈帧中局部变量表引用的对象 | 最常见，方法执行期间阻止对象回收 |
| 静态属性引用 | 方法区中类的静态变量引用的对象 | 静态集合（如 static List）是内存泄漏重灾区 |
| 常量引用 | 运行时常量池中的引用（如字符串常量池） | `String.intern()` 引入的引用也属于此类 |
| JNI 引用 | Native 方法栈中引用的对象 | Native 代码泄漏排查困难，需借助 Native 工具 |
| 虚拟机内部引用 | Class 对象、常驻异常对象、系统类加载器等 | 几乎不参与应用层 GC 分析 |
| 同步锁持有对象 | 被 `synchronized` 锁持有的对象 | 死锁导致对象无法释放的场景 |

> GC Roots 的选择体现了"宁可漏杀不可错杀"的原则——误把存活对象回收会导致 JVM 崩溃，所以 GC Roots 范围倾向于保守，宁可多纳入一些引用。

### finalize() 的两次标记过程

一个对象被判定为不可达后，并不会立即被回收。JVM 会执行一个**两次标记**的过程：

1. **第一次标记与筛选**：对象经过可达性分析发现没有到 GC Roots 的引用链时，被第一次标记。随后进行一次筛选——如果对象没有覆盖 `finalize()` 方法，或 `finalize()` 已经被虚拟机调用过，则直接进入"即将回收"集合。

2. **第二次标记**：筛选通过的对象会被放入一个名为 **F-Queue** 的低优先级队列中，由一条专门的 Finalizer 线程去执行其 `finalize()` 方法。GC 之后会对 F-Queue 中的对象进行第二次标记——如果在 `finalize()` 中对象重新与 GC Roots 建立了关联（比如将自己赋给某个静态变量），则移出"即将回收"集合。

### 为什么 finalize() 不推荐使用

`finalize()` 在 Java 设计之初就被定位为 C++ 析构函数的替代方案，但在实际生产中几乎没有合理的使用场景：

- **执行时机不确定**：`finalize()` 的执行完全由 GC 线程调度，应用代码无法控制其执行时机。对象标记为可回收后，可能需要经历多次 GC 才能真正执行 `finalize()`，甚至在整个 JVM 生命周期内都可能不被执行。
- **性能代价高**：每个覆盖了 `finalize()` 的对象都需要额外的 GC 处理流程——Finalizer 线程的处理速度远慢于 GC 回收速度，当大量对象等待 `finalize()` 时会严重拖慢 GC。
- **可靠性无法保证**：若 `finalize()` 中抛出未捕获的异常，Finalizer 线程会忽略该异常继续处理下一个对象，没有任何日志输出。你在日志中甚至看不到它失败过。
- **资源释放的正确方式**：对于外部资源（文件句柄、网络连接、Native 内存），使用 `try-with-resources` 或显式的 `close()` 方法才是可靠的做法。

> JDK 9 中 `finalize()` 已被标记为 `@Deprecated`，JDK 18 标记为 `@Deprecated(forRemoval = true)`。如果你的代码或依赖库中还在使用 `finalize()`，现在是时候迁移了。

### 内存泄漏的常见模式

内存泄漏不是"内存被吃光了"，而是"不再需要的对象无法被回收"。以下是生产环境中最常见的三种模式：

**静态集合持有引用**：静态字段的生命周期与类相同，意味着只要类未被卸载，其引用的对象就永远不会被回收。

```java
// 泄漏模式：静态 Map 持续放入对象，永不清理
static Map<String, Object> cache = new HashMap<>();
// 每次请求都放入新的缓存对象，缓存的 key 却从不失效
```

排查线索：通过 MAT（Eclipse Memory Analyzer Tool）分析 heap dump，找到静态字段的引用链。

**ThreadLocal 未清理**：线程池中的线程不会销毁，ThreadLocal 的值如果不调用 `remove()`，就会在线程的生命周期内持续存活，同时引用着 ThreadLocalMap 中的 Entry。虽然 JDK 8 之后 Entry 使用弱引用指向 ThreadLocal 对象本身，但 Entry 的 value 仍然是强引用，只要线程不死，value 就不会被回收。

```java
// 泄漏模式：线程池 + ThreadLocal 未 remove
ThreadLocal<HeavyObject> local = new ThreadLocal<>();
// ... 使用后忘记调用 local.remove()
```

排查线索：`jmap -histo:live <pid>` 查看 ThreadLocal 相关对象的数量是否异常增长。

**监听器/回调未移除**：将对象注册为监听器后，事件源会持有该对象的引用。如果对象不再需要而忘记注销，就会泄漏。这在 GUI 和事件驱动架构中尤为常见。

```java
// 泄漏模式：注册监听器后未在适当时机移除
eventBus.register(this);  // 事件总线持有 this
// 对象生命周期结束时忘记调用 eventBus.unregister(this)
```

***

## 回收方法区

方法区的垃圾收集效果通常不如堆，但了解其机制对排查元空间 OOM 和类加载器泄漏至关重要。

### 废弃常量的回收

方法区中的常量池（特别是运行时常量池）在满足以下条件时可以回收常量：

- 没有任何地方引用这个常量。例如，字符串常量池中的某个字面量——如果没有任何 `String` 对象引用该字面量，且虚拟机中没有任何地方使用它，该常量即可被回收。

常量的回收类似于堆中对象的回收，判断逻辑简单。真正复杂的在于**类型卸载**。

### 类卸载的三个条件

一个类的 Class 对象被回收需要同时满足以下三个苛刻条件：

1. **该类的所有实例都已被回收**：Java 堆中不存在该类及其子类的任何实例对象。

2. **加载该类的类加载器已经被回收**：这是最难满足的条件，因为类加载器本身也作为一个对象存在于堆中。除非是用户自定义的类加载器且在特定场景下被明确弃用（如 OSGi 容器中的模块卸载、JSP 热部署），否则引导类加载器、扩展类加载器和系统类加载器永远不会被回收。

3. **该类的 Class 对象没有在任何地方被引用**：无法通过反射访问到该类的 Class 对象。

> 在实际生产环境中，类卸载最常见的触发场景是大量使用动态代理、CGLIB、字节码增强（如 Spring AOP、Hibernate 实体代理）且类加载器被反复创建和销毁的场景。如果在 JFR（JDK Flight Recorder）中观察到类卸载事件频繁但元空间依然持续增长，说明类卸载的条件未能同时满足——通常是类加载器泄漏。

元空间的垃圾收集可以通过参数精细控制：

| 参数 | 作用 | 实战建议 |
|------|------|---------|
| `-XX:MaxMetaspaceSize` | 元空间上限 | 生产环境必须设置，防止 Native 内存耗尽 |
| `-XX:MetaspaceSize` | 触发首次元空间 GC 的阈值 | 不宜过小，否则频繁 GC；不宜过大，延迟首次 GC |
| `-XX:MinMetaspaceFreeRatio` | GC 后最小空闲比例 | 与 `MaxMetaspaceFreeRatio` 配合控制元空间伸缩 |
| `-XX:+TraceClassUnloading` | 记录类卸载日志 | 排查类加载器泄漏时的关键开关 |

***

## 垃圾收集算法

理解 GC 算法的本质是理解"在什么条件下用什么方式回收内存"，以及"每种方式的优缺点如何影响生产行为"。

### 分代收集理论

现代 JVM 的 GC 设计建立在两个**弱分代假设**之上：

- **绝大多数对象都是朝生夕灭的**：方法中的临时变量、循环中的中间对象、请求上下文中的短生命周期对象，大多数在创建后很快就不再使用。
- **熬过越多次 GC 的对象就越难以消亡**：经过多次 GC 仍然存活的对象，往往是缓存、连接池、单例等长生命周期对象。

基于这两个假设，JVM 将堆划分为**新生代**（Young Generation）和**老年代**（Old Generation），不同代采用不同的收集策略：

- **新生代**：对象死亡率高，使用**标记-复制**算法，只需复制少量存活对象，效率高。
- **老年代**：对象存活率高，使用**标记-清除**或**标记-整理**算法，避免频繁复制开销。

按照回收范围，GC 可分为以下类型：

| 类型 | 回收范围 | 触发场景 | 特点 |
|------|---------|---------|------|
| Minor GC / Young GC | 仅新生代 | Eden 区满 | 频繁，速度快，STW 时间短 |
| Major GC / Old GC | 仅老年代 | 老年代空间不足 | **仅 CMS 有此行为**，其他收集器无单独的 Old GC |
| Mixed GC | 新生代 + 部分老年代 | G1 的垃圾占比达到阈值 | **仅 G1 有此行为** |
| Full GC | 整个堆 + 方法区 | 空间分配担保失败、System.gc() 等 | STW 时间长，应尽量避免 |

### 标记-清除算法

标记-清除（Mark-Sweep）是最基础的 GC 算法，分为两个阶段：

1. **标记**：从 GC Roots 出发，标记所有可达的存活对象。
2. **清除**：遍历整个堆，回收所有未被标记的对象。

**主要缺陷**：

- **执行效率不稳定**：如果堆中存活对象多，标记阶段耗时长；如果可回收对象多，清除阶段耗时长。两个阶段的执行时间都随堆大小和对象数量线性增长。
- **内存空间碎片化**：清除后产生大量不连续的内存碎片。当需要分配一个大对象时，即使堆中总体空闲空间足够，也可能因为找不到连续空间而提前触发下一次 GC。

> 在实际生产中，标记-清除导致的碎片化问题在 CMS 收集器上表现最为突出。当老年代碎片严重到无法分配一个"并不算大"的对象时，JVM 会触发一次 Full GC 来整理内存碎片——这次 Full GC 的 STW 时间可能长达数秒甚至数十秒。

### 标记-复制算法

标记-复制（Mark-Copy）将可用内存划分为大小相等的两块，每次只使用其中一块。当这一块用完时，将存活的对象复制到另一块上，然后一次性清理整个已使用的半区。

**优点**：
- **实现简单高效**：只需复制少数存活对象，内存分配时直接使用指针碰撞（Bump the Pointer），无需考虑碎片问题。
- **适合新生代**：新生代中 98% 的对象在第一次 Minor GC 时就死亡，只需复制极少量存活对象。

**缺点**：
- **空间浪费**：始终有一半内存处于闲置状态。HotSpot 的做法是在新生代中分配一个较大的 Eden 区和两个较小的 Survivor 区（默认比例 8:1:1），以 10% 的空间浪费换取高效回收。

> 在实际生产中，标记-复制带来的空间浪费是可控的——因为新生代只占整个堆的一小部分（通常 1/3 到 1/2），10% 的 Survivor 空间浪费相对于整体堆内存来说微不足道。但如果 Survivor 空间不足以容纳所有存活对象，就需要依赖**空间分配担保**机制将多余对象直接送入老年代——这也是"过早晋升"（Premature Promotion）问题的根源。

### 标记-整理算法

标记-整理（Mark-Compact）在标记-清除的基础上增加了一步：标记完成后，将所有存活对象向内存一端移动，然后直接清理边界外的所有内存。

**优点**：消除了内存碎片，内存分配回归指针碰撞模式，效率最高。

**缺点**：移动存活对象并更新所有引用（对象内部的指针、虚拟机栈中的引用、静态字段的引用等）必须在 STW 阶段完成——用户线程不能在使用旧地址的同时让 GC 移动对象。这意味着标记-整理的 STW 时间比标记-清除更长。

> 这里有一个经典的吞吐量 vs 延迟的权衡：标记-清除不移动对象，STW 时间短，但内存碎片化导致分配效率低，整体吞吐量下降；标记-整理移动对象，STW 时间长，但内存连续分配效率高，整体吞吐量更高。HotSpot 中 Parallel Scavenge 选择标记-整理来追求吞吐量，CMS 选择标记-清除来追求低延迟，而 G1 和 ZGC 则试图在两者之间找到新的平衡点。

***

## HotSpot 实现细节

GC 算法是理论，HotSpot 的实现细节才是生产环境中影响 STW 时间的真正因素。

### 根节点枚举与 OopMap

GC 的第一步是找到所有 GC Roots，这个过程称为**根节点枚举**（Root Scanning）。在早期的简单实现中，需要遍历所有线程的栈帧，逐一检查每个槽位是否是一个对象引用——这会非常耗时。

HotSpot 使用 **OopMap**（Ordinary Object Pointer Map）来加速：在 JIT 编译生成的代码中，HotSpot 会在特定位置（安全点）记录下当时栈和寄存器中哪些位置存放着对象引用。这样 GC 发生时，只需读取对应安全点的 OopMap 即可快速定位所有根引用，无需逐个排查。

### 安全点与安全区域

**安全点**（Safepoint）是 HotSpot 中唯一可以生成 OopMap 的位置——只有程序执行到安全点时，HotSpot 才能安全地暂停所有线程进行 GC。安全点通常设置在：

- 方法调用之前
- 循环跳转的后沿（循环末尾，即下一次循环开始前）
- 异常跳转的位置

HotSpot 使用**主动式中断**策略：当需要暂停用户线程时，设置一个全局标志位，各个线程在执行过程中主动轮询该标志，发现需要中断时就挂起自己。在 x86 平台上，这通过一条 `test` 指令（读取内存页）实现——只需将标志位所在的内存页设为不可读，线程执行到该指令时就会产生自陷信号，进入异常处理器挂起。

> **安全点过长是 STW 抖动的主要原因之一。** 如果一个线程正在执行一个长循环（例如 `for (int i = 0; i < Integer.MAX_VALUE; i++)`），HotSpot 可能为了性能不会在循环内部插入安全点——这意味着该线程需要跑完整个循环才能到达安全点，导致所有其他线程都处于等待状态。排查方法是通过 `-XX:+PrintSafepointStatistics` 查看安全点操作的时间分布。

**安全区域**（Safe Region）解决的是安全点机制的盲区：处于 Sleep 或 Blocked 状态的线程无法主动轮询安全点标志。安全区域通过"进入-离开"协议解决——线程进入安全区域时标记自己已进入，离开时检查 GC 是否已完成。如果 GC 尚未完成，线程必须等待。

### 记忆集与卡表

在分代收集的架构下，新生代 GC（Minor GC）只需要扫描新生代中的对象。但这里有一个**跨代引用**问题：老年代中的对象可能引用了新生代中的对象。如果每次 Minor GC 都要扫描整个老年代来找这些引用，就完全丧失了分代收集的优势。

**记忆集**（Remembered Set）正是为此而设计：它是一种从非收集区域指向收集区域的指针集合的抽象数据结构。对于 Minor GC 来说，记忆集记录了老年代中哪些区域存在指向新生代的引用。

**卡表**（Card Table）是记忆集最常用的实现方式。HotSpot 将老年代划分为若干 512 字节大小的**卡页**（Card Page），用一个字节数组实现卡表——每个字节对应一个卡页，如果卡页中存在跨代引用，则对应字节标记为"脏"（Dirty）。Minor GC 时，只需扫描脏卡页中对应的老年代对象即可。

> 卡表维护的开销来自**写屏障**：每次引用类型字段赋值时，JIT 编译器会插入一段额外代码（写后屏障），将赋值涉及的卡页标记为脏。在高并发写入场景下，大量写屏障操作可能造成性能开销——这就是"伪共享"（False Sharing）问题在 GC 中的体现：多个线程并发修改相邻的对象引用时，会频繁更新同一张卡表中的相邻字节，导致 CPU 缓存行失效。

### 三色标记法

并发标记阶段需要一边标记一边应对用户线程修改引用关系。主流 GC 算法使用**三色标记**（Tri-color Marking）来追踪标记进度：

| 颜色 | 含义 | 状态转换 |
|------|------|---------|
| 白色 | 未访问，可达性分析结束时仍为白色的对象不可达 | 初始状态 → 被访问后变灰 |
| 灰色 | 已访问但引用未完全扫描 | 被访问后变灰 → 引用全部扫描完变黑 |
| 黑色 | 已访问且所有引用都已扫描，安全存活 | 不再转换 |

可达性分析结束时，不应该存在灰色对象——如果有，说明标记过程尚未完成。

### 漏标问题与解决方案

并发标记中可能出现**漏标**：由于用户线程在标记期间修改了引用关系，导致一个本来应该是黑色的对象被错误地标记为白色。漏标需要同时满足两个条件：

1. 插入了一条或多条从黑色对象到白色对象的引用（新增引用）。
2. 删除了全部从灰色对象到该白色对象的直接或间接引用（删除保护）。

两种解决方案：

**增量更新**（Incremental Update）：破坏条件 1。当黑色对象插入指向白色对象的新引用时，将该黑色对象重新标记为灰色，在重新标记阶段再次扫描。CMS 采用此方案，使用**写后屏障**实现。

**原始快照**（SATB，Snapshot At The Beginning）：破坏条件 2。当灰色对象删除指向白色对象的引用时，将该引用记录下来，在最终标记阶段以记录中的灰色对象为根重新扫描。G1 和 Shenandoah 采用此方案，使用**写前屏障**实现。

> SATB 的设计哲学是"宁可错杀不可放过"——它按照 GC 开始时刻的对象快照来判定存活，即使某个对象在 GC 过程中变成了垃圾，SATB 也会将其视为存活对象，留到下一次 GC 再处理。这会产生一些"浮动垃圾"（Floating Garbage），但换来了并发标记的正确性保证。在 G1 中，浮动垃圾会在下一次 Mixed GC 中被回收；而在 CMS 中，浮动垃圾只能等待下一次 CMS 周期——这恰恰是 CMS 需要预留内存空间、容易触发 Concurrent Mode Failure 的原因之一。

***

## 经典收集器

HotSpot 内置了多种垃圾收集器，每种收集器在延迟、吞吐量和内存占用之间做出不同的权衡。理解每种收集器的工作原理和适用场景，是生产环境 GC 调优的基础。

在介绍具体收集器之前，先明确两个基本概念：

- **并行**（Parallel）：多条 GC 线程同时工作，但用户线程处于等待状态（STW 期间）。
- **并发**（Concurrent）：GC 线程与用户线程同时运行，用户线程不会完全停顿。

### Serial / Serial Old

**Serial** 是最古老、最简单的收集器——新生代，单线程，标记-复制算法，收集期间必须暂停所有用户线程。

**Serial Old** 是 Serial 的老年代版本，单线程，标记-整理算法。它最主要的"存在意义"是作为 CMS 在 Concurrent Mode Failure 时的后备方案，以及配合 Serial 用于 Client 模式下的桌面应用。

适用场景：
- 单核处理器或小内存场景（< 100 MB 堆）
- 桌面应用（Swing GUI 等）
- Docker 容器中资源极度受限的微型服务

> 在 2024 年的服务器端，Serial 几乎不再使用，除非你在 256 MB 内存的单核 VPS 上运行一个迷你 Java 应用。但在某些嵌入式场景（如 ARM 设备）中，Serial 的简单性仍有价值。

开启参数：`-XX:+UseSerialGC`（同时启用 Serial + Serial Old）。

### ParNew

**ParNew** 是 Serial 的**多线程并行版本**，新生代，标记-复制算法。它与 Serial 的核心区别是使用多条线程并行收集，默认线程数与 CPU 核心数相同（可通过 `-XX:ParallelGCThreads` 调整）。

ParNew 在单核处理器下效率不如 Serial（线程切换开销），但在多核服务器上能显著缩短新生代 GC 的 STW 时间。

值得注意的是：ParNew **只能与 CMS 配合使用**，不能与 Parallel Old 配合。如果使用 `-XX:+UseConcMarkSweepGC`，新生代默认使用 ParNew。

ParNew 的存在源于历史原因——它是唯一能与 CMS 搭配的新生代并行收集器。JDK 9 之后 CMS 被废弃，ParNew 也随之淡出。

### Parallel Scavenge / Parallel Old

**Parallel Scavenge** 是新生代**并行**收集器，标记-复制算法。**它的设计目标是最大化吞吐量**（Throughput），而非最小化停顿时间。

吞吐量 = 用户代码运行时间 /（用户代码运行时间 + GC 运行时间）。例如，`-XX:GCTimeRatio=99` 表示目标吞吐量为 99%（即 GC 时间不超过总时间的 1%）。

Parallel Scavenge 的核心特性是**自适应调节策略**（`-XX:+UseAdaptiveSizePolicy`，默认开启）：只需设置基本的内存参数和优化目标（最大停顿时间 `-XX:MaxGCPauseMillis` 或吞吐量 `-XX:GCTimeRatio`），JVM 会自动动态调整新生代大小、Eden/Survivor 比例、晋升老年代年龄等参数。

**Parallel Old** 是 Parallel Scavenge 的老年代配套，并行，标记-整理算法。

适用场景：
- 批处理、科学计算等对吞吐量要求高、对停顿不太敏感的场景
- 后台离线任务（数据 ETL、报表生成、定时任务）

> 这是 JDK 8 及之前的服务端默认收集器组合。在后端服务中，如果系统是计算密集型而非延迟敏感型，Parallel 仍然是很好的选择——最大吞吐量意味着在单位时间内完成最多的业务处理，批处理任务的价值远大于微秒级的延迟差异。

开启参数：`-XX:+UseParallelGC`（同时启用 Parallel Scavenge + Parallel Old）。

### CMS

**CMS（Concurrent Mark Sweep）** 是老年代**并发**收集器，标记-清除算法，**目标是最短回收停顿时间**。它是 JDK 5 到 JDK 8 期间低延迟场景的主流选择。

CMS 的收集过程分为四个阶段：

| 阶段 | 是否 STW | 工作内容 | 耗时特征 |
|------|---------|---------|---------|
| 初始标记 | 是 | 标记 GC Roots 直接关联的对象 | 非常短（毫秒级），单线程 |
| 并发标记 | 否 | 从 GC Roots 出发遍历整个对象图 | 最长，多线程，与用户线程并发 |
| 重新标记 | 是 | 修正并发标记期间变动的引用（增量更新） | 较短，多线程并行 |
| 并发清除 | 否 | 清除死亡对象 | 较长，多线程，与用户线程并发 |

CMS 的默认回收线程数 =（CPU 核心数 + 3）/ 4。在 4 核机器上只使用 1 条线程，在 8 核机器上使用 2 条线程——在核心数较少的机器上，CMS 的并发线程会明显挤占用户线程的 CPU 资源，降低吞吐量。

**CMS 的三大致命缺陷：**

**1. Concurrent Mode Failure**：并发标记和并发清除期间，用户线程仍在产生新对象。如果老年代在 CMS 回收完成前就被填满，JVM 会冻结所有用户线程，启动 Serial Old 进行单线程 Full GC，导致长达数秒甚至数十秒的停顿。

触发条件：老年代使用率超过阈值（由 `-XX:CMSInitiatingOccupancyFraction` 设定（JDK 6 早期曾为 92%，后因生产频繁 Concurrent Mode Failure，在 JDK 6u23 中调回约 68%））。

排查方法：
- 检查 GC 日志中是否出现 "Concurrent Mode Failure" 或 "concurrent mode failure"。
- 适当降低 `CMSInitiatingOccupancyFraction`（如调整为 70-80），让 CMS 更早启动，给并发回收留出更多缓冲空间。
- 增加老年代大小，从根本上延缓填满时间。
- 若堆内存已足够大，考虑切换到 G1 或 ZGC。

**2. Promotion Failed**：Minor GC 时，Survivor 空间放不下存活对象，需要向老年代"晋升"，但老年代碎片化严重，找不到足够连续空间。JVM 同样会触发 Full GC。

排查方法：
- GC 日志中出现 "promotion failed"。
- 增大 Survivor 区大小或调整 `-XX:SurvivorRatio`。
- 检查是否存在对象过早晋升（Premature Promotion），即大量短生命周期对象因 Survivor 区太小被迫进入老年代。

**3. 内存碎片化**：CMS 只做标记-清除，不做整理。随着时间推移，老年代碎片会越来越严重，最终即使总体空闲空间足够，也分配不了稍大的对象，触发 Full GC。可以通过 `-XX:+UseCMSCompactAtFullCollection`（Full GC 后整理）和 `-XX:CMSFullGCsBeforeCompaction`（N 次 Full GC 后整理一次）来缓解，但这只是弥补手段，无法根治。

> CMS 最痛苦的场景是：系统平稳运行了很长时间，老年代碎片慢慢积累，某天因为没有连续空间分配一个"中等大小"的对象，触发 Full GC，导致整个服务停顿数十秒，所有请求超时，客户告警。这种"突袭式"的停顿是 CMS 在生产环境中最令人头疼的问题。

**CMS 已被废弃**：JDK 9 中标记为不推荐（deprecated），JDK 14 中被正式移除。如果你的系统还在使用 CMS，现在就应该开始向 G1 迁移。

***

## G1 收集器

G1（Garbage First）是对 CMS 的"响应式替代"——它不是等老年代快满了才触发 GC，而是基于**停顿时间模型**主动选择回收价值最高的 Region。JDK 9 起，G1 成为服务端默认收集器。

### Region 化内存布局

G1 最大的创新是抛弃了固定大小的新生代/老年代分区，将整个堆划分为多个大小相等的 **Region**（通过 `-XX:G1HeapRegionSize` 设定，取值 1~32 MB，必须是 2 的 N 次幂）。每个 Region 可以动态扮演 Eden、Survivor、Old 或 **Humongous**（巨大对象）角色。

Humongous Region 用于存放大小超过 Region 一半的**巨大对象**。如果对象超过一个 Region 的容量，会占用 N 个连续的 Region。Humongous 对象直接被视作老年代的一部分，**不会在新生代中被复制**。

> **生产实践：G1 的巨大对象问题。** 如果一个 Region 是 4 MB，那么任何超过 2 MB 的对象都是巨大对象。当你频繁创建 2.5 MB 的 byte 数组时，G1 会不断分配 Humongous Region，导致老年代空间快速增长，甚至触发 Full GC。排查方法：通过 GC 日志中的 "Humongous" 关键字搜索，或使用 `-XX:+PrintAdaptiveSizePolicy` 观察 Region 分布。如果确认是巨大对象导致的，考虑增大 `-XX:G1HeapRegionSize`（如从 4 MB 调整到 16 MB），或从应用层面控制大对象的创建频率。

### Mixed GC 与停顿预测

G1 的核心思想是**面向回收集的局部收集**：跟踪每个 Region 中垃圾堆积的"价值"（回收收益），维护一个优先级列表，每次根据用户设定的停顿时间目标（`-XX:MaxGCPauseMillis`，默认 200 ms），优先回收收益最大的那些 Region。

G1 的收集周期包含四个阶段：

| 阶段 | STW | 工作内容 |
|------|-----|---------|
| 初始标记 | 是 | 标记 GC Roots 直接关联对象，借用 Minor GC 的 STW 完成 |
| 并发标记 | 否 | 遍历对象图，使用 SATB 处理引用变更，计算每个 Region 的存活对象占比 |
| 最终标记 | 是 | 处理 SATB 剩余的少量记录 |
| 筛选回收 | 是 | 并行复制存活对象到空 Region，回收旧 Region |

**筛选回收阶段就是 Mixed GC**——它会回收所有新生代 Region，同时选择回收收益最高的若干个老年代 Region。G1 的核心价值在于：**它用可控的短停顿（Mixed GC）替代了不可控的长停顿（Full GC）**。

> **停顿时间的"软目标"**：G1 的停顿时间目标是尽力而为的（best-effort），不是硬保证。如果 Region 中存活对象过多，单次 Mixed GC 的停顿时间可能远超目标值。调优的关键是设置合理的目标值——一般建议 100~300 ms。设置过低（如 10 ms）会导致 G1 每次只回收极少量 Region，老年代清理速度跟不上产生速度，最终触发 Full GC。

### SATB 与并发标记

G1 使用 **SATB（Snapshot At The Beginning）** 解决并发标记中的漏标问题：在并发标记开始时，G1 对整个对象图拍一张"逻辑快照"。在标记期间，如果用户线程删除了一条从灰色对象到白色对象的引用，SATB 通过写前屏障将这个删除操作记录下来，最终标记阶段以这些记录中的对象为根重新扫描。

SATB 的代价是**浮动垃圾**：在并发标记期间新产生的垃圾（原本可达、标记过程中变成不可达的对象）会被 SATB 当作存活对象保留到下一次 GC。这是以"少量浮动垃圾"换取"并发标记安全"的经典设计权衡。

### 实战：G1 的常见问题

**Evacuation Failure（转移失败）**：Mixed GC 的筛选回收阶段需要将存活对象复制到新的空 Region。如果没有足够的空 Region 来容纳复制后的对象，G1 会后端将剩余存活对象留在原 Region 中——本次 Mixed GC 无法回收这些 Region，下次再试。频繁的 Evacuation Failure 会导致 GC 效率急剧下降。

排查方法：GC 日志中出现 "to-space exhausted" 或 "Evacuation Failure"。解决方案包括增大堆内存、增大 `-XX:G1ReservePercent`（预留空间比例，默认 10%）、或适当降低 `-XX:InitiatingHeapOccupancyPercent`（触发并发标记的堆占用阈值，默认 45%）。

**Full GC 的触发**：G1 的设计理想是不触发 Full GC，但以下情况仍会导致：

- Mixed GC 回收速度跟不上对象分配速度，老年代逐步被填满。
- 巨大对象频繁分配，Humongous Region 无法被正常回收。
- `System.gc()` 显式调用（可通过 `-XX:+DisableExplicitGC` 禁止）。

***

## 低延迟收集器

当你的系统要求 GC 停顿时间低于 10 ms 时，经典收集器和 G1 都无法满足。**ZGC** 和 **Shenandoah** 正是为此而生——它们的目标是在任意堆大小下，将 STW 停顿控制在毫秒级。

### ZGC

ZGC（The Z Garbage Collector）从 JDK 11 开始实验性引入，**JDK 15 正式发布为 Production 级别**。它支持 TB 级堆（理论上 16 TB，实际 4 TB 已有验证），目标是将停顿时间控制在 **10 ms 以内**，且停顿时间不随堆大小或存活对象数量增长。

JDK 版本演进：

| JDK 版本 | ZGC 状态 | 关键变化 |
|----------|---------|---------|
| JDK 11 | 实验性（Experimental） | 仅支持 Linux/x64 |
| JDK 13 | 实验性增强 | 最大堆从 4 TB 扩展到 16 TB，支持主动返还未使用内存给操作系统 |
| JDK 14 | 实验性扩展 | 支持 macOS、Windows |
| JDK 15 | **正式发布（Production）** | 生产可用，Windows 和 macOS 仍有限制 |
| JDK 16 | 增强 | 支持并发线程栈扫描（减少根节点枚举的 STW） |
| JDK 17 LTS | 稳定 | 长期支持版本，生产推荐 |
| JDK 21 LTS | **Generational ZGC** | 分代 ZGC，进一步提升吞吐量 |

> JDK 17 是 ZGC 作为 LTS 的第一个版本，也是目前（2024-2025）生产环境部署 ZGC 最广泛使用的版本。建议在 JDK 17+ 上使用 ZGC。

#### 核心技术：染色指针

ZGC 的核心技术是**染色指针**（Colored Pointers）——在 64 位指针中，只有低 48 位用于寻址，ZGC 将其中的高 4 位用于存储元数据：

| 标志位 | 含义 |
|--------|------|
| Marked 0 / Marked 1 | 三色标记状态，交替使用 |
| Remapped | 对象是否已从重分配集复制到新地址 |
| Finalizable | 对象只能通过 `finalize()` 方法访问 |

使用染色指针后，ZGC 可管理的最大内存为 16 TB（44 位地址空间），且**不再支持指针压缩**（Compressed Oops）。这意味着在堆小于 32 GB 的场景下，ZGC 的内存占用会略高于支持指针压缩的收集器。

#### 并发整理与指针自愈

ZGC 的"并发整理"能力来自于染色指针与**读屏障**的配合：

1. 并发重分配阶段，ZGC 将存活对象复制到新的 Region。
2. 旧对象的染色指针被标记为 Remapped，同时维护一个**转发表**（Forwarding Table）记录新旧地址映射。
3. 当用户线程通过不正确的指针访问旧对象时，**读屏障**会拦截访问，根据转发表将引用修正为新地址，并"自愈"该指针——这就是 ZGC 的**指针自愈**（Self-healing）能力。

因为指针自愈的存在，ZGC 不需要一次性地修正所有旧引用——未修正的引用在下一次被访问时会自动修复。这部分"遗留工作"会被延后到下一次 GC 的并发标记阶段完成，从而节省了一次遍历对象图的开销。

#### ZGC 关键参数

| 参数 | 作用 | 建议值 |
|------|------|--------|
| `-XX:+UseZGC` | 启用 ZGC | JDK 17+ 生产环境直接使用 |
| `-XX:ZCollectionInterval` | GC 间隔（秒），0 表示不强制 | 高实时系统可设为 5~10，避免 GC 过于频繁 |
| `-XX:ZFragmentationLimit` | 碎片化阈值（%），触发整理 | 默认 25，一般无需修改 |
| `-XX:ConcGCThreads` | 并发 GC 线程数 | 默认约 CPU 核心数的 12.5%，可调至 25% |
| `-Xms` / `-Xmx` | 堆大小 | 建议设相同值，避免堆伸缩带来的性能抖动 |
| `-XX:+ZGenerational` | JDK 21+ 启用分代 ZGC | 提升吞吐量，非延迟敏感场景推荐 |

### Shenandoah

Shenandoah 由 Red Hat 主导开发，目标同样是**极低停顿时间**。与 ZGC 最大的区别在于实现路径：ZGC 使用染色指针，Shenandoah 使用 **Brooks Pointers（转发指针）**。

Shenandoah 的 JDK 演进：

| JDK 版本 | Shenandoah 状态 |
|----------|----------------|
| JDK 12 | 实验性引入（仅支持 Linux/x64） |
| JDK 15 | **正式发布（Production）** |
| JDK 17 LTS | 稳定，生产可用 |

> Shenandoah 的一个关键区别：它不是 Oracle JDK 的一部分，而是 OpenJDK 项目。在 Oracle 官方 JDK 发行版中不可用，但在 Adoptium、Red Hat OpenJDK、Amazon Corretto 等发行版中可用。

#### Brooks Pointers 与并发复制

Shenandoah 在每个对象头的前面增加一个**转发指针**（Brooks Pointer），指向对象的当前有效地址：

- 非并发移动时，转发指针指向对象自身。
- 并发移动时，转发指针通过 CAS 操作原子地更新为指向新副本的地址。

当用户线程访问一个对象时，Shenandoah 的**读屏障**会检查转发指针——如果对象已被移动，自动将访问重定向到新地址。写屏障同样需要处理转发，因为写入一个已被移动的对象应该修改新副本，而非旧地址。

#### Shenandoah 的收集步骤

Shenandoah 的阶段比 G1 和 ZGC 更多（共 9 个阶段），但只有 3 个阶段需要 STW：

| 阶段 | STW | 说明 |
|------|-----|------|
| 初始标记 | 是 | 标记 GC Roots 直接关联对象 |
| 并发标记 | 否 | 遍历对象图 |
| 最终标记 | 是 | 处理 SATB 剩余记录，选出回收集 |
| 并发清理 | 否 | 清理无存活对象的 Region |
| **并发回收** | **否** | **真正并发地复制存活对象到新 Region** |
| 初始引用更新 | 是 | 确保所有收集器线程完成对象移动（极短） |
| 并发引用更新 | 否 | 线性扫描物理内存，修正引用 |
| 最终引用更新 | 是 | 修正 GC Roots 中的引用 |
| 并发清理 | 否 | 回收旧 Region |

Shenandoah 不使用分代收集——整个堆一视同仁。这是它的简化之道，也是它的性能上限所在：在年轻代对象密集的场景下，Shenandoah 无法像分代收集器那样靠 Minor GC 高效回收短生命周期的对象。

#### Shenandoah 关键参数

| 参数 | 作用 | 建议值 |
|------|------|--------|
| `-XX:+UseShenandoahGC` | 启用 Shenandoah | JDK 17+ |
| `-XX:ShenandoahGCHeuristics` | 启发式策略 | `adaptive`（默认）/ `static` / `compact` / `aggressive` |
| `-XX:ShenandoahAllocSpikeFactor` | 分配突发因子 | 默认 5，突发分配场景可调大 |
| `-XX:ShenandoahPacing` | 分配速度调节 | 默认开启，控制并发回收节奏 |

### ZGC vs Shenandoah vs G1：低延迟选型决策树

| 维度 | G1 | ZGC | Shenandoah |
|------|----|----|------------|
| 目标停顿 | < 200 ms | < 10 ms | < 10 ms |
| 最大堆 | 几十 GB | 16 TB | 几十 GB |
| 分代收集 | 是 | JDK 21+ 支持 | 否 |
| 并发整理 | 否（STW 时复制） | 是 | 是 |
| 指针压缩 | 支持 | 不支持 | 支持 |
| JDK 发行版 | 所有 | 所有 | OpenJDK（非 Oracle JDK） |
| 成熟度 | 最成熟 | 成熟 | 较成熟 |

**选型决策树：**

```
你的 GC 停顿目标是多少？
  ├── < 10 ms（金融交易、广告竞价、实时风控）
  │     ├── Oracle JDK → ZGC
  │     └── OpenJDK 发行版 → ZGC 或 Shenandoah
  ├── 10~200 ms（Web 服务、API 网关、微服务）
  │     └── G1（JDK 9+ 默认，最成熟）
  └── 无严格要求（批处理、离线计算）
        └── Parallel GC（最大吞吐量）
```

> 对于绝大多数 Web 后端应用（延迟敏感但非极致低延迟），G1 已经足够。ZGC 的价值在堆很大（32 GB+）且停顿要求极严（< 10 ms）的场景下才会凸显。盲目追求 ZGC 可能导致吞吐量下降（读屏障和染色指针有额外开销），反而得不偿失。

***

## 其他收集器

### Epsilon（空操作收集器）

Epsilon（JDK 11 引入）是一个"什么都不做"的垃圾收集器——它只负责内存分配，不进行任何回收。当堆内存耗尽时，JVM 直接退出。

适用场景：
- **性能测试**：对比有无 GC 时的程序性能，量化 GC 开销。
- **内存压力测试**：在已知内存充足的条件下运行，测试程序的内存上限。
- **短生命周期应用**：运行时间很短、内存足够、退出时 JVM 自动回收所有内存的程序。
- **延迟关键的任务**：已知不会产生垃圾的路径上进行极限测试。

开启参数：`-XX:+UseEpsilonGC -Xlog:gc`（配合日志确认没有 GC 活动）。

### 各收集器适用场景总结

| 收集器 | 算法 | 目标 | 适用场景 |
|--------|------|------|---------|
| Serial | 复制 / 标记-整理 | 简单高效 | 嵌入式、桌面应用、微型容器 |
| Parallel | 复制 / 标记-整理 | 最大吞吐量 | 批处理、离线计算、后台任务 |
| CMS | 标记-清除（并发） | 低延迟 | 已废弃，不建议新系统使用 |
| G1 | 复制 / 标记-整理（局部） | 可控停顿 | 通用 Web 服务、微服务、API 网关 |
| ZGC | 标记-整理（并发） | 极低延迟 | 金融交易、实时系统、大堆低延迟 |
| Shenandoah | 标记-整理（并发） | 极低延迟 | 与 ZGC 类似，OpenJDK 发行版 |
| Epsilon | 无回收 | 无 GC 开销 | 测试、基准、极短生命周期应用 |

***

## 内存分配与回收策略

理解 GC 只解决了一半问题。另一半是理解 JVM 如何在堆中分配对象，以及分配策略如何影响 GC 行为。

### 对象优先在 Eden 分配

大多数情况下，对象在新生代的 Eden 区中分配。当 Eden 区没有足够空间时，JVM 发起一次 Minor GC。Minor GC 后仍然存活的对象，根据年龄判断进入 Survivor 区还是老年代。

### 大对象直接进入老年代

大对象（如长字符串、大数组）需要大量连续内存空间。为了避免在 Eden 和两个 Survivor 之间来回复制的大开销，超过一定阈值的大对象直接在老年代分配。阈值由 `-XX:PretenureSizeThreshold` 参数设定（只对 Serial 和 ParNew 有效）。

> **实战隐患**：如果在循环中频繁创建刚好超过阈值的"大对象"（比如每次请求都 new 一个 5 MB 的缓存数组），这些对象会绕过新生代直接进入老年代，导致老年代被快速填满，频繁触发 Full GC。排查方法是通过 GC 日志观察老年代增长速率，如果与请求量呈线性关系，可以怀疑是"伪大对象"导致的。

### 长期存活对象晋升

对象在新生代每熬过一次 Minor GC，年龄（Age）就增加 1。当年龄达到阈值（默认 15，由 `-XX:MaxTenuringThreshold` 设定）后，对象被晋升到老年代。

### 动态对象年龄判定

HotSpot 并不严格要求对象年龄必须达到 `MaxTenuringThreshold` 才能晋升。还有一个**动态年龄判定**规则：如果 Survivor 空间中**相同年龄**的所有对象大小总和超过了 Survivor 空间的一半，那么年龄**大于等于**该年龄的对象都可以直接进入老年代。

> 这个规则的实际效果是：如果一个 Survivor 区中，年龄为 3 的对象加起来已经占了 Survivor 区的一半以上，那么年龄 ≥ 3 的所有对象都会被晋升。这意味着实际晋升年龄通常远小于 15——这也是为什么生产环境中 `MaxTenuringThreshold` 设置得再大，也可能观察到大量对象早早进入了老年代。

### 空间分配担保

发生 Minor GC 之前，JVM 必须先检查老年代最大可用连续空间是否大于新生代所有对象的总空间，或大于历次晋升到老年代对象的平均大小。如果满足，则允许进行 Minor GC；否则先触发一次 Full GC。

如果空间分配担保失败（即 Minor GC 后大量对象需要晋升到老年代，但老年代空间不足），JVM 同样会触发 Full GC——而且是**在 Minor GC 进行中触发的 Full GC**，停顿时间叠加，后果严重。

> **排查线索**：GC 日志中出现 "Full GC (Allocation Failure)" 前的 Minor GC 日志，检查 Survivor 区使用率是否接近 100%，以及老年代使用率是否已经很高。如果确认是空间担保失败，可以尝试减小 SurvivorRatio（增大 Survivor 空间），或提前增大老年代大小。

***

## JDK 21~25 的新发展

GC 技术的演进在最近几个 JDK 版本中并未停下脚步。以下变化对生产环境已经开始产生影响。

### Generational ZGC（JDK 21）

JDK 21 引入了**分代 ZGC**（JEP 439），在 ZGC 中加入了分代收集：

- 新生代对象使用 Minor GC 快速回收，减少了对全堆并发标记的依赖。
- 老年代对象继续使用 ZGC 的并发标记和并发整理。
- 吞吐量显著提升（Oracle 官方测试提升约 10%~20%），堆大小 > 32 GB 时提升更明显。

开启参数：`-XX:+UseZGC -XX:+ZGenerational`（JDK 21 中默认关闭，需显式开启 `-XX:+ZGenerational`；JDK 25 中已移除该参数，纯分代成为默认）。

> 分代 ZGC 让 ZGC 从"纯低延迟"向"兼顾吞吐量"迈进了一大步。如果你的系统已经升级到 JDK 21+，且还在使用 G1 但受困于偶尔的长时间停顿，分代 ZGC 是值得评估的升级方向。

### 紧凑对象头（JDK 25 产品特性）

Project Lilliput 的目标是将 HotSpot 的对象头从 96~128 bit（64 位 + 压缩 Oops）缩减到 64 bit。JDK 24 中作为 JEP 450 实验性引入，**JDK 25 通过 JEP 519 提升为正式产品特性**，无需再解锁实验性参数。

紧凑对象头的直接收益：
- **降低内存占用**：64 字节以下的常见对象，内存节省约 10%~20%。SPECjbb2015 基准测试显示堆空间减少约 22%，CPU 时间减少约 8%。
- **提高缓存效率**：更紧凑的对象意味着 CPU 缓存行中可以容纳更多对象，提高缓存命中率。JSON 解析器基准测试速度提升约 10%。
- **减轻 GC 压力**：相同堆大小下可容纳更多对象，GC 频率降低约 15%。
- **扩大压缩指针范围**：标准对象头下压缩指针上限为 32 GB，紧凑对象头下可扩展至 64 GB。

开启参数：`-XX:+UseCompactObjectHeaders`（JDK 25，产品特性，无需 `-XX:+UnlockExperimentalVMOptions`）。

> 紧凑对象头是一项底层改动，对应用层几乎透明，但对内存密集型应用（如缓存中间件、大规模数据处理）的收益显著。Amazon 已在数百个生产服务中验证该特性（包括反向移植到 JDK 21 和 JDK 17）。建议对内存敏感的应用在预发环境充分验证后开启，尤其是 monitor 竞争极其激烈的场景需关注潜在的性能回归。

### Generational Shenandoah（JDK 25）

Shenandoah 的低延迟设计与 ZGC 类似，但实现路径不同（使用 Brooks Pointer 而非染色指针）。JDK 25 通过 **JEP 521** 将 Shenandoah 的**分代模式**从实验性特性提升为产品特性。

- JDK 24 中分代模式作为实验性功能引入（JEP 404），需配合 `-XX:+UnlockExperimentalVMOptions` 使用。
- **JDK 25 中分代模式默认可用**，不再需要解锁实验性参数，启用方式：`-XX:+UseShenandoahGC -XX:ShenandoahGCMode=generational`。
- **默认仍为非分代模式**：与 ZGC 不同，Shenandoah 在 JDK 25 中默认仍使用单代模式，分代模式需显式开启。
- 分代模式的收益：降低持续内存占用、降低 CPU 和功耗、减少分配尖刺期间的退化收集和 Full GC。
- 仍支持压缩指针（Compressed Oops），平台支持范围比 ZGC 更广。

> 如果你的应用对延迟敏感但堆大小 ≤ 16 GB，Shenandoah 是一个值得与 ZGC 并列评估的选项。JDK 25 中分代模式的成熟使其在吞吐量与延迟之间取得了更好的平衡。

### ZGC 非分代模式移除（JDK 24）

ZGC 的分代演进在 JDK 23~24 之间完成了最终切换：

- **JEP 474（JDK 23）**：将 ZGC 的默认模式从非分代切换为分代，`-XX:+ZGenerational` 默认值为 `true`。
- **JEP 490（JDK 24）**：彻底移除非分代 ZGC 的代码和测试，`-XX:+ZGenerational` 选项被标记为废弃（obsolete）。
- **JDK 25 中的行为**：`-XX:+UseZGC` 始终启用分代 ZGC。即使传入 `-XX:-ZGenerational`，JVM 也会忽略该选项并打印废弃警告，仍然使用分代模式。

这意味着从 JDK 25 开始，ZGC 用户无需再考虑分代与否的问题——分代是唯一的实现方式。对于绝大多数工作负载，分代 ZGC 的吞吐量优于非分代版本；仅极少数天生非分代特征的工作负载可能观察到轻微的性能回归。

### G1 持续优化（JDK 22~24）

G1 作为 JDK 9+ 的默认收集器，其优化仍在持续进行：

#### JEP 423 Region Pinning（JDK 22）

JNI 的 `GetPrimitiveArrayCritical` / `ReleasePrimitiveArrayCritical` 机制会在临界区（Critical Region）内禁止 GC，以防止 GC 移动被 Native 代码直接引用的 Java 对象。在 G1 中，这种实现曾导致严重的延迟问题——如果某个线程长时间处于 JNI 临界区，其他线程触发 GC 时会被阻塞，极端情况下应用可能停滞数分钟。

JEP 423 引入了 **Region Pinning** 机制：G1 不再在 JNI 临界区期间完全禁用 GC，而是将包含临界对象的特定 Region "钉住"（Pin），允许 GC 继续回收其他 Region。这彻底消除了 JNI 临界区导致的 GC 停滞问题。

#### JEP 475 Late Barrier Expansion（JDK 24）

G1 的写屏障（Write Barrier）用于记录跨 Region 引用，是 G1 并发标记正确性的核心。此前，G1 屏障在 C2 JIT 编译流水线的早期阶段展开，导致 C2 的编译开销增加约 10%~20%。

JEP 475 将 G1 屏障的展开时机推迟到 C2 流水线的后期，显著降低了 C2 的编译时间和内存开销，同时保持生成代码的质量不变。这一改动对云原生和容器化部署尤为重要——更短的 JIT 编译时间意味着更快的启动和更低的内存峰值。

***

## Reference 机制深入

上一节介绍了对象的存活判断和 `finalize()` 的两次标记过程。但 JVM 对 Reference 的处理远比概念层面复杂——SoftReference、WeakReference、PhantomReference、FinalReference 这四类引用不仅仅是"不同级别的弱引用"，它们在 JVM 内部的实现机制有本质差异。这一节以 ZGC 为例，深入 OpenJDK 17 源码层面，拆解 Reference 的完整处理链路。

### 五种引用类型全景

在 JVM 的 Reference 体系中，按"强度"从高到低排列：

| 引用类型 | JDK 类 | 被引用对象回收条件 | 典型应用场景 |
|---------|--------|------------------|-------------|
| 强引用（Strong） | 默认 | 永远不会被 GC | 绝大多数 Java 对象 |
| 软引用（Soft） | `SoftReference` | 内存不足，且较久未被访问时 | 内存敏感缓存（Guava Cache、KryoPool） |
| 弱引用（Weak） | `WeakReference` | 下一次 GC 必然回收 | `ThreadLocal` 的 Entry、`WeakHashMap` |
| 虚引用（Phantom） | `PhantomReference` | 和 WeakReference 一致，但用于跟踪回收状态 | `DirectByteBuffer` 的 Cleaner、堆外内存释放 |
| 终结引用（Final） | `FinalReference`（JVM 内部） | 与 WeakReference 一致，但 object 会被复活 | 与 `finalize()` 配套，JDK 9 已废弃 |

> **关键认知**：`SoftReference`、`WeakReference`、`PhantomReference` 本身也是普通的 Java 对象（都继承自 `java.lang.ref.Reference<T>`）。它们在 GC 标记阶段会被当作普通对象扫描——GC 线程顺着 GC Roots 的引用链遍历时，会发现这些 Reference 对象也是"可达"的。但 JVM 并不是简单地因为 Reference 对象可达就保留其 referent，而是在标记阶段通过特殊的判断逻辑实现了不同引用类型的语义。这是理解 Reference 机制的核心起点。

### Reference 处理的全链路：从 GC 线程到 Java 线程

Reference 的完整处理需要 GC 线程和 Java 线程协同完成，分为以下阶段：

1. **Concurrent Mark 阶段（GC 线程）**：GC 线程遍历对象图时，遇到 Reference 类型的对象，会检查其 referent 是否仍然有强引用或软引用。如果没有，则将该 Reference 对象插入 `_discovered_list` 链表，同时清除 referent 字段（对于 Cleaner 这类 PhantomReference 会保留 referent 以便后续清理资源）。对于 FinalReference，还会额外将 referent 重新标记为存活（复活）。

2. **Concurrent Process Non-Strong References 阶段（GC 线程）**：GC 线程将 `_discovered_list` 中的所有 Reference 对象批量转移到 JVM 内部的全局链表 `_reference_pending_list` 中，随后唤醒 ReferenceHandler 线程。

3. **ReferenceHandler 线程（Java 线程，优先级 `Thread.MAX_PRIORITY`）**：从 `_reference_pending_list` 中逐个摘下 Reference 对象。如果是 `Cleaner` 类型，直接调用 `clean()` 方法释放资源；否则通过 `enqueueFromPending()` 将其加入创建时注册的 `ReferenceQueue` 中。

4. **业务线程感知（Java 线程）**：应用代码通过轮询 `ReferenceQueue` 或重写 `Cleaner` 的 `clean()` 方法来感知对象已被 GC，执行后续资源清理。

> **Cleaner 的资源释放模式**：`DirectByteBuffer` 的 Cleaner 并不走 ReferenceQueue 路径——它被 ReferenceHandler 线程直接从 `_reference_pending_list` 摘下后调用 `clean()` 方法释放 Native Memory。Cleaner 内部还维护了一个全局双向链表来持有自身的强引用，防止 Cleaner 对象在 referent 存活期间被 GC 回收。`clean()` 执行时会先从链表中移除自己，断开唯一的强引用链，下一次 GC 时 Cleaner 自身也会被回收。

### SoftReference：LRU 时钟算法量化"内存不足"

教科书常说"内存不足时 SoftReference 会被回收"，但何为"内存不足"？JVM 是如何精确量化这个概念的？

#### 无条件回收的场景

并非所有 GC 中 SoftReference 都会被保留。在 `ZReferenceProcessor` 中，SoftReference 的处理策略由 `_soft_reference_policy` 控制。当 GC 原因属于以下四种之一时，JVM 会将 `_soft_reference_policy` 设置为 `always_clear`——**无条件回收所有 SoftReference**：

- `_gc_locker`：JNI 临界区触发的 GC
- `_wb_full_gc`：WhiteBox API 触发的 Full GC
- `_metadata_GC_threshold`：元空间扩容触发的 GC
- `_allocation_profiler`：分配分析器触发的 GC

```java
// SoftReference 中的关键字段
public class SoftReference<T> extends Reference<T> {
    static private long clock;    // 每次 GC 更新为当前时间戳（毫秒）
    private long timestamp;       // 上次调用 get() 时更新为 clock 的值
}
```

当 GC 原因不属于上述四种时，JVM 使用 **LRUMaxHeapPolicy** 策略——结合 LRU（最近最少使用）和最大堆约束来判断哪些 SoftReference 可以被回收。

#### clock 与 timestamp：LRU 的量化依据

- `clock` 是 `SoftReference` 的静态字段，在每次 GC 的末尾由 `soft_reference_update_clock()` 更新为当前时间戳。
- `timestamp` 是每个 SoftReference 实例的字段，在应用程序调用 `get()` 时更新为当时的 `clock` 值。

因此 `clock - timestamp` 表示这个 SoftReference 有多久没有被访问过——差值越大，说明该软引用越"冷"，JVM 越倾向于回收其 referent。这正是 LRU 策略的语义来源。

```c
// LRUMaxHeapPolicy 的判断逻辑
// _max_heap 表示本轮 GC 中允许 SoftReference 保留其 referent 对象的最大堆空间
// _max_heap = free_heap + soft_heap
// 其中 soft_heap = last_gc_heap_used - last_gc_minus_avg
// free_heap = 当前 GC 完成后的空闲堆大小
// last_gc_heap_used = 上一次 GC 之后堆的使用量
// last_gc_minus_avg = 最近几次 GC 的平均空闲堆大小
```

JVM 遍历 `_discovered_list` 中的 SoftReference 时，按 `clock - timestamp` 从大到小排序（即按"最久未被访问"排序）。只有前 `_max_heap / MB` 个 SoftReference 被允许保留其 referent（不被回收），其余的 referent 都会被清除。

> **关键含义**：即使在内存看似充足的情况下，如果一个 SoftReference 很久没有被访问，它的 referent 仍然可能被回收——这不是因为内存不足，而是 LRU 策略认为它"不值得保留"。因此，"内存充足时 SoftReference 不会被回收"这个说法并不准确。

### PhantomReference vs WeakReference：FinalReference 复活揭示的差异

从概念上看，PhantomReference 和 WeakReference 几乎相同：都是当 referent 没有强引用和软引用时，Reference 对象被加入队列。那么 PhantomReference 能跟踪对象回收状态，WeakReference 是否也可以？

在大多数场景下，WeakReference 确实可以。但当对象重写了 `finalize()` 方法时（即存在 FinalReference），两者表现出**本质差异**：

**场景**：一个 Java 对象在堆中同时被 WeakReference（或 PhantomReference）和 FinalReference 引用，除此之外没有任何强引用和软引用。

**WeakReference + FinalReference 的情况**：
- 发生 GC 时，WeakReference 的语义触发——referent 没有强引用和软引用，因此指向它的 WeakReference 被加入 `_reference_pending_list`，进入后续的 ReferenceHandler 处理流程。
- 同时，FinalReference 的语义也触发——JVM 将 referent 重新标记为存活（`finalizable` 视图），执行其 `finalize()` 方法。
- 结果：**WeakReference 已入队，但 referent 实际上还活着**。如果你此时检查 WeakReference 的队列，会误判对象已被回收。

**PhantomReference + FinalReference 的情况**：
- 发生 GC 时，JVM 在处理 PhantomReference 之前会先检查 `is_inactive()`。由于 referent 被 FinalReference 标记为 `finalizable` 后仍然存活，PhantomReference 的 `is_inactive()` 返回 `true`，**本轮 GC 不会将 PhantomReference 加入 `_discovered_list`**。
- 只有等到下一轮 GC（此时 `finalize()` 已执行完毕，FinalReference 与 referent 的关联已被 `clear()`），referent 真正死亡时，PhantomReference 才会被入队。
- 结果：**PhantomReference 入队的时机与 referent 真正被回收的时机严格一致**。

```c
// ZGC 中 should_discover 的判断流程（简化）
bool ZReferenceProcessor::should_discover(oop reference, ReferenceType type) const {
    if (is_inactive(reference, referent, type)) return false;  // 已处理过
    if (is_strongly_live(referent)) return false;              // 有强引用
    if (is_softly_live(reference, type)) return false;         // 有软引用且符合保留条件
    return true;  // 加入 _discovered_list
}
```

> **结论**：正是因为 `is_inactive()` 这一层判断，PhantomReference 在 FinalReference 复活场景下能够正确等待对象真正死亡后才入队，而 WeakReference 则会产生"对象还活着但引用已入队"的误报。这就是 PhantomReference 设计为"不能通过 `get()` 获取对象"的根本原因——它连接的对象可能已经被复活，此时读取它是没有意义的。PhantomReference 唯一正确的用途就是和 ReferenceQueue 配合，精确跟踪对象的垃圾回收状态。

### FinalReference：为什么它让 GC 变慢

FinalReference 是 JVM 内部使用的一种 Reference 类型，不由用户直接创建。当一个 Java 类覆盖了 `finalize()` 方法，JVM 在创建该类的实例时，会额外创建一个与之关联的 Finalizer 对象（FinalReference 的子类）。

#### 复活机制

这是 FinalReference 与其他 Reference 最大的区别。当 GC 在标记阶段发现一个 referent 对象仅被 FinalReference 引用时，JVM 的处理是：

1. 通过 `ZBarrier::mark_barrier_on_oop_field` 将 referent 在标记位图（`_livemap`）中的 bit 位标记为 1，即重新标记为存活。
2. 将 referent 对象的地址视图调整为 `finalizable`，表示该对象只能通过 `finalize()` 方法访问。
3. 将 FinalReference 本身加入 `_discovered_list`，走后续的处理链路。

```c
// ZGC 中 FinalReference 的复活关键代码
if (is_final_reference(reference, type)) {
    // 标记 referent 为存活，finalizable = true
    ZBarrier::mark_barrier_on_oop_field(referent_addr, true /* finalizable */);
}
// 然后将 FinalReference 加入 _discovered_list
```

#### 处理链路与为什么"拖拖拉拉"

复活后的 referent 在本轮 GC 中不会被回收。FinalReference 随后通过以下链路传递：

`_discovered_list` → `_reference_pending_list` → ReferenceHandler 线程 → Finalizer 的 `ReferenceQueue` → FinalizerThread 线程 → 执行 `finalize()` → 调用 `clear()` 断开引用 → 下一轮 GC 时 referent 和 FinalReference 一起被回收。

在这个过程中，有两个环节导致延迟：

**1. FinalizerThread 的调度延迟**：FinalizerThread 是守护线程，优先级为 `Thread.MAX_PRIORITY - 2`，低于 ReferenceHandler（`MAX_PRIORITY`）。在线程竞争激烈的环境下，FinalizerThread 可能很久才被 OS 调度到。在此之前，referent 对象虽然已"逻辑上死亡"但无法被回收。

**2. 多轮 GC 才能彻底回收**：referent 被复活后至少需要再经历一轮 GC 才能真正被回收。如果 `finalize()` 方法执行慢，中间可能已经发生多轮 GC。在这期间的每一轮 GC 中，FinalReference 和它的 referent 都占据着堆空间但无法回收——GC 日志中会出现大量 "Finalizer" 对象。

**3. unfinalized 双向链表**：Finalizer 类内部维护了一个静态的双向链表 `unfinalized`，所有 Finalizer 对象在创建时被插入该链表。这个链表提供了 Finalizer 对象的强引用链，防止 Finalizer 自身在 referent 存活期间被 GC 回收。只有当 FinalizerThread 在 `runFinalizer()` 中将其从链表移除后，Finalizer 对象才变得可以被回收。

> **生产环境警示**：覆盖 `finalize()` 的对象从"不可达"到"真正被回收"之间，可能经历数秒甚至更长的延迟。如果你的应用中有大量对象覆盖了 `finalize()`，`jmap -histo:live` 会看到大量的 `java.lang.ref.Finalizer` 实例，占用可观堆空间。这是排查 GC 异常缓慢的必查项。JDK 9 已标记 `finalize()` 为 `@Deprecated`，JDK 18 标记为 `@Deprecated(forRemoval = true)`，新系统应使用 `Cleaner` 或 `try-with-resources` 替代。

### System.gc() 内部发生了什么

`System.gc()` 的调用并不简单等同于"触发一次 GC"。不同垃圾收集器对它的处理方式差异很大，而且它的行为可以通过 JVM 参数调节。

#### 通用处理框架

无论使用哪种收集器，`System.gc()` 最终都会调用到对应 `CollectedHeap` 子类的 `collect()` 方法，并传入 GC 原因 `GCCause::_java_lang_system_gc`。在触发 GC 之前，JVM 会先判断是否设置了 `-XX:+DisableExplicitGC`——如果设置，大部分收集器的 `System.gc()` 将不产生任何效果。

#### 各收集器行为对照

| 收集器 | System.gc() 行为 | Java 线程是否阻塞 |
|--------|-----------------|------------------|
| SerialGC | `VM_SerialGCSystemGC` → VMThread 执行 Full GC | 是，阻塞直到 Full GC 完成 |
| ParallelGC | `VM_ParallelGCSystemGC` → VMThread 执行 Full GC | 是，阻塞直到 Full GC 完成 |
| G1 | `VM_G1CollectFull` 执行 Full GC。若设置 `-XX:+ExplicitGCInvokesConcurrent`，则改为 `VM_G1TryInitiateConcMark` 执行并发标记 | Full GC 时阻塞；Concurrent 时不阻塞 |
| ZGC | 发送同步 GC 请求，执行一轮完整的同步 GC 周期 | 是，阻塞直到 GC 周期完成 |
| Shenandoah | 设置 `_gc_requested` 标志，ShenandoahControlThread 定期检测并执行。若设置 `-XX:+ExplicitGCInvokesConcurrent` 则为 Concurrent GC，否则为 Full GC | Concurrent 时不阻塞；Full GC 时阻塞 |

#### DisableExplicitGC 的例外

`-XX:+DisableExplicitGC` 对 SerialGC、ParallelGC、G1、Shenandoah 生效（直接忽略 `System.gc()` 调用），但对 ZGC 不完全适用——ZGC 在某些场景下仍会响应。此外，`-XX:+DisableExplicitGC` 并不阻止 JVM 内部的显式 GC 调用，只阻止来自 `System.gc()` 和 `DiagnosticCommand` 的调用。

#### GC 后的收尾工作：Cleaner 的执行时机

GC 完成后，`VM_GC_Operation::doit_epilogue()` 会被调用。这个阶段做两件事：
- 检查 `_reference_pending_list` 是否有待处理的 Reference 对象（Cleaner 就在其中）。
- 通过 `Heap_lock->notify_all()` 唤醒 ReferenceHandler 线程，启动 Cleaner 的执行。

这就是 `DirectByteBuffer` 释放 Native Memory 的关键环节：如果 `System.gc()` 触发的 GC 回收了不再使用的 DirectByteBuffer，GC 结束之后，ReferenceHandler 线程被唤醒，立即执行 Cleaner 的 `clean()` 方法，通过 `UNSAFE.freeMemory()` 将 Native Memory 归还给 OS。

> **性能考量**：因为 `System.gc()` 对大多数收集器会触发 Full GC（或至少一次完整的 GC 周期），它会暂停所有 Java 业务线程。在生产环境中直接调用 `System.gc()` 是危险的——可能导致意外的长时间停顿。如果确实需要释放堆外内存，更好的做法是适时调用 `System.gc()` 配合 `-XX:+ExplicitGCInvokesConcurrent`（对 G1 和 Shenandoah），以并发 GC 代替 Full GC。

***

## 写在最后

GC 调优不是魔法，也没有银弹。理解这篇内容的目的不是让你记住每个参数，而是在系统出现问题时，能快速建立"症状 → 收集器行为 → 根因"的分析链路。

几个核心原则值得反复记住：

- **不要过早优化 GC**：先让应用正确运行，通过监控（JFR、Prometheus + JMX Exporter）观察 GC 的实际行为，再决定是否需要调整。大多数应用的 GC 问题不是参数问题，而是代码中的内存泄漏或对象生命周期管理问题。

- **选择收集器的优先顺序**：G1（JDK 9+ 默认）→ 根据停顿目标考虑 ZGC → 非延迟敏感且追求吞吐量时考虑 Parallel。CMS 已经进入历史，不要再用于新系统。

- **监控比调参更重要**：比起背参数，更重要的是建立 GC 监控——GC 频率、停顿时间分布、各区域使用率趋势。线上 GC 问题 80% 的根因可以通过监控数据在分钟级定位。

- **堆不是越大越好**：32 GB 以上无法使用指针压缩，对象引用从 4 字节变成 8 字节，实际可用内存增加并不线性。如果不是必须要大堆，控制在 32 GB 以内往往是更好的选择。

> ZGC 和 Shenandoah 的出现标志着 JVM GC 已经从"尽量缩短停顿"进入了"彻底消除长停顿"的时代。对于延迟敏感的应用，现在是时候把 ZGC 纳入评估雷达了。
