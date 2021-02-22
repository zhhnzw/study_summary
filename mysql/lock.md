## 锁

### 共享锁(shared lock)和排他锁(exclusive lock)

也叫**读锁**(read lock)和**写锁**(write lock)

#### 读锁（如`LOCK TABLE test READ`）

| 加读锁的session                       | 其他session                                   |
| ------------------------------------- | --------------------------------------------- |
| 当前session可以查询该表记录           | 其他session也可以                             |
| 当前session不可以再查询其他没锁定的表 | 其他session可以查询或更新未锁定的表           |
| 当前session不可以再插入或更新锁定的表 | 当前session插入或更新锁定的表会一直等待获得锁 |

#### 写锁（如`LOCK TABLE test WRITE`）

| 加写锁的session                             | 其他session                                         |
| ------------------------------------------- | --------------------------------------------------- |
| 当前session对锁定表的查询、更新、插入都可以 | 其他session对该锁定表的任何操作都被阻塞，等待获得锁 |

#### 结论

读锁会阻塞写，但是不会阻塞读，而写锁则会把读和写都阻塞。

#### 查看表锁定情况

① `SHOW status LIKE 'table%'`

`Table_lock_imediate`产生表级锁定的次数，表示可以立即获得锁的查询次数，每立即获得锁，值+1。

`Table_locks_waited`：出现表级锁定争用而发生等待的次数（不能立即获得锁的次数，每等待一次锁值+1），此值高则说明存在着较严重的表级锁争用情况。

② `SHOW status LIKE 'innodb_row_lock%'`

`innodb_row_lock_current_waits`：当前正在等待锁定的数量。

`innodb_row_lock_time`：从系统启动到现在锁定总时间长度。

`innodb_row_lock_time_avg`：每次等待所花平均时间。

`innodb_row_lock_time_max`：从系统启动到现在等待最长的一次所花的时间。

`innodb_row_lock_time_waits`：系统启动后到现在总共等待的次数。

③ `SHOW profiles`

是MySQL提供的可以用来分析当前会话中语句执行的资源消耗情况的工具，可用于sql调优的测量。默认情况下处于关闭状态，并保存最近15次的运行结果。使用步骤：

1. 开启该功能。`SET profiling=on;`，查看：`SHOW variables LIKE 'profiling';`
2. 运行一些SQL，然后执行`SHOW profiles`
3. 查看详细运行参数：`SHOW profile cpu,block io for query {上一步的某个Query_ID};`

### 表锁和行锁

表级写锁：一个session在对表进行写操作(插入、删除、更新等)前，需要先获得写锁，这会阻塞其他session对该表的所有读写操作。

表级读锁：没有写锁时，其他读取的session就能获得读锁，而读锁之间是不相互阻塞的。

#### 区别

表锁：开销小，加锁快，不会出现死锁，锁粒度大，发生锁冲突概率高，并发低。

行锁：开销大，加锁慢，会出现死锁，锁粒度小，发生锁冲突概率低，并发高。

#### 应用场景

表锁适用于大部分业务场景都是读，写操作很少的场景；而行锁可以最大程度的支持并发处理(同时也带来了最大的锁开销)，比如大并发下的扣库存操作

另：服务器会为诸如ALTER TABLE之类的语句使用表级写锁

#### 细节

for update关键字**悲观锁**。InnoDB默认是行级别的锁，当`WHERE`的字段有索引的时候，是行级锁，否则是表级别。

例子: 假设表foods ，存在有id跟name、status三个字段，id是主键，status有索引。

例1:  明确指定主键，并且有此记录，行级锁

```sql
SELECT * FROM foods WHERE id=1 FOR UPDATE;
SELECT * FROM foods WHERE id=1 and name=’咖啡色的羊驼’ FOR UPDATE;
```

例2: 明确指定主键/索引，若查无此记录，无锁

``` sql
SELECT * FROM foods WHERE id=-1 FOR UPDATE;
```

例3: 无主键/索引，表级锁

``` sql
SELECT * FROM foods WHERE name=’咖啡色的羊驼’ FOR UPDATE;
```

例4: name字段没加单引号，MySQL底层对此字段调用了类型转换函数，导致索引失效，表级锁

``` sql
SELECT * FROM foods WHERE name=111 FOR UPDATE;
```

#### 间隙锁

当用范围条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件的已有数据的索引项加锁，对于键值条件范围内不存在的记录，叫做间隙。InnoDB也会对这个间隙加锁，这就是间隙锁。

例5: (范围条件检索数据时，行级锁，开区间，如下例，锁定了2、3、4行)

``` sql
SELECT * FROM foods WHERE id>1 AND id <5 FOR UPDATE;
```

注: `UPDATE`语句与上述6个例子特点类似，有索引就是行锁，否则是表锁；`INSERT`语句是行锁

for update的注意点

* for update 仅适用于InnoDB，并且必须开启事务，在begin与commit之间才生效。
* 要测试for update的锁表情况，可以利用MySQL的Command Mode，开启二个视窗来做测试。

Q: 当开启一个事务进行for update的时候，另一个事务也对同一条记录for update的时候会一直等着，直到第一个事务结束吗？

A: 会的。除非第一个事务commit或者rollback或者断开连接，第二个事务会立马拿到锁进行后面操作。

Q:  如果没查到记录会锁表吗？

A: 表级锁时，不管是否查询到记录，都会锁定表（MyISAM）; 行级锁不会（InnoDB）。

### 乐观锁

乐观锁不是真正的锁，只是一种并发控制的思想。

1. 先查出`version`字段，假设查出来的值为1
2. `update user set point = point + 20, version = version + 1 where userid=1 and version=1`

### 死锁

是指两个或者多个**事务**在同一资源上互相占用，并请求锁定对方占用的资源，从而导致恶性循环。

```sql
START TRANSACTION;
UPDATE StockPrice SET close=45.50 WHERE stock_id=4 and date = '2002-05-01';
UPDATE StockPrice SET close=19.80 WHERE stock_id=3 and date = '2002-05-02';
COMMIT;
```

```sql
START TRANSACTION;
UPDATE StockPrice SET high=20.12 WHERE stock_id=3 and date = '2002-05-02';
UPDATE StockPrice SET high=47.20 WHERE stock_id=4 and date = '2002-05-01';
COMMIT;
```

如果凑巧，两个事务都执行了第一条`UPDATE`语句，更新了一行数据，同时也锁定了该行数据, 接着每个事务都尝试去执行第二条`UPDATE`语句，却发现该行已经被对方锁定，然后两个事务都等待对方释放锁，同时又持有对方需要的锁，则陷入死循环。

死锁解决方案：死锁发生以后，只有部分或者完全回滚其中一个事务，才能打破死锁。大多数情况下只需要重新执行因死锁回滚的事务即可。