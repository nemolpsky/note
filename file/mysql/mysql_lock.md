## Mysql锁问题
在开发过程中有时候会遇到数据库死锁的问题，这个时候很容易让系统不可用，所以需要了解数据库什么时候会加锁，以及如何解决死锁问题。


1. 行锁和表锁

   其实Mysql中的锁类型如果细分的话，可以分为很多种类，但是这里只大致介绍一下更新操作中的表锁和行锁，顾名思义，行锁就是只锁住一行，表锁则是锁住整个表。

   - 表锁
   
     在执行update语句的时候，如果where条件里面有用到索引就会指锁住索引相关的数据，比如下面这个weather表，有2个索引，分别是主键索引和temp字段的二级索引，里面有四条数据。

     ```
     mysql> desc weather;
     +--------+-------------+------+-----+---------+----------------+
     | Field  | Type        | Null | Key | Default | Extra          |
     +--------+-------------+------+-----+---------+----------------+
     | id     | int(11)     | NO   | PRI | NULL    | auto_increment |
     | date   | date        | YES  |     | NULL    |                |
     | temp   | int(11)     | YES  | MUL | NULL    |                |
     | gender | varchar(11) | YES  |     | NULL    |                |
     +--------+-------------+------+-----+---------+----------------+
     4 rows in set (0.01 sec)

     mysql> show index from weather \G;
     *************************** 1. row ***************************
             Table: weather
        Non_unique: 0
          Key_name: PRIMARY
      Seq_in_index: 1
       Column_name: id
         Collation: A
       Cardinality: 4
          Sub_part: NULL
            Packed: NULL
              Null:
        Index_type: BTREE
           Comment:
     Index_comment:
     *************************** 2. row ***************************
             Table: weather
        Non_unique: 1
          Key_name: index
      Seq_in_index: 1
       Column_name: temp
         Collation: A
       Cardinality: 4
          Sub_part: NULL
            Packed: NULL
              Null: YES
        Index_type: BTREE
           Comment:
     Index_comment:
     2 rows in set (0.00 sec)

     mysql> select * from weather;
     +----+------------+------+--------+
     | id | date       | temp | gender |
     +----+------------+------+--------+
     |  1 | 2015-01-01 |   12 | F      |
     |  2 | 2015-01-02 |   25 | F      |
     |  3 | 2015-01-03 |   20 | M      |
     |  4 | 2015-01-04 |   30 | M      |
     +----+------------+------+--------+
     4 rows in set (0.00 sec)
     ```

     现在分别开两个命令行来执行命令，第一条是手动开启事务然后更新第一条数据，但是执行完后不提交，第二条是直接更新第二条数据，可以看到第二条的事务执行超时直接回滚了，因为date字段是没有索引，直接把整个表都锁住了。

     ```
     mysql> BEGIN;
     Query OK, 0 rows affected (0.28 sec)

     mysql> UPDATE weather SET gender = 'fs' WHERE DATE = '2015-01-01';
     Query OK, 0 rows affected (0.00 sec)
     Rows matched: 1  Changed: 0  Warnings: 0

     mysql> UPDATE weather SET gender = 'fs' WHERE DATE = '2015-01-02';
     ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
     ```

   - 行锁

     如果改成使用带索引的temp字段就会发现，不会锁表了，只会缩住索引所在的那一行。

     ```
     mysql> BEGIN;
     Query OK, 0 rows affected (0.00 sec)

     mysql> UPDATE weather SET gender = 'fs' WHERE temp = 12;
     Query OK, 0 rows affected (0.00 sec)
     Rows matched: 1  Changed: 0  Warnings: 0

     mysql> UPDATE weather SET gender = 'fs' WHERE temp = '25';
     Query OK, 1 row affected (0.07 sec)
     Rows matched: 1  Changed: 1  Warnings: 0
     ```

     如果给gender字段添加索引，在当做条件更新，会发现会把所有符合索引条件的数据都锁住，因为id为1和2的两条数据都符合gender为F的条件，索引第一条SQL直接把id为1和2的两条数据都锁住了，而更新id为3的数据则没有问题，所以很多情况下更新语句都是会拿id进行更新，这样只会锁住主键id符合的那条数据。

     ```
     mysql> BEGIN;
     Query OK, 0 rows affected (0.00 sec)

     mysql> UPDATE weather SET temp = '100' WHERE gender = 'F';
     Query OK, 2 rows affected (0.00 sec)
     Rows matched: 2  Changed: 2  Warnings: 0

     mysql> UPDATE weather SET temp = '100' WHERE id = 2;
     Ctrl-C -- sending "KILL QUERY 6" to server ...
     Ctrl-C -- query aborted.
     ERROR 1317 (70100): Query execution was interrupted
     mysql> UPDATE weather SET temp = '100' WHERE id = 3;
     Query OK, 1 row affected (0.12 sec)
     Rows matched: 1  Changed: 1  Warnings: 0
     ```

2. 锁问题

   像上面这种阻塞事务在真实环境下也是会遇到的，解决方法可以是使用SHOW FULL PROCESSLIST来kill掉阻塞的语句，或者也可以查询general_log日志来查看是哪条语句阻塞住了。
   ```
   // 查看日志是否开启
   SHOW VARIABLES LIKE '%general%';

   // 开启日志
   SET GLOBAL general_log=ON;

   // 查询innodb的information_schema中的信息，查看数据库中的锁等待和锁信息
   // 可以根据blocking_trx_id在general_log日志中查询阻塞的SQL
   mysql> SELECT
       -> r.trx_mysql_thread_id waiting_thread,
       -> r.trx_id waiting_trx_id,r.trx_query waiting_query,
       -> b.trx_id blocking_trx_id, b.trx_query blocking_query,
       -> b.trx_mysql_thread_id blocking_thread,b.trx_started,
       -> b.trx_wait_started
       -> FROM information_schema.innodb_lock_waits w
       -> INNER JOIN information_schema.innodb_trx b
       -> ON b.trx_id =w.blocking_trx_id
       -> INNER JOIN
       -> information_schema.innodb_trx r
       -> ON r.trx_id=w.requesting_trx_id \G
   *************************** 1. row ***************************
     waiting_thread: 6
     waiting_trx_id: 244441
      waiting_query: UPDATE weather SET temp = '100' WHERE id = 2
    blocking_trx_id: 244440
     blocking_query: NULL
    blocking_thread: 10
        trx_started: 2019-12-11 21:50:02
   trx_wait_started: NULL
   1 row in set (0.00 sec)

   // 直接查询INFORMATION_SCHEMA.INNODB_LOCKS信息也可以获取到blocking_trx_id
   mysql>  SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS \G;
   *************************** 1. row ***************************
       lock_id: 244443:101:3:3
   lock_trx_id: 244443
     lock_mode: X
     lock_type: RECORD
    lock_table: `test`.`weather`
    lock_index: PRIMARY
    lock_space: 101
     lock_page: 3
      lock_rec: 3
     lock_data: 2
   *************************** 2. row ***************************
       lock_id: 244440:101:3:3
   lock_trx_id: 244440
     lock_mode: X
     lock_type: RECORD
    lock_table: `test`.`weather`
    lock_index: PRIMARY
    lock_space: 101
     lock_page: 3
      lock_rec: 3
     lock_data: 2
   2 rows in set (0.00 sec)
   ```

   上面就是一个锁问题，前面的事务阻塞了后面事务，再看下面这个例子，首先执行一个长时间的查询操作，再执行DDL操作，会发现DDL操作被阻塞住了，然后使用SHOW FULL PROCESSLIST语句查看，可以看到第一个事务执行了很久，阻塞住了后面的，id是10，使用kill 10命令就可以干掉这个

   ```
   mysql> select id,sleep(120) from weather;

   mysql> alter table weather add column time datetime;

   mysql> SHOW FULL PROCESSLIST \G;
   *************************** 1. row ***************************
        Id: 3
      User: root
      Host: localhost:5264
        db: test
   Command: Sleep
      Time: 116
     State:
      Info: NULL
   *************************** 2. row ***************************
        Id: 6
      User: root
      Host: localhost:6161
        db: test
   Command: Query
      Time: 24
     State: Waiting for table metadata lock
      Info: alter table weather add column time datetime
   *************************** 3. row ***************************
        Id: 10
      User: root
      Host: localhost:7184
        db: test
   Command: Query
      Time: 37
     State: User sleep
      Info: select id,sleep(120) from weather
   *************************** 4. row ***************************
        Id: 11
      User: root
      Host: localhost:7546
        db: NULL
   Command: Sleep
      Time: 884
     State:
      Info: NULL
   *************************** 5. row ***************************
        Id: 15
      User: root
      Host: localhost:7688
        db: test
   Command: Query
      Time: 0
     State: init
      Info: SHOW FULL PROCESSLIST
   5 rows in set (0.00 sec)
   ```

## 总结

锁问题一旦出现就会对系统造成很大问题，要尽量避免。
- 设置合适的索引，避免更新等操作造成锁表。
- 尽量避免一个事务内执行很多操作，可以拆分成小事务执行。
- 不要在高峰时期对线上的表进行DDL操作。