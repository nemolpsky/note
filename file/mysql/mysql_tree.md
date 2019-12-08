## Mysql底层实现

Mysql底层是使用B+树来实现，上一篇文章介绍了二叉搜索树、B树和B+树的一些特点，其实可以发现使用B+树的原因就是可以保证树尽量的矮，也就是层级尽量低，因为每查找新的一层就是一次新的IO操作，这可比在内存中查找要慢的多了，所以要尽量减少IO操作，B树是节点可以存储多个键和数据，并且保证叶子节点都在同一层，所以会比二叉搜索树矮很多，而B+树则是在这方面继续优化，数据全都存在叶子节点，非叶子节点存键，这样树又会变的更低，而且叶子节点也是顺序相连的，对范围查找性能也会比较好。


1. 索引存储结构

   - 聚簇索引

     聚簇索引是在InnoDB引擎下为主键建立的索引，没有主键则会默认隐式的建立一个，假设一个表里面有id、name和age三个字段，存储结构大致上就是下面这张图的形式，B+树结构，数据全都存储在叶子节点中。

     ![聚簇索引](https://github.com/nemolpsky/Note/raw/master/file/mysql/image/mysql_tree1.png)


   - 非聚簇索引

     非聚簇索引也叫做二级索引，比如单独为name字段建立索引，也是会建立一个B+树结构的索引，但是键全都是name字段，叶子节点存储的数据反而是id，也就是主键，所以非聚簇索引查询是先根据字段值查找到主键id，然后再拿这个主键id聚簇索引上查询所有数据，这也就是回表查询，

     ![非聚簇索引](https://github.com/nemolpsky/Note/raw/master/file/mysql/image/mysql_tree2.png)

     下面这张图是建立一个联合索引，包含name和age字段，和上面的单个索引很像，只不过键是存储了多个字段。

     ![非聚簇索引](https://github.com/nemolpsky/Note/raw/master/file/mysql/image/mysql_tree3.png)

2. 数据库存储结构

   整个数据库存储其实是分为三层，InnoDB引擎层，文件系统层和磁盘扇区层，数据其实都存在磁盘中，再读取到文件系统层最后再读取到InnoDB引擎层。这里稍微介绍一下，在计算机中磁盘存储数据最小单元是扇区，一个扇区的大小是512字节，而文件系统(例如XFS/EXT4)他的最小单元是块，一个块的大小是4k，而对于我们的InnoDB存储引擎也有自己的最小储存单元-页(Page)，一个页的大小是16K(可以设置)，下面是一个只有一个字符的txt文档和数据库的数据文件(后缀为ibd的文件)，可以看到文档占用空间是4K，数据文件也全都是16K的倍数。

   ![存储结构](https://github.com/nemolpsky/Note/raw/master/file/mysql/image/mysql_tree4.png)
   ![存储结构](https://github.com/nemolpsky/Note/raw/master/file/mysql/image/mysql_tree5.png)
   ![存储结构](https://github.com/nemolpsky/Note/raw/master/file/mysql/image/mysql_tree6.png)

3. B+树存储数据量

   上面我们已经知道了InnoDB引擎每一页的大小，如果再知道B+树的高度和一行数据的大小就可以算出大致的数据存储量。

   因为B+树非叶子节点上存的全是键，也就是指向叶子节点的指针，所以数据总量应该就是非叶子节点中的指针数量 * 叶子节点中的记录行数。假设行数据是1K，而每个叶子节点都是一页，那就可以存储16条数据，而假设树的高度是2，也就是说需要计算根节点能存放多少指针，假设每个指针6字节，而每个主键ID假设是int类型，长度为4字节，加起来就是8字节，16KB=16384字节，16384/8=2043.5，所以2043.5 * 16 = 32696条数据，如果是高度为3的B+树就是32696 * 2043.5 = 66814276条数据，因为第二层全都是指针，当然这只是大致上的理论算法，实际上肯定还有很多因素要考虑。

   这也表示了B+树的特征，非常的矮，即使数量这么多，也只要几次IO查询，这也是为什么数据库底层会使用B+树结构。


4. 查询数据库的树高

   ```
   // 有一张500W数据的表
   mysql> select count(*) from table2;
   +----------+
   | count(*) |
   +----------+
   |  4913975 |
   +----------+
   1 row in set (9.95 sec)

   // 可以看到该表的B+树高度是3
   mysql>    SELECT
       ->    i.name AS key_name,t.name AS table_name,index_id,TYPE,i.space,i.page_no
       ->    FROM
       ->    information_schema.`INNODB_SYS_INDEXES` i
       ->    INNER JOIN
       ->    information_schema.`INNODB_SYS_TABLES` t
       ->    ON i.table_id = t.table_id AND i.space <> 0 WHERE t.name LIKE '%mybatis/table2%';
   +----------+----------------+----------+------+-------+---------+
   | key_name | table_name     | index_id | TYPE | space | page_no |
   +----------+----------------+----------+------+-------+---------+
   | PRIMARY  | mybatis/table2 |       90 |    3 |    77 |       3 |
   +----------+----------------+----------+------+-------+---------+
   1 row in set (0.00 sec)

   ```
