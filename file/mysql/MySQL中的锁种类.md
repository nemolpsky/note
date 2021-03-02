### MySQL中的锁种类

MySQL中上锁其实是基于索引，最基本的记录锁一般不会出什么问题，但是间隙锁是MySQL的一种独有的锁，使用不当的话等同于锁住全表需要详细了解下，以免发生死锁问题。

---


#### 1. 排他锁和共享锁

行锁可以分为排他锁和共享锁两种类型，都是在InnoDB引擎下才有的行锁，并且在不同的隔离级别下有着不同的特性。

- 排他锁

    排他锁是更新和删除的时候使用的，英文是exclusive lock，所以被称为X锁，这个蛮好理解的，同一时间同一行数据只能有一个排他锁，也只能有一个事务持有，避免并发修改和删除，这个时候其他事务对被锁住的数据什么都做不了。如果是REPEATABLE READ隔离级别，也就是MySQL默认的隔离级别下，其他事务没有获取锁也能读取这行数据，但是读取的是快照，但如果是更高级别的SERIALIZABLE时，即使是读取也会被阻塞住，参考[MySQL一致性读详解](https://github.com/nemolpsky/note/blob/master/file/mysql/MySQL%E4%B8%80%E8%87%B4%E6%80%A7%E8%AF%BB.md)中的3、4小节。

- 共享锁

    共享锁英文是shared lock，所以被称为S锁，读取的时候会使用，它不会阻塞其他事务读取，而且是每个读取的时候都会加一个共享锁，也就是说同一时间同一行数据上可能会有多个共享锁，但是注意，是多个共享锁，如果A事务对某行数据上了S锁，B事务也可以对这行数据上个S锁，但是不能上X锁(也就是更新操作)，上X锁会被阻塞。

    |A事务| B事务 |
    |  :-:    |:-:        |
    |begin;||
    |select * from table1 where score = 1 LOCK IN SHARE MODE; score字段上有索引，该行数据会上一个S锁，LOCK IN SHARE MODE是使用共享锁，否则是直接读快照||
    ||update table1 set score=2 where score=1; 更新同一行数据，需要上X锁，会阻塞住|

    下面这个例子证明了使用S锁就不会新建快照。

    |A事务| B事务 |
    |  :-:    |:-:        |
    |begin;||
    |select * from table1 where score = 1 LOCK IN SHARE MODE; 查出一行数据，但是不会建立快照||
    ||insert into table1(score) values(2);|
    |select * from table1; 这个时候会查出上面B事务插入的数据，这表示现在才建立了快照，否则快照应该是只有一条数据||
    ||insert into table1(score) values(3); 再插入一条数据|
    |select * from table1; 还是两条数据，查询的快照||
    |commit; ||
    |select * from table1; 现在可以查询到3条数据，快照更新了||
    
---

#### 2. 意图锁
英文名是Intention Locks，它也有两种类型，意图共享锁和意图排他锁，英文分别是intention shared lock和intention exclusive lock，简称分别是IS和IX。需要注意下，意图锁不会阻塞任何东西，意图锁的作用就是能够显示某个事务将要锁定一行数据，或者是正在锁定一行数据。所以上一个S锁之前会上一个IS锁，同样上一个X锁之前也会上一个IX锁，而且在上S和X锁之前会检查行数据上的意图锁，是否匹配。

其实它的作用类似于提供一个快速获取全表锁的情况，举个例子，MySQL中的排他锁是会阻塞其它加锁操作的，假如A事务想要给全表数据上一个排他锁，首先肯定要判断是否有别的排他锁存在，如果是的话就要等这个锁释放，锁的范围是全表，那么数据库怎么获取锁的状况？就是依靠意图锁，否则它只能全表扫描查看是否有哪行数据被锁住。

||X|IX|S|IS|
|:-:|:-:|:-:|:-:|:-:|
|X|冲突|冲突|冲突|冲突|
|IX|冲突|不冲突|冲突|不冲突|
|S|冲突|冲突|不冲突|不冲突|
|IS|冲突|不冲突|不冲突|不冲突|

---


#### 3. 记录锁
Record Locks，也是基于索引加锁的，例如```SELECT * FROM table1 WHERE ranks = 1 FOR UPDATE;```这样查询单条数据的就是记录锁，哪怕查询的字段没有索引也不影响，因为使用InnoDB引擎的话每条数据都默认会有一个聚簇索引，有主键的话，主键就是聚簇索引，如果没有主键也有一个隐式的聚簇索引。但是有一点要注意，如果这个时候使用的筛选字段不是唯一索引，查询又为空，就会产生间隙锁。

---

#### 4. 间隙锁
Gap Locks，间隙锁是锁住一个范围，只存在于默认隔离级别下，是为了解决幻读问题。现在使用下面这张表来进行测试。

```
create table table1
(
    id    int auto_increment
        primary key,
    score int null,
    ranks int null,
    constraint score_key
        unique (score)
);

create index ranks_index
    on table1 (ranks);

INSERT INTO testDatabase.table1 (id, score, ranks) VALUES (5, 5, 5);
INSERT INTO testDatabase.table1 (id, score, ranks) VALUES (7, 7, 7);
INSERT INTO testDatabase.table1 (id, score, ranks) VALUES (10, 10, 10);
```

比如下面这个例子，实际上生成了3个间隙锁，(-∞,5)，(5,7)，(7,10)，注意是间隙锁，锁的就是这四条数据之间的间隙，也就是没有数据的这段范围，所以可以看到都是开区间。另外注意看，对数据的更新也会阻塞，这是上面说到的记录锁，锁住每行数据，实际使用中往往就是间隙锁+记录锁出现，MySQL对这种组合有个官方的称呼，叫Next-Key Locks。它的范围是(-∞,5]，(5,7]，(7,10]，变成了左开右闭的形式，闭其实就是把记录锁包括进来了。

|A事务| B事务 |
|  :-:    |:-:        |
|begin;||
|select * from table1 where score between 5 and 7 for update;||
||insert into table1(id, score, ranks) VALUE (1,1,1); 阻塞|
||insert into table1(id, score, ranks) VALUE (2,2,2); 阻塞|
||insert into table1(id, score, ranks) VALUE (3,3,3); 阻塞|
||insert into table1(id, score, ranks) VALUE (4,4,4); 阻塞|
||insert into table1(id, score, ranks) VALUE (6,6,6); 阻塞|
||insert into table1(id, score, ranks) VALUE (8,8,8); 阻塞|
||insert into table1(id, score, ranks) VALUE (9,9,9); 阻塞|
||insert into table1(id, score, ranks) VALUE (11,11,11); 不阻塞|
||update table1 set ranks=55 where ranks=5; 阻塞|
||update table1 set ranks=77 where ranks=7; 阻塞|
||update table1 set ranks=100 where ranks=10; 阻塞|


如果把ranks字段去掉索引，会执行了全表扫描，生成的Next-Key Lock会覆盖去表，(-∞,3],（3,7],（7,10]，(10,+∞)。官方文档上的例子也是如此。所以一定要注意，基于没有索引的字段加锁查询和锁表没有区别。
|A事务| B事务 |
|  :-:    |:-:        |
|begin;||
|select * from table1 where ranks=7 for update;||
||mysql> insert into table1(id,score,ranks) values(1,1,1); 失败|
||mysql> insert into table1(id,score,ranks) values(2,2,2); 失败|
||mysql> update table1 set ranks=33 where ranks=3; 失败|
||mysql> insert into table1(id,score,ranks) values(4,4,4); 失败|
||mysql> insert into table1(id,score,ranks) values(6,6,6); 失败|
||mysql> insert into table1(id,score,ranks) values(8,8,8); 失败|
||mysql> update table1 set ranks=77 where ranks=7; 失败|
||mysql> insert into table1(id,score,ranks) values(9,9,9); 失败|
||mysql> update table1 set ranks=100 where ranks=10; 成功|


间隙锁是MySQL为了解决并发和性能的一种产物，可以说最复杂的就是间隙锁了，再看最后一个例子，可以发现A事务使用共享锁也会触发一个间隙锁，因为id为5的数据不存在，所以会锁住[3,7)，注意id为7的数据没有锁住，间隙锁区间是左闭右开，最重要的是后面B事务使用排他锁却可以成功，两者都可以触发间隙锁，但是并没有因此产生阻塞。

|A事务| B事务 |
|  :-:    |:-:        |
|begin;||
|select * from table1 where ranks=5 lock in share mode;||
||mysql> insert into table1(id,score) values(2,2,2); 成功|
||mysql> insert into table1(id,score) values(4,4,4); 失败|
||select * from table1 where ranks=5 for update; 成功|
||update table1 set ranks=77 where ranks=7; 成功|


间隙锁是为了解决幻读的产物，又不能影响到MySQL上其它原有的设计，所以除了上面一些通用的规则之外还会有很多细节，总之加锁的时候要慎重，考虑周全，试验下。

- https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-gap-locks
- https://time.geekbang.org/column/article/75173
- https://time.geekbang.org/column/article/75659