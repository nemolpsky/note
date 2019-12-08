## Mysql优化

1. limit优化

   table1表中大约有500W的数据，假设需要[4900000,4900010]范围的数据，一般都是会想到利用索引，但是会发现索引虽然有用到却还是很慢，因为即使是使用到了索引但是数据库也需要从头遍历找到从哪开始是4900000条件数据，所以会很慢。

   ```
   mysql> EXPLAIN
       -> SELECT * FROM table1 WHERE table1.`t1_id`>0 LIMIT 4900000,10;
   +----+-------------+--------+-------+---------------+---------+---------+------+---------+-------------+
   | id | select_type | table  | type  | possible_keys | key     | key_len | ref  | rows    | Extra       |
   +----+-------------+--------+-------+---------------+---------+---------+------+---------+-------------+
   |  1 | SIMPLE      | table1 | range | PRIMARY       | PRIMARY | 4       | NULL | 2443662 | Using where |
   +----+-------------+--------+-------+---------------+---------+---------+------+---------+-------------+
   1 row in set (0.00 sec)

   mysql> SELECT * FROM table1 WHERE table1.`t1_id`>0 LIMIT 4900000,10;
   +---------+------------+------+---------+
   | t1_id   | name       | age  | t2_id   |
   +---------+------------+------+---------+
   | 4900921 | name182585 |   16 | 4900921 |
   | 4900922 | name182586 |   21 | 4900922 |
   | 4900923 | name182587 |   57 | 4900923 |
   | 4900924 | name182588 |   22 | 4900924 |
   | 4900925 | name182589 |   39 | 4900925 |
   | 4900926 | name182590 |   29 | 4900926 |
   | 4900927 | name182591 |   30 | 4900927 |
   | 4900928 | name182592 |   60 | 4900928 |
   | 4900929 | name182593 |   10 | 4900929 |
   | 4900930 | name182594 |   73 | 4900930 |
   +---------+------------+------+---------+
   10 rows in set (11.21 sec)
   ```

   基于这种状况其实可以每次查询都拿上次分页返回的最大id值或者是创建时间来做条件，直接把小于这部分的筛选掉再获取，因为每次都把前面的筛选掉了，所以数据量即使增大，每次分页查询的时间也不会有太大的变化。

   ```
   mysql> EXPLAIN
       -> SELECT * FROM table1 WHERE table1.`t1_id`>=4900921 LIMIT 10;
   +----+-------------+--------+-------+---------------+---------+---------+------+-------+-------------+
   | id | select_type | table  | type  | possible_keys | key     | key_len | ref  | rows  | Extra       |
   +----+-------------+--------+-------+---------------+---------+---------+------+-------+-------------+
   |  1 | SIMPLE      | table1 | range | PRIMARY       | PRIMARY | 4       | NULL | 29736 | Using where |
   +----+-------------+--------+-------+---------------+---------+---------+------+-------+-------------+
   1 row in set (0.00 sec)

   mysql> SELECT * FROM table1 WHERE table1.`t1_id`>=4900921 LIMIT 0,10;
   +---------+------------+------+---------+
   | t1_id   | name       | age  | t2_id   |
   +---------+------------+------+---------+
   | 4900921 | name182585 |   16 | 4900921 |
   | 4900922 | name182586 |   21 | 4900922 |
   | 4900923 | name182587 |   57 | 4900923 |
   | 4900924 | name182588 |   22 | 4900924 |
   | 4900925 | name182589 |   39 | 4900925 |
   | 4900926 | name182590 |   29 | 4900926 |
   | 4900927 | name182591 |   30 | 4900927 |
   | 4900928 | name182592 |   60 | 4900928 |
   | 4900929 | name182593 |   10 | 4900929 |
   | 4900930 | name182594 |   73 | 4900930 |
   +---------+------------+------+---------+
   10 rows in set (0.00 sec)
   ```

2. 类型匹配

   user表中有个varchar(32)类型的字段password，这个字段有建索引，查询的时候这个字段条件如果没有输入单引号数据库也会将数据库中的字符串自动转换为数字进行对比，但是这样就会造成索引失效。

   ```
   mysql> desc user;
   +-----------------+-------------+------+-----+---------+-------+
   | Field           | Type        | Null | Key | Default | Extra |
   +-----------------+-------------+------+-----+---------+-------+
   | user_id         | bigint(20)  | YES  |     | NULL    |       |
   | user_name       | varchar(12) | YES  |     | NULL    |       |
   | mobile          | varchar(13) | YES  |     | NULL    |       |
   | birthday        | datetime    | YES  |     | NULL    |       |
   | reg_time        | datetime    | YES  |     | NULL    |       |
   | password        | varchar(32) | YES  | MUL | NULL    |       |
   | reg_ip          | varchar(24) | YES  |     | NULL    |       |
   | last_login_time | datetime    | YES  |     | NULL    |       |
   | last_login_ip   | varchar(24) | YES  |     | NULL    |       |
   | login_counts    | int(8)      | YES  |     | NULL    |       |
   +-----------------+-------------+------+-----+---------+-------+
   10 rows in set (0.01 sec)

   mysql> EXPLAIN
       -> SELECT * FROM USER WHERE PASSWORD = 123456;
   +----+-------------+-------+------+---------------+------+---------+------+------+-------------+
   | id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
   +----+-------------+-------+------+---------------+------+---------+------+------+-------------+
   |  1 | SIMPLE      | USER  | ALL  | password      | NULL | NULL    | NULL |    6 | Using where |
   +----+-------------+-------+------+---------------+------+---------+------+------+-------------+
   1 row in set (0.00 sec)

   mysql> EXPLAIN
       -> SELECT * FROM USER WHERE PASSWORD = '123456';
   +----+-------------+-------+------+---------------+----------+---------+-------+------+-----------------------+
   | id | select_type | table | type | possible_keys | key      | key_len | ref   | rows | Extra                 |
   +----+-------------+-------+------+---------------+----------+---------+-------+------+-----------------------+
   |  1 | SIMPLE      | USER  | ref  | password      | password | 131     | const |    1 | Using index condition |
   +----+-------------+-------+------+---------------+----------+---------+-------+------+-----------------------+
   1 row in set (0.00 sec)
   ```

3. join更新、删除数据

   mysql5.6中引入了Materialization(物化)这一关键特性用于子查询(比如在IN/NOT IN子查询以及FROM子查询)优化，但是只对查询起作用，更新和删除必须改成join的格式，否则会很慢。

   table1表中有500W条数据，使用了一个in子查询，使用explain查看执行计划是可以看到使用了物化进行优化。

   ```
   mysql> EXPLAIN
       -> SELECT * FROM table1 t1
       -> WHERE  t1.t1_id IN (SELECT t1_id
       ->                 FROM   (SELECT t1_id
       ->                         FROM   table1 t1
       ->                         WHERE  age = 55
       ->                         LIMIT  1) t
       ->                         );
   +----+--------------+-------------+--------+---------------+---------+---------+-------------------+---------+----------
   ---+
   | id | select_type  | table       | type   | possible_keys | key     | key_len | ref               | rows    | Extra
      |
   +----+--------------+-------------+--------+---------------+---------+---------+-------------------+---------+----------
   ---+
   |  1 | PRIMARY      | <subquery2> | ALL    | NULL          | NULL    | NULL    | NULL              |    NULL | NULL
      |
   |  1 | PRIMARY      | t1          | eq_ref | PRIMARY       | PRIMARY | 4       | <subquery2>.t1_id |       1 | NULL
      |
   |  2 | MATERIALIZED | <derived3>  | ALL    | NULL          | NULL    | NULL    | NULL              |       1 | NULL
      |
   |  3 | DERIVED      | t1          | ALL    | NULL          | NULL    | NULL    | NULL              | 4906952 | Using where |
   +----+--------------+-------------+--------+---------------+---------+---------+-------------------+---------+----------
   ---+
   4 rows in set (0.00 sec)

   mysql> SELECT * FROM table1 t1
       -> WHERE  t1.t1_id IN (SELECT t1_id
       ->                 FROM   (SELECT t1_id
       ->                         FROM   table1 t1
       ->                         WHERE  age = 55
       ->                         LIMIT  1) t
       ->                         );
   +-------+--------+------+-------+
   | t1_id | name   | age  | t2_id |
   +-------+--------+------+-------+
   |    39 | name39 |   55 |    39 |
   +-------+--------+------+-------+
   1 row in set (0.00 sec)
   ```

   下面再试着把这条语句改成更新，可以看到速度非常慢，要3.47秒左右，也没有使用物化，其中DEPENDENT SUBQUERY是嵌套子查询，相当慢，应该尽量避免，EXISTS子句也是嵌套查询所以应该尽量使用join代替。

   ```
   mysql> EXPLAIN
       -> UPDATE table1 t1
       -> SET age = 555
       -> WHERE  t1.t1_id IN (SELECT t1_id
       ->                 FROM   (SELECT t1_id
       ->                         FROM   table1 t1
       ->                         WHERE  age = 55
       ->                         LIMIT  1) t
       ->                         );
   +----+--------------------+------------+--------+---------------+---------+---------+------+---------+-------------+
   | id | select_type        | table      | type   | possible_keys | key     | key_len | ref  | rows    | Extra       |
   +----+--------------------+------------+--------+---------------+---------+---------+------+---------+-------------+
   |  1 | PRIMARY            | t1         | index  | NULL          | PRIMARY | 4       | NULL | 4906952 | Using where |
   |  2 | DEPENDENT SUBQUERY | <derived3> | system | NULL          | NULL    | NULL    | NULL |       1 | NULL        |
   |  3 | DERIVED            | t1         | ALL    | NULL          | NULL    | NULL    | NULL | 4906952 | Using where |
   +----+--------------------+------------+--------+---------------+---------+---------+------+---------+-------------+
   3 rows in set (0.00 sec)

   mysql> UPDATE table1 t1
       -> SET age = 555
       -> WHERE  t1.t1_id IN (SELECT t1_id
       ->                 FROM   (SELECT t1_id
       ->                         FROM   table1 t1
       ->                         WHERE  age = 55
       ->                         LIMIT  1) t
       ->                         );
   Query OK, 1 row affected (3.47 sec)
   Rows matched: 1  Changed: 1  Warnings: 0

   ```

   而修改成join格式之后几乎不需要什么时间。

   ```
   mysql> EXPLAIN
       -> UPDATE table1 t1
       -> JOIN  (SELECT t1_id
       ->           FROM  table1 t1
       ->           WHERE  age = 55
       ->           LIMIT  1) t
       ->       ON t1.`t1_id` = t.t1_id
       -> SET age = 55;
   +----+-------------+------------+--------+---------------+---------+---------+-------+---------+-------------+
   | id | select_type | table      | type   | possible_keys | key     | key_len | ref   | rows    | Extra       |
   +----+-------------+------------+--------+---------------+---------+---------+-------+---------+-------------+
   |  1 | PRIMARY     | <derived2> | system | NULL          | NULL    | NULL    | NULL  |       1 | NULL        |
   |  1 | PRIMARY     | t1         | const  | PRIMARY       | PRIMARY | 4       | const |       1 | NULL        |
   |  2 | DERIVED     | t1         | ALL    | NULL          | NULL    | NULL    | NULL  | 4906952 | Using where |
   +----+-------------+------------+--------+---------------+---------+---------+-------+---------+-------------+
   3 rows in set (0.00 sec)

   mysql> UPDATE table1 t1
       -> JOIN  (SELECT t1_id
       ->           FROM  table1 t1
       ->           WHERE  age = 55
       ->           LIMIT  1) t
       ->       ON t1.`t1_id` = t.t1_id
       -> SET age = 55;
   Query OK, 0 rows affected (0.00 sec)
   Rows matched: 1  Changed: 0  Warnings: 0
   ```

4. 条件下推

   很多时候回写出这种在子查询中分组筛选后，再根据子查询的结果继续筛选的语句，其实可以直接在子查询中完成条件筛选，速度会快很多，可以看到一个是直接利用索引查询，另一个是子查询全表后还继续在子查询中查询。

   ```
   mysql> EXPLAIN
       -> SELECT * FROM (
       ->  SELECT age,COUNT(*) FROM table1 t1
       ->  GROUP BY age
       -> ) r WHERE r.age = 40;
   +----+-------------+------------+-------+---------------+-------------+---------+-------+---------+-------------+
   | id | select_type | table      | type  | possible_keys | key         | key_len | ref   | rows    | Extra       |
   +----+-------------+------------+-------+---------------+-------------+---------+-------+---------+-------------+
   |  1 | PRIMARY     | <derived2> | ref   | <auto_key0>   | <auto_key0> | 5       | const |      10 | NULL        |
   |  2 | DERIVED     | t1         | index | age           | age         | 5       | NULL  | 4906952 | Using index |
   +----+-------------+------------+-------+---------------+-------------+---------+-------+---------+-------------+
   2 rows in set (0.00 sec)

   mysql> SELECT * FROM (
       ->  SELECT age,COUNT(*) FROM table1 t1
       ->  GROUP BY age
       -> ) r WHERE r.age = 40;
   +------+----------+
   | age  | COUNT(*) |
   +------+----------+
   |   40 |    49100 |
   +------+----------+
   1 row in set (0.89 sec)

   mysql> EXPLAIN
       -> SELECT age,COUNT(*) FROM table1 t1
       -> WHERE t1.age = 40
       -> GROUP BY age;
   +----+-------------+-------+------+---------------+------+---------+-------+-------+-------------+
   | id | select_type | table | type | possible_keys | key  | key_len | ref   | rows  | Extra       |
   +----+-------------+-------+------+---------------+------+---------+-------+-------+-------------+
   |  1 | SIMPLE      | t1    | ref  | age           | age  | 5       | const | 96966 | Using index |
   +----+-------------+-------+------+---------------+------+---------+-------+-------+-------------+
   1 row in set (0.00 sec)

   mysql> SELECT age,COUNT(*) FROM table1 t1
       -> WHERE t1.age = 40
       -> GROUP BY age;
   +------+----------+
   | age  | COUNT(*) |
   +------+----------+
   |   40 |    49100 |
   +------+----------+
   1 row in set (0.02 sec)
   ```

5. 相关名词介绍

   - 物化

     在SQL执行过程中，第一次需要子查询结果时执行子查询并将子查询的结果保存为临时表，后续对子查询结果集的访问将直接通过临时表获得。 与此同时，优化器还具有延迟物化子查询的能力，先通过其它条件判断子查询是否真的需要执行。物化子查询优化SQL执行的关键点在于对子查询只需要执行一次。其执行计划中的查询类型是MATERIALIZED与，之相对的执行方式嵌套子查询，其执行计划中的查询类型为"DEPENDENT SUBQUERY"。

   - 条件下推

     尽量将筛选条件放到子查询中，可以大大加快查询效率。