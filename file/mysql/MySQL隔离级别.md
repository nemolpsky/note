### MySQL隔离级别

MySQL事务有以下四大特性

- 原子性，同一个事务内所有的操作都是一个整体，要么全失败，要么全成功。
- 一致性，要保证所有的操作满足数据库的完整性约束，说白了就是遵守所有数据库的规范约束，不能出现非法数据，比如更新完一条数据之后会发生主键冲突的问题，应该报错回滚执行。
- 隔离性，不同的事务之间的操作不能影响到其它的事务。
- 持久性，数据库的数据应该是持久化保存的，不会因为宕机或者异常操作造成数据丢失。


---


#### 1. READ UNCOMMITTED（读未提交）

最低的隔离级别，会造成脏读。下面A事务把id为1的列score数据修改成5，还没提交B事务就已经可以读取到修改的值。

|A事务                                                         | B事务                                                           |
|:-:                                                           |:-:                                                               |
|                                                              |SET @@session.transaction_isolation = 'READ-UNCOMMITTED';         |
|                                                              |select * from table1; 查询出id为1，score为10的数据                |
|SET @@session.transaction_isolation = 'READ-UNCOMMITTED';     |                                                                  |
|begin;                                                        |                                                                  |
|update table1 set score=15 where id = 1; 修改成功，但没有提交|                                                                  |
|                                                              |select * from table1;能查到B事务把score修改为15                  |
---

#### 2. READ COMMITTED（读提交）

解决了```读未提交```的脏读问题，只有提交事务之后才会读取到数据的变更，但是又引入了不可重复读问题，也就是同一个事务内读取多次可能获得结果不同。

下面B事务在当前事务内分别读取两次，第一次读取是在修改完但是没提交事务的时候，第二次是修改完提交成功后的时候，这两次读取的结果不一样，因为读提交是没法读取到A事务没提交之前做的修改，所以两次结果不一致，这就是不可重复读的问题。

|A事务                                                       | B事务                                                       |
|:-:                                                         |:-:                                                          |
|SET @@session.transaction_isolation = 'READ-COMMITTED';     |                                                             |
|                                                   begin;   |                                                             |
|select * from table1; 查询出一条id为1，score为15的数据      |                                                             |
|update table1 set score=20 where id = 1; 成功执行更新操作   |                                                             |
|                                                            |select * from table1;查询出id为1的数据，score仍然是15         |
|commit;                                                     |                                                             |
|                                                            |select * from table1;查询出id为1的数据，score变为是20         |

---

#### 3. REPEATABLE READ（可重复读）

这是MySQL默认的隔离级别，比```读提交```级别更高，能够解决不可重复读，比如那个例子，B事务两次查询会得到一样的结果，MySQL是利用了[MVCC](https://github.com/nemolpsky/note/blob/master/file/mysql/MySQL%E4%B8%80%E8%87%B4%E6%80%A7%E8%AF%BB.md)来解决不可重复读的问题。

但是会存在幻读问题，比如下面的例子，表中已经有一条id为1的数据，B事务插入一条数据，使用自增id生成id为2的数据并提交，A事务是查不出来这条数据的，因为上面MVCC机制，A事务当前读取的时候快照数据，但是当A事务也想插入一条数据时会发现id为2的数据已经存在，所以插入的数据生成id为3，这就叫幻读。

其实幻读和不可重复读很像，都是原来不存在再查一遍又存在了，区别在于不可重复读是针对修改操作，而幻读是插入操作。MySQL对幻读也有做处理，内部设计了一种被称为[间隙锁](https://github.com/nemolpsky/note/blob/master/file/mysql/MySQL%E4%B8%AD%E7%9A%84%E9%94%81%E7%A7%8D%E7%B1%BB.md#4-%E9%97%B4%E9%9A%99%E9%94%81)的排他锁来解决幻读问题。

但是要注意，在使用快照读时，幻读问题是避免不了的，间隙锁是在当前读才会实现，也就是update更新或者for update上锁时才会触发当前读，而不是快照读。


|A事务                                                                         | B事务                                                            |
|  :-:                                                                         |:-:                                                               |
|SET @@session.transaction_isolation = 'REPEATABLE-READ';                      |                                                                  |
|begin;                                                                        |                                                                  |
|                                                                              |begin;                                                            |
|select * from table1; 查出id为1的数据                                         |                                                                  |
|                                                                              |insert into table1(score) values(25);                             |
|                                                                              |commit;                                                           |
|                                                                              |select * from table1;查出2条数据，id分别为1和2，2是B事务刚刚插入的 |
|select * from table1;还是只能查出id为1的数据                                  |                                                                   |
|insert into table1(score) values(30);                                         |                                                                  |
|select * from table1;查出2条数据，id分别为1和3，3是A事务刚刚插入的，没有id为2的 |                                                                  |
|commit;                                                                       |                                                                  |
|select * from table1; 可以查出3条数据，id分别是1，2，3                         |                                                                  |

---


#### 4. SERIALIZABLE（序列化）

这个是最高的隔离级别，串行执行，也就是任何的写操作执行的时候都必定是会锁住数据，这个时候无论其他的操作是读操作还是写操作都无法执行，都必须等这个写操作执行完。既然是串行化那必定是没有并发问题了，但是效率也是最差的。

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
