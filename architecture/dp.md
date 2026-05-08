---
title: DDD 之 Domain Primitive
category: architecture
date: 2022-05-31
summary: 不是讲架构，而是一种比较好用的开发思想。
---

## Java Bean

* 所有属性为 `private`
* 提供默认构造方法
* 提供 `getter` 和 `setter`
* 实现 `Serializable` 接口

### 问题

以前被告知的好处是：使用 getter 和 setter 可以自行添加业务逻辑。但慢慢地，共识已经变成不允许在里面添加任何逻辑了——如果加了反而可能引入不必要的 bug。现在唯一的作用大概只是控制某些属性的可见性和可修改性。

---

## Data Object (DO)

* 首先是一个 Java Bean
* 数据库模型的映射
* **日常开发使用的数据模型**，一般放置在 `entity` 或 `domain` 包下
```java
// 一个典型的DO类
@Data
@TableName("tb_user")
public class User implements Serializable {
    @TableId
    private Long id;
    private String name;
    private String phone;
    private String address;
}
```

### DO应该如何设计和使用

* 是一个 Java Bean
* 字段与数据库字段一一对应
* 将其活动范围限制在基础设施层

### DTO Assembler & Data Converter

* DTO Assembler：DTO 和 Entity 互相转化
* Data Converter：DO 和 Entity 互相转化

---

## 简单的 Web 三层架构

UI 层（Controller）、业务层（Service、Facade 等）、基础设施层（Mapper、Repository 等）

- 问题：
    * 上层对下层直接依赖，耦合度过高（本次不涉及）
    * 业务层没有自己的数据模型，将 DO 当成了 Entity（领域）使用

- 正确的数据模型对应关系：
    * UI 层：VO、DTO、Query、Command
    * 业务层：Entity（domain）
    * 基础设施层：DO、DTO（RPC）

### 将 DO 当作 Entity 使用带来的问题

**一段用户注册的代码示例：**

```java
// 业务层
public class RegistrationServiceImpl implements RegistrationService {

    private UserRepository userRepo;

    public User register(String name, String phone, String address) {

        // 校验逻辑
        if (name == null || name.length() == 0) {
            throw new ValidationException("name");
        }
        if (phone == null || !isValidPhoneNumber(phone)) {
            throw new ValidationException("phone");
        }
        // 省略其他字段校验逻辑

        // 业务逻辑
        if (userRepo.findByName(name) != null) {
            throw new ValidationException("姓名不能重复");
        }
        if (userRepo.findByPhone(phone) != null) {
            throw new ValidationException("手机号不能重复");
        }
        // 省略其他业务逻辑

        // 创建用户DO
        User user = new User();
        user.setName(name);
        user.setPhone(phone);
        user.setAddress(address);

        // 调用基础设施层方法持久化，返回结果
        return userRepo.save(user);
    }
}
```
* 问题1：清晰度不够。属性使用的是没有特殊意义的原始类型，并非一个具体的概念——只能靠变量名区分，无法通过变量类型来区分。
```java
    User register(String name, String phone, String address) 

    // 上述注册方法在运行时，参数全是 String 类型
    User register(String, String, String);

    // 在 Controller 层这样调用不会报错，甚至能直接运行；若遗漏参数校验，数据可能直接落库，产生脏数据
    service.register("张三", "浙江省杭州市", "13211110000");

    // 因为 name 和 phone 类型都是 String，所以只能在方法名里加上 ByXxx 来区分
    User findByName(String name);

    User findByPhone(String phone);

    // 多个参数也只能通过变量名区分
    User findByNameOrPhone(String name, String phone);

    // 参数顺序搞反了照样能编译运行，出了问题很难排查
    findByNameOrPhone(user.getPhone(), user.getName());
```
* 问题2：数据验证和错误处理代码四处散落。每次用到 Name、Phone 这种有业务意义的参数时，都要各处添加相同的校验代码，产生大量重复，维护成本非常高，而且无法保证每个调用方都以正确的方式进行了校验。
* 问题3：业务代码的清晰性受损。如果需要对参数做进一步解析——比如从 address 里提取省市区信息——就不得不在调用处额外添加一段解析代码。
* 问题4：可测试性差。如果想单独测试校验 address 参数的逻辑，该如何编写测试用例？
```java
    // 弊端：不能保证调用方使用相同的调用方式
    @Test
    void testAddress() {
        String address = " ";
        //
        if (address == null) {
            throw new ValidationException();
        }
        if (!address.contains("省")) {
            throw new ValidationException();
        }
        // 其他省略

    }

    // 抽离出静态工具类——弊端：业务逻辑分散
    @Test
    void testAddress() {
        String address = " ";
        //
        if (AddressUtils.validate(address)) {
            throw new ValidationException();
        }
    }
```

#### 现有的解决方案

- 抽离成公共代码
  - 不在同一个类里也需要复制粘贴，以后修改得多处改动。
- 抽离成静态工具类
    * 工具类的设计原则是对修改封闭的，而校验代码本身是一段业务逻辑，不可避免会被修改；
    * 大量静态工具类会让业务逻辑四处分散，增加维护难度；
    * 无法控制其他开发人员如何使用这些静态方法，调用方甚至需要额外去了解工具类的用法。

***根本问题就是：name、address、phone 都是具有业务意义的概念，并不能简单地用原始类型去描述。***

---

## Java Primitive 原始类型

八大原始类型，以及对应的包装类、String、BigDecimal、BigInteger、枚举等，都可以视作 Java 语言和 Java Bean 的基础。

* 不从任何事物发展而来
* 初级的形成或生长的早期阶段

## Domain Primitive 领域的基础

* DP 是一个传统意义上的 Value Object（表示一个有含义的值的对象），拥有 Immutable 的特性
* DP 是一个完整的概念整体，拥有精准定义
* DP 使用业务域中的原生语言
* DP 可以是业务域的最小组成部分，也可以构建复杂组合

## 如何创建一个DP？

* 隐形概念显性化
    * 创建一个 Type（数据类型）去显性地表示一个概念
    * 将这个概念相关的逻辑完整地收集到一个 Class（类）里
```java
// 参考 Integer、String 等原始类型去设计 DP
public class Name {
    // 用 final 修饰属性，创建完即不可变
    @Getter
    private final String name;

    // 构造方法不对外暴露
    private Name(String name) {
        this.name = name;
    }

    // 使用静态方法创建对象
    public static Name valueOf(String name) throws ValidationException {
        // 参数校验
        if (name == null || name.isBlank()) {
            throw new ValidationException("不能为空");
        }
        if (!Pattern.compile("[\\u4e00-\\u9fa5]+").matcher(name).matches()) {
            throw new ValidationException("姓名只能为中文！");
        }
        // 其他省略

        // 创建对象并返回
        return new Name(name);
    }
}

// DP类应该定义行为方法
public class Address {

    private final String address;

    // 其他方法省略

    // 一个获取省份的行为方法
    public String getProvince() {
        return this.address.split("省")[0];
    }
}
```
* 隐性上下文显性化
    * 拓展一个简单概念的上下文，将多个概念组合成一个独立的完整概念
```java
// 钱这个概念其实隐性地包含有币种，只是我们默认是人民币，但不代表币种不存在
// 金额和币种才能完整地构成 Money 这个概念，可以先将其定义，方便以后拓展
public class Money {

    private BigDecimal amount;

    // 币种可以是一个 DP 或者一个枚举
    private Currency currency;
}
```
* 封装多对象行为（不再拓展）
    * 一个概念涉及多个对象之间的复杂业务逻辑，将其封装为 DP，简化原始代码。

## DP的使用

```java
// DP的使用

// 接口方法
User find(Name name);

User find(Phone phone);

User find(Name name, Phone phone);

// 类型不匹配，编译出错，及时发现问题
find(user.getPhone(), user.getName());

// 测试用例
@Test
void testAddress() {
    String address = " ";
    // 创建一个Address对象即可。
    Address.valueOf(address);
}
```
**一旦创建必然合法，定义的行为方法能保证安全使用，即完全可控——所有业务逻辑全部封装在类中。**

## DP适合的场景

* 有格式限制的字符串：比如 Name、PhoneNumber、OrderNumber、ZipCode、Address 等
* 有限制的整数：比如 OrderId（>0）、Percentage（0%~100%）、Quantity（>=0）等
* 浮点数：一般用到的 Double 或 BigDecimal 都是有业务含义的，比如 Temperature、Money、Amount、ExchangeRate、Rating 等
* 复杂的数据结构：比如 Map 等，尽量把 Map 的所有操作封装起来，仅暴露必要行为
### 完整示例：

```java
/**
 * 有范围长整型 DP 基类 <br>
 *
 * @author kaikoo
 */
@EqualsAndHashCode(callSuper = true)
public abstract class RangedLong extends Number implements Type {

  private final long value;

  /**
   * @param value 数值
   * @param fieldName 字段名称
   * @param min 最小值
   * @param minInclusive 是否包含最小值
   * @param max 最大值
   * @param maxInclusive 是否包含最大值
   */
  protected RangedLong(
      long value,
      String fieldName,
      Long min,
      Boolean minInclusive,
      Long max,
      Boolean maxInclusive) {
    if (min != null) {
      var cmp = Long.compare(value, min);
      if ((minInclusive && cmp < 0) || (!minInclusive && cmp <= 0)) {
        throw IllegalArgumentExceptions.forMinValue(fieldName, min, minInclusive);
      }
    }
    if (max != null) {
      var cmp = Long.compare(value, max);
      if ((maxInclusive && cmp > 0) || (!maxInclusive && cmp >= 0)) {
        throw IllegalArgumentExceptions.forMaxValue(fieldName, max, maxInclusive);
      }
    }
    this.value = value;
  }

  @JsonValue
  public long getValue() {
    return this.value;
  }

  protected static long parseLong(Object o, String fieldName) {
    if (o == null) {
      throw IllegalArgumentExceptions.forIsNull(fieldName);
    } else if (o instanceof Long l) {
      return l;
    } else if (o instanceof String s) {
      try {
        return Long.parseLong(s);
      } catch (NumberFormatException e) {
        throw IllegalArgumentExceptions.forWrongPattern(fieldName);
      }
    }
    throw IllegalArgumentExceptions.forWrongClass(fieldName);
  }

  @Override
  public int intValue() {
    return (int) this.value;
  }

  @Override
  public long longValue() {
    return this.value;
  }

  @Override
  public float floatValue() {
    return (float) this.value;
  }

  @Override
  public double doubleValue() {
    return (double) this.value;
  }
}

/**
 * Long 类型 ID
 *
 * @author kaikoo
 */
@EqualsAndHashCode(callSuper = true)
public class LongId extends RangedLong implements Identifier {

  protected LongId(long value, String fieldName) {
    super(value, fieldName, 0L, false, null, null);
  }

  @JsonCreator
  public static LongId of(long l) {
    return new LongId(l, "LongId");
  }

  public static LongId valueOf(Object o, String fieldName) {
    return new LongId(parseLong(o, fieldName), fieldName);
  }

  @Override
  public String identifier() {
    return String.valueOf(getValue());
  }
}
```

---

## 项目改造

1. 确定概念，创建 DP 类，收集概念的所有相关业务逻辑和行为
2. 替换所有创建和使用旧类型的地方
3. 创建新的接口方法
4. 修改外部调用方
```java
    // Controller 层
    @PostMapping("/user")
    public Boolean register(@RequestBody @Valid UserCreateCommand command) {
        // 做一些参数的非空校验

        // 参数封装为 DP 对象
        return registerService.register(
            Name.valueOf(command.getName()),
            Phone.valueOf(command.getPhone()),
            Address.valueOf(command.getAddress()));
    }

    // Service 层
    public User register(Name name, Phone phone, Address address) {

        // 不再需要参数校验逻辑，DP 对象创建出来后就必然是合法的

        // 业务逻辑（其实可以移到 Entity 的行为方法中）
        if (userRepo.find(name) != null) {
            throw new ValidationException("姓名不能重复");
        }
        if (userRepo.find(phone) != null) {
            throw new ValidationException("手机号不能重复");
        }

        // 省略其他业务逻辑

        // 创建 Entity
        User user = User.builder().name(name).phone(phone).address(address).build();

        // 调用持久层方法持久化，返回结果
        return userRepo.save(user);
    }
```

---

## Entity 实体 业务模型

* 属于领域对象，业务层使用
* 由 DP 组成
* 生命周期存在于内存中，不需要序列化
* 拥有业务行为
* 不对外开放属性的修改
* 字段和方法与业务语言一致

DO 作为数据模型的可靠性无法保障——暴露了构造方法和 setter，可以随意创建一个不符合业务规范的对象，且在整个生命周期内可以随时通过 setter 修改属性，无法保证数据的可靠性和一致性。

***下面是一个 Entity 的案例：***

```java
/**
 * 用户账户
 *
 * @author KaiKoo
 */
@JsonDeserialize(builder = Account.AccountBuilder.class) // 设置反序列化使用Builder
@EqualsAndHashCode(callSuper = true)
@Getter
@Builder
public class Account extends Entity<AccountId> {

  @Setter(AccessLevel.PROTECTED)
  private AccountId id;
  @DiffIgnore private Instant createTime;
  @DiffIgnore private Instant updateTime;
  private AccountState state;

  // 行为方法使用依赖反转
  public void save(AccountService accountService) {
    if (this.id == null) {
      // 新增逻辑：设置初始状态
      this.state = AccountState.of(AccountStateEnum.INIT);
    } else {
      // 更新逻辑
      if (!accountService.allowModify(this)) {
        throw new BusinessException("不允许修改");
      }
    }
    accountService.save(this);
  }

  // 行为方法，只能在 Entity 内部修改属性
  public void invalidate() {
    // 领域对象创建时已保证合法，可以忽略空指针问题
    if (AccountStateEnum.ACTIVE.equals(this.state.getValue())) {
      this.state = AccountState.of(AccountStateEnum.TERMINATED);
    } else {
      throw new BusinessException("当前状态无法失效。");
    }
  }
}
```

---

## 总结

DP 就是为一个有业务意义的字段创建一个类，并把相关的所有代码全部写在这个类中——其实平常大家应该都有过这样的设计想法，只是 Java Bean 的影响太深了。

Entity 由 DP 组成，字段不需要和数据库一一对应，也不需要持久化。
