## 锁

乐观锁不是真正的锁，只是一种并发控制的思想

### 共享锁(shared lock)和排他锁(exclusive lock)

也叫读锁(read lock)和写锁(write lock)

读锁是共享的，多个客户端在同一时刻可以同时读取同一个资源，而互不干扰

写锁是排他的，一个写锁会阻塞其他的写锁和读锁

这样可以确保在给定的时间里，只有一个用户能执行写入，并防止其他用户读取正在写入的同一资源

### 表锁和行锁

表级写锁：一个用户在对表进行写操作(插入、删除、更新等)前，需要先获得写锁，这会阻塞其他用户对该表的所有读写操作。

表级读锁：没有写锁时，其他读取的用户就能获得读锁，而读锁之间是不相互阻塞的。

**区别**

表锁：开销小，加锁快，不会出现死锁，锁粒度大，发生锁冲突概率高，并发低。

行锁：开销大，加锁慢，会出现死锁，锁粒度小，发生锁冲突概率低，并发高。

**应用场景**

表锁适用于大部分业务场景都是读，写操作很少的场景；而行锁可以最大程度的支持并发处理(同时也带来了最大的锁开销)，比如大并发下的扣库存操作

另：服务器会为诸如ALTER TABLE之类的语句使用表级写锁

**细节**

for update关键字。InnoDB默认是行级别的锁，当有明确指定的主键时候，是行级锁。否则是表级别。

例子: 假设表foods ，存在有id跟name、status三个字段，id是主键，status有索引。

例1:  (明确指定主键，并且有此记录，行级锁)

```sql
SELECT * FROM foods WHERE id=1 FOR UPDATE;
SELECT * FROM foods WHERE id=1 and name=’咖啡色的羊驼’ FOR UPDATE;
```

例2: (明确指定主键/索引，若查无此记录，无锁)

``` sql
SELECT * FROM foods WHERE id=-1 FOR UPDATE;
```

例3: (无主键/索引，表级锁)

``` sql
SELECT * FROM foods WHERE name=’咖啡色的羊驼’ FOR UPDATE;
```

例4: (主键/索引不明确，表级锁)

``` sql
SELECT * FROM foods WHERE id<>’3’ FOR UPDATE;
SELECT * FROM foods WHERE id LIKE ‘3’ FOR UPDATE;
```

for update的注意点

* for update 仅适用于InnoDB，并且必须开启事务，在begin与commit之间才生效。
* 要测试for update的锁表情况，可以利用MySQL的Command Mode，开启二个视窗来做测试。

Q: 当开启一个事务进行for update的时候，另一个事务也有for update的时候会一直等着，直到第一个事务结束吗？

A: 会的。除非第一个事务commit或者rollback或者断开连接，第二个事务会立马拿到锁进行后面操作。

Q:  如果没查到记录会锁表吗？

A: 表级锁时，不管是否查询到记录，都会锁定表; 行级锁不会。

[参考](https://www.jianshu.com/p/64fdb29b67aa)

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