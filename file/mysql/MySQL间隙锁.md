## MySQL间隙锁

MySQL默认是RR隔离级别，引入了快照读的机制来解决不可重复读问题，但是还是依然有幻读问题，间隙锁就是为了解决幻读的问题，不过只是部分解决，极端情况下依然会有幻读问题。

说到间隙锁不得不提Next-Key Locks锁，就是行锁+间隙锁，以下有几条关键规则。


1. 加锁的基本单位是Next-Key Locks，规则都是前开后闭区间。
2. 查找过程中只有访问到的索引都会加锁。
3. 索引上的等值查询，给唯一索引加锁的时候，Next-Key Locks退化为行锁。
4. 索引上的等值查询，向右遍历时且最后一个值不满足当前where条件时，Next-Key Locks退化为前开后开区间。

---

#### 一、精准查询

```
CREATE TABLE `test` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);

```

|A事务                                                      | B事务                                                    | C事务                                                    |
|:-:                                                        |:-:                                                       |:-:                                                       |
|begin;                                                     |                                                          |                                                         |
|update t set d = d+1 where id = 7;                         |                                                          |                                                         |
|                                                           |insert into t values(8,8,8);阻塞                          |                                                         |
|                                                           |                                                          |update t set d = d+1 where id =10;成功                    |

A事务执行了一个精准查询，但是该值不存在，所以生成了一个(5,10]的间隙锁，注意5是开区间，所以不包含5，根据上面第四条规则，7不等于10，所以生成了(5,10)的间隙锁，因此插入8会阻塞，插入10会成功。

---

#### 二、非唯一索引上锁

```
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);

```

|A事务                                                      | B事务                                                    | C事务                                                    |
|:-:                                                        |:-:                                                       |:-:                                                       |
|select id from t where c =5 lock in share mode;            |                                                          |                                                         |
|                                                           |update t set d = d+1 where id =5;成功                     |                                                         |
|                                                           |                                                          |insert into t values(7,7,7);阻塞                         |

A事务给select加了一个读锁，按上面的规则最终是生成了(5,10)的间隙锁，但是注意上面第2条规则，间隙锁是针对读取的索引上锁，c已经有索引了，所以不会锁住id索引，B事务的更新只涉及id索引，不涉及c字段的索引，所以可以成功。

---

#### 三、死锁

```
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);

```

|A事务                                                      | B事务                                                    |
|:-:                                                        |:-:                                                       |
|begin;                                                     |                                                          |
|select * from t where id = 9 for update;                   |                                                          |
|                                                           |begin;                                                    |
|                                                           |select * from t where id = 9 for update;                  |
|                                                           |insert into values(9,9,9);阻塞                            |
|insert into values(9,9,9); 死锁                            |                                                          |

间隙锁很复杂，规则又多，所以使用不当就会导致死锁，上面两个事务就会导致死锁
- A生成(5,10)的间隙锁
- B生成(5,10)的间隙锁，这里要注意间隙锁只有在写入的时候会阻塞，读不会
- B插入9的数据被阻塞，等待A释放锁
- A也插入9的数据被阻塞，等待B释放锁
- 两者锁的范围是一样的，又相互等待对方释放，造成死锁