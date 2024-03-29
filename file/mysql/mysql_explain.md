## explain
1. id
   
   表示查询的标识符，每个查询都有一个唯一的标识符。

2. select_type
 
   - SIMPLE，简单的SELECT查询，不包含UNION和子查询等。
   - PRIMARY，使用了union查询，并且是最外层的查询，也就是第一个查询。
   - UNION，使用了union查询，第一个查询之后的查询。
   - DEPENDENT UNION，和UNION类似，但是查询结果会被最外层也就是最后的查询所依赖。
   - UNION RESULT，使用union查询后合并的结果集。
   - SUBQUERY，子查询中的第一个SELECT。
   - DEPENDENT SUBQUERY，类似DEPENDENT UNION查询结果会被最外层的查询所依赖。
   - DERIVED，查询的结果作为一个临时表来让外层查询的数据来源。

3. table

   一般是查询的表名，还有下列几个特殊情况
   - select_type是UNION RESULT的情况下，会显示结果集是来源于哪几个查询id的，例如<1,2>
   - 在出现select_type是DERIVED的情况下，使用该派生表的查询会显示引用的id
   - 显示引用的物化表的id

4. partitions

   如果设置了分区，会显示数据是查询的哪个分区，否则会显示NULL

5. type

   用来表示查询的时候表是以什么方式联接查询的
   - system，在直接查询一些系统值，而且只会有一条数据的时候会是这个类型
   - const，在设置了主键或者唯一索引的情况下，一些字段一定是唯一的，这个时候设置特定的搜索条件只会查询一条数据，优化器会把这列数据视为常量，读取速度非常快
   - eq_ref，当两个表联接查询的时候如果联接的字段在两个表中都是主键或唯一索引
     ```
     // id是主键
     EXPLAIN
     SELECT * FROM test1 t1 INNER JOIN test2 t2 ON t1.id = t2.id
     ```

   - ref，和eq_ref类似，区别在于使用了索引，但不是主键或唯一索引，所以会匹配到多条数据
     ```
     // name不是唯一索引
     EXPLAIN
     SELECT * FROM test1 t1 INNER JOIN test2 t2
     ON t1.name = t2.name
     ```
   - fulltext，全文索引
   - ref_or_null，和ref类似，但是会额外搜索包含空值的字段，但是按照下方官网给的例子显示的不是这个类型
     ```
     SELECT * FROM ref_table
     WHERE key_column=expr OR key_column IS NULL;
     ```
   - index_merge，如果筛选条件是多个字段就会合并索引查询，按下列的语法有的可以有的不行
     ```
     EXPLAIN
     SELECT * FROM test2 WHERE NAME = 'a' OR id =1
     ```
   - unique_subquery，有时候在in查询中会代替eq_ref，只是一个索引查找函数，替换了子查询以提高效率。
   - index_subquery，类型类似于unique_subquery。但它适合子查询中筛选条件是非唯一索引
   - range，查询给定范围的数据，使用=、<>、>、>=、<、<=、IS NULL、<=>、BETWEEN或者IN操作符
     ```
     // id是主键
     EXPLAIN
     SELECT * FROM test2 WHERE  id > 1
     ```
    - index，查询整个索引树，但是比全表扫描要快
    - ALL，全表扫描，不适用索引，速度最慢

6. possible_keys

   列出所有可能会使用的索引，但是不一定会真的使用，NULL表示没有索引

7. key
   
   实际使用的索引，NULL表示没有使用

8. key_len

   使用的索引长度

9. ref
   
   显示是哪个表的哪个字段被用于和索引进行比较

10. rows
    
    Mysql预估本次查询需要检索的数据行数，只是一个估计值

11. filtered
    
    表示过滤掉的百分比，100就表示没有过滤，用这个比例乘以rows就可以得到真实的检索行数


12. Extra

    显示一些额外信息，表示Mysql执行的相关策略  
    - Using index，使用了索引，如果还使用了覆盖索引，即只使用索引中的数据，不需要回表查询。
    - Using where，使用了WHERE过滤条件。
    - Using temporary，查询时需要创建一个临时表，这种情况下可能会增加查询的开销。
    - Using filesort，需要对结果集进行排序操作，如果不使用索引对数据进行排序操作，效率可能比较低。
    - Using join buffer，执行Join操作时使用了Join Buffer，这种情况下会把Join的结果存放在Buffer中，然后再返回给客户端。
    - Impossible where，表示WHERE条件不满足，无法查询到数据。
    - Select tables optimized away，表示优化了查询，即不需要访问任何表就可以得到结果。例如，查询列的值都是常量。
    - Distinct，使用了Distinct去重操作。
    - Fulltext，使用了全文索引来加速查询。
    - Range checked for each record，在执行索引扫描时，需要对每一行数据都进行判断，这种情况下可能会影响查询效率。
    - Using index condition，在Using index的基础上使用了索引里数据作为条件直接过滤。
