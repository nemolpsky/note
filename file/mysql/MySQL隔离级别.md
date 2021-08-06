### MySQL隔离级别

MySQL事务有以下四种特点，它其实是为了保证并发请求下数据的正常，其实这四点如果不用生涩的专业术语讲的话就是要保证并发安全，其中最重要最复杂的就是隔离性，日常开发中碰到的问题基本都是因为隔离性造成的。

- 原子性，同一个事务内所有的操作都是一个整体，要么全失败，要么全成功。
- 一致性，要保证所有的操作满足数据库的完整性约束，说白了就是遵守所有数据库的规范约束，不能出现非法数据，比如更新完一条数据之后会发生主键冲突的问题，应该报错回滚执行。
- 隔离性，不同的事务之间的操作不能影响到其它的事务。
- 持久性，数据库的数据应该是持久化保存的，不会因为宕机或者异常操作造成数据丢失。
---


#### 1. READ UNCOMMITTED（读未提交）

最低的隔离级别，顾名思义，一个事务会读取到没有提交的事务所做的修改，也就是脏读。比如下面A事务可以读取到B事务未提交的数据，如果A读取到后B又回滚了，那么读取到的数据就有问题，也就是脏读。

|A事务| B事务 |
|  :-:    |:-:        |
|SET @@session.transaction_isolation = 'READ-UNCOMMITTED';||
|select * from table1; 查询出id为1，score为10的数据||
||SET @@session.transaction_isolation = 'READ-UNCOMMITTED';|
||begin;|
||update table1 set score=15 where id = 1; 修改成功，但没有提交|
|select * from table1;能查到B事务把score修改为15||
---

#### 2. READ COMMITTED（读提交）

解决了```READ UNCOMMITTED```的脏读问题，只有提交事务之后才会读取到数据的变更，但是又引入了不可重复读问题，也就是读取多次可能获得结果不同，比如下面B事务分别两次读取都会得到不同结果，因为一次是数据修改未做修改时的旧数据，另一次是提交后的新数据。

|A事务| B事务 |
|  :-:    |:-:        |
|SET @@session.transaction_isolation = 'READ-COMMITTED';||
|begin;||
|select * from table1; 查询出一条id为1，score为15的数据||
|update table1 set score=20 where id = 1; 成功执行更新操作||
||select * from table1;查询出id为1的数据，score仍然是15|
|commit;||
||select * from table1;查询出id为1的数据，score变为是20|

---

#### 3. REPEATABLE READ（可重复读）

是MySQL默认的隔离级别，比```READ UNCOMMITTED```级别更高，它解决了不重复读的问题，比如再执行上面那个例子，即使A事务做了修改提交B事务两次查询依然会得到一样的结果，MySQL是利用了[MVCC](https://github.com/nemolpsky/note/blob/master/file/mysql/MySQL%E4%B8%80%E8%87%B4%E6%80%A7%E8%AF%BB.md)来解决不可重复读的问题。

但是这种情况下会有幻读问题，比如下面的例子，B事务插入提交一条数据数据，使用自增id生成id为2的数据，A事务是查不出来这条数据的，但是当A事务再插入一条数据提交时会发现id为2的数据已经存在，所以生成id为3的数据，这就叫幻读。其实幻读和不可重复读很像，都是原来不存在再查一遍又存在了，区别在于不可重复读是针对修改操作，而幻读是插入操作。MySQL对幻读也有做处理，内部设计了一种被成为[间隙锁](https://github.com/nemolpsky/note/blob/master/file/mysql/MySQL%E4%B8%AD%E7%9A%84%E9%94%81%E7%A7%8D%E7%B1%BB.md#4-%E9%97%B4%E9%9A%99%E9%94%81)的排他锁来解决幻读问题。


|A事务| B事务 |
|  :-:    |:-:        |
|SET @@session.transaction_isolation = 'REPEATABLE-READ'; ||
|begin;||
||begin;|
|select * from table1; 查出id为1的数据||
||insert into table1(score) values(25);|
||commit;|
||select * from table1;查出2条数据，id分别为1和2，2是B事务刚刚插入的|
|select * from table1;还是只能查出id为1的数据||
|insert into table1(score) values(30);||
|select * from table1;查出2条数据，id分别为1和3，3是A事务刚刚插入的，没有id为2的||
|commit;||
|select * from table1; 可以查出3条数据，id分别是1，2，3||

---


#### 4. SERIALIZABLE（序列化）

这个是最高的隔离级别，也最好理解，串行执行，也就是任何的写操作执行的时候都必定是会锁住数据，这个时候无论其他的操作是读操作还是写操作都无法执行，都必须等这个写操作执行完。既然是串行化那必定是没有并发问题了，但是效率也是最差的。

|A事务| B事务 |
|  :-:    |:-:        |
||SET @@session.transaction_isolation = 'SERIALIZABLE';|
||begin;|
||insert into table1(score) values(45);|
|SET @@session.transaction_isolation = 'SERIALIZABLE';||
|begin;||
|select * from table1; 这个查询会一直卡住，直到B事务提交||
||commit; 提交后，A事务的查询才能够执行|

---



### 总结

总的来说隔离级别越高，性能越差，但是也越安全，尤其是可重复读这个级别，是MySQL的默认级别，因此基于这个级别，MySQL还有其他的一些机制来弥补它的不足，有很多种锁在不同的隔离级别下也有着不同的效果，如果想要理解MySQL锁的机制，先了解不同的隔离级别之间的差异是必不可少的。

|隔离级别|脏读|不可重复读|幻读|
|:-:|:-:|:-:|:-:|
|读未提交|	可以出现|	可以出现|	可以出现|
|读提交|	不允许出现|	可以出现|	可以出现|
|可重复读|	不允许出现|	不允许出现|	可以出现|
|序列化	|不允许出现	|不允许出现	|不允许出现|