### MySQL一致性读详解

因为MySQL的默认隔离级别是可重复读(REPEATABLE READ)，解决了不可重复读的问题，不可重复读的问题本质其实就是事务必须提交，数据才可见，而这又是为了解决脏读问题。MySQL是利用MVCC(多版本控制并发)机制来实现不同隔离级别下不同的效果，绝大多数数据库都是采用MVCC来解决这个问题。

这篇文章其实本质上就是讲MVCC如何实现各个隔离级别，不了解的先看看[MySQL隔离级别](https://github.com/nemolpsky/note/blob/master/file/mysql/MySQL%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB.md)，对隔离级别的特性有了理解再来看。


---


#### 1. MVCC

按照维基百科的介绍来讲，MVCC(多版本并发控制，Multiversion concurrency control)就是一个事务执行的时候，它的读取操作不是直接读取数据库中的数据，而是基于数据建立一份快照，而每个快照都会有一个版本。而如果一个事务是执行写操作，也不是真的写，而是创建一个新的快照版本，到提交的时候才真正的写覆盖。


---


#### 2. REPEATABLE READ下的MVCC
因为MVCC是基于快照实现，所以有时候在MySQL中也被称为快照读，官方文档的介绍是这样的，每次查询都会新建当前最新数据的快照，所以别的事务修改数据提交也影响不了当前快照数据，以此解决不可重复读的问题。

>The query sees the changes made by transactions that committed before that point of time, and no changes made by later or uncommitted transactions. The exception to this rule is that the query sees the changes made by earlier statements within the same transaction. This exception causes the following anomaly: If you update some rows in a table, a SELECT sees the latest version of the updated rows, but it might also see older versions of any rows. If other sessions simultaneously update the same table, the anomaly means that you might see the table in a state that never existed in the database.


但是正如文档最后一段所说，又引入了幻读问题，快照中没有最新数据，但是别的事务可能已经新增了数据，而更新操作是基于当前读，如果你也更新数据，就会发现快照数据查询没查到别的事务新增的数据，新增数据却会有冲突。

>
If the transaction isolation level is REPEATABLE READ (the default level), all consistent reads within the same transaction read the snapshot established by the first such read in that transaction. You can get a fresher snapshot for your queries by committing the current transaction and after that issuing new queries.


再看下面这个操作，A事务的第一条select语句会创建一个快照的，后面第事务提交后的select语句又会创建第二个快照。注意，默认是select操作才会触发快照，只是单纯的begin则不会。

|A事务                                                      | B事务                                                    |
|:-:                                                        |:-:                                                       |
|begin;                                                     |                                                          |
|select * from table1; 无数据                               |                                                          |
|                                                           |begin;                                                    |
|                                                           |insert into table1(score) values(20); 插入一条id为6的数据 |
|select * from table1; 无数据                               |                                                          |
|insert into table1(score) values(25); 插入一条id为7的数据  |                                                          |
|                                                           |commit;                                                   |
|commit;                                                    |                                                          |
|select * from table1; 两条数据都查询到了                    |                                                          |

还有一点要注意的就是，快照不是局限于单张表，只要是select语句就会快照整个库的数据。
|A事务                                                       | B事务                                                  |
|:-:                                                         |:-:                                                     |
|begin;                                                      |                                                        |
|select * from table2; 建立快照                              |                                                        |
|                                                            |insert into table1(score) values(1);                    |
|select * from table1; 查不到数据                            |                                                        |



如果想要在事务刚开始时就建立快照，可以使用```START TRANSACTION WITH CONSISTENT SNAPSHOT;```语句。

|A事务                                                        | B事务                                                       |
|:-:                                                          |:-:                                                          |
|START TRANSACTION WITH CONSISTENT SNAPSHOT;                  |                                                             |
|                                                             |insert into table1(score) values(1);                         |
|select * from table1; 已经建立快照，所以查不到数据            |                                                             |




---

#### 3. MVCC对DML语句的作用

官方文档上有特别标注，DML语句(删除和修改)不使用MVCC，以下是官方例子。

```
The snapshot of the database state applies to SELECT 
statements within a transaction, not necessarily to DML 
statements. If you insert or modify some rows and then 
commit that transaction, a DELETE or UPDATE statement issued 
from another concurrent REPEATABLE READ transaction could 
affect those just-committed rows, even though the session 
could not query them. If a transaction does update or delete 
rows committed by a different transaction, those changes do 
become visible to the current transaction. For example, you might encounter a situation like the following:

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

下面这个例子中，A事务查询不到B事务插入的数据，因为A事务查询的是快照，但是当A事务执行delete和update操作时，是可以影响到B事务的数据，因为这时是当前读，也就是读取当前最新数据，而不是读取快照数据。

不过这也表明MySQL的默认隔离级别可重复读还是存在幻读问题，如果这个时候A事务的逻辑是插入一条数据，它在插入前查询数据库来确保没有重复，而B事务也插入一条相同的数据，因为A事务查询使用的是快照读，它查不到B事务刚插入的数据，但是当A事务自己也插入这条数据时就会造成数据重复，所以MySQL还有[间隙锁](https://github.com/nemolpsky/note/blob/master/file/mysql/MySQL%E4%B8%AD%E7%9A%84%E9%94%81%E7%A7%8D%E7%B1%BB.md#4-%E9%97%B4%E9%9A%99%E9%94%81)的机制来解决幻读问题。
|A事务                                                 |B事务                                               |
|:-:                                                   |:-:                                                 |
|begin;                                                |                                                    |
|select * from table1; 没有数据                        |                                                    |
|                                                      |insert into table1(score) values(1);                |
|select * from table1; 没有数据                        |                                                    |
|delete from table1 where id>0; 提示影响了一行数据     |                                                    |
|                                                      |select * from table1; 可以查到数据                  |
|commit;                                               |                                                    |
|                                                      |select * from table1; 数据被删除了                  |

---

### 官方文档
> https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html
