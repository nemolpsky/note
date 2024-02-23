## Mysql索引基础

MySQL有各种各样的引擎来满足不同的场景，同样的也提供各种类型的索引满足不同的数据场景。


1. 索引的存储分类

   索引是在MYSQL的存储引擎层中实现的，每种存储引擎都有各自的索引，下面是4种比较常见的索引。

   - B+Tree 索引：最常见的索引类型，大部分引擎都支持[B+树索引](https://github.com/nemolpsky/algorithm/blob/master/file/data/tree.md)。
   - HASH 索引：只有Memory引擎支持，使用场景简单，实现就是每行数据都计算出一个hashCode，适合等值查询，会很快。
   - R-Tree 索引(空间索引)：空间索引是MyISAM的一种特殊索引类型，主要用于地理空间数据类型。
   - Full-text (全文索引)：全文索引也是MyISAM的一种特殊索引类型，主要用于全文索引，InnoDB从MYSQL5.6版本提供对全文索引的支持。

   | 索引类型 | MyISAM引擎 | InnoDB引擎 | Memory引擎 |
   |---|---|---|---|
   | B-Tree索引 | 支持 | 支持 | 支持 |
   | HASH索引 | 不支持 | 不支持 | 支持 |
   | R-Tree索引 | 支持 | 不支持 | 不支持 |
   | Full-text索引 | 不支持 | 暂不支持 | 不支持 |


2. B-TREE索引类型

   底层结构是平衡多路查找树，这种结构可以更好的利用存储空间，加快查找速度，每个节点可以用多个子节点，但是维护成本比较高。

   - 普通索引
     
     最基本的索引类型，没有唯一性之类的限制。

     ```
     // 创建索引
     CREATE INDEX 索引名 ON 表名(列名1，列名2,...);
     // 修改表
     ALTER TABLE 表名ADD INDEX 索引名 (列名1，列名2,...);
     // 创建表时指定索引
     CREATE TABLE 表名 ( [...], INDEX 索引名 (列名1，列名 2,...) );     
     ```


   - UNIQUE索引
     
     唯一的，不允许重复的索引
     ```     
     // 创建索引
     CREATE UNIQUE INDEX 索引名 ON 表名(列的列表);
     // 修改表
     ALTER TABLE 表名ADD UNIQUE 索引名 (列的列表);
     // 创建表时指定索引
     CREATE TABLE 表名( [...], UNIQUE 索引名 (列的列表) );
     ```     

   - PRIMARY KEY索引

     是一种唯一性索引，但它必须指定为PRIMARY KEY，每个表只能有一个主键。（主键相当于聚合索引，是查找最快的索引）
     
     - 主键一般在创建表的时候指定：“CREATE TABLE 表名( [...], PRIMARY KEY (列的列表) ); ”
     - 修改表的方式加入主键：“ALTER TABLE 表名ADD PRIMARY KEY (列的列表); ”

4. 查看索引

   可以使用下列两条语句来查询一个表的索引

   ```
   SHOW INDEX FROM tableName;
   SHOW KEYS FROM tableName;
   ```
   
   每个字段代表的意思

   - Table：表的名称
   - Non_unique：如果索引不能包括重复词，则为0。如果可以，则为1
   - Key_name：索引的名称
   - Seq_in_index：索引中的列序列号，从1开始
   - Column_name：列名称
   - Collation：列以什么方式存储在索引中。在MySQL中，有值‘A’（升序）或NULL（无分类）。
   - Cardinality：索引中唯一值的数目的估计值。通过运行ANALYZE TABLE或myisamchk -a可以更新。基数根据被存储为整数的统计数据来计数，所以即使对于小型表，该值也没有必要是精确的。基数越大，当进行联合时，MySQL使用该索引的机会就越大。
   - Sub_part：如果列只是被部分地编入索引，则为被编入索引的字符的数目。如果整列被编入索引，则为NULL。
   - Packed：指示关键字如何被压缩。如果没有被压缩，则为NULL。
   - Null：如果列含有NULL，则含有YES。如果没有，则该列含有NO。
   - Index_type：用过的索引方法（BTREE, FULLTEXT, HASH, RTREE）。
   - Comment：更多评注。


5. 索引选择性

   - 较频繁的作为查询条件的字段应该创建索引
     
     尤其是尽量选择会出现在where子句或join子句中出现的字段

   - 唯一性太差的字段不适合单独创建索引，即使频繁作为查询条件

     例如，存放出生日期的列具有不同的值，很容易区分行，而用来记录性别的列，只有"M"和"F",则对此进行索引没有多大用处，因此不管搜索哪个值，都会得出大约一半的行，可以使用下列语句计算一下不重复的数据和所有数据之间的比例，比例越低就表示大部分数据是一样的，建立索引也没有必要。

     ```
     SELECT COUNT(ac_code_show)/COUNT(*) AS Selectivity FROM ac_code;
     ```

   - 更新非常频繁的字段不适合创建索引
   - 不会出现在 WHERE 子句中的字段不该创建索引
