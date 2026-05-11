## 类加载机制概述

Java 虚拟机把描述类的数据从 Class 文件加载到内存，并对数据进行**验证、准备、解析和初始化**，最终形成可以被虚拟机直接使用的 Java 类型。与编译时连接的语言不同，Java 的**加载、连接和初始化**都是在程序运行期间完成的。这种**动态加载**和**动态连接**的设计虽然带来了轻微的性能开销，却赋予了 Java 极高的扩展性和灵活性。

> **关键约定**：后文中的"类型"同时涵盖类和接口；"Class 文件"泛指任何二进制字节流，不限于磁盘文件，也包括网络、内存或动态生成的字节流。

***

## 类的生命周期

一个类型从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期经历七个阶段：

**加载（Loading）→ 验证（Verification）→ 准备（Preparation）→ 解析（Resolution）→ 初始化（Initialization）→ 使用（Using）→ 卸载（Unloading）**

其中验证、准备、解析三个阶段统称为**连接（Linking）**。

加载、验证、准备、初始化和卸载这五个阶段的顺序是确定的，必须按部就班地**开始**。解析阶段则不一定：某些情况下可以在初始化之后再开始，这是为了支持 Java 的运行时绑定特性（动态绑定 / 晚期绑定）。需要注意的是，这些阶段通常是**交叉混合进行**的，在一个阶段执行的过程中调用、激活另一个阶段。例如加载阶段尚未完成时，连接阶段的部分动作（如一部分字节码文件格式验证）可能已经开始，但两个阶段的开始时间仍然保持着固定的先后顺序。

***

## 类加载的时机

### 六种主动引用（必须初始化）

《Java 虚拟机规范》严格规定了**有且只有**六种情况必须立即对类进行初始化：

1. **new / getstatic / putstatic / invokestatic 指令**：使用 new 关键字实例化对象、读取或设置类型的静态字段（被 final 修饰且编译期常量已入池的字段除外）、调用类型的静态方法；
2. **反射调用**：使用 java.lang.reflect 包的方法对类型进行反射调用时；
3. **父类初始化**：初始化类时，若其父类尚未初始化，会先触发父类的初始化（**接口不需要**父接口先初始化）；
4. **虚拟机启动主类**：虚拟机启动时，包含 main() 方法的主类会被初始化；
5. **MethodHandle 解析**：JDK 7 动态语言支持中，MethodHandle 实例解析为 REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial 四种类型的方法句柄时；
6. **接口默认方法**：接口中定义了 default 方法，其实现类发生初始化时，该接口需要先被初始化。

### 三种被动引用（不会初始化）

除上述六种主动引用外，所有其他引用类型的方式都不会触发初始化，称为**被动引用**：

1. **通过子类引用父类的静态字段**：只会触发父类初始化，不会触发子类初始化；
2. **通过数组定义引用类**：`new SuperClass[10]` 不会触发 SuperClass 的初始化，但会触发虚拟机自动生成的数组类（如 `[LSuperClass`）的初始化；
3. **引用编译期常量**：编译期通过**常量传播优化**已将常量值存入调用类的常量池中，运行时引用的是调用类自身的常量池，不会触发定义类的初始化。

被动引用的第三条规则在优化常量访问的同时，也可能带来隐蔽的陷阱：如果将一个常量从编译期可确定的值改为运行期计算值（如从字面量改为方法返回值），原本不触发初始化的代码路径突然会触发初始化，可能导致意料之外的类加载顺序问题。

> **生产注意**：接口在初始化时并不要求父接口全部完成初始化，只有真正使用到父接口时才会触发。这与类的初始化规则不同，排查初始化顺序问题时容易忽略。

***

## 类加载的五个阶段

### 加载

加载阶段虚拟机需要完成三件事：

- 通过类的全限定名获取定义此类的**二进制字节流**；
- 将字节流代表的静态存储结构转换为方法区的**运行时数据结构**；
- 在内存中生成代表此类的 java.lang.Class 对象，作为方法区数据的访问入口。

加载阶段的灵活性是类加载机制中最具扩展性的一环。字节流来源不限于 Class 文件，还包括 ZIP / JAR、网络、动态代理生成、JSP 编译、数据库读取甚至加密文件解密后的字节流。加载阶段也是开发人员可控性最强的阶段，可以通过自定义类加载器重写 `findClass()` 方法来控制字节流的获取方式，赋予应用程序获取运行代码的动态性。

数组类由虚拟机直接在内存中动态构造，不通过类加载器创建。但数组的元素类型仍由类加载器完成加载，数组类的可访问性与元素类型一致。

### 验证

验证是连接的第一步，目的是确保 Class 文件的字节流符合《Java 虚拟机规范》的全部约束，保护虚拟机自身安全。验证阶段包含四个子阶段：

| 验证阶段 | 检查内容 |
|---------|---------|
| **文件格式验证** | 魔数、版本号、常量池类型、索引指向是否合法等 |
| **元数据验证** | 语义分析：是否有父类、是否继承了 final 类、是否实现了接口方法等 |
| **字节码验证** | 数据流和控制流分析：类型转换是否合法、跳转指令是否指向正确位置等 |
| **符号引用验证** | 解析阶段前的校验：类 / 字段 / 方法是否存在、访问权限是否合法等 |

字节码验证是整个验证过程中最复杂的环节。早期的类型推导需要对方法体进行复杂的数据流分析，验证时间长。JDK 6 之后引入了**类型检查（Type Checking）**替代类型推导，通过 StackMapTable 属性描述方法体中各基本块开始时本地变量表和操作栈的状态，虚拟机只需检查状态是否合法即可，显著缩短了验证时间。这也是 JDK 6 要求编译器生成 StackMapTable 属性的原因。

> **生产调优**：验证阶段在类加载过程中占据了相当大的性能比重。对于可信代码，可通过 `-noverify` 或 `-Xverify:none` 关闭验证以缩短启动时间（**不推荐在生产环境使用**，除非对字节来源有绝对把控）。在 GraalVM Native Image 中，由于类在构建期已完成加载和验证，运行时完全跳过类加载阶段，这也是 Native Image 启动极快的原因之一。

### 准备

准备阶段为类变量（静态变量）分配内存并设置**系统要求的初始零值**。JDK 7 之前分配在永久代，**JDK 8 之后随 Class 对象存储在堆中**。

注意区分：**准备阶段仅设置零值**，真正的初始值赋值在初始化阶段执行 `<clinit>()` 时完成。但如果类字段被 `static final` 修饰且属于**编译期常量**，则准备阶段会直接赋值为常量值。

```java
public static int value = 123;      // 准备阶段 value = 0
public static final int CONST = 123; // 准备阶段 CONST = 123
```

> **面试常考点**：`static final` 修饰的常量如果在编译期无法确定值（如依赖方法调用或运行时计算），则仍然只在准备阶段设零值，初始化阶段才赋值。这也是区分"编译期常量"和"运行期常量"的关键。

### 解析

解析是将常量池内的**符号引用**替换为**直接引用**的过程。符号引用是一组描述符，与虚拟机内存布局无关；直接引用是直接指向目标内存地址的指针、偏移量或句柄。

解析阶段在虚拟机实现中有较大的灵活性，虚拟机可以在类加载器加载时就完成解析，也可以等到**真正使用符号引用之前**才进行解析。但 **`invokedynamic` 指令**是个例外：必须等到程序执行到这条指令时才进行解析。Lambda 表达式和接口默认方法底层均依赖 `invokedynamic`。

解析动作主要针对类 / 接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符这七类符号引用。

### 初始化

初始化阶段是类加载过程的最后一步，执行类构造器 **`<clinit>()`** 方法。只有主动引用才会触发初始化。

#### `<clinit>()` 方法规则

- 由 Javac 编译器自动生成，由**类中所有类变量的赋值动作**和**静态语句块中的语句**合并而成，顺序与源码中出现顺序一致；
- 子类的 `<clinit>()` 执行前，虚拟机会保证父类的 `<clinit>()` 已经执行完毕；
- 不是必需的，没有静态字段赋值和静态语句块的类不会生成 `<clinit>()`；
- 接口也可以生成 `<clinit>()`（用于初始化接口成员变量），但执行时**不需要先执行父接口的 `<clinit>()`**；
- 虚拟机会保证 `<clinit>()` 在多线程环境中被**正确地加锁同步**，多个线程同时初始化一个类时，只有一个线程能执行 `<clinit>()`，其他线程阻塞等待。

> **死锁场景**：`<clinit>()` 的线程安全机制可能导致死锁。若类 A 的 `<clinit>()` 中触发了类 B 的初始化，而类 B 的 `<clinit>()` 中又触发了类 A 的初始化，且两个线程分别持有 A 和 B 的初始化锁，就会形成死锁。生产环境中的类初始化循环依赖需要特别警惕。

一个容易忽略的细节是：`<clinit>()` 方法中只能访问定义在静态语句块之前的静态变量，定义在后面的静态变量只能赋值不能访问。Javac 编译器会对此进行合法性检查，这是 Java 语法层面的限制，也是 `<clinit>()` 执行顺序与源码顺序一致的具体体现。

***

## 类加载器与双亲委派模型

类加载器负责"通过一个类的全限定名获取定义此类的二进制字节流"。每一个类加载器都有独立的类名称空间，**任意一个类都必须由加载它的类加载器和这个类本身才能确定其在 Java 虚拟机中的唯一性**。换言之，同一个 Class 文件被不同的类加载器加载，在虚拟机中就是两个不同的类。这也是 Tomcat 能够实现 Web 应用隔离的根本原因：每个 WebApp 使用独立的类加载器加载相同的类，彼此互不干扰。

### 三层类加载器架构

JDK 9 之前，Java 采用三层类加载器架构：

| 类加载器 | 实现语言 | 加载范围 | 说明 |
|---------|---------|---------|------|
| **Bootstrap ClassLoader** | C++（HotSpot） | `<JAVA_HOME>/lib` 目录下能被虚拟机识别的类库 | 无法被 Java 程序直接引用，获取时返回 null |
| **Extension ClassLoader** | Java | `<JAVA_HOME>/lib/ext` 目录或 `java.ext.dirs` 指定路径 | JDK 9 后被 Platform ClassLoader 取代 |
| **Application ClassLoader** | Java | 用户类路径（`CLASSPATH`）上的类 | 也称为系统类加载器，`ClassLoader.getSystemClassLoader()` 返回值 |

JDK 9 引入模块化后，类加载器架构也做出了相应调整。模块化系统下的类加载不再单纯依赖双亲委派，而是优先将类加载请求委派给该类归属模块的类加载器完成：

| 类加载器 | 加载范围 | 说明 |
|---------|---------|------|
| **Bootstrap ClassLoader** | java.base 等核心模块 | JDK 9 后 HotSpot 也采用 Java 类与虚拟机配合实现 |
| **Platform ClassLoader** | 平台模块（如 java.sql、java.xml 等） | 替代 Extension ClassLoader |
| **Application ClassLoader** | 应用程序模块和类路径 | 加载用户代码和第三方依赖 |

### 双亲委派的工作过程

双亲委派模型要求：除 Bootstrap ClassLoader 外，其余类加载器都应有自己的父加载器。

**工作过程**：一个类加载器收到类加载请求时，首先不会自己去尝试加载，而是把请求**委派给父类加载器**去完成；每一层类加载器都是如此，只有当父加载器反馈自己无法完成加载时，子加载器才会尝试自己去完成加载。

这个逻辑实现在 `java.lang.ClassLoader.loadClass()` 方法中：先检查是否已加载，未加载则委派父加载器，父加载器为空时默认使用 Bootstrap ClassLoader；父加载器加载失败抛出 `ClassNotFoundException` 后，才调用自己的 `findClass()` 方法尝试加载。

### 为什么双亲委派是安全的

双亲委派的核心价值在于**保证基础类型的一致性和安全性**。Java 的核心类库（如 `java.lang.Object`、`java.lang.String`）由 Bootstrap ClassLoader 加载，无论哪个应用程序都无法通过自定义类加载器来加载一个同包同名的"恶意"核心类。如果自定义类加载器尝试加载 `java.lang.System` 或 `java.lang.String`，最终都会被委派到 Bootstrap ClassLoader，加载的是真正的 JDK 核心类。

即使强行使用 `defineClass()` 加载以 `java.lang` 开头的类，虚拟机会抛出 `java.lang.SecurityException: Prohibited package name: java.lang`。

> **安全边界**：双亲委派从架构上隔离了用户代码对核心类库的篡改，这是 Java 安全模型的基石之一。自定义类加载器即便存在漏洞，也难以突破这层边界去替换核心类。

值得注意的是，双亲委派模型并非绝对不可突破。如果直接在启动参数中使用 `-Xbootclasspath/a:` 将自定义 Jar 追加到 Bootstrap ClassLoader 的搜索路径中，就可以让 Bootstrap 加载用户类。这在某些需要替换 JDK 内置实现的场景（如替换 `java.util.zip.ZipFile`）中有实际应用，但也带来了极大的安全风险，生产环境中应严格管控。

***

## 打破双亲委派模型

双亲委派模型是推荐而非强制。历史上主要出现过三次较大规模的"被破坏"：

### 第一次：JDK 1.2 的向前兼容

双亲委派在 JDK 1.2 才被引入，但 ClassLoader 抽象类从 Java 第一个版本就已存在。为了兼容已有自定义类加载器代码，设计团队无法阻止 `loadClass()` 被覆盖，只能在 ClassLoader 中新增 `findClass()` 方法，引导用户将类加载逻辑写在 `findClass()` 中而非 `loadClass()` 中。

### 第二次：SPI 机制与线程上下文类加载器

这是双亲委派模型**自身的缺陷**导致的。双亲委派解决了基础类型由上层加载器加载的一致性问题，但基础类型如果需要**回调用户代码**该怎么办？

典型的例子是 **JDBC**：`java.sql.Driver` 接口由 Bootstrap ClassLoader 加载，但具体的数据库驱动实现（如 `com.mysql.cj.jdbc.Driver`）在应用程序的 ClassPath 下，Bootstrap ClassLoader 无法加载。同样的困境也存在于 JNDI、JCE、JAXB、JBI 等 SPI 场景。

解决方案是引入**线程上下文类加载器（Thread Context ClassLoader）**：通过 `Thread.currentThread().getContextClassLoader()` 获取（默认为 Application ClassLoader），SPI 使用它去加载实现类，如 `Class.forName("com.mysql.cj.jdbc.Driver", true, cl)`。

线程上下文类加载器可以通过 `Thread.setContextClassLoader()` 设置。这是一种**父加载器请求子加载器完成类加载**的逆向行为，打破了双亲委派的层次结构。JDK 6 提供的 `java.util.ServiceLoader` 配合 `META-INF/services` 配置，为 SPI 加载提供了相对优雅的解决方案。

### 第三次：OSGi 与模块化热部署

OSGi 的每个 Bundle 都有自己的类加载器，类加载器结构不再是树状而是**网状**。收到类加载请求时，OSGi 的搜索顺序为：

1. 以 `java.*` 开头的类，委派给父加载器；
2. 委派列表中的类，委派给父加载器；
3. Import 列表中的类，委派给 Export 该类的 Bundle 的类加载器；
4. 查找当前 Bundle 的 ClassPath，使用自己的类加载器；
5. 查找 Fragment Bundle；
6. 查找 Dynamic Import 列表的 Bundle。

只有前两点符合双亲委派原则，其余均在**平级类加载器**之间进行。OSGi 为了实现热部署付出了高复杂度的代价，业界对其优劣存在争议，但其在类加载器运用上的设计思路值得深入研究。

***

## 实战视角：生产环境中的类加载问题

### 启动慢：类加载导致的启动瓶颈

大型 Spring Boot 应用的启动时间中，**类加载和验证**往往占据显著比例。以下是常见的优化思路：

| 优化手段 | 适用场景 | 注意事项 |
|---------|---------|---------|
| **AppCDS** | JDK 10 基础 / JDK 12+ 动态归档，启动类路径稳定 | 生成共享归档文件，多 JVM 实例共享元数据 |
| **Class Data Sharing (CDS)** | JDK 5+ 基础支持，JDK 12+ 扩展至 AppCDS | 仅对 Bootstrap / App ClassLoader 加载的类有效 |
| **GraalVM Native Image** | 云原生、Serverless 场景 | AOT 编译，启动毫秒级，但丧失部分动态性 |
| **延迟加载非必要类** | 启动时不需要的组件 | 使用 `@Lazy`、条件装配减少初始加载量 |
| **并行类加载** | 多核环境 | JDK 7+ 的 `ClassLoader` 已支持并行加载 |

> **案例**：某微服务启动耗时 45 秒，其中 18 秒花在类加载。通过 AppCDS 生成共享归档后，启动时间降至 28 秒，类加载阶段缩短到 6 秒左右。

### NoClassDefFoundError vs ClassNotFoundException

这两个异常是生产环境最常见的类加载问题，排查时需要明确区分：

| 异常 | 触发时机 | 典型原因 | 排查思路 |
|-----|---------|---------|---------|
| **ClassNotFoundException** | 显式加载时（`Class.forName()`、`ClassLoader.loadClass()`） | 类路径缺失、Jar 包未引入、类名拼写错误 | 检查 `CLASSPATH` / 依赖树（`mvn dependency:tree`） |
| **NoClassDefFoundError** | 隐式链接时（new / 继承 / 实现接口） | 编译期存在但运行时缺失、类加载失败被缓存、Jar 包版本冲突 | 检查是否因异常导致首次加载失败；排查 `LinkageError` 链 |

**关键区别**：`NoClassDefFoundError` 通常意味着这个类在**编译期存在**，但运行时因为某种原因（如静态初始化失败、依赖缺失、版本不兼容）导致链接失败。Java 虚拟机会**缓存加载失败的结果**，后续对该类的引用直接抛出 `NoClassDefFoundError`，而不再尝试重新加载。

> **排查技巧**：遇到 `NoClassDefFoundError` 时，优先查看应用启动日志中是否有更早的 `ExceptionInInitializerError` 或 `ClassNotFoundException`。如果某个类的 `<clinit>()` 抛出了异常，后续所有对该类的引用都会变成 `NoClassDefFoundError`。另一个常见原因是 Fat Jar 打包时不同版本的同名类被覆盖，导致运行时加载到的类版本与编译时不一致。

### `<clinit>()` 死锁场景

一个典型的生产死锁案例：类 A 的 `<clinit>()` 中触发了类 B 的初始化，而类 B 的 `<clinit>()` 中又触发了类 A 的初始化。当线程 1 初始化类 A、线程 2 初始化类 B 时，两个线程分别持有对方的初始化锁，形成死锁。线程 dump 中会显示线程卡在 `java.lang.Class.init()` 的 monitor entry 等待上。

> **预防建议**：避免在静态初始化块中触发其他类的初始化，尤其是存在循环依赖风险的模块。Spring 的 `@DependsOn` 可以显式控制初始化顺序，但根本解法是重构消除循环依赖。

### Tomcat / Jetty 的类加载隔离机制

Web 容器需要解决的核心问题是：**同一个容器的不同 Web 应用之间不能互相干扰**，包括类隔离和库隔离。

**Tomcat 的类加载器架构**：

| 类加载器 | 加载范围 | 双亲委派方向 |
|---------|---------|------------|
| **Bootstrap** | JVM 核心类库 | 顶层 |
| **System** | `CLASSPATH` | 向上委派 |
| **Common** | `$CATALINA_BASE/lib`（共享库） | 向上委派 |
| **Catalina** | Tomcat 自身类 | 向上委派到 Common |
| **Shared** | 多 Web 应用共享（已废弃） | 向上委派到 Common |
| **WebApp** | `WEB-INF/classes` 和 `WEB-INF/lib` | **先本地后委派**（打破双亲委派） |

Tomcat 的 **WebApp ClassLoader** 打破了双亲委派：**优先加载 Web 应用本地的类**，本地找不到时才向上委派。这保证了不同 Web 应用可以使用不同版本的 Spring、Hibernate 等库而不会冲突。每个 Web 应用看到的类环境是独立的，即使两个应用分别依赖 Spring 4 和 Spring 5，也不会相互影响。

> **Tomcat 配置**：`conf/catalina.properties` 中的 `server.loader` 和 `shared.loader` 已废弃，推荐使用 `Common` 加载器统一处理共享依赖。

**Jetty 的类加载策略**类似，通过 `WebAppClassLoader` 实现。Jetty 还提供了 `WebAppContext.setParentLoaderPriority(true)` 来切换回标准双亲委派模式（通常不推荐，会破坏隔离性）。

> **容器对比**：Tomcat 和 Jetty 都实现了 Web 应用隔离，但默认策略不同。Tomcat 严格遵循"先本地后父加载器"，Jetty 默认也是子加载器优先，但可以通过配置切换。了解容器的类加载策略对于排查 `NoClassDefFoundError` 和 `ClassCastException` 至关重要，尤其是在同一个容器部署多个应用且共享部分库时。

### Spring Boot 的 LaunchedURLClassLoader

Spring Boot 可执行 Jar 采用**嵌套 Jar** 结构（`BOOT-INF/classes/` 和 `BOOT-INF/lib/`），标准 `URLClassLoader` 无法直接加载嵌套 Jar 中的类。Spring Boot 自定义了 `LaunchedURLClassLoader`：

- 继承 `URLClassLoader`，支持从嵌套 Jar 中加载类和资源；
- 使用 `jar:file:/path/to/app.jar!/BOOT-INF/lib/nested.jar!/` 形式的 URL；
- 加载顺序：`BOOT-INF/classes` > `BOOT-INF/lib` 中的 Jar > 父加载器。

> **排查注意**：Spring Boot 应用在生产环境如果使用了 `-Dloader.path` 加载外部 Jar，需要确认外部 Jar 的加载顺序和版本优先级，避免与 `BOOT-INF/lib` 内的依赖产生冲突。另外，Spring Boot 的 `PropertiesLauncher` 模式支持通过 `loader.path` 加载外部依赖目录，这在需要动态替换依赖而不重新打包的场景中很有用，但也增加了类加载的复杂度。

### Jar Hell 与依赖冲突解决

**Jar Hell** 指类路径中存在多个版本的同一个 Jar，或不同 Jar 包含相同的包名类名，导致类加载器加载到"错误"的版本。

**常见症状**：

- `NoSuchMethodError` / `NoSuchFieldError`：运行时加载的类版本与编译时不一致；
- `ClassCastException`：同一个类被不同类加载器加载，强制转换失败；
- `AbstractMethodError`：接口实现类编译时实现了方法 A，但运行时接口版本要求方法 B。

**排查工具与思路**：

| 工具 / 命令 | 用途 |
|-----------|------|
| `mvn dependency:tree` / `gradle dependencies` | 查看依赖树，定位冲突版本 |
| `jar -tf app.jar | grep ClassName` | 查找包含某类的 Jar |
| `arthas` 的 `sc` / `sm` 命令 | 查看已加载的类和类加载器 |
| `mvn dependency:analyze-duplicate` | 查找重复依赖 |

**解决思路**：

1. **Maven `<dependencyManagement>` / Gradle `strictly`**：统一锁定版本；
2. **`<exclude>` 排除传递依赖**：排除不需要的版本；
3. **Shade / Relocation**：使用 Maven Shade Plugin 将冲突包 relocate 到不同包名（如 `com.google.common` → `shaded.com.google.common`）；
4. **OSGi / JPMS**：通过模块化实现真正的隔离（重量级，需架构支持）。

> **快速定位**：使用 Arthas 的 `sc -d com.example.ConflictingClass` 可以查看类的类加载器、加载来源 Jar 和 HashCode，迅速判断是否存在多版本共存。另一个常用命令是 `classloader -t`，可以打印出类加载器的继承树，直观展示当前 JVM 中的类加载器层次结构。

***

## JDK 9 模块化系统（JPMS）

JDK 9 引入的 Java Platform Module System（JPMS）是实现**可配置封装隔离机制**的重要升级。模块不仅充当代码容器，还包含：

- 依赖其他模块的列表（`requires`）；
- 导出的包列表（`exports`）；
- 开放的包列表（`opens`）；
- 使用的服务列表（`uses`）；
- 提供的服务实现列表（`provides ... with`）。

### 模块路径 vs 类路径

| 维度 | 类路径（ClassPath） | 模块路径（ModulePath） |
|-----|-------------------|---------------------|
| **依赖检查** | 运行时才发现缺失 | 启动时验证依赖完备性，缺失则启动失败 |
| **访问控制** | public 对所有代码可见 | public 仅在导出的包中对依赖模块可见 |
| **隔离性** | 无隔离，Jar 间完全可见 | 强隔离，未导出的包不可访问 |
| **兼容性** | JDK 9+ 仍完全支持 | 新特性，需显式适配 |

### 三种模块形态

| 形态 | 说明 | 可见性 |
|-----|------|--------|
| **具名模块（Named Module）** | 包含 `module-info.class` 的模块 | 只能访问 `requires` 声明的依赖模块和导出的包 |
| **自动模块（Automatic Module）** | 无 `module-info.class` 但放在模块路径上的 Jar | 默认导出所有包，默认依赖所有其他模块 |
| **匿名模块（Unnamed Module）** | 类路径上的所有 Jar 被视为一个匿名模块 | 几乎无隔离，可访问所有导出包 |

模块描述符 `module-info.java` 示例：声明模块名、依赖（`requires`）、导出包（`exports`）、开放包（`opens`）以及服务提供（`provides ... with`）。

```java
module com.example.myapp {
    requires java.sql;
    requires org.slf4j;

    exports com.example.myapp.api;
    exports com.example.myapp.model to com.example.client;

    opens com.example.myapp.internal to org.hibernate;

    uses com.example.myapp.spi.Plugin;
    provides com.example.myapp.spi.Plugin with com.example.myapp.internal.MyPlugin;
}
```

### 可读性与可访问性

JPMS 引入了两层访问控制：

- **可读性（Readability）**：模块 A `requires` 模块 B，则 A 对 B **可读**；
- **可访问性（Accessibility）**：模块 B 必须 `exports` 包 P，模块 A 才能访问 P 中的 public 类型。

即使模块 A 对模块 B 可读，如果 B 没有导出目标包，A 也无法访问其中的类。这在 JDK 9 之前是不存在的：只要类是 public 的，任何代码都能访问。

> **迁移注意**：JDK 9 后，JDK 内部类（如 `sun.misc.Unsafe`、`sun.nio.ch` 包下的类）不再默认开放。直接使用这些内部 API 的应用在迁移到 JDK 9+ 时会遇到 `IllegalAccessError`。这也是很多依赖 `sun.misc.Unsafe` 的库（如 Netty、Cassandra 的早期版本）在 JDK 9+ 需要适配的根本原因。后续 JDK 版本通过 `VarHandle` 和 `MemorySegment` 等标准 API 逐步替代了 Unsafe 的核心功能。

***

## JDK 17：强封装与 `--add-opens` 问题

JDK 16 起，**强封装（Strong Encapsulation）** 成为默认行为；JDK 17 作为 LTS 版本，全面执行这一策略。所有 JDK 内部元素（包括包和模块）默认都对代码**不可访问**，即使通过反射也不行。

### `--add-opens` 与 `--add-exports`

当代码需要通过反射访问未开放的包时，需要使用 JVM 参数显式开放，如 `java --add-opens java.base/java.lang=ALL-UNNAMED -jar app.jar`。

| 参数 | 作用 | 持久性 |
|-----|------|--------|
| `--add-exports` | 导出包，允许编译期和运行期访问 | 运行时 |
| `--add-opens` | 开放包，仅允许运行期反射访问 | 运行时 |

### 升级 JDK 17 时的常见问题

以下是在生产环境中迁移到 JDK 17 时高频出现的类加载 / 访问问题：

**1. 序列化框架（如 Kryo、FST）**

Kryo 等序列化库常通过反射访问 `sun.nio.ch` 或 `java.lang` 包下的内部类。升级 JDK 17 后会抛出 `InaccessibleObjectException`，提示 `module java.base does not "opens ..." to unnamed module`。

**解决**：添加 `--add-opens` 参数，或升级到框架的最新版本（通常已适配 JDK 17）。

**2. Spring / Hibernate 等框架的内部访问**

Spring 在 `ReflectionUtils` 中访问 JDK 内部类，Hibernate 在字节码增强时访问 `java.lang` 包。Spring Framework 5.3+ 和 Spring Boot 2.5+ 已适配 JDK 17，但旧版本需要手动添加 opens。

**3. 测试框架（Mockito、PowerMock）**

Mockito 的 inline mock maker 需要访问 `java.instrument` 模块，PowerMock 因大量依赖内部 API 在 JDK 17 下几乎无法使用。

**4. 第三方库的 `setAccessible(true)`**

大量遗留库在反射时无条件调用 `setAccessible(true)`。JDK 17 下这会抛出 `InaccessibleObjectException`。需要评估：

- 库是否有支持 JDK 17 的新版本；
- 是否可以通过 `--add-opens` 临时绕过；
- 是否必须寻找替代方案。

> **最佳实践**：生产环境迁移到 JDK 17 前，先在测试环境使用 `--illegal-access=deny`（JDK 17 默认值）运行全量测试，收集所有非法访问点。优先升级依赖版本，仅在必要时使用 `--add-opens` 作为临时过渡方案。避免在生产脚本中堆砌数十个 `--add-opens`，这往往意味着依赖版本过旧或代码存在架构性依赖内部 API 的问题。

### 从 JDK 8 到 JDK 17 的类加载变化速查

| 特性 | JDK 8 | JDK 17 |
|-----|-------|--------|
| **类加载器** | Bootstrap / Ext / App | Bootstrap / Platform / App |
| **核心类库位置** | `rt.jar`、`tools.jar` | 模块化，无 `rt.jar` |
| **扩展目录** | `lib/ext` | 移除，由 Platform ClassLoader 加载平台模块 |
| **内部 API 访问** | 警告或直接允许 | 默认拒绝，需 `--add-opens` |
| **类路径隔离** | 无 | JPMS 强隔离 |
| **AppCDS** | 仅 Bootstrap | 支持 Application ClassLoader |
| **密封类** | 无 | `sealed` / `permits` 语法 |

***

## JDK 21：动态 Agent 加载警告（JEP 451）

JDK 21 引入了 **JEP 451**，当通过 Attach API 向运行中的 JVM **动态加载 Agent** 时，JVM 会发出警告。这一改动的目标是准备在未来的 JDK 版本中默认禁止动态 Agent 加载，以提升运行时代码完整性。

**重要区分：**

| 加载方式 | 是否受影响 | 说明 |
|:--|:--|:--|
| 启动时加载（`-javaagent`、`-agentlib`） | ❌ 不受影响 | 命令行指定的 Agent 不会触发警告 |
| 动态 attach（`jcmd <pid> JVMTI.agent_load` 等） | ⚠️ 触发警告 | 通过 Attach API 在运行期加载 |
| 监控工具（`jcmd`、`jconsole`） | ❌ 不受影响 | 纯监控和管理用途的 attach 不受影响 |

**生产影响：**

- 使用动态 attach 的 APM 工具（部分实现）、Arthas、以及某些动态诊断脚本会在 JDK 21+ 中看到警告日志
- 目前警告不会阻止 Agent 加载，但未来版本可能默认禁止
- 临时缓解：`-XX:+EnableDynamicAgentLoading` 可抑制警告（仍建议迁移到启动时加载）

> **迁移建议**：如果生产环境依赖动态 Agent 加载，应评估将其改为启动时通过 `-javaagent` 加载。对于必须使用动态 attach 的场景（如紧急线上诊断），保留 `-XX:+EnableDynamicAgentLoading` 作为过渡方案，同时关注 OpenJDK 后续版本的完整性策略演进。

## Project Leyden：AOT 类加载与链接（JDK 24~25）

Project Leyden 是 OpenJDK 长期项目，目标是在不牺牲 Java 动态性的前提下，通过 AOT（Ahead-of-Time）缓存显著改善启动速度和预热时间。

### 核心 JEP 与工作流程

| JEP | 版本 | 内容 |
|:--|:--|:--|
| JEP 483 | JDK 24 | Ahead-of-Time Class Loading & Linking：在训练运行中捕获类的加载和链接状态，存入 AOT 缓存 |
| JEP 514 | JDK 25 | Ahead-of-Time Command-Line Ergonomics：简化命令行，训练与生产使用相同参数 |
| JEP 515 | JDK 25 | Ahead-of-Time Method Profiling：将方法执行画像存入缓存，JIT 启动即生成优化代码 |

**使用流程：**

```bash
## 训练运行（自动生成 AOT 缓存）
java -XX:AOTCache=app.aot -cp app.jar com.example.Main

## 生产运行（直接加载缓存）
java -XX:AOTCache=app.aot -cp app.jar com.example.Main
```

JEP 514 的关键简化在于：训练运行和生产运行使用**完全相同的命令行**。如果 `-XX:AOTCache` 指定的缓存文件不存在，JVM 自动创建；如果存在，则自动加载并恢复类的加载/链接状态和方法画像。

### 收益与限制

- **启动时间**：常见提升 2–4 倍，类加载密集型应用（如 Spring Boot）收益最大
- **预热时间**：JIT 编译器无需等待热点代码收集，启动后立即生成优化后的本地代码
- **与 Native Image 的区别**：Leyden **完全保留 Java 动态性**——反射、动态类加载、JVMCI、Lambda 等均不受限制，无需配置可达性元数据
- **限制**：仅支持内置类加载器（Bootstrap / Platform / App），用户自定义类加载器加载的类无法被缓存

> **生产建议**：JDK 25 中 Leyden 的 AOT 缓存流程已趋于成熟，建议对启动时间敏感的 Serverless、容器化微服务和 CI/CD 流水线任务进行评估。训练运行应尽可能模拟真实生产负载，以确保缓存中的方法画像具有代表性。

***

## 总结

类加载机制是 Java 运行时的基石，理解它不仅有助于排查 `ClassNotFoundException`、`NoClassDefFoundError`、`NoSuchMethodError` 等生产问题，更是理解 Spring Boot、Tomcat、OSGi 等框架底层设计的前提。

从双亲委派的安全设计，到 SPI 和线程上下文类加载器的逆向突破；从 Tomcat 的 WebApp 隔离，到 Spring Boot 的嵌套 Jar 加载；从 JDK 9 JPMS 的封装隔离，到 JDK 17 强封装带来的 `--add-opens` 迁移挑战——类加载机制始终伴随着 Java 生态的演进，在扩展性与安全性之间不断寻找平衡。

对于资深后端开发而言，熟练掌握类加载机制，意味着能够快速定位依赖冲突和版本不兼容问题，合理设计模块边界和类加载隔离策略，平稳完成 JDK 升级过程中的兼容性适配，并在启动性能优化中识别类加载瓶颈并采取有效措施。

更深入地看，类加载机制的知识还能帮助开发者在设计插件化架构、热更新系统时做出正确的技术选型——无论是选择自定义 ClassLoader 实现插件隔离，还是引入 OSGi 实现模块化热部署，抑或利用 JPMS 构建原生模块化应用，背后的核心逻辑都建立在类加载机制的深刻理解之上。
