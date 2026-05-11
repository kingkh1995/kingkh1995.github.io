## Java 内存模型概述

Java 内存模型（Java Memory Model，JMM）是一组**规范**而非具体的物理实现。它定义了 Java 程序中各种变量（实例字段、静态字段和数组元素）的访问规则，用于屏蔽不同硬件架构和操作系统的内存访问差异，确保 Java 程序在所有平台下都能获得一致的并发语义。

JMM 的核心职责可以概括为两点：

- **可见性**：规定一个线程对共享变量的修改，何时对其他线程可见；
- **有序性**：规定编译器和处理器可以对指令进行何种程度的重排序，同时保证多线程执行结果的正确性。

> **关键区别**：JMM 与 JVM 运行时数据区（堆、栈、方法区等）是两个不同层面的概念。JMM 描述的是抽象的内存访问语义，而 JVM 内存结构描述的是具体的内存区域划分。

***

## 主内存与工作内存

### 抽象模型

JMM 定义了线程与主内存之间的抽象关系：

- **主内存（Main Memory）**：存储所有共享变量，对应 JVM 堆和方法区中的数据。由于多个线程都可以访问，主内存是线程安全问题的根源。
- **工作内存（Working Memory）**：每个线程私有的数据区域，保存了该线程使用的主内存变量的**副本**。工作内存是抽象概念，对应 CPU 寄存器、高速缓存和缓冲区。

线程对共享变量的所有操作都必须在工作内存中进行，不能直接读写主内存。线程间无法直接访问对方的工作内存，变量传递只能通过主内存完成。

### 8 种原子操作

JMM 定义了 8 种用于主内存与工作内存之间交互的原子操作：

| 操作 | 作用域 | 说明 |
|------|--------|------|
| **lock** | 主内存 | 将变量标识为线程独占状态 |
| **unlock** | 主内存 | 释放被锁定的变量 |
| **read** | 主内存 | 将变量值从主内存传输到工作内存，供 load 使用 |
| **load** | 工作内存 | 将 read 得到的值放入工作内存的变量副本 |
| **use** | 工作内存 | 将变量值传递给执行引擎 |
| **assign** | 工作内存 | 将执行引擎接收到的值赋给工作内存变量 |
| **store** | 工作内存 | 将变量值从工作内存传送到主内存，供 write 使用 |
| **write** | 主内存 | 将 store 得到的值写入主内存变量 |

> JMM 只要求 read 和 load、store 和 write 必须按顺序执行，但不要求是**连续执行**。也就是说，read 之后可以插入其他指令，然后再执行 load。

### 操作规则

8 种操作必须满足以下规则：

- read 和 load、store 和 write 必须成对出现，顺序执行；
- 不允许线程丢弃最近的 assign 操作，变量在工作内存中改变后必须把变化同步回主内存；
- 不允许线程无原因地（没有发生过 assign）将数据从工作内存同步回主内存；
- 新的变量只能在主内存中诞生，不允许在工作内存中直接使用一个未被初始化（load 或 assign）的变量；
- 一个变量同一时刻只允许一条线程对其进行 lock 操作，但 lock 可以被同一线程重复执行多次，unlock 也必须重复相同次数才能释放；
- 如果对一个变量执行 lock 操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行 load 或 assign 操作初始化；
- 如果一个变量事先没有被 lock 操作锁定，则不允许对它执行 unlock 操作；
- 对一个变量执行 unlock 操作之前，必须先把此变量同步回主内存（执行 store 和 write）。

***

## happens-before 原则

happens-before 是 JMM 中判断数据是否存在竞争、线程是否安全的主要依据。如果操作 A happens-before 操作 B，那么 A 的执行结果对 B 可见，且 A 在 B 之前执行。

> **注意**：happens-before 关系并不等同于时间上的先后顺序。JMM 允许重排序，只要重排序后的执行结果与按 happens-before 顺序执行的结果一致即可。

### 程序次序规则（Program Order Rule）

在一个线程内，按照程序代码顺序，书写在前面的操作 happens-before 书写在后面的操作。

> 这条规则只保证**单线程内**的执行顺序，不保证多线程间的可见性。

### 监视器锁规则（Monitor Lock Rule）

一个 unlock 操作 happens-before 后面对同一个锁的 lock 操作。这里强调的是**同一个锁**。

```java
synchronized (lock) {
    x = 1;  // 1
}           // 2: unlock
synchronized (lock) {
    // 3: lock，保证能看到 x = 1
}
```

### volatile 变量规则（Volatile Variable Rule）

对一个 volatile 变量的写操作 happens-before 后面对这个变量的读操作。

```java
volatile int v;
// 线程 A
v = 1;      // 写操作

// 线程 B
int r = v;  // 读操作，保证能看到 v = 1
```

### 线程启动规则（Thread Start Rule）

Thread 对象的 start() 方法 happens-before 此线程的每一个动作。

```java
Thread t = new Thread(() -> {
    // 这里的操作保证能看到 start() 之前的所有修改
});
x = 42;
t.start();
```

### 线程终止规则（Thread Termination Rule）

线程中的所有操作都 happens-before 对此线程的终止检测。可以通过 Thread.join() 方法结束、Thread.isAlive() 的返回值等手段检测到线程已经终止执行。

```java
Thread t = new Thread(() -> {
    x = 42;  // 线程内的操作
});
t.start();
t.join();   // join 返回后，保证能看到 x = 42
```

### 线程中断规则（Thread Interruption Rule）

对线程 interrupt() 方法的调用 happens-before 被中断线程的代码检测到中断事件的发生（通过 Thread.interrupted() 或 Thread.isInterrupted()）。

### 对象终结规则（Finalizer Rule）

一个对象的初始化完成（构造函数执行结束）happens-before 它的 finalize() 方法的开始。

### 传递性（Transitivity）

如果 A happens-before B，且 B happens-before C，那么 A happens-before C。

```java
// 线程 A
x = 1;          // A
volatileV = 2;  // B，volatile 写

// 线程 B
int r1 = volatileV;  // C，volatile 读（B happens-before C）
int r2 = x;          // D（A happens-before B，B happens-before C，由传递性 A happens-before D）
```

***

## 并发三大特性

### 原子性

原子性是指一个操作或一系列操作要么全部执行成功，要么全部不执行，不存在中间状态。

JMM 中基本数据类型的读取和赋值操作（除 long 和 double 外）是原子性的。但**复合操作**（如 `i++`）不是原子的，它实际上包含读取、计算、写入三个步骤。

对于 64 位的 long 和 double，JMM 允许将 64 位的读写操作划分为两个 32 位的操作来执行。但声明为 volatile 的 long 和 double 读写具有原子性。

保证原子性的方式：

- **synchronized 关键字**：通过 monitor 进入和退出保证代码块的原子性；
- **Lock 接口**：显式锁机制，功能与 synchronized 类似但更加灵活；
- **原子类**：`AtomicInteger`、`AtomicLong` 等，基于 CAS 实现无锁原子操作。

### 可见性

可见性是指一个线程修改了共享变量的值，其他线程能够立即得知这个修改。

在没有同步机制的情况下，线程对共享变量的修改可能只存在于自己的工作内存中，其他线程无法感知。

保证可见性的方式：

- **volatile 关键字**：强制将修改刷新到主内存，并使其他线程的工作内存缓存失效；
- **synchronized 关键字**：unlock 之前将修改的变量刷新到主内存，lock 时清空工作内存重新读取；
- **final 关键字**：被 final 修饰的字段在构造函数中初始化完成后，其他线程就能看到它的最终值。

### 有序性

有序性是指程序执行的顺序按照代码的先后顺序执行。但为了提高性能，编译器和处理器常常会对指令进行重排序。

重排序分三种类型：

| 类型 | 发生阶段 | 说明 |
|------|----------|------|
| **编译器优化重排序** | 编译阶段 | 不改变单线程语义的前提下，重新安排语句执行顺序 |
| **指令级并行重排序** | 处理器执行阶段 | 将多条指令重叠执行，不存在数据依赖时可改变执行顺序 |
| **内存系统重排序** | 缓存和缓冲区 | 由于使用缓存和读写缓冲区，加载和存储操作看起来像在乱序执行 |

保证有序性的方式：

- **volatile 关键字**：通过内存屏障禁止特定类型的指令重排序；
- **synchronized 关键字**：同一时刻只允许一个线程执行，自然保证了有序性。

***

## volatile 关键字

volatile 是 Java 中最轻量级的同步机制。它确保变量的修改对所有线程立即可见，并禁止指令重排序。

### 可见性实现机制

volatile 变量的读写操作在 JVM 层面会生成特殊的字节码指令：

- **写操作**：会在写操作后插入一个 `lock` 前缀指令（对应 x86 的 `LOCK#` 信号），该指令会将当前处理器缓存行的数据写回到系统内存，同时使其他 CPU 里缓存了该内存地址的数据无效；
- **读操作**：直接从主内存读取最新值，而不是使用工作内存中的缓存。

> 在 x86 架构下，volatile 读的开销与普通读几乎没有差别，因为 x86 本身具有较强的一致性保证。volatile 写的开销主要在于刷新写缓冲区。

### 禁止指令重排序

volatile 通过插入**内存屏障**（Memory Barrier）来禁止特定类型的重排序：

- 在每个 volatile **写操作之前**插入 **StoreStore 屏障**：确保 volatile 写之前的普通写操作对其他线程可见；
- 在每个 volatile **写操作之后**插入 **StoreLoad 屏障**：确保 volatile 写之后的普通读操作不会被重排序到 volatile 写之前；
- 在每个 volatile **读操作之后**插入 **LoadLoad 屏障**：确保 volatile 读之后的普通读操作不会被重排序到 volatile 读之前；
- 在每个 volatile **读操作之后**插入 **LoadStore 屏障**：确保 volatile 读之后的普通写操作不会被重排序到 volatile 读之前。

### 内存屏障详解

内存屏障是一种 CPU 指令，用于确保屏障前后的内存操作不会被重排序。

#### LoadLoad 屏障

语义：Load1 → LoadLoad → Load2，确保 Load1 的数据装载先于 Load2 及后续装载指令。

典型场景：在 volatile 读之后插入，防止后续普通读被重排序到 volatile 读之前。

#### LoadStore 屏障

语义：Load1 → LoadStore → Store2，确保 Load1 的数据装载先于 Store2 及后续存储指令。

典型场景：在 volatile 读之后插入，防止后续普通写被重排序到 volatile 读之前。

#### StoreStore 屏障

语义：Store1 → StoreStore → Store2，确保 Store1 的数据对其他处理器可见（刷新到内存）先于 Store2 及后续存储指令。

典型场景：在 volatile 写之前插入，防止前面的普通写与 volatile 写重排序。

#### StoreLoad 屏障

语义：Store1 → StoreLoad → Load2，确保 Store1 的数据对其他处理器可见先于 Load2 及后续装载指令。

StoreLoad 屏障是一个**全能屏障**，它同时具有其他三种屏障的效果。但它的开销也是最大的，因为需要将写缓冲区中的数据全部刷新到内存。

> **x86 架构的特殊性**：x86 处理器本身只允许 StoreLoad 重排序，因此 x86 下 volatile 写后只需插入 StoreLoad 屏障，其他屏障是空操作（NOP）。这也是为什么 volatile 在 x86 上的性能相对较好的原因。

### MESI 缓存一致性协议与 volatile

现代 CPU 使用 MESI（Modified、Exclusive、Shared、Invalid）等缓存一致性协议来保证多核缓存的数据一致性。那么既然有 MESI，为什么还需要 volatile？

原因有二：

1. **MESI 并非绝对可靠**：处理器和编译器为了性能优化，可能不完全遵守 MESI 的即时同步语义。volatile 是语言层面的保证，不受具体 CPU 实现的影响；
2. **volatile 不仅解决可见性，还解决有序性**：MESI 只保证缓存一致性，不保证指令执行的顺序。volatile 通过内存屏障禁止指令重排序，这是 MESI 无法提供的。

> 一个常见的误区是认为 volatile 只是解决可见性问题。实际上，在某些场景下（如 DCL），volatile 的核心作用是**禁止指令重排序**。

***

## 实战视角：ConcurrentHashMap 的演进

### JDK 7：分段锁（Segment）

JDK 7 中的 ConcurrentHashMap 采用**分段锁**设计：

- 将整个哈希表划分为 16 个 Segment（默认），每个 Segment 是一个独立的 ReentrantLock；
- 不同 Segment 上的操作可以并发执行，同一 Segment 上的操作需要竞争锁；
- get 操作不需要加锁，利用 volatile 保证可见性。

分段锁的优势在于将全局锁细化为多个局部锁，提高了并发度。但仍有局限：

- 每个 Segment 是一个小型的 HashMap，扩容时需要锁定整个 Segment；
- 统计 size 时需要遍历所有 Segment 并加锁，性能较差；
- 随着哈希表增大，单个 Segment 内部仍可能产生热点。

### JDK 8：CAS + synchronized

JDK 8 对 ConcurrentHashMap 进行了彻底重构：

- **取消 Segment**：直接使用 Node 数组 + 链表/红黑树；
- **put 操作**：
  - 若目标位置为空，使用 CAS 直接插入；
  - 若存在哈希冲突，使用 synchronized 锁定链表头节点；
- **get 操作**：依然无锁，依赖 volatile 保证可见性；
- **size 统计**：引入 CounterCell 数组分散热点，使用 `@sun.misc.Contended` 避免伪共享。

### 为什么去掉分段锁？

1. **synchronized 性能提升**：JDK 6 引入偏向锁、轻量级锁后，synchronized 的性能已经大幅提高，在很多场景下与 ReentrantLock 差距不大；
2. **锁粒度更细**：synchronized 只锁定链表头节点（或红黑树根节点），而 Segment 锁定的是一段连续的哈希桶。新设计的锁粒度更细，冲突概率更低；
3. **代码简化**：去掉 Segment 后，数据结构和逻辑更加简洁，维护成本降低；
4. **扩容优化**：JDK 8 支持**多线程协助扩容**（transfer 方法），多个线程可以并发地迁移数据，而 JDK 7 的 Segment 扩容需要单线程完成。

> 这个演进体现了并发设计的一个重要趋势：**在无竞争或低竞争场景下优先使用无锁算法（CAS），在确实存在竞争时使用最轻量级的锁（synchronized 锁定头节点）**。

***

## 实战视角：双重检查锁定的风险

### 经典 DCL 写法

双重检查锁定（Double-Checked Locking，DCL）是一种常见的延迟初始化优化模式：

```java
public class Singleton {
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {                    // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {            // 第二次检查
                    instance = new Singleton();    // 对象创建
                }
            }
        }
        return instance;
    }
}
```

### 指令重排序问题

`instance = new Singleton()` 并非原子操作，在 JVM 层面可分为三步：

1. **分配内存**：为 Singleton 对象分配内存空间；
2. **初始化对象**：调用构造方法，初始化对象字段；
3. **设置引用**：将内存地址赋值给 instance 变量。

在单线程环境下，步骤 2 和 3 可以重排序（因为两者没有数据依赖），即先执行 3 再执行 2。这在单线程下没有问题，因为只要保证在最终使用对象前完成初始化即可。

但在多线程环境下，重排序会导致严重问题：

- 线程 A 执行到步骤 3（instance 已指向内存地址，但对象尚未初始化）；
- 线程 B 执行第一次检查，发现 `instance != null`，直接返回未完全初始化的对象；
- 线程 B 使用这个对象时，可能访问到字段的默认值而非构造方法设置的值。

### volatile 的必要性

在 JDK 5 之前，DCL 被认为是不可用的，因为 volatile 的语义不足以阻止这种重排序。JDK 5 增强了 volatile 的内存语义，明确禁止了上述重排序：

- **volatile 写操作**与之前的普通写操作（构造方法中的字段赋值）不能重排序；
- **volatile 读操作**与之后的普通读操作（使用对象字段）不能重排序。

因此，**DCL 中的 volatile 是必不可少的**。如果省略 volatile，即使使用 synchronized，也无法保证对象正确发布。

> 更安全的替代方案：使用静态内部类或枚举实现单例，利用类加载机制保证线程安全，无需显式同步。

***

## 实战视角：CAS 与 ABA 问题

### CAS 原理

CAS（Compare-And-Swap）是一种乐观锁实现，包含三个操作数：内存位置 V、预期值 A、新值 B。当且仅当 V 的值等于 A 时，才将 V 的值更新为 B，否则不做任何操作。

Java 中的 CAS 通过 `sun.misc.Unsafe` 类的本地方法实现，最终映射为 CPU 的 `cmpxchg` 指令。从 JDK 9 开始，推荐使用 `java.lang.invoke.VarHandle` 替代 Unsafe 的部分功能。

CAS 的优点是无阻塞、高性能，适用于低竞争场景。但高竞争下，频繁的失败重试会导致 CPU 空转（自旋），反而降低性能。

### ABA 问题

ABA 问题是 CAS 的经典缺陷：

- 线程 1 读取变量值为 A；
- 线程 2 将变量值改为 B，随后又改回 A；
- 线程 1 执行 CAS，发现值仍是 A，操作成功。但实际上变量的值已经经历了 A → B → A 的变化，线程 1 无法感知中间状态。

在某些场景下，ABA 不会造成问题（例如简单的计数器）。但在依赖中间状态变化的场景中，ABA 可能导致严重错误。

### 解决 ABA 问题

**AtomicStampedReference**

Java 提供了 `AtomicStampedReference` 来解决 ABA 问题。它在引用之外附加了一个**版本号**（Stamp），每次修改时版本号递增。CAS 操作需要同时比较引用值和版本号，只有当两者都匹配时才更新。

```java
AtomicStampedReference<Integer> ref =
    new AtomicStampedReference<>(100, 0);

int[] stampHolder = new int[1];
Integer value = ref.get(stampHolder);

// CAS 时需要提供预期值和预期版本号
boolean success = ref.compareAndSet(
    value, 200, stampHolder[0], stampHolder[0] + 1
);
```

> 在大多数业务场景中，ABA 问题并不常见。但在实现无锁数据结构（如无锁队列、栈）时，必须考虑 ABA 问题。JDK 8 的 `StampedLock` 也采用了类似的戳记机制。

***

## JDK 9+ 新特性：VarHandle

### 概述

`java.lang.invoke.VarHandle` 是 JDK 9 引入的变量句柄机制，用于替代 `sun.misc.Unsafe` 的部分功能。它提供了对变量进行原子性、有序性和可见性控制的轻量级方式。

VarHandle 支持的操作模式：

| 模式 | 说明 |
|------|------|
| **read** | 普通读、volatile 读、opaque 读 |
| **write** | 普通写、volatile 写、opaque 写 |
| **CAS** | compareAndSet、compareAndExchange |
| **原子性更新** | getAndSet、getAndAdd |

### 生产环境使用场景

VarHandle 在以下场景中可以替代 Unsafe：

1. **高性能计数器**：使用 `getAndAdd` 实现无锁计数器，语义与 `AtomicInteger` 相同但开销更低；
2. **内存布局控制**：通过 `MethodHandles.lookup().findVarHandle()` 访问字段，配合 `@Contended` 注解避免伪共享；
3. **无锁数据结构**：实现自定义的无锁队列、栈等数据结构，精确控制内存序。

```java
public class Counter {
    private volatile int count;
    private static final VarHandle COUNT_HANDLE;

    static {
        try {
            COUNT_HANDLE = MethodHandles.lookup()
                .findVarHandle(Counter.class, "count", int.class);
        } catch (Exception e) {
            throw new Error(e);
        }
    }

    public void increment() {
        COUNT_HANDLE.getAndAdd(this, 1);
    }
}
```

### 与 Unsafe 的对比

| 特性 | Unsafe | VarHandle |
|------|--------|-----------|
| 官方支持 | 否（内部 API） | 是（标准 API） |
| 模块系统支持 | JDK 9+ 受限 | 完全支持 |
| 内存序控制 | 有限 | 完整（plain、opaque、acquire/release、volatile） |
| 性能 | 极高 | 接近 Unsafe |

> **建议**：在新项目中优先使用 VarHandle 替代 Unsafe。对于仅需要原子操作的简单场景，`AtomicInteger`、`AtomicLong` 等原子类仍然是最安全、最可读的选择。

***

## Memory Order 与 VarHandle 深入

### 从 `volatile` 到 Memory Order：为什么需要更细粒度的控制

回顾前文，我们已经知道：

- **MESI 协议**通过缓存状态机（M/E/S/I）保证多核环境下缓存数据的一致性；
- **Store Buffer** 将写操作延迟提交以提升吞吐，**Invalidation Queue** 将失效消息暂存以减少等待；
- **内存屏障**（LoadLoad、LoadStore、StoreStore、StoreLoad）用于抑制编译器和 CPU 的重排序，确保 `volatile` 的语义正确。

这套机制在传统的锁（`synchronized`）和 `volatile` 编程模型中运作良好，但存在一个问题：**它们只有"开"和"关"两种状态** —— 要么完全不保证（普通变量），要么完全保证（`volatile`/锁）。对于 lock-free 编程而言，这种二元的同步模型过于粗糙，导致大量不必要的性能开销。

举个例子：一个生产者线程写数据，消费者线程读数据。在 `volatile` 模型下，我们只能对整个变量施加全序（total order）保证，但实际上**只需要保证"生产者的写 happens-before 消费者的读"**。这种单向的因果关系，用完整的 `volatile` 语义是过度同步的，尤其在不具备强内存模型的平台（如 ARM）上，会引入多余的 `DMB` 屏障指令，拖累性能。

JDK 9 通过 `VarHandle` 引入的四种 Memory Order 模式，正是为了解决这个问题——**让程序员可以根据具体场景选择最弱但足够的内存序保证**，在正确性和性能之间取得更精确的平衡。

### 四种 Memory Order 模式详解

这四种模式按一致性强度从弱到强排列：

| 模式 | VarHandle 方法 | 一致性强度 | 核心语义 |
|------|---------------|:--------:|---------|
| **Plain** | `get()` / `set()` | 无 | 允许任意重排序，同 JDK 9 之前普通变量 |
| **Opaque** | `getOpaque()` / `setOpaque()` | 弱 | 位原子性 + 最终可见 + 写有序 + 无循环先行 |
| **Acquire/Release** | `getAcquire()` / `setRelease()` | 中 | 单向屏障：Acquire 读阻后续，Release 写阻前置 |
| **Volatile** | `getVolatile()` / `setVolatile()` | 强 | 全序保证，等同 `volatile` 变量语义 |

#### Plain — 无保证

Plain 模式就是 JDK 9 之前对普通共享变量的访问方式。编译器、CPU 和缓存系统都可以在单线程语义允许的范围内自由重排序相关指令。

```java
class Point {
    int x, y;
    private static final VarHandle X, Y;

    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            X = l.findVarHandle(Point.class, "x", int.class);
            Y = l.findVarHandle(Point.class, "y", int.class);
        } catch (ReflectiveOperationException e) {
            throw new Error(e);
        }
    }

    // Plain 读写：无任何跨线程可见性或有序性保证
    void move(int dx, int dy) {
        X.set(this, (int) X.get(this) + dx);  // 可能被重排序
        Y.set(this, (int) Y.get(this) + dy);
    }
}
```

适用场景：仅单线程访问的变量，或程序本身通过其他同步机制（如锁的 unlock/lock）已经保证了 happens-before 关系。

#### Opaque — 位原子性 + 最终可见

Opaque 模式在 Plain 的基础上增加了四个有限保证：

1. **先行无环**：对 Opaque 变量的读/写不会被重排序为因果循环。例如流水线执行中，multiply 的结果不会在 add 的输入被读取前就写回，避免了"未来的结果影响过去的读取"；
2. **一致性**：对同一变量的连续写不会被编译器合并为单次写（`a=1; a=2` 不会被优化为 `a=2`），每次写都实际发生；
3. **行进（Progress）**：写操作最终对读操作可见。如下面的示例，线程 B 以 `getOpaque` 自旋读取，最终一定能看到线程 A 写入的值（这在使用 Plain 读时可能被编译器优化为死循环）：

```java
volatile boolean flag = false;
VarHandle FLAG;

// 线程 A：写操作最终可见
FLAG.setOpaque(this, true);

// 线程 B：不会因编译器优化而陷入死循环
while (!(boolean) FLAG.getOpaque(this)) {};
```

4. **位原子性**：对于 `long`、`double`（64 位）类型的读写保证原子性，即使平台不支持 64 位原子内存操作。

> **关键区别**：Opaque 保证的是"位原子性 + 最终可见"，而不是"立即可见"。在多线程间，Opaque 写的可见时机是不确定的，只是保证了**早晚会**被其他线程看到。

#### Acquire/Release — 单向屏障

RA 模式的核心是**单向内存屏障**，这在因果一致性系统中是最自然的语义：

- **Release 写**（`setRelease`）：在当前线程内，**所有在 Release 写之前的普通内存操作，都不会被重排序到 Release 写之后**。相当于在该写操作前插入了一道向上的屏障。
- **Acquire 读**（`getAcquire`）：在当前线程内，**所有在 Acquire 读之后的普通内存操作，都不会被重排序到 Acquire 读之前**。相当于在该读操作后插入了一道向下的屏障。

用一句话概括：**Release 之前的操作不会被延迟到 Release 之后，Acquire 之后的操作不会被提前到 Acquire 之前**。

经典的生产者-消费者示例：

```java
volatile int ready;  // Initially 0, with VarHandle READY
int dinner;          // 普通变量，无特殊内存序

// 线程 1（生产者）
dinner = 17;                      // 1. 准备数据（Plain 写）
READY.setRelease(this, 1);        // 2. 发布信号（Release 写）

// 线程 2（消费者）
if ((int) READY.getAcquire(this) == 1) {  // 3. 接收信号（Acquire 读）
    int d = dinner;                        // 4. 读取数据 — 保证看到 17
}
```

这里的关键在于：Release 写保证步骤 1 不会被重排序到步骤 2 之后，Acquire 读保证步骤 4 不会被重排序到步骤 3 之前。两者结合产生了"生产者写完数据 → 消费者看到数据"的因果链，**而无需 `dinner` 本身是 `volatile`**。

> **RA 模式的局限**：RA 模式在多线程写入同一变量的场景下**不提供全局有序性**。以下代码中，如果使用 `setRelease/getAcquire`，可能出现在两个线程中都读到 0 的情况（`rx == ry == 0`），因为 Release 不能阻止后面的读被重排到前面：

```java
volatile int x, y;  // Initially zero

// Thread 1                 // Thread 2
X.setRelease(this, 1);      Y.setRelease(this, 1);
int ry = Y.getAcquire(this); int rx = X.getAcquire(this);
// 使用 Release/Acquire 时，rx 和 ry 可能都是 0
// 使用 Volatile 时，rx 和 ry 至少有一个是 1
```

因此 RA 模式更适合 **ownership 模型**（只有一个 writer，多个 reader），而不是多写者竞争的场景。

#### Volatile — 全序保证

Volatile 模式就是 `volatile` 变量的默认语义，提供了最强的内存序保证：**所有 Volatile 变量的内存访问是全序的（totally ordered）**。在任意两个线程的视角中，所有 volatile 读写操作的顺序是一致的。

在前面 Release/Acquire 的竞争示例中，将 `setRelease/getAcquire` 替换为 `setVolatile/getVolatile` 就能保证至少有一个线程读到对方写入的值。这是因为 Volatile 模式的写操作后面被插入了 **StoreLoad 屏障**（最强屏障），确保写操作对后续任何线程的读可见。其背后的实现机制与前文 [volatile 关键字](#volatile-关键字) 和 [内存屏障详解](#内存屏障详解) 章节中描述的一致。

### Memory Order 在不同 CPU 架构上的映射

理解 Memory Order 对性能的影响，关键在于了解不同平台如何**实际实现**这些语义：

| Memory Order 操作 | x86（强内存模型） | ARM（弱内存模型） |
|-------------------|-------------------|-------------------|
| **Plain 读/写** | 无额外指令（x86 天然保证 LoadLoad、LoadStore、StoreStore 有序） | 无额外指令，但允许大量重排序 |
| **Opaque 读/写** | 同 Plain，无额外开销 | 同 Plain，无额外开销（仅编译器层面保证） |
| **Acquire 读** | 无额外指令（x86 读本身具有 Acquire 语义） | `LDAR` 指令或 `DMB ISHLD` 屏障 |
| **Release 写** | 无额外指令（x86 只允许 StoreLoad 重排） | `STLR` 指令或 `DMB ISH` 屏障 |
| **Volatile 写** | `LOCK` 前缀或 `MFENCE`（需 StoreLoad 屏障） | `DMB ISH` 全屏障 |
| **Volatile 读** | 无额外指令（同 Acquire 读） | `LDAR` 或 `DMB ISHLD` |

> **关键洞察**：x86 架构遵循 **TSO（Total Store Order）** 模型，只允许 StoreLoad 一种重排序，因此 Plain、Opaque 和 Acquire 读/Release 写在实际运行时几乎零开销。而 ARM 采用弱内存模型，允许所有类型的重排序，不同 Memory Order 之间才有显著的性能差异。

这也是为什么在 x86 上 `volatile` 写比 `volatile` 读更重（写后需插入 StoreLoad 屏障），而 Acquire/Release 模式在 x86 上性能极佳的原因所在。如果程序需要部署到 ARM 服务器，合理选择 Memory Order 的意义就更为明显了。

### Lock-Free 编程实践

#### AQS 中的 VarHandle 应用

`AbstractQueuedSynchronizer`（AQS）是 `java.util.concurrent` 的基石，其内部的 CLH 队列是展示 VarHandle 分级使用的最佳范例。

CLH 节点的字段定义：

```java
static final class Node {
    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;

    // CAS 操作设置 next 字段
    final boolean compareAndSetNext(Node expect, Node update) {
        return NEXT.compareAndSet(this, expect, update);
    }

    // 对 prev 字段使用 Plain 模式写入
    final void setPrevRelaxed(Node p) {
        PREV.set(this, p);
    }

    private static final VarHandle NEXT;
    private static final VarHandle PREV;
    private static final VarHandle THREAD;
    private static final VarHandle WAITSTATUS;

    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            NEXT = l.findVarHandle(Node.class, "next", Node.class);
            PREV = l.findVarHandle(Node.class, "prev", Node.class);
            THREAD = l.findVarHandle(Node.class, "thread", Thread.class);
            WAITSTATUS = l.findVarHandle(Node.class, "waitStatus", int.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }
}
```

入队（enqueue）的核心逻辑：

```java
private Node addWaiter(Node mode) {
    Node node = new Node(mode);
    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);          // Plain 写：node 尚未发布
            if (compareAndSetTail(oldTail, node)) { // CAS（volatile 语义）
                oldTail.next = node;                // volatile 写：发布 node
                return node;
            }
        } else {
            initializeSyncQueue();
        }
    }
}
```

仔细分析这里的内存序选择逻辑：

- **`setPrevRelaxed`（Plain 写）**：当前 `node` 是新创建的局部对象，尚未被任何其他线程看见（还未发布）。此时写入 `prev` 字段不存在竞争，Plain 模式即可，无需 volatile 开销；
- **CAS 设置 `tail`**：`compareAndSet` 本身隐含 volatile 语义，确保多线程竞争下只有一个线程成功修改 `tail`；
- **`oldTail.next = node`（volatile 写）**：这一步将 `node` 正式发布给所有线程。由于 `next` 字段是 `volatile` 的，后续出队线程通过 volatile 读就能看到完整的 `node` 对象。

而出队（dequeue）只在被唤醒的单个线程中执行：

```java
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

由于出队没有多线程竞争（只有持有锁的线程才能执行），`setHead` 中的字段清理直接使用普通赋值即可，无需 VarHandle。

> **设计智慧**：AQS 对同一变量的不同访问阶段使用了不同强度的内存序——对象未发布时用 Plain 节省开销，发布时用 volatile 保证可见性，出队时无竞争则回归普通赋值。这种"**按需施加，够用就好**"的哲学，正是 Memory Order 分级设计的精髓。

#### StampedLock 中的 acquireFence

`StampedLock` 提供了乐观读模式，允许读线程在不加锁的情况下读取数据，读取完毕后通过 `validate` 检查期间是否有写操作发生。这里涉及到一个微妙的内存序问题：

```java
class Point {
    private double x, y;                       // 普通变量，非 volatile
    private final StampedLock sl = new StampedLock();

    double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead();   // 获取乐观读戳记
        double currentX = x;                   // Plain 读
        double currentY = y;                   // Plain 读
        if (!sl.validate(stamp)) {             // 验证是否有写操作介入
            // 重试...
        }
        return Math.hypot(currentX, currentY);
    }
}
```

问题在于：`currentX` 和 `currentY` 是 Plain 读，而 `validate(stamp)` 内部会读取 `volatile long state` 字段。根据前文 [限制 Reorderings](#volatile-关键字) 的分析，**Normal Load 与 Volatile Load 之间不会插入任何内存屏障**，这意味着 `x`、`y` 的读取可能被重排序到 `validate` 之后。如果恰好在此期间发生了写操作修改了 `x` 或 `y`，那么 `validate` 读到的新戳记与 `x`、`y` 读到的旧值就会不一致。

`StampedLock.validate` 的源码给出了答案：

```java
private transient volatile long state;

public boolean validate(long stamp) {
    VarHandle.acquireFence();              // 插入 Acquire 屏障
    return (stamp & SBITS) == (state & SBITS);
}
```

`VarHandle.acquireFence()` 是一个**裸 Acquire 屏障**——它确保 **Fence 之后的内存操作（读取 `state`）不会重排序到 Fence 之前的内存操作（读取 `x`、`y`）之前**。这就保证了 `x`、`y` 一定在 `state` 的验证之前完成读取。简言之：

- **不加 acquireFence**：读 `x` → 读 `y` → 验证 `state` 这三步可能乱序，写操作可能在读 `x` 之后、验证 `state` 之前完成，导致验证通过但数据不一致；
- **加了 acquireFence**：读 `x` → 读 `y` → acquireFence → 验证 `state` 是严格有序的。如果验证通过，说明写操作（会修改 `state` 并触发 volatile 写）一定在 acquireFence 之前就已结束，那么 `x` 和 `y` 读到的就是写操作之前或之后的完整值。

> 这个例子完美说明了 Memory Order 在实际工程中的价值：在 `x` 和 `y` 两个普通变量和 `validate` 的 volatile 读之间，只需要一道单向的 Acquire 屏障就足够保证正确性，而不是将 `x`、`y` 都声明为 `volatile`（那样会显著增加写侧的负担）。

### VarHandle、Unsafe 与 Atomic 类的选择

前文已经对比了 VarHandle 与 Unsafe 的基本差异。从 Memory Order 的角度进一步补充：

| 维度 | Atomic 类 | VarHandle | Unsafe |
|------|-----------|-----------|--------|
| **API 层级** | 最高（对象级封装） | 中（字段级访问） | 最低（内存偏移量） |
| **内存序控制** | 仅 volatile | Plain、Opaque、Acquire/Release、Volatile | 仅 volatile + 部分 fence |
| **类型安全** | 编译期检查 | 运行时检查（无 unchecked 警告） | 无 |
| **锁自由编程** | 不适合自定义结构 | 首选 | 可行但不应使用 |
| **可移植性** | 完美 | 完美 | 依赖 JVM 内部实现 |

选择建议的决策逻辑：

1. **简单计数器/累加器**：直接用 `AtomicInteger`、`AtomicLong`、`LongAdder`（高竞争下更优）。你不需要也不应该自己用 VarHandle 来实现已有标准库覆盖的功能；
2. **自定义无锁数据结构**：使用 VarHandle。`compareAndSet`、`getAcquire`、`setRelease` 的组合能让你精确控制每个字段的内存序，而不需要为整个结构付出全 `volatile` 的开销；
3. **字段偏移量操作**（如 `Unsafe.objectFieldOffset`）：VarHandle 已完全替代。通过 `MethodHandles.lookup().findVarHandle()` 获取，类型安全且不依赖内部 API；
4. **直接内存操作**（堆外内存）：目前的 VarHandle 尚不能完全替代 Unsafe 的 `allocateMemory`/`freeMemory` 等堆外内存操作。JDK 后续版本通过 `MemorySegment`（Project Panama）来填补这一空白。

> **核心原则**：在 Memory Order 的选择上，始终坚持"**用最弱但足够的内存序**"。这个原则与前面 ConcurrentHashMap 演进中"优先无锁（CAS），必要时加最细粒度的锁"的思路一脉相承——都是通过更精细的控制来逼近性能与正确性的最优平衡点。

***

## 总结

JMM 是理解 Java 并发编程的基石。掌握以下几点，可以帮助你在生产环境中写出正确的并发代码：

- **happens-before 原则**是判断线程安全的核心依据，8 条规则覆盖了 Java 中所有同步机制；
- **volatile** 不仅保证可见性，更重要的是通过内存屏障禁止指令重排序，这是 DCL 正确性的关键；
- **4 种内存屏障**（LoadLoad、LoadStore、StoreStore、StoreLoad）是实现有序性的底层机制；
- **ConcurrentHashMap 的演进**反映了并发设计的趋势：优先无锁（CAS），必要时使用最细粒度的锁；
- **CAS 的 ABA 问题**在实现无锁数据结构时必须考虑，`AtomicStampedReference` 是标准解决方案；
- **VarHandle** 是 JDK 9+ 中替代 Unsafe 的标准方式，在需要精细控制内存序的场景下值得使用。

> 并发编程的本质是在**性能**和**正确性**之间寻找平衡。理解 JMM 的语义，才能在不牺牲正确性的前提下，做出合理的性能优化决策。
