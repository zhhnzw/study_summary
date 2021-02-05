## MySQL 基础

### 连接管理

每个客户端连接都会在服务器进程中拥有一个线程，这个连接的查询只会在这个单独的线程中执行，该线程只能在某个CPU核心中运行。

### 数据类型

### 存储引擎

| 对比项   | InnoDB                                                       | MyISAM                                                       |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 外键     | 支持                                                         | 不支持                                                       |
| 事务     | 支持                                                         | 不支持                                                       |
| 行表锁   | 行锁<br>操作时只锁住该行记录<br>适合高并发的场景             | 表锁<br>即使操作一条记录也会锁住整个表<br>不适合高并发的场景 |
| 缓存     | 不仅缓存索引<br>还缓存真实数据<br>对内存要求较高<br>内存大小对性能有决定性影响 | 只缓存索引<br>不缓存真实数据                                 |
| 占用空间 | 大                                                           | 小                                                           |
| 关注点   | 事务                                                         | 性能                                                         |

### SQL

#### 连接查询（[Join](join.md)）

#### 事务
