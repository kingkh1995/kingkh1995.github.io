## 2.1 环境准备

### 2.1.1 系统要求

| 组件 | 版本/要求 |
|------|-----------|
| JDK | 8+ |
| MySQL | 5.7+（需 InnoDB 引擎） |
| 操作系统 | Linux / macOS / Windows |
| 开发框架 | Spring Boot / Dubbo / SOFABoot |

### 2.1.2 创建数据库

在快速入门示例中，我们使用**一个数据库 + 三个数据源**模拟三个微服务的数据库隔离。

```sql
CREATE DATABASE seata_demo DEFAULT CHARSET utf8mb4;
```

### 2.1.3 创建 UNDO_LOG 表（AT 模式必需）

```sql
CREATE TABLE IF NOT EXISTS `undo_log` (
  `branch_id`     BIGINT       NOT NULL COMMENT 'branch transaction id',
  `xid`           VARCHAR(128) NOT NULL COMMENT 'global transaction id',
  `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
  `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
  `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
  `log_created`   DATETIME(6)  NOT NULL COMMENT 'create datetime',
  `log_modified`  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
  UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB AUTO_INCREMENT = 1 DEFAULT CHARSET = utf8mb4 COMMENT ='AT transaction mode undo table';

ALTER TABLE `undo_log` ADD INDEX `ix_log_created` (`log_created`);
```

> 不同数据库的 UNDO_LOG 建表 SQL 可在 [GitHub 查看](https://github.com/apache/incubator-seata/tree/2.x/script/client/at/db)

### 2.1.4 创建业务表

```sql
DROP TABLE IF EXISTS `storage_tbl`;
CREATE TABLE `storage_tbl` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `commodity_code` varchar(255) DEFAULT NULL,
  `count` int(11) DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY (`commodity_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

DROP TABLE IF EXISTS `order_tbl`;
CREATE TABLE `order_tbl` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(255) DEFAULT NULL,
  `commodity_code` varchar(255) DEFAULT NULL,
  `count` int(11) DEFAULT 0,
  `money` int(11) DEFAULT 0,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

DROP TABLE IF EXISTS `account_tbl`;
CREATE TABLE `account_tbl` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(255) DEFAULT NULL,
  `money` int(11) DEFAULT 0,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

---

## 2.2 启动 Seata Server

### 2.2.1 下载

从 [GitHub Releases](https://github.com/apache/incubator-seata/releases) 下载 `seata-server-{version}.tar.gz` 并解压。

### 2.2.2 启动命令

```shell
# Linux / macOS
sh seata-server.sh -p 8091 -h 127.0.0.1 -m file

# Windows
cmd seata-server.bat -p 8091 -h 127.0.0.1 -m file
```

### 2.2.3 参数说明

| 参数 | 简写 | 默认值 | 说明 |
|------|------|--------|------|
| `--host` | `-h` | `0.0.0.0` | 注册中心暴露的地址，其他服务通过此地址访问 |
| `--port` | `-p` | `8091` | 监听端口 |
| `--storeMode` | `-m` | `file` | 日志存储模式：`file`（文件）、`db`（数据库） |
| `--help` | | | 查看帮助信息 |

---

## 2.3 Dubbo + Seata 示例

### 2.3.1 用例场景

一个用户购买商品的业务逻辑，由 3 个微服务共同完成：

```
用户下单
    │
    ├──▶ 仓储服务 (Storage)：扣除库存数量
    ├──▶ 订单服务 (Order)：创建订单
    └──▶ 账户服务 (Account)：扣减余额
```

### 2.3.2 服务接口定义

**仓储服务**：

```java
public interface StorageService {
    /**
     * 扣除存储数量
     */
    void deduct(String commodityCode, int count);
}
```

**订单服务**：

```java
public interface OrderService {
    /**
     * 创建订单
     */
    Order create(String userId, String commodityCode, int orderCount);
}
```

**账户服务**：

```java
public interface AccountService {
    /**
     * 从用户账户中借出
     */
    void debit(String userId, int money);
}
```

### 2.3.3 主要业务逻辑（无分布式事务——会出问题）

```java
public class BusinessServiceImpl implements BusinessService {

    private StorageService storageService;
    private OrderService orderService;

    public void purchase(String userId, String commodityCode, int orderCount) {
        storageService.deduct(commodityCode, orderCount);
        orderService.create(userId, commodityCode, orderCount);
    }
}
```

```java
public class OrderServiceImpl implements OrderService {

    private OrderDAO orderDAO;
    private AccountService accountService;

    public Order create(String userId, String commodityCode, int orderCount) {
        int orderMoney = calculate(commodityCode, orderCount);
        accountService.debit(userId, orderMoney);

        Order order = new Order();
        order.userId = userId;
        order.commodityCode = commodityCode;
        order.count = orderCount;
        order.money = orderMoney;

        return orderDAO.insert(order);
    }
}
```

> 以上代码的问题：若 `orderService.create()` 中 `accountService.debit()` 成功但订单插入失败，则已扣减的余额无法回滚。

### 2.3.4 Spring XML 数据源配置

修改 `dubbo-account-service.xml`、`dubbo-order-service.xml`、`dubbo-storage-service.xml` 中的数据库连接：

```xml
<property name="url" value="jdbc:mysql://x.x.x.x:3306/xxx" />
<property name="username" value="xxx" />
<property name="password" value="xxx" />
```

### 2.3.5 运行顺序

完整的示例代码在 [seata-samples/at-samples](https://github.com/apache/incubator-seata-samples/tree/master/at-sample)。

按顺序启动以下服务：

1. **Account**（账户服务）
2. **Storage**（仓储服务）
3. **Order**（订单服务）
4. **Business**（业务入口服务，触发分布式事务）

---

## 2.4 @GlobalTransactional 注解

在 `purchase` 方法上添加 `@GlobalTransactional` 注解即可开启分布式事务：

```java
import io.seata.spring.annotation.GlobalTransactional;

public class BusinessServiceImpl implements BusinessService {

    private StorageService storageService;
    private OrderService orderService;

    @GlobalTransactional
    public void purchase(String userId, String commodityCode, int orderCount) {
        storageService.deduct(commodityCode, orderCount);
        orderService.create(userId, commodityCode, orderCount);
    }
}
```

**关键说明**：
- `@GlobalTransactional` 是 Seata 分布式事务的**入口**
- 当该方法被调用时，TM 会向 TC 注册一个全局事务并生成 XID
- XID 通过 Dubbo/Spring Cloud 的**调用链透传**机制传递到各微服务
- 任何 RM 执行失败 → TC 通知所有 RM 回滚 → 全局事务回滚
- 所有 RM 执行成功 → TC 通知所有 RM 提交 → 全局事务提交
- 使用 AT 模式时无需修改任何业务代码——`@GlobalTransactional` 就是唯一需要的改动

