### MySQL死锁检测
MySQL默认开启了死锁检测，处理的方式也很简单，回滚小的事务，根据有影响的行数多少来判断哪个事务是比较小的。MySQL的锁机制其实设计比较完善了，性能相当不错了，普通的中小程序基本上都不会出现死锁，但是一旦出现，如果不是业务代码SQL冗余嵌套太多就是并发数比较大，解决起来也会比较麻烦。


#### 1. 死锁检测参数
下面两个参数分别是默认开启死锁检测和设置锁等待时间为50秒，超过50秒就回滚。

```
mysql> show variables like '%innodb_deadlock_detect%'; 
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_deadlock_detect | ON   |
+------------------------+-------+
1 row in set (0.00 sec)

mysql> SHOW GLOBAL VARIABLES LIKE 'innodb_lock_wait_timeout%'; 
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_lock_wait_timeout | 50  |
+--------------------------+-------+
1 row in set (0.00 sec)
```

为了便于测试，可以把死锁检测关掉，再把锁超时时间增大。

```
set GLOBAL innodb_deadlock_detect = "OFF";
set GLOBAL innodb_lock_wait_timeout = 5000;
```

---

#### 2. 死锁例子

建立下面这样结构的表，插入两条数据。
```
create table test_table
(
    id    int auto_increment
        primary key,
    score int null
);
INSERT INTO testDatabase.test_table (id, score) VALUES (1, 1);
INSERT INTO testDatabase.test_table (id, score) VALUES (10, 10);
```

下面这个例子其实是比较常见的一个例子，先查一下当前数据，再插入一条新数据。从感觉上应该不会有问题，锁的不是同一条数据，写入的也是不同的数据，甚至都不是相邻的数据，但是如果了解过MySQL的间隙锁就会明白为什么会锁住了。因为AB两个事务同时用间隙锁锁住了一个区间，间隙锁自己之间是不会排斥的，但是它会阻止其它事务在这个区间进行插入操作，这样一来死锁的条件就达成了，双方各持有一个锁，我要拿你的，你要拿我的。

|A事务| B事务 |
|  :-:    |:-:        |
|begin;||
|select * from test_table where id=2 for update;||
||begin;|
||select * from test_table where id=7 for update;|
|insert into test_table(id,score) value(3,3); 阻塞||
||insert into test_table(id,score) value(8,8); 阻塞|


如果没有关闭死锁检测的时候，必定会有一个事务出现这个提示，大意就是检测到有事务在获取锁的时候发生死锁，回滚事务。
```
mysql> insert into test_table(id,score) value(8,8);
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

如果关闭死锁检测，提示就会变，告诉你锁等待超时了
```
mysql> insert into test_table(id,score) value(8,8);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```


查看当前执行的语句列表，可以看到刚才那两条插入语句，等待了40多秒。
```
mysql> mysql> SHOW FULL PROCESSLIST;
+----+-----------------+------------------+--------------+---------+------+------------------------+---------------------------------------------+
| Id | User            | Host             | db           | Command | Time | State                  | Info                                        |
+----+-----------------+------------------+--------------+---------+------+------------------------+---------------------------------------------+
|  5 | event_scheduler | localhost        | NULL         | Daemon  | 2613 | Waiting on empty queue | NULL                                        |
|  8 | root            | 172.17.0.1:63458 | testDatabase | Sleep   | 2510 |                        | NULL                                        |
|  9 | root            | localhost        | testDatabase | Query   |   44 | update                 | insert into test_table(id,score) value(3,3) |
| 10 | root            | 172.17.0.1:63462 | testDatabase | Sleep   |   82 |                        | NULL                                        |
| 15 | root            | localhost        | testDatabase | Query   |   38 | update                 | insert into test_table(id,score) value(8,8) |
| 18 | root            | 172.17.0.1:63490 | testDatabase | Sleep   |  449 |                        | NULL                                        |
| 19 | root            | localhost        | testDatabase | Query   |    0 | init                   | SHOW FULL PROCESSLIST                       |
+----+-----------------+------------------+--------------+---------+------+------------------------+---------------------------------------------+
7 rows in set (0.00 sec)
```


还可以查看详细的死锁日志，可以看到里面讲了两个事务是哪张表的，持有什么锁，等待什么锁，最后回滚了其中一个事务。
```
mysql> show engine innodb status\G;
LATEST DETECTED DEADLOCK
------------------------
2021-02-03 09:25:04 0x7f0342cf7700
*** (1) TRANSACTION:
TRANSACTION 6199, ACTIVE 80 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 9, OS thread handle 139652409308928, query id 725 localhost root update
insert into test_table(id,score) value(3,3)

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 20 page no 4 n bits 72 index PRIMARY of table `testDatabase`.`test_table` trx id 6199 lock_mode X locks gap before rec
Record lock, heap no 4 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000182a; asc      *;;
 2: len 7; hex 8100000110012a; asc       *;;
 3: len 4; hex 8000000a; asc     ;;


*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 20 page no 4 n bits 72 index PRIMARY of table `testDatabase`.`test_table` trx id 6199 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000182a; asc      *;;
 2: len 7; hex 8100000110012a; asc       *;;
 3: len 4; hex 8000000a; asc     ;;


*** (2) TRANSACTION:
TRANSACTION 6200, ACTIVE 23 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 15, OS thread handle 139652407539456, query id 726 localhost root update
insert into test_table(id,score) value(8,8)

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 20 page no 4 n bits 72 index PRIMARY of table `testDatabase`.`test_table` trx id 6200 lock_mode X locks gap before rec
Record lock, heap no 4 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000182a; asc      *;;
 2: len 7; hex 8100000110012a; asc       *;;
 3: len 4; hex 8000000a; asc     ;;


*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 20 page no 4 n bits 72 index PRIMARY of table `testDatabase`.`test_table` trx id 6200 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000182a; asc      *;;
 2: len 7; hex 8100000110012a; asc       *;;
 3: len 4; hex 8000000a; asc     ;;

*** WE ROLL BACK TRANSACTION (2)
------------
TRANSACTIONS
------------
Trx id counter 6212
Purge done for trx's n:o < 6208 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421127423176272, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421127423173704, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421127423171992, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421127423171136, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421127423170280, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 6211, ACTIVE 339 sec
2 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 15, OS thread handle 139652407539456, query id 1090 localhost root
---TRANSACTION 6210, ACTIVE 345 sec
2 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 9, OS thread handle 139652409308928, query id 1089 localhost root
```

---

#### 3. 解决方法

解决死锁的方法很简单，打破这种竞态关系。[MySQL中的锁种类](https://github.com/nemolpsky/note/blob/master/file/mysql/MySQL%E4%B8%AD%E7%9A%84%E9%94%81%E7%A7%8D%E7%B1%BB.md)讲了间隙锁的产生以及锁住的范围，当两个间隙锁重叠的时候就会产生这种死锁，可以在执行锁定前执行不带锁的查询，确保数据不为空，如果查询的字段是唯一索引，而且也不为空就不会产生间隙锁。

还有就是避免长事务，比如有时候一个for循环处理很多条数据的业务逻辑，而这些数据本身没有任何关联性，所以最好独立一个事务提交，事务越长，锁住的范围就越大。

此外有一点要注意的是，死锁检测本身也是会耗费性能的，所以有些场景可以试着关闭死锁检测，调短锁超时时间，这对解决死锁本身是没有帮助的，但是配合合理的重试机制可以解决因为死锁长期僵持带来的资源耗费，比如连接数，以及对后续事务的影响，但是这种方法考虑的因素比较多，需要慎重。