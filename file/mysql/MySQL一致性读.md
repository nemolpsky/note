### MySQL一致性读详解

因为MySQL的默认隔离级别是可重复读(REPEATABLE READ)，解决了不可重复读的问题，不可重复读的问题本质其实就是事务必须提交，数据才可见，而这又是为了解决脏读问题。MySQL是利用MVCC(多版本控制并发)机制来实现不同隔离级别下不同的效果，绝大多数数据库都是采用MVCC来解决这个问题。

这篇文章其实本质上就是讲MVCC如何实现各个隔离级别，不了解的先看看[MySQL隔离级别]()，对隔离级别的特性有了理解再来看。


---


#### 1. MVCC

按照维基百科的介绍来讲，MVCC就是一个事务执行的时候，它的读取操作不是直接读取数据库中的数据，而是基于数据建立一份快照，而每个快照都会有一个版本。而如果一个事务是执行写操作，也不是真的写，而是创建一个新的快照版本，到提交的时候才真正的写覆盖。


---


#### 2. REPEATABLE READ下的MVCC
因为MVCC是基于快照实现，所以有时候在MySQL中也被称为快照读，官方文档的介绍是这样的，大致上就是说快照其实就是一次事务查询之前的最新数据快照，所以即便是别的事务修改提交了，当前事务也看不到，因为读取的是快照，所以这就解决了不可重复读的问题，快照是不会变的。

```
The query sees the changes made by transactions that committed before that point of time, and no changes made by later or uncommitted transactions. The exception to this rule is that the query sees the changes made by earlier statements within the same transaction. This exception causes the following anomaly: If you update some rows in a table, a SELECT sees the latest version of the updated rows, but it might also see older versions of any rows. If other sessions simultaneously update the same table, the anomaly means that you might see the table in a state that never existed in the database.
```

但是正如文档最后一段所说，又引入了幻读问题，你读取快照的时候明明没有数据，但是实际上别的事务已经修改了，你一提交，快照也会更新版本，你读取最新的快照又读取到数据了，这就是幻读。

```
If the transaction isolation level is REPEATABLE READ (the default level), all consistent reads within the same transaction read the snapshot established by the first such read in that transaction. You can get a fresher snapshot for your queries by committing the current transaction and after that issuing new queries.
```

再看下面这个操作，解决了不可重复读，引入了幻读，快照就是在A事务的第一条select语句下新建的，就像文档说的那样，想要获取最新的快照提交当前事务然后发出新的查询，注意的是，一定是select操作才会触发快照，只是单纯的begin则不会。

|A事务| B事务 |
|  :-:    |:-:        |
|begin;||
|select * from table1; 无数据||
||begin;|
||insert into table1(score) values(20); 插入一条id为6的数据|
|select * from table1; 无数据||
|insert into table1(score) values(25); 插入一条id为7的数据||
||commit;|
|commit;||
|select * from table1; 两条数据都查询到了||

还有一点要注意的就是，只要是select语句就会触发快照，不要求是select同一张表。
|A事务| B事务 |
|  :-:    |:-:        |
|begin;||
|select * from table2; 建立快照||
||insert into table1(score) values(1);|
|select * from table1; 查不到数据||
|A事务| B事务 |
|  :-:    |:-:        |
|begin;||
||insert into table1(score) values(1);|
|select * from table2; 建立快照||
|select * from table1; 可以查到数据||


文档上还说了，以select语句触发建立快照，如果使用```START TRANSACTION WITH CONSISTENT SNAPSHOT;```语句则可以在事务刚开始时就建立快照。

|A事务| B事务 |
|  :-:    |:-:        |
|START TRANSACTION WITH CONSISTENT SNAPSHOT;||
||insert into table1(score) values(1);|
|select * from table1; 已经建立快照，所以查不到数据||

#### 3. REPEATABLE COMMITTED下的MVCC

在REPEATABLE COMMITTED隔离级别下，快照的新建规则就简单多了，就像前面说的，读提交隔离级别可以解决脏读，但是不能解决不可重复读，但是没有幻读的问题。

|A事务| B事务 |
|  :-:    |:-:        |
|select * from table1;||
||insert into table1(score) values(1);|
|select * from table1; 可以查询到数据||


---

#### 4. MVCC对DML语句的作用

官方文档上有特别标注，DML语句，也就是删除和修改不按上面那套规则走，只要执行了，事务就能查的到，还给了例子。

```
The snapshot of the database state applies to SELECT statements within a transaction, not necessarily to DML statements. If you insert or modify some rows and then commit that transaction, a DELETE or UPDATE statement issued from another concurrent REPEATABLE READ transaction could affect those just-committed rows, even though the session could not query them. If a transaction does update or delete rows committed by a different transaction, those changes do become visible to the current transaction. For example, you might encounter a situation like the following:

SELECT COUNT(c1) FROM t1 WHERE c1 = 'xyz';
-- Returns 0: no rows match.
DELETE FROM t1 WHERE c1 = 'xyz';
-- Deletes several rows recently committed by other transaction.

SELECT COUNT(c2) FROM t1 WHERE c2 = 'abc';
-- Returns 0: no rows match.
UPDATE t1 SET c2 = 'cba' WHERE c2 = 'abc';
-- Affects 10 rows: another txn just committed 10 rows with 'abc' values.
SELECT COUNT(c2) FROM t1 WHERE c2 = 'cba';
-- Returns 10: this txn can now see the rows it just updated.
```

做个简单的例子，事务A虽然查询的时候查不到事务B插入的数据，那是因为事务查询的是快照，但是如果执行的delete和update操作时，事务A是可以查到事务B的数据，查的可不是快照，而是真的数据，这个时候可是真的会对数据进行修改。
|A事务| B事务 |
|  :-:    |:-:        |
|begin;||
|select * from table1; 没有数据||
||insert into table1(score) values(1);|
|select * from table1; 没有数据||
|delete from table1 where id>0; 提示影响了一行数据||
||select * from table1; 可以查到数据|
|commit;||
||select * from table1; 数据被删除了|

---

### 官方文档
> https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html