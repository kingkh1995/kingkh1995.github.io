## IOC

### IOC 容器注解

- @Component：注解在类上，@Repository、@Service、@Controller 都注解了 @Component 注解，属于其衍生注解，只有语义区别。@ComponentScan（@SpringBootApplication 上注解）用于扫描指定路径下含有 @Component 注解的类。

- @Bean：注解在方法上，可以指定 initMethod 和 destroyMethod，适用于装配第三方库的类。

- @Configuration：用于配置类，也注解了 @Component 注解，proxyBeanMethods 参数表示是否代理 @Bean 方法，即 @Bean 方法多次调用只返回同一个单例对象，**默认开启，建议关闭，能提高启动速度**。

- @ConfigurationProperties：使用指定的前缀从 properties 文件中绑定属性，可以注解在类和方法上，**需要被 IOC 容器管理**。

- @EnableConfigurationProperties：基于 @Import，导入指定的 @ConfigurationProperties 注解的类。
  
- **@Import**：用于导入 Bean，ImportSelector、DeferredImportSelector、ImportBeanDefinitionRegistrar 必须依赖 @Import 使用，但 @Import 可以单独使用。

### FactoryBean、BeanFactory、ApplicationContext

- FactoryBean：属于设计模式，是创建 Bean 的一种方式，核心方法是 getObject()。

- BeanFactory：即 IOC 容器的最顶级接口，负责配置、创建、管理 Bean。
  
- ApplicationContext：BeanFactory 的子接口，预先加载 Bean，还支持国际化、资源文件读取和事件传播。

### @Scope

配合 @Bean 和 @Component 使用，指定 bean 的作用域。可以新增，使用 ConfigurableBeanFactory 的 registerScope 方法注册，或添加 CustomScopeConfigurer，**除了 Singleton 和 Prototype 之外都会生成 Scope 代理对象**。

- value：等同于 scopeName
  - singleton：单例模式，默认作用域；
  - prototype：多例模式，每次使用都会创建一个实例；
  - request：每次 HTTP 请求都会创建一个实例，仅在当前 HTTP 请求内有效；
  - session：为每个 HTTP 会话创建一个新的实例，仅在当前会话内有效；

- proxyMode：代理模式
  - DEFAULT：按照默认策略
  - NO：无代理 
  - INTERFACES：基于接口代理
  - TARGET_CLASS：基于类代理（cglib）

- @RefreshScope：Spring Cloud Bus 中使用，当配置中心配置更新后会通知服务器刷新作用域为 refresh 的 bean，代理模式为 TARGET_CLASS，对应的 Scope 类型为 RefreshScope，实现为从缓存获取 bean。

### Bean 的生命周期

1. Bean 容器（通常是 ApplicationContext）先创建所有的 PostProcessor 实例；
2. 找到 bean 的定义，通过配置文件或扫描类文件；
3. 使用反射创建 Bean 实例，如果是通过构造器注入，递归创建并注入属性；
4. 如果实现了 BeanNameAware 等 Aware 接口则调用对应方法；
5. 如果容器内存在 BeanPostProcessor，则执行对应的 postProcessBeforeInitialization() 方法；
6. 如果 bean 实现了 InitializingBean，则执行 afterPropertiesSet() 方法；
   - SmartInitializingSingleton 会在所有的单例模式 Bean 创建完成后执行，在 InitializingBean 之后；
7. 执行 Bean 配置的 init 方法；
8. 执行 BeanPostProcessor 的 postProcessAfterInitialization() 方法；
9. 销毁 Bean 时，执行 DestructionAwareBeanPostProcessor 的 postProcessBeforeDestruction() 方法；
10. 如果实现了 DisposableBean 接口，则执行 destroy() 方法；
11. 执行 Bean 配置的 destroy 方法。

- @PostConstruct 和 @PreDestroy 是 jdk 规范，在 **InitDestroyAnnotationBeanPostProcessor** 的 BeforeInitialization 和 BeforeDestruction 阶段执行。
- MergedBeanDefinitionPostProcessor：BeanPostProcessor 处理完成之后，实例化所有 MergedBeanDefinitionPostProcessor，每次 bean 创建完后执行，合并 Bean 定义（postProcessMergedBeanDefinition）先于 postProcessBeforeInitialization 执行。
- BeanDefinitionRegistryPostProcessor：继承 BeanFactoryPostProcessor，用于注册及处理 BeanDefinition，**实现该接口会导致类被提前加载**。
  - postProcessBeanDefinitionRegistry：在 BeanDefinition 创建完成后，合并 Bean 定义之前（因为 MergedBeanDefinitionPostProcessor 属于 BeanPostProcessor），实例化阶段之前执行，用于注册 BeanDefinition，此时未完成属性注入。
  - postProcessBeanFactory：实例化阶段之前执行，ConfigurableListableBeanFactory 创建之后触发，用于修改 ConfigurableListableBeanFactory，也可以使用 registerSingleton 注册 bean，**会直接注册单例 bean**，不经过 BeanPostProcessor 处理。
- ConfigurationPropertiesBeanRegistrar：用于注册所有配置类的 BeanDefinition，最终会创建配置类，但不会注入。
- ConfigurationPropertiesBindingPostProcessor：BeanPostProcessor，在 postProcessBeforeInitialization 阶段进行配置类属性注入。
- AutowiredAnnotationBeanPostProcessor：用于注解注入及通过 setter 方法注入，在 postProcessProperties 方法中注入，执行阶段在合并 bean 定义之后。
- InitializingBean 和 SmartInitializingSingleton：InitializingBean 执行更早，**会在实例化之后执行**，实例化的时间决定了执行时间，如果 BeanPostProcessor 则会在所有 Bean 实例化之前；SmartInitializingSingleton 则默认是在所有单例 bean 全部实例化完成后才会执行，所以如果是后置处理，应该使用 SmartInitializingSingleton。

---

## DI

### 注入方式

Spring 支持多种依赖注入方式，底层统一由 `AutowiredAnnotationBeanPostProcessor`（处理 @Autowired/@Value）和 `CommonAnnotationBeanPostProcessor`（处理 @Resource/@PostConstruct/@PreDestroy）等后置处理器完成。

- **构造器注入**（推荐）：对象构造完成即可用
  - 优点：属性可设为 final 保证不可变性；强制依赖明确；天然避免循环依赖（构造阶段就能暴露问题）；
  - 缺点：依赖过多时构造器参数冗长。

- **setter 方法注入**：对象构造后通过 setter 重新注入
  - 优点：灵活度最高，适合可选依赖和后续重新注入；
  - 缺点：属性不能是 final；循环依赖在构造阶段无法被发现。

- **字段注入**：通过反射直接注入到私有字段
  - 优点：代码简洁；
  - 缺点：与 Spring 框架强耦合；无法用于 final 字段；不利于单元测试（无法通过构造器传入 mock）；隐藏了类的真实依赖数量。

- **方法注入**：在任意方法上注入依赖，方法会在 Bean 初始化期间被调用
  - 适用于需要在注入时执行额外逻辑的场景。

- **接口注入**：实现特定 Aware 接口（如 BeanFactoryAware、ApplicationContextAware），容器回调注入
  - 属于 Spring 特有的侵入式注入方式。

### @Autowired

后置处理器为 `AutowiredAnnotationBeanPostProcessor`，默认匹配方式 **byType**，存在多个实现类时，则使用 **byName** 匹配。要指定具体的 bean 可以：
- 添加 `@Qualifier("beanName")` 注解；
- 在实现类上添加 `@Primary` 注解标记首选 bean；
- 使用 `@Autowired(required = false)` 标记为可选依赖，找不到时注入 null。

支持注解在构造器、字段、setter 方法、任意方法及其参数上。

### @Resource

属于 JSR-250（jakarta.annotation）依赖注入注解，后置处理器为 `CommonAnnotationBeanPostProcessor`。默认 **byName**（通过 name 属性或字段名），其次 **byType**，也可通过 name 和 type 属性具体指定。

与 @Autowired 的区别：
- 不支持 `required` 配置；
- 无法注解在方法参数上；
- 不支持 `@Qualifier`，但可通过 name 属性达到同样效果。

### @Inject（JSR-330）

属于 JSR-330 标准依赖注入注解（`jakarta.inject.Inject`），功能与 @Autowired 类似，默认 byType。需要引入 `jakarta.inject:jakarta.inject-api` 依赖。不支持 `required` 属性，需配合 `@Nullable` 或 `Provider<T>` 实现可选注入。

### ObjectProvider 与 Provider

`ObjectProvider<T>` 是 Spring 对 `ObjectFactory<T>` 的扩展接口，提供了更丰富的延迟查找能力：

```java
public interface ObjectProvider<T> extends ObjectFactory<T>, Iterable<T> {
    T getObject(Object... args) throws BeansException;
    T getIfAvailable() throws BeansException;      // 存在则返回，不存在返回 null
    T getIfUnique() throws BeansException;          // 唯一则返回，多个则返回 null
    T getIfAvailable(Supplier<T> defaultSupplier);  // 不存在时使用默认供应商
    // ...
}
```

典型使用场景：
- **延迟查找**：注入 `ObjectProvider<ExpensiveBean>`，仅在调用 `getObject()` 时才创建 bean；
- **可选依赖**：`getIfAvailable()` 替代 `@Autowired(required = false)`；
- **集合注入**：`ObjectProvider<MyService>` 可直接迭代所有匹配类型的 bean；
- **条件创建**：`getIfAvailable(() -> new DefaultService())` 提供默认实现。

JSR-330 的 `Provider<T>` 与 `ObjectProvider<T>` 类似，通过 `get()` 方法获取实例。

### @Lookup 方法注入

`@Lookup` 用于方法注入，允许 Spring 容器通过 CGLIB 动态生成子类重写标注的方法，每次调用时返回容器中查找的 bean。**典型场景是 singleton bean 需要获取 prototype bean 的新实例**：

```java
public abstract class CommandManager {
    public Object process(Object commandState) {
        // 每次调用都会获取新的 MyCommand 实例
        Command command = createCommand();
        command.setState(commandState);
        return command.execute();
    }

    @Lookup("myCommand")
    protected abstract Command createCommand();
}
```

底层原理：Spring 使用 CGLIB 创建 CommandManager 的子类，重写 `createCommand()` 方法，将其实现改为 `ApplicationContext.getBean("myCommand", Command.class)`。

XML 中等价配置为 `<lookup-method name="createCommand" bean="myCommand"/>`。

### @Value

用于注入基本类型、String、SpEL 表达式值：
- `@Value("${app.name}")`：从配置文件读取属性；
- `@Value("#{systemProperties['os.name']}")`：SpEL 表达式；
- `@Value("${app.timeout:3000}")`：带默认值。

### @Lazy

设置为延迟加载，在依赖注入时为 Bean 生成代理类，首次被调用时才创建 Bean。**可用于解决构造器注入时的循环依赖问题**。也可注解在 @Configuration 类或 @Bean 方法上延迟整个配置类的初始化。

### 三级缓存与循环依赖

Spring 通过三级缓存解决 singleton bean 的循环依赖问题（仅限 setter/字段注入，构造器注入无法解决）：

| 缓存 | 数据结构 | 作用 |
|------|----------|------|
| 一级缓存 | `singletonObjects`（ConcurrentHashMap） | 存放完全初始化好的成品 Bean |
| 二级缓存 | `earlySingletonObjects`（HashMap） | 存放提前暴露的半成品 Bean（已实例化但未注入属性），以及 AOP 代理对象 |
| 三级缓存 | `singletonFactories`（HashMap） | 存放 ObjectFactory 工厂函数，其 `getObject()` 方法返回半成品 Bean（会经过 SmartInstantiationAwareBeanPostProcessor 的 `getEarlyBeanReference` 处理） |

**完整流程**（以 A ↔ B 循环依赖为例）：

1. 创建 A：实例化 A（调用构造器）→ 将 A 的 ObjectFactory 加入三级缓存 → 开始注入属性；
2. A 依赖 B：发现 B 不存在 → 递归创建 B；
3. 创建 B：实例化 B → 将 B 的 ObjectFactory 加入三级缓存 → 开始注入属性；
4. B 依赖 A：从一级缓存未找到 → 从二级缓存未找到 → **从三级缓存找到 A 的 ObjectFactory** → 调用 `getObject()` 获取 A 的早期引用（若有 AOP 则在此创建代理）→ **将 A 移入二级缓存，从三级缓存移除** → 将 A 的早期引用注入 B；
5. B 完成：属性注入完成 → 初始化 → **将 B 放入一级缓存**，清除二、三级缓存中的 B；
6. 回退到 A：获取到 B 的成品 → 注入 A → A 完成初始化 → **将 A 放入一级缓存**，清除二、三级缓存中的 A。

**为什么需要三级缓存？** 三级缓存的核心价值在于 **AOP 代理的延迟创建**。如果没有三级缓存，Bean 实例化后就必须立即判断是否需要创建代理，但此时属性还未注入，AOP 的 `getEarlyBeanReference` 无法正确执行。三级缓存的 ObjectFactory 将"获取早期引用"的逻辑延迟到真正需要时（即其他 Bean 依赖它时）才执行。

**构造器注入无法解决循环依赖**：因为构造器注入在实例化阶段就需要依赖对象，而此时 Bean 还未放入任何缓存。

---

## AOP

### 核心概念

- **JoinPoint（连接点）**：程序执行过程中的一个点，如方法执行、异常抛出；
- **Pointcut（切入点）**：匹配一组 JoinPoint 的表达式，决定在哪些连接点上应用增强；
- **Advice（通知/增强）**：在特定 JoinPoint 上执行的动作；
- **Aspect（切面）**：Pointcut + Advice 的组合，模块化横切关注点；
- **Target（目标对象）**：被代理的原始对象；
- **Proxy（代理对象）**：AOP 框架创建的、包含增强逻辑的对象；
- **Weaving（织入）**：将增强应用到目标对象创建代理的过程。

### 代理

Spring AOP 默认使用 JDK 动态代理，如果没有实现接口则使用 cglib 动态代理，**只是使用了 AspectJ 的注解**。通过 `@EnableAspectJAutoProxy` 或 `<aop:aspectj-autoproxy/>` 开启。

#### 动态代理

**运行时增强**，在程序运行期间创建代理类的字节码文件。

- JDK 动态代理：基于接口实现，核心为 InvocationHandler，生成一个实现代理接口的匿名类，方法执行使用反射机制（**JDK8 后效率与 cglib 不再有差距**）。
    ```    
    Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
    ```

- cglib 动态代理：基于子类继承（**所以无法处理 final**），核心为 MethodInterceptor，使用字节码处理框架 ASM 动态生成被代理类的子类。
    ```    
    Enhancer.create(Class type, MethodInterceptor callback)
    ```

#### 静态代理

**编译时增强**，效率高，代理类在编译阶段生成，程序运行前就已经存在。

- AspectJ 静态代理：使用特定的编译器针对特定的语法，在编译期间生成代理类的字节码文件。

### Advice 类型

| 类型 | 注解 | 执行时机 |
|------|------|----------|
| 前置通知 | `@Before` | 目标方法执行之前 |
| 后置通知 | `@After` | 目标方法执行之后（无论是否异常） |
| 返回通知 | `@AfterReturning` | 目标方法正常返回之后，可获取返回值 |
| 异常通知 | `@AfterThrowing` | 目标方法抛出异常之后，可获取异常对象 |
| 环绕通知 | `@Around` | 包围目标方法的执行，最强大的通知类型 |

```java
@Aspect
@Component
public class LoggingAspect {
    
    @Around("execution(* com.example.service.*.*(..))")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = pjp.proceed();  // 执行目标方法
            return result;
        } finally {
            long cost = System.currentTimeMillis() - start;
            System.out.println(pjp.getSignature() + " 耗时: " + cost + "ms");
        }
    }
}
```

### @Pointcut

- execution(方法表达式)：匹配方法的执行
- within(类型表达式)：匹配 target（目标对象）的类型
  - **target.getClass().equals(X.class)**
- this(类型全限定名)：匹配 proxy（代理对象）的具体类型
  - **X.class.isAssignableFrom(proxy.getClass())**
  - **注意**：基于接口的代理则需要指定为被代理类的接口才能生效
- target(类型全限定名)：匹配 target 的具体类型
  - **X.class.isAssignableFrom(target.getClass())**
- args(参数类型列表)：匹配指定参数列表的方法
- @within(注解类型)：匹配方法归属类型上的注解
  - **method.getDeclaringClass().getAnnotation(A.class)**
- @target(注解类型)：匹配 target 上的注解
  - **target.getClass().getAnnotation(A.class)**
- @args(注解类型的参数列表)：匹配指定注解类型的参数列表的方法，即参数类型含有指定的注解
- @annotation(注解类型)：匹配方法上的注解
- bean(bean 名称)：匹配 IOC 容器中指定的 bean
- pointcut 引用：指向具体的方法，使用该方法定义的 pointcut
- pointcut 组合：支持 &&、\|\|、!

### AOP 下循环依赖问题如何处理

AOP 的 AbstractAutoProxyCreator 为 SmartInstantiationAwareBeanPostProcessor 类型，**其 getEarlyBeanReference 方法实现为使用半成品 Bean 创建代理对象并返回，同时还会将其加入二级缓存中**；在其 postProcessAfterInitialization 方法中会为 Bean 创建代理对象，如果二级缓存中已经存在代理对象则直接获取。

若 A、B 互相依赖，A 先创建，则 B 注入 A 时，注入的是通过 getEarlyBeanReference 方法创建的半成品 A 的代理对象；B 依赖注入完成后在 postProcessAfterInitialization 方法中创建了 B 的代理对象，然后加入一级缓存内；递归回退到 A 的依赖注入后，A 直接从一级缓存中获取到 B 的代理对象注入；A 依赖注入完成后，执行 postProcessAfterInitialization 方法时，直接从二级缓存中获取到自己的代理对象，然后加入一级缓存中。

---

## Spring MVC

### DispatcherServlet

DispatcherServlet 是前端控制器，属于 Servlet 容器的 Bean，负责统一分发和协调处理所有 HTTP 请求。

### 请求处理流程

1. 客户端请求会被提交到 DispatcherServlet（前端控制器）；
2. DispatcherServlet 请求一个或多个 HandlerMapping（处理器映射器），并返回一个执行链（HandlerExecutionChain）；
3. DispatcherServlet 将执行链返回的 Handler 信息发送给 HandlerAdapter（处理器适配器）；
4. HandlerAdapter 根据 Handler 信息找到并执行相应的 Handler（即 Controller）；
5. Handler 执行完毕后返回给 HandlerAdapter 一个 ModelAndView 对象，HandlerAdapter 再将其返回给 DispatcherServlet；
6. DispatcherServlet 接收到 ModelAndView 对象后，请求 ViewResolver（视图解析器）对其进行解析；
7. ViewResolver 根据 View 信息匹配到相应的视图结果，并返回给 DispatcherServlet；
8. DispatcherServlet 将 Model 交给接收到的 View 进行渲染，然后返回给客户端。

### 常用注解

- @Controller：标识一个类为 Spring MVC 的控制器，由 DispatcherServlet 负责扫描和注册。
- @RestController：组合注解（@Controller + @ResponseBody），表示该控制器的所有方法返回值直接写入 HTTP 响应体，适用于 RESTful API。
- @RequestMapping：映射 HTTP 请求到处理方法，支持 value（路径）、method（请求方法）、params、headers 等属性。
- @GetMapping / @PostMapping / @PutMapping / @DeleteMapping / @PatchMapping：@RequestMapping 的派生注解，分别对应 GET、POST、PUT、DELETE、PATCH 请求。
- @RequestBody：将 HTTP 请求体内容绑定到方法参数，通常配合 JSON 反序列化使用。
- @ResponseBody：将方法返回值直接写入 HTTP 响应体，跳过视图解析。
- @PathVariable：从 URL 路径中提取变量，如 `/users/{id}` 中的 `id`。
- @RequestParam：从查询参数或表单数据中提取值，支持 required 和 defaultValue 配置。
- @ResponseStatus：指定方法返回的 HTTP 状态码。

### HandlerInterceptor 拦截器

拦截器在 Handler 执行前后介入请求处理流程，需实现 `HandlerInterceptor` 接口：

```java
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        // 在 Handler 执行前调用，返回 false 则中断请求
        return checkAuth(req);
    }
    
    @Override
    public void postHandle(HttpServletRequest req, HttpServletResponse res, Object handler, ModelAndView mv) {
        // Handler 执行后、视图渲染前调用
    }
    
    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res, Object handler, Exception ex) {
        // 视图渲染完成后调用，用于资源清理
    }
}
```

注册方式：实现 `WebMvcConfigurer` 接口的 `addInterceptors()` 方法，或通过 `@Configuration` 类配置。

**拦截器 vs 过滤器**：过滤器基于 Servlet 规范，在 DispatcherServlet 之前执行；拦截器基于 Spring 框架，在 HandlerMapping 之后、Handler 前后执行，可访问 Spring 容器中的 bean。

### 异常处理

- `@ExceptionHandler`：在 Controller 内处理该 Controller 抛出的特定异常；
- `@ControllerAdvice` / `@RestControllerAdvice`：全局异常处理，作用于所有 Controller；
- `@ResponseStatus`：标注在异常类或处理方法上，指定返回的 HTTP 状态码。

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getMessage()));
    }
}
```

---

## Spring 事务

### 编程式事务

- TransactionManager：事务管理器 marker 接口。

- PlatformTransactionManager：平台事务管理器接口，定义了 getTransaction、commit 以及 rollback 操作。

- TransactionDefinition：事务定义，包括隔离级别、传播行为、超时时间、回滚规则以及是否只读。
  - PROPAGATION_REQUIRED：**默认**，存在事务则加入，不存在则创建新事务；
  - PROPAGATION_REQUIRES_NEW：创建新事务，挂起外部事务，两者互不影响；
  - PROPAGATION_NESTED：创建新事务，如果存在外部事务则作为其嵌套事务，**外部事务会影响当前事务，但当前事务不会影响外部事务**；
  - PROPAGATION_MANDATORY：要求当前必须存在事务，否则抛出异常；
  - PROPAGATION_SUPPORTS：存在事务则加入，否则以非事务方式执行；
  - PROPAGATION_NOT_SUPPORTED：以非事务方式执行，挂起外部事务；
  - PROPAGATION_NEVER：当前不允许存在事务，否则抛出异常。

- TransactionStatus：事务状态，用于创建 savepoint 以及 flush 操作。

- TransactionTemplate：封装了事务操作，使用 PlatformTransactionManager 和 TransactionDefinition 创建。

- TransactionSynchronizationManager：工具类，事务同步管理器，用于监听 Spring 的事务操作。

### 声明式事务

基于 AOP 实现，故需要被 Spring 管理、只能拦截 public 方法、内部调用无法生效。通过 `@EnableTransactionManagement` 开启。

- @Transactional：用于注解在类和方法上开启事务，若不配置 rollbackFor 参数，则只会在 RuntimeException 和 Error 时回滚；
- @GlobalTransaction：Seata 的分布式事务开启注解。

### 事务隔离级别

- ISOLATION_DEFAULT：使用数据库默认隔离级别；
- ISOLATION_READ_UNCOMMITTED：允许脏读、不可重复读和幻读；
- ISOLATION_READ_COMMITTED：阻止脏读，允许不可重复读和幻读（Oracle 默认）；
- ISOLATION_REPEATABLE_READ：阻止脏读和不可重复读，允许幻读（MySQL 默认）；
- ISOLATION_SERIALIZABLE：阻止所有并发问题，性能最低。

### 事务失效场景

- **内部调用**：同一类中方法 A 调用方法 B，B 的 @Transactional 不生效（因为绕过了代理对象）；
- **非 public 方法**：Spring AOP 默认只拦截 public 方法；
- **异常被 catch**：异常在方法内部被捕获未抛出，事务不会回滚；
- **数据库引擎不支持**：如 MySQL 的 MyISAM 引擎不支持事务。

### TransactionSynchronizationManager

事务同步管理器，用于注册事务同步回调，在事务提交前/后、完成后执行特定逻辑：

```java
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            // 事务提交后执行，如发送 MQ 消息
        }
    }
);
```

---

## Spring Data JPA

通过 JdkDynamicAopProxy 创建代理对象，默认通过 Hibernate 执行数据库操作。

- Repository：marker 接口；
- CrudRepository：仅 CRUD 操作，Spring Data Redis 则使用该接口；
- PagingAndSortingRepository：在 CrudRepository 基础上支持分页排序；
- QueryByExampleExecutor：单独接口，支持 Example 方式查询；
- JpaSpecificationExecutor：单独接口，支持 Specification 动态查询；
- JpaRepository：标准 JPA 操作接口，继承 PagingAndSortingRepository 和 QueryByExampleExecutor。

### 方法名查询

Spring Data JPA 支持通过方法名自动生成查询语句，常用关键字：

| 关键字 | 示例 | SQL 等价 |
|--------|------|----------|
| And | findByLastnameAndFirstname | WHERE lastname = ? AND firstname = ? |
| Or | findByLastnameOrFirstname | WHERE lastname = ? OR firstname = ? |
| Between | findByStartDateBetween | WHERE start_date BETWEEN ? AND ? |
| Like | findByFirstnameLike | WHERE firstname LIKE ? |
| NotLike | findByFirstnameNotLike | WHERE firstname NOT LIKE ? |
| OrderBy | findByAgeOrderByLastnameDesc | ORDER BY lastname DESC |
| IsNull | findByAgeIsNull | WHERE age IS NULL |
| In | findByAgeIn(Collection ages) | WHERE age IN (?, ?, ...) |

---

## Spring Boot

### @SpringBootApplication

**@SpringBootConfiguration** 表示当前类为配置类；**@EnableAutoConfiguration** 开启自动配置，通过 `@Import(AutoConfigurationImportSelector.class)` 实现；**@ComponentScan** 开启包扫描，默认路径为当前类所在的包。

### AutoConfigurationImportSelector

即 Spring 的 SPI 机制。Spring Boot 2.7 之前使用 SpringFactoriesLoader 扫描 `META-INF/spring.factories` 下配置的所有扩展点；Spring Boot 2.7+ 改为扫描 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件，并按类名排序加载。

#### 启动流程

通过 SpringApplication.run() 方法启动，步骤为：
1. 启动监听器；
2. 环境构建；
3. 创建容器；
4. 前置处理；
5. **刷新容器（启动 Spring IOC 容器）**；
6. 后置处理；
7. 发出事件；
8. 执行 Runner。

### Starter 机制

Starter 是一组依赖描述符，将常用的功能依赖打包在一起。引入一个 Starter 即可自动引入相关依赖和自动配置：

- spring-boot-starter-web：包含 Spring MVC、Tomcat、Jackson 等 Web 开发所需依赖；
- spring-boot-starter-data-jpa：包含 Spring Data JPA、Hibernate 等 ORM 依赖；
- spring-boot-starter-test：包含 JUnit、Mockito、AssertJ 等测试依赖；
- spring-boot-starter-actuator：包含生产监控端点。

### 条件注解

Spring Boot 自动配置的核心机制，根据特定条件决定是否注册 Bean：

| 注解 | 条件 |
|------|------|
| `@ConditionalOnClass` | 类路径中存在指定类 |
| `@ConditionalOnMissingBean` | 容器中不存在指定类型的 Bean |
| `@ConditionalOnProperty` | 配置文件中存在指定属性且满足条件 |
| `@ConditionalOnWebApplication` | 当前是 Web 应用环境 |
| `@ConditionalOnMissingClass` | 类路径中不存在指定类 |

### CommandLineRunner 与 ApplicationRunner

在 Spring Boot 应用启动完成后执行的回调接口：

- **CommandLineRunner**：接收原始字符串数组参数 `run(String... args)`；
- **ApplicationRunner**：接收封装后的 `ApplicationArguments` 对象，支持选项参数解析（`--key=value`）。

两者都在 ApplicationContext 刷新完成后 `ApplicationRunner` 和 `CommandLineRunner` 类型的 bean 会被调用，执行顺序可通过 `@Order` 注解控制。

---

## Spring 中使用的设计模式

- 单例：singleton 作用域的 bean，**类并不是单例模式**，只是 IOC 容器中每个名称的 bean 只能存在一个；
- 工厂：BeanFactory、ApplicationContext；
- 代理：AOP；
- 模板方法：BeanPostProcessor、TransactionManager；
- 观察者：ApplicationEvent；
- 适配器：HandlerAdapter；
- 装饰器：ServletRequestWrapper、TransactionAwareCacheDecorator；
- 责任链：interceptor、filter；
- 策略：ResourceLoader。
