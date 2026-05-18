## MySQL 核心概念（InnoDB 存储引擎）

本文系统梳理 MySQL（InnoDB 存储引擎）的核心知识体系，涵盖基础架构、数据存储、事务与锁机制、MVCC 多版本并发控制、索引策略、主从复制以及 SQL 优化等关键主题。

***

## 三大范式

第一范式（1NF）：字段不可再拆分。
第二范式（2NF）：表中每一个非主键字段必须完全依赖于整个主键，而不是只依赖于主键的一部分（消除部分依赖）。
第三范式（3NF）：在满足 2NF 的基础上，任何非主键字段不依赖于其他非主键字段（消除传递依赖）。

***

## 基础架构

客户端 → 服务器（连接器 → 查询缓存 → 分析器 → 优化器 → 执行器） → 存储引擎 → 文件系统

- 连接器：身份认证和权限相关。
- 查询缓存：MySQL 8.0 版本已移除。执行查询语句前会先查询缓存。
- 分析器：解析 SQL 语法结构。
- 优化器：选择执行计划，为 SQL 语句选择最优方案。
- 执行器：执行语句，首先判断权限，再从存储引擎中读取数据返回。
- 存储引擎：插件式架构，主要负责数据的存储和读取，支持 InnoDB、MyISAM、Memory 等多种存储引擎。

> **MySQL 8.0 改动**：
> - **查询缓存彻底移除**（5.7 已标记废弃），不再支持 `query_cache_type` 参数
> - **原子 DDL**：InnoDB 支持原子性的 DDL 操作，DDL 失败时自动回滚，不再留下中间状态
> - **不可见列**：支持 `ALTER TABLE ... MODIFY COLUMN ... INVISIBLE`，列存在但 `SELECT *` 不显示
> - **数据字典重构**：从 `.frm`、`.trg` 等文件迁移到 InnoDB 系统表（`mysql.ibd`），元数据管理更统一

***

## 数据存储

依次可以分为：表空间、段、区、页、行。

- Buffer Pool：使用缓冲池来减小 CPU 和磁盘速度上的差异。

- 表空间：作为存储结构的最高层，分为系统表空间（ibdata1，存储数据字典和 undo log）和用户表空间（file-per-table，每个表独立 .ibd 文件，MySQL 5.6+ 默认）。

- 段：表空间由各个段构成，包括索引段、数据段、回滚段。

- 区：段由区组成，每个区固定大小是 1 MB。

- 页：区由连续的页组成，每个页的大小默认为 16 KB，所以默认情况下一个区包括 64 个连续的页；页是磁盘读取的最小单位，页中的用户记录行是使用链表的方式存储；页也包括页目录（slot），用于二分查找快速定位，每个 slot 包含 4 到 8 个数据行。

- 行：页中存储的具体的记录。组成包括：变长字段长度列表（变长分配的空间等于实际数据字节大小）、字段 NULL 标志位（列为空则在对应的位上置为 1，同时不存储该列值，也导致索引字段为空会进行额外的操作）、记录头信息、MVCC 版本信息、列数据、（如果未声明主键还会存在一个内置的记录 ID）。

### 分区表

分区表是一个独立的逻辑表，但是底层由多个物理子表组成。MySQL 使用 `PARTITION BY` 定义每个分区存放的数据，**分区表的索引只是在各个底层表上各自加上一个完全相同的索引**，主要目的是将数据按照一个较粗的粒度分在不同的表中（如历史数据归纳）。

***

## redolog & undolog & binlog

- redo log：用于崩溃恢复，只有数据库启动时才会读取；为日志文件组，顺序写，同时是环形写，会覆盖旧的文件；记录的是物理页上的操作。
  - **redo log buffer**：操作同时还会写入 redo log buffer 中，后台线程会每隔 1 秒或缓冲区大小达到阈值后，把缓冲区中的内容写到文件系统缓存（page cache），然后调用 fsync 刷盘。

- undo log：用来事务回滚和 MVCC；记录的是修改前的数据版本（行记录的旧值），快照读通过 undo log 计算得到记录原值，事务则通过 undo log 中的历史数据进行回滚；**undo log 一定存在对应的 redo log**；有一个后台线程会定时处理所有事务都不会再使用的 undo log。

- binlog：属于 MySQL 服务器层，用于数据同步；记录的是原始逻辑；追加写不会覆盖旧记录；有三种格式：statement、row、mixed。
  - statement：原始 SQL，如果包含 now() 函数等则会导致主从执行结果不一致，MySQL 会标记为不安全语句并可能自动切换格式；
  - row：会额外记录下操作数，需要更大的资源；
  - mixed：自动判断选择 statement 或 row。

***

## 事务

### ACID

- 原子性（Atomicity）：事务内的所有操作要么全部成功，要么全部失败；
- 一致性（Consistency）：执行事务前后，数据的完整性不被破坏，**C 是目的，而 AID 是手段**；
- 隔离性（Isolation）：并发执行的事务之间互不影响；
- 持久性（Durability）：事务一旦提交，它对数据的改变就是永久的。

### 并发事务带来的问题

- 脏读：**读取到其他事务未提交的数据**；
- 不可重复读：事务内多次读取同一数据内容不一致，**读取到其他事务已提交的数据**；
- 幻读：事务内读取多行数据，之后的读取会多出几条数据，**其他事务仍然可以在当前事务读取范围内新增数据**。

### 隔离级别

- Read Uncommited：读未提交，会出现脏读、不可重复读、幻读；
- Read Committed (RC)：读已提交，允许读取到并发事务提交的数据，可以阻止脏读；
    > 通过加锁可以解决不可重复读的问题，但是仍然无法解决幻读。 
- Repeatable Read (RR)：可重复读，多次读取（**快照读**）结果一致，理论上可能出现幻读；
    > **InnoDB 默认隔离级别，使用 MVCC（快照读） + Next-key Lock（当前读）防止了幻读。**
- Serializable：可序列化，事务并发执行的结果与串行执行一致。
    > InnoDB 下会由 MVCC 退化为基于锁的并发控制，所有的读均默认为当前读。

### 事务执行流程

1. 开启事务，InnoDB 在第一次修改数据时分配事务 ID（自增值），锁在执行过程中按需获取，没有获取到锁则等待；
2. 执行器先通过存储引擎找到对应的数据页，如果 Buffer Pool（缓冲池）存在数据则直接取出，没有则从磁盘读取并放入缓冲池；
3. 在数据页内找到具体的记录，修改后写入 Buffer Pool；
4. 存储引擎生成 redo log 和 undo log 到内存中，**将 redo log 状态设为预提交**；
5. 将 redo log 写入文件并调用 fsync 刷盘（undo log 的修改在 Buffer Pool 中，由 redo log 的 WAL 机制保护）；
6. 事务提交，服务器生成 binlog 并写入 binlog 文件中，调用 fsync 保证刷盘；
7. 将 redo log 状态改为已提交，并释放所有锁。

- *如果 redo log 在提交阶段发生故障，服务器恢复后会通过 XID（XA 事务 ID）找到对应的 binlog 日志，并继续提交事务，恢复数据。*

> **MySQL 8.0 改动**：
> - **事务持久化**：崩溃恢复流程优化，`innodb_undo_log_truncate` 支持在线回收 undo log 空间
> - **跳过锁等待**：`innodb_deadlock_detect` 可关闭死锁检测（适用于高并发写入场景，依赖 `innodb_lock_wait_timeout` 超时机制）

### AUTOCOMMIT 机制

MySQL 默认采用自动提交模式，即如果不显式使用 START TRANSACTION 语句来开始一个事务，那么每个查询操作都会被当做一个事务并自动提交。

***

## MVCC

多版本并发控制协议，InnoDB 会为每条记录添加额外字段：
- DB_TRX_ID：最后更新该记录的事务 ID
- DB_ROLL_PTR：回滚指针，指向该记录的 undo log
- DB_ROW_ID：如果该表没有主键且没有唯一非空索引，会使用该 ID 生成聚簇索引
- 删除位：标识是否被删除

优点是快照读不需要获取锁，提高了系统的并发度；缺点是需要维护每条记录的版本信息，且在检索行时需要判断版本是否可见，降低了查询的效率，同时还需要定期清理及时回收空间。

### 更新数据流程

1. 获取排他锁；
2. 修改记录；
3. 写 redo log 和 undo log；
4. 设置当前事务 ID，将回滚指针指向 undo log 历史数据。

### ReadView

配合 MVCC 使用，组成：当前事务 ID、当前进行中的事务 ID 集合、当前进行中的事务 ID 集合的最小值、当前将要分配的下一个事务 ID（即事务 ID 的上限）。

**RR 级别只会在第一个查询时创建 ReadView，而 RC 级别每次查询都会创建一个 ReadView，这是 RR 级别快照读不会出现幻读的关键原因。**

查询过程：
   1. 查询到某条记录后，判断该版本记录的 trx_id 是否与 ReadView 中的 creator_trx_id 相等，相等则表示可读直接返回，否则进行以下判断：
      1. 小于 ReadView 记录的最小事务号，则可读；
      2. 大于等于 ReadView 记录的最大事务号，则不可读；
      3. 在两者之间，**则在 ReadView 记录的进行中的事务 ID 集合中查找当前版本事务 ID，如果找不到则表示创建当前 ReadView 时该事务已经提交故可读，否则表示事务还未提交不可读**。
   2. 如果当前版本不可读，通过回滚指针沿着 undo log 链向上查找历史版本，重复上面步骤。

> **MySQL 8.0 改动**：
> - **ReadView 复用优化**：RR 级别下同一事务内多个一致性读可复用同一个 ReadView，减少开销
> - **undo log 在线截断**：`innodb_undo_log_truncate=ON` 时，后台线程自动清理不再需要的 undo log，避免 `ibdata` 无限膨胀

***

## 锁

### 锁策略

- X 锁（排他锁-写锁）：`lock table ... write` / `select ... for update`
  - 同一时刻只能由一个事务加锁，与任何类型的锁都不兼容。

- S 锁（共享锁-读锁）：`lock table ... read` / `lock in share mode`
  - 可以同时被多个并行事务加锁，保证数据不能被其他事务修改；
  - **申请到 S 锁后可以继续申请 X 锁，如果还有其他并行事务也持有 S 锁则会进入等待，可能产生死锁。**

### 锁类型

**表级锁主要用于执行 DDL 语句。**

**意向锁**：表级锁，包括意向共享锁（IS 锁）和意向排他锁（IX 锁）。
- "表明"加锁的意图，申请行级锁前要先申请对应的意向锁，**申请表级锁时只需要先判断是否存在其他表级锁，再判断是否存在意向锁即可**。
- 完全由数据库引擎自己维护，用户无法手动操作。

**行级锁：**
- 记录锁（Record Lock）：锁住单条数据，**是锁住索引而不是记录本身**。
  - 更新或删除单条数据前默认会先获取其排他记录锁；
- 间隙锁（Gap Lock）：锁住索引的一个范围，**用于阻塞插入意向锁**。
  - 间隙锁之间不会互相阻塞。
- 插入意向锁：**不属于意向锁，而是由 INSERT 操作产生的一种间隙锁**，表明在加锁区间的插入意图。
  - 插入意向锁不会互相阻塞，插入操作时才会判断数据之间是否冲突；
  - 实际上不会阻塞任何锁，包括间隙锁，且只会被间隙锁阻塞。
- 临键锁（Next-key Lock）：记录锁 + 间隙锁，锁定索引上的一个范围，包括记录本身，为 InnoDB 在 RR 级别下的加锁方式。

### 2PL

二阶段锁，事务期间加锁和解锁分为两个完全不相交的阶段，加锁阶段只加锁，解锁阶段只释放锁。

### RR 级别加锁行为

- 快照读：不加任何类型的锁。

- 加 S 锁：在使用的索引上对记录加共享 Next-key Lock，如果需要回表查询，则在主键索引上加上对应记录的共享记录锁。

- 加 X 锁 / UPDATE / DELETE：在使用的索引上对记录加排他的 Next-key Lock，同时在主键索引上加上对应记录的排它记录锁。
    - UPDATE 和 DELETE 操作在执行前都会先执行一个当前读操作；
    - **因为加锁是引擎层面的行为，如果无法使用索引会对所有行加上间隙锁和记录锁后返回，不过执行器在过滤时会通知引擎释放掉不匹配行的记录锁，虽然这明显违背了 2PL，注意间隙锁仍然不会释放。**

- INSERT：在插入数据之前，要在所有索引上对要插入的范围加上插入意向锁，在插入数据成功后会在插入的行上加上排它记录锁。
    - 如果检测到唯一键冲突，则会申请冲突行的 S 锁；t0 为成功插入的事务，t1 和 t2 是检查到唯一键冲突的事务，t0 提交或回滚后，t1 和 t2 都会成功申请到冲突位置的 S 锁；如果 t0 提交了，则 t1 和 t2 获取到 S 锁后会抛出主键冲突错误；如果 t0 回滚了，则 t1 和 t2 都需要再获取到 X 锁才可以插入数据，但由于对方都获取到了 S 锁，则永远不可能获取到 X 锁，故出现死锁，根据死锁机制，第一个尝试获取 X 锁的事务会成功，其他事务则抛出死锁错误；
    - 如果有自增列（一张表只允许一个自增列）会维护加一个表级排他锁以获取到自增值，不过这个表级锁会在插入完成后释放，**MySQL 5.7+ 默认使用互斥量（`innodb_autoinc_lock_mode=2`）替代表级锁实现轻量级自增分配**。

- INSERT ... ON DUPLICATE KEY UPDATE：检测到唯一键冲突会直接申请加排它锁。

### 组合索引加锁

- **idx(a,b)**
- **a** : 1 | 3 | 3 | 3 | 3 | 5
- **b** : 1 | 1 | 2 | 2 | 3 | 5
- **c** : 1 | 2 | 3 | 1 | 4 | 5
- **id**: 3 | 5 | 1 | 2 | 4 | 6

1. `delete from t where a>1 and a<5 and b=2 and c=1`：组合索引上，gap 锁 5 个（a 值 1 到 5 之间），X 锁两个（b=2），聚簇索引 X 锁两个（id=1,2）

2. `delete from t where a=3 and b>1 and b < 3 and c=1`：组合索引上，gap 锁 3 个（a 值等于 3 且 b 值大于 1 小于 3），X 锁两个（b=2），聚簇索引 X 锁两个（id=1,2）

### 死锁产生条件

- 多个并发事务（2个或者以上）；
- 每个事务都持有锁（或者是已经在等待锁）；
- 每个事务都需要再继续持有锁（为了完成事务逻辑，还必须更新更多的行）；
- 事务之间产生加锁的循环等待，形成死锁。

### 产生死锁原因

1. 两个事务行锁加锁顺序不一致；
   - 批量更新时，按固定的顺序（如 ID 顺序）操作。
2. 间隙锁与插入意向锁互斥：一个事务持有间隙锁，另一个事务尝试插入时需要获取插入意向锁而被阻塞，若双方互相等待则产生死锁；
   - 事务级别调整到RC。
3. 唯一索引插入导致死锁，第一个事务插入后回滚之后，其他两个事务通过一个当前读获取到了共享间隙锁，导致互相等待；
   - 使用insert on duplicate key update。
4. 同一加锁行为，由于 index_merge 使用了多个索引，这多个索引对主键行锁的加锁顺序不一致。
   - 使用force index；
   - 创建组合索引。

***

## 索引

### 索引类型

- B 树索引：
    - 使用 B+ 树实现，有序存储，一般为 1-3 层，非叶子结点只保存索引值，在叶子结点才保存被索引的数据，同时叶子结点之间通过指针顺序连接。
    - 优点是减少了磁盘 IO 的次数，且每次查询都是稳定的，因为数据行到根节点的高度是相同的。

- 哈希索引：使用哈希表实现，无序存储，查询快，但无法排序和范围搜索。
    
- **自适应哈希索引**：InnoDB 会自发的在 B+ 树索引基础上对被频繁使用的索引值在内存中创建一个哈希索引以提高查找效率，此为完全存储引擎的行为，用户只能开启或关闭该功能。

### 索引策略

- 聚簇索引：将数据行和索引保存在一起，而不是通过指针指向数据行。
    - InnoDB 的主键索引是聚簇索引，将数据行保存在 B 树索引的叶子结点中；
    - 优点是不再需要磁盘 IO 去查找数据，缺点是更新代价很高，会导致"页分裂"。

- 非聚簇索引：也称为二级索引、普通索引。
    - InnoDB 的二级索引的叶子结点保存的是主键值而不是数据行指针，所以通过二级索引查找数据时可能需要两次 B 树查找。
    - **二级索引默认会将主键作为最后一列，即默认都是组合索引。**

- 唯一索引：索引值必须是唯一的，可以为Null，主键索引是唯一索引且不能为Null。

- 组合索引：多列索引，使用最左前缀匹配原则，所以应该将区分度高的列放在左边。

- 前缀索引：针对字符串列的索引不需要索引完整的字符串，只索引字符串前几位，以提升索引效率，因为只能使用左模糊进行查询。

- 覆盖索引：如果索引覆盖了查询需要的所有数据行，则不需要再去读取数据行，称为覆盖索引。
  
- 索引下推：5.6 版本推出，用于优化非聚簇索引查询效率，**如果存在某些被索引的列的判断条件，MySQL 服务器会将这一部分判断条件传递给存储引擎**，然后由存储引擎可以提前过滤掉不匹配的记录，减少回表查询的次数。

### index_merge

index_merge 是 MySQL 5.1 后引入的一项索引合并优化技术，**它允许对同一个表同时使用多个索引进行查询，并对多个索引的查询结果进行合并后返回**。在使用 index_merge 技术后，会同时执行两个索引，故可能导致死锁，可以使用 `force index` 操作避免。

### 索引失效场景

1. 查询范围太大，即服务器认为走全表扫描会更快，包含使用负向扫描（NOT）的场景；
2. 数据隐式转换，关联查询时字符集不同也会导致隐式转换字符；
3. 对列使用函数；
4. 对列进行运算；
5. like 使用左模糊；
6. 组合索引不符合最左匹配原则。

### 索引选择策略

扫描行数是主要因素但并不是唯一的判断标准，优化器还会结合是否使用临时表、是否排序等因素进行综合判断。会统计基数用于预估行，即选取一定数量的数据页，统计其不同值得到平均值。

1. 主键索引优先；
2. 可以使用覆盖索引；
3. 扫描行数少；
4. 可以用于排序；
5. 索引大小。

### show index from

- cardinality: 表示索引在当前列唯一值的数量，为估计值。

> **MySQL 8.0 改动**：
> - **降序索引**：支持 `INDEX ... DESC`，InnoDB 真正支持降序索引扫描（5.7 语法支持但实际仍按升序存储）
> - **函数索引 / 表达式索引**：支持对表达式建索引，如 `INDEX ((col1 + col2))`，底层使用隐藏虚拟列实现
> - **不可见索引**：支持 `ALTER TABLE ... ALTER INDEX ... INVISIBLE`，优化器忽略该索引但不删除，用于安全测试索引效果
> - **直方图统计**：`ANALYZE TABLE ... UPDATE HISTOGRAM` 为无索引列收集数据分布统计，优化器可据此更准确估算行数
> - **自增列持久化**：MySQL 8.0 自增计数器自动持久化到 redo log 和数据字典，重启后 AUTO_INCREMENT 值不再丢失（5.7 重启可能回退到 `SELECT MAX(col)+1` 逻辑）

***

## 主从复制

1. 主库将数据库中数据的变化写入到 binlog；
2. 从库连接主库，主库会创建一个 binlog dump 线程来发送 binlog，从库中的 I/O 线程负责接收更新的 binlog；
3. 从库的 I/O 线程将接收的 binlog 写入到 relay log 中；
4. 从库的 SQL 线程读取 relay log 同步数据本地（再执行一遍 SQL）。

为异步复制，主库只保证将数据变更写入 binlog，但不关心是否被从库接收，缺点就是如果主库出现宕机且还未来的及将新写入的 binlog 发送给从库，此时若将从库升级为主库则会出现数据丢失。

> **MySQL 8.0 改动**：
> - **并行复制增强**：`binlog_transaction_dependency_tracking` 支持 WRITESET / WRITESET_SESSION 模式，基于写集合判断事务依赖，大幅提升从库回放并行度
> - **MGR（Group Replication）成熟**：内置基于 Paxos 的多主/单主高可用方案，支持自动故障切换和流控
> - **半同步复制优化**：`rpl_semi_sync_master_wait_point` 支持 AFTER_SYNC 模式，降低主从不一致窗口
> - **binlog 过期策略改进**：`binlog_expire_logs_seconds` 替代 `expire_logs_days`，支持秒级精度控制

***

## EXISTS & IN
1. 如果无法使用索引的情况下，MySQL 会把 IN 的查询语句改成 EXISTS 去执行；
2. IN 查询在内部表和外部表上都可以使用到索引，Exists 查询仅在内部表上可以使用到索引；
3. 当子查询结果集很大，而外部表较小的时候，Exists 的 BNL 算法作用开始显现，并弥补外部表无法用到索引的缺陷，查询效率会优于 IN；而当子查询结果集较小，而外部表很大的时候，IN 的外表索引优势占主要作用，此时 IN 的查询效率会优于 Exists。

**Block Nested Loop**：将外层循环的行 / 结果集存入 join buffer，内层循环的每一行与整个 buffer 中的记录做比较，从而减少内层循环的次数。

> **MySQL 8.0 改动**：
> - **哈希连接（Hash Join）**：8.0.18+ 引入，当连接条件无索引时优先使用 Hash Join 替代 BNL，性能提升显著
> - **子查询物化优化**：IN 子查询结果集自动物化为临时表并建索引，避免重复执行
> - **CTE（公共表表达式）**：支持 `WITH` 语法，可递归查询（`WITH RECURSIVE`），替代复杂的自连接和临时表

***

## EXPLAIN

- id：查询的编号，分为简单查询和复杂查询，复杂类型分为：简单子查询（SELECT 子句中）、派生表（FROM 子句中）、UNION 查询。
- select_type：查询类型，SIMPLE：简单查询；PRIMARY：复杂查询的最外层；SUBQUERY：简单子查询；DERIVED：派生表；UNION：UNION 查询；UNION RESULT：UNION 结果集，id 为 null，它是一个匿名临时表。
- table：表名或者对应的别名，派生表的外层查询则是 `<derivedN>`（N 为派生表查询的 id）。
- type：访问类型，以下效率从差到优：
  - ALL：全表扫描；
  - index：只比 ALL 快一点，需要遍历整个索引树；
  - range：范围扫描索引树，即是一个有限制的 index；
  - ref：是索引查找，返回匹配某个值的行，可能是多个行，因此属于查找和扫描的混合，**查找非唯一索引或唯一索引的非唯一前缀时才会发生**；
  - eq_ref：等值查找，明确只会返回一行，即在查找唯一性索引时发生；
  - const、system：命中主键或唯一索引，且查询条件是常量值。
  - NULL：意味着在优化阶段就可以分解查询条件，不需要再访问表或索引。
- possible_keys：显示可以使用哪些索引，在优化早期创建的，可能对后续优化过程并没有作用。
- key：实际使用的索引，表示最小的查询成本应该使用的索引。
- key_len：表示索引的**最大字节数**，根据前缀模式可以推算出使用到的列。
- ref：记录了在索引中查找值使用的列或常量。
- rows：为了找到需要的行而大概需要读取的行数。
- filtered：MySQL 5.7+ 默认包含，悲观的估计大概符合条件的行大概占需要扫描的行的百分比，使用 ALL、index、range 访问时会使用这个值。
- Extra：
  - Using index：表示使用覆盖索引；
  - Using index condition：表示使用了索引下推；
  - Using where：表示服务器将在检索到行之后再过滤，**意味着可以考虑优化索引结构**；
  - Using temporary：表示对查询结果排序、GROUP BY、DISTINCT 等操作时会建立临时表；
  - Using filesort：表示会对结果使用外部索引排序而不是读取出行，**可能是在内存或磁盘**；
  - Using join buffer：使用了连接缓存，即表连接时未使用到索引；

> **MySQL 8.0 改动**：
> - **EXPLAIN ANALYZE**：不仅显示执行计划，还实际执行查询并返回各阶段的实际耗时和行数，定位性能瓶颈更精准
> - **EXPLAIN FORMAT=TREE**：以树形结构展示执行计划，比传统表格更直观，便于理解嵌套连接关系
> - **优化器跟踪**：`optimizer_trace` 功能增强，可详细查看优化器为何选择某个执行计划

***

## 数据库调优策略

- 选择适合的数据库
- 优化表设计：合理的字段数据类型、合理的表结构、冷热分离、增加中间表、增加冗余字段
- 优化查询：SQL 优化，索引优化
- 增加缓存层
- 库级别优化：读写分离，垂直拆分、水平拆分

> **MySQL 8.0 改动**：
> - **角色（Roles）**：支持 `CREATE ROLE` 批量管理权限，简化用户权限分配
> - **密码策略增强**：`validate_password` 组件支持密码历史检查、双密码机制（轮换时新旧密码同时有效）
> - **资源组**：`CREATE RESOURCE GROUP` 可为不同线程绑定 CPU 核心和优先级，实现资源隔离
> - **JSON 增强**：支持 JSON 聚合函数（`JSON_ARRAYAGG`、`JSON_OBJECTAGG`）、JSON 路径表达式优化、JSON 表函数（`JSON_TABLE` 可将 JSON 展开为关系表）
> - **窗口函数**：`ROW_NUMBER()`、`RANK()`、`DENSE_RANK()`、`LAG()`、`LEAD()`、`SUM() OVER()` 等，替代复杂的自连接和用户变量

***

## MySQL 8.0 窗口函数

MySQL 8.0 引入窗口函数（Window Functions），允许在不折叠行的前提下对一组相关行进行计算。相比传统的自连接和用户变量方案，窗口函数语法更简洁、性能更优。

### 基本语法

```sql
函数名(表达式) OVER (
  PARTITION BY 分组列
  ORDER BY 排序列
  窗口帧子句
)
```

- `PARTITION BY`：将结果集划分为多个分区，函数在每个分区内独立计算（类似 GROUP BY 但不折叠行）
- `ORDER BY`：定义分区内行的排序，影响排序类和偏移类函数的结果
- **窗口帧**：定义当前行周围的计算范围，仅对聚合类窗口函数有效，默认 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

### 窗口函数分类

#### 排序类

| 函数 | 说明 | 示例场景 |
|------|------|---------|
| `ROW_NUMBER()` | 为每行分配唯一序号，从 1 开始，不重复 | 分页、去重、Top N |
| `RANK()` | 相同值排名相同，跳过后续序号（1, 2, 2, 4） | 含并列的排名 |
| `DENSE_RANK()` | 相同值排名相同，不跳过后续序号（1, 2, 2, 3） | 连续排名 |
| `NTILE(n)` | 将分区均分为 n 个桶，返回每行所属桶号 | 分位数分析 |

```sql
-- Top N 问题：每个部门工资前 2 的员工
select dept, name, salary
from (
  select dept, name, salary,
    dense_rank() over (partition by dept order by salary desc) as rn
  from employees
) t
where rn <= 2;
```

#### 聚合类

所有标准聚合函数（`SUM`、`AVG`、`COUNT`、`MAX`、`MIN`）均可作为窗口函数使用。

```sql
select order_date, amount,
  sum(amount) over (order by order_date) as running_total,
  avg(amount) over (
    order by order_date
    rows between 2 preceding and current row
  ) as moving_avg_3d
from orders;
```

#### 偏移类

| 函数 | 说明 |
|------|------|
| `LAG(expr, n, default)` | 获取分区内前 n 行的值 |
| `LEAD(expr, n, default)` | 获取分区内后 n 行的值 |
| `FIRST_VALUE(expr)` | 获取窗口帧内第一行的值 |
| `LAST_VALUE(expr)` | 获取窗口帧内最后一行的值（注意默认窗口帧边界） |
| `NTH_VALUE(expr, n)` | 获取窗口帧内第 n 行的值 |

```sql
-- 计算相邻订单的时间间隔
select order_date,
  lag(order_date) over (order by order_date) as prev_date,
  datediff(order_date, lag(order_date) over (order by order_date)) as days_gap
from orders;
```

> **注意**：`LAST_VALUE()` 的默认窗口帧是 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`，即只到当前行，因此 `LAST_VALUE()` 默认返回当前行自身。如需获取分区内真正的最后一行，需显式指定窗口帧：
> ```sql
> last_value(amount) over (
>   order by order_date
>   rows between unbounded preceding and unbounded following
> )
> ```

#### 分布类

| 函数 | 说明 |
|------|------|
| `PERCENT_RANK()` | 相对排名百分比，公式 `(rank - 1) / (总行数 - 1)` |
| `CUME_DIST()` | 累积分布，小于等于当前行的行数 / 总行数 |
| `PERCENTILE_CONT(p)` | 连续百分位数（线性插值） |
| `PERCENTILE_DISC(p)` | 离散百分位数（取实际存在的值） |

```sql
-- 计算员工工资在部门内的百分位排名
select name, dept, salary,
  percent_rank() over (partition by dept order by salary) as pct_rank
from employees;
```

### 窗口帧详解

窗口帧仅对聚合类窗口函数有效，有两种模式：

| 模式 | 说明 |
|------|------|
| `ROWS` | 基于物理行位置定义窗口 |
| `RANGE` | 基于逻辑值范围定义窗口（ORDER BY 列值相同的行视为同一帧） |

```sql
-- ROWS：严格按行位置，无论值是否相同
rows between 2 preceding and 2 following

-- RANGE：按值范围，值相同的行一起计算
range between interval 1 day preceding and interval 1 day following

-- 常用简写
rows unbounded preceding          -- 从分区开头到当前行
rows between unbounded preceding and current row  -- 默认行为（RANGE 模式下）
rows between current row and unbounded following  -- 从当前行到分区末尾
```

### 执行顺序

窗口函数在 `WHERE`、`GROUP BY`、`HAVING` 之后执行，在 `ORDER BY` 之前执行。因此：

- 窗口函数**不能**出现在 `WHERE` 子句中（需用子查询或 CTE 包裹后过滤）
- 窗口函数**可以**出现在 `SELECT` 和 `ORDER BY` 子句中
- 多个窗口函数可以共存于同一查询，各自定义独立的 `OVER` 子句

```sql
-- 错误：窗口函数不能直接在 WHERE 中使用
select * from t where row_number() over (order by id) > 10;

-- 正确：使用 CTE 包裹
with numbered as (
  select *, row_number() over (order by id) as rn from t
)
select * from numbered where rn > 10;
```

### 性能优势

相比 MySQL 5.7 及之前的替代方案，窗口函数有显著优势：

| 场景 | MySQL 5.7 方案 | MySQL 8.0 窗口函数 |
|------|---------------|-------------------|
| 行号 | 用户变量 `@rn := @rn + 1` | `ROW_NUMBER()` |
| 累计求和 | 自连接 + SUM | `SUM() OVER (ORDER BY)` |
| 相邻行比较 | 自连接 | `LAG()` / `LEAD()` |
| Top N | 子查询 + COUNT | `DENSE_RANK() OVER` |
| 会话划分 | 用户变量 + 条件判断 | `LAG()` + `SUM() OVER` |

用户变量方案依赖 SQL 执行顺序，在 MySQL 5.7 中行为未完全定义，且无法并行执行。窗口函数由优化器统一规划，语义明确且支持并行。

***

## SQL 题

### 取出每个科目所有分数排名前2的成绩

1. 使用子查询：
```sql
select * 
from t t0 
where 2 >
(
  select count(distinct t1.score)
  from t t1
  where t1.subject = t0.subject and t1.score > t0.score
);
```

2. 使用 exists（等价改写）：
```sql
select * 
from t t0 
where (
  select count(distinct t1.score)
  from t t1
  where t1.subject = t0.subject and t1.score > t0.score
) < 2;
```
**以上两种方式效果相同，主表都是全表扫描，子查询会走 subject 索引。**

*如果是找出第 N 高的成绩，则将 < 改为 = 号即可。*

###  找出连续三天以上访客超过100的记录

```sql
select distinct t1.*
from t t1, t t2, t t3
where t1.people > 100 and t2.people > 100 and t3.people > 100
and
(
    (t1.id - t2.id = 1 and t1.id - t3.id = 2 and t2.id - t3.id =1)
    or (t2.id - t1.id = 1 and t2.id - t3.id = 2 and t1.id - t3.id =1)
    or (t3.id - t2.id = 1 and t2.id - t1.id =1 and t3.id - t1.id = 2)
)
order by t1.id;
```

> **注意**：此解法假设 `id` 是连续无间隙的，实际数据库中因删除、回滚等原因可能导致 ID 不连续。更通用的方案是使用窗口函数（如 `ROW_NUMBER()`）或日期算术。

### 树节点

```sql
select id, 
case when t.id in (select t1.id from tree t1 where t1.p_id is null) then 'Root'
when t.id in (select p_id from tree t2) then 'Inner'
else 'Leaf' end
as type
from tree t;
```

> **注意**：使用 `IN` 而非 `=` 可兼容多根节点场景（多条记录的 `p_id` 为 NULL）。

### Gaps & Islands（间隙与孤岛）

**Islands 问题**：找出连续区间。例如找出连续登录超过 3 天的用户。

```sql
-- MySQL 8.0+ 使用窗口函数
select user_id, min(login_date) as start_date, max(login_date) as end_date, count(*) as days
from (
  select user_id, login_date,
    row_number() over (partition by user_id order by login_date) as rn
  from logins
) t
group by user_id, date_sub(login_date, interval rn day)
having count(*) >= 3;
```

**核心思路**：对于连续日期序列，`login_date - row_number` 的差值是恒定的。利用这个差值分组即可识别连续区间。

**Gaps 问题**：找出序列中的缺失部分。例如找出缺失的 ID 区间。

```sql
-- 找出缺失的 ID 区间（假设 ID 范围 1~100）
select t1.id + 1 as gap_start, min(t2.id) - 1 as gap_end
from t t1
join t t2 on t1.id < t2.id
group by t1.id
having t1.id + 1 < min(t2.id);
```

**核心思路**：自连接找到每个 ID 之后的下一个 ID，如果差值大于 1 则说明中间存在间隙。

### IN-OUT 时间差计算

**题目**：表 `records` 含 `type`（IN/OUT）和 `date`（时间）。规则：连续多条 IN 取最早，连续多条 OUT 取最晚，IN 和 OUT 成组配对计算时间差。

```sql
-- MySQL 8.0+ 窗口函数解法
with grouped as (
  -- 检测 type 变化点，累计求和分配组号
  select type, date,
    sum(case when type != lag(type) over (order by date) then 1 else 0 end)
      over (order by date) as grp
  from records
),
paired as (
  -- 每组内聚合：IN 取最小值，OUT 取最大值
  select
    min(case when type = 'IN' then date end) as in_time,
    min(case when type = 'OUT' then date end) as out_time
  from grouped
  group by grp
)
-- 配对计算时间差
select timediff(out_time, in_time) as duration
from paired
where in_time is not null and out_time is not null;
```

**核心思路**：利用 `LAG()` 检测相邻记录的 type 变化点，通过 `SUM() OVER()` 累计分配组号，将连续相同 type 的记录归入同一孤岛（Island），再按组聚合取极值后配对计算。

### 用户留存率计算

**题目**：表 `logins` 含 `user_id` 和 `login_date`，计算次日、3 日、7 日留存率。

```sql
-- MySQL 8.0+ 窗口函数解法
with first_login as (
  -- 每个用户首次登录日期
  select user_id, min(login_date) as first_date
  from logins
  group by user_id
),
retention as (
  select
    f.first_date,
    count(distinct f.user_id) as new_users,
    count(distinct case when datediff(l.login_date, f.first_date) = 1 then f.user_id end) as day1_retained,
    count(distinct case when datediff(l.login_date, f.first_date) = 3 then f.user_id end) as day3_retained,
    count(distinct case when datediff(l.login_date, f.first_date) = 7 then f.user_id end) as day7_retained
  from first_login f
  left join logins l on f.user_id = l.user_id
  group by f.first_date
)
select
  first_date,
  new_users,
  round(day1_retained * 100.0 / new_users, 2) as day1_rate,
  round(day3_retained * 100.0 / new_users, 2) as day3_rate,
  round(day7_retained * 100.0 / new_users, 2) as day7_rate
from retention;
```

**核心思路**：先找出每个用户的首次登录日期作为基准，再通过 `DATEDIFF` 判断后续登录与首次登录的时间差，按不同天数窗口统计留存用户数。

### 累计求和与移动平均

**题目**：表 `sales` 含 `sale_date` 和 `amount`，计算每日累计销售额和 7 日移动平均。

```sql
select
  sale_date,
  amount,
  sum(amount) over (order by sale_date) as cumulative_sum,
  round(avg(amount) over (
    order by sale_date
    rows between 6 preceding and current row
  ), 2) as moving_avg_7d
from sales;
```

**核心思路**：`SUM() OVER (ORDER BY)` 实现累计求和；`ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` 定义滑动窗口范围，计算过去 7 天（含当天）的平均值。

### 会话划分

**题目**：表 `events` 含 `user_id` 和 `event_time`，若两次操作间隔超过 30 分钟则视为新会话，划分每个用户的会话。

```sql
with gaps as (
  select user_id, event_time,
    lag(event_time) over (partition by user_id order by event_time) as prev_time
  from events
),
sessions as (
  select user_id, event_time,
    sum(case when prev_time is null or timestampdiff(minute, prev_time, event_time) > 30
        then 1 else 0 end)
      over (partition by user_id order by event_time) as session_id
  from gaps
)
select user_id, session_id,
  min(event_time) as session_start,
  max(event_time) as session_end,
  count(*) as event_count
from sessions
group by user_id, session_id;
```

**核心思路**：先用 `LAG()` 找出相邻事件的时间差，标记间隔超过阈值的断点，再通过累计求和分配会话 ID，最后按会话聚合。

### 行列转换

**题目**：表 `scores` 含 `student`、`subject`、`score`，将科目行转为列。

```sql
-- 行转列（PIVOT）
select student,
  max(case when subject = '语文' then score end) as chinese,
  max(case when subject = '数学' then score end) as math,
  max(case when subject = '英语' then score end) as english
from scores
group by student;

-- 列转行（UNPIVOT，MySQL 8.0+ 使用 JSON_TABLE 或 UNION ALL）
select student, '语文' as subject, chinese as score from scores
union all
select student, '数学', math from scores
union all
select student, '英语', english from scores;
```

**核心思路**：行转列利用 `CASE WHEN` + 聚合函数将多行值压缩到单行；列转行通过 `UNION ALL` 将多列展开为多行。

### 漏斗分析

**题目**：表 `funnel` 含 `user_id`、`step`（步骤 1~4）、`step_time`，计算每一步的转化率。

```sql
with step_counts as (
  select
    count(distinct case when step >= 1 then user_id end) as step1_users,
    count(distinct case when step >= 2 then user_id end) as step2_users,
    count(distinct case when step >= 3 then user_id end) as step3_users,
    count(distinct case when step >= 4 then user_id end) as step4_users
  from funnel
)
select
  step1_users,
  step2_users,
  step3_users,
  step4_users,
  round(step2_users * 100.0 / step1_users, 2) as step1_to_2_rate,
  round(step3_users * 100.0 / step2_users, 2) as step2_to_3_rate,
  round(step4_users * 100.0 / step3_users, 2) as step3_to_4_rate
from step_counts;
```

**核心思路**：利用 `COUNT(DISTINCT CASE WHEN step >= N)` 统计到达每一步的独立用户数，再逐级计算转化率。
