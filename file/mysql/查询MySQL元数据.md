## MySQL元数据

MySQL5.0之后会提供一个内置的```information_schema```数据库来记录各种元数据信息，比如数据库中各个表的属性，每个表的字段属性和索引信息等等。要注意的是这个数据库是一个虚拟数据库，也就是说并不会有真正的物理数据存储在磁盘文件中，它里面所有的表都是视图。

下面可以看到```information_schema```中有很多视图，这里只介绍```TABLES```、```COLUMNS```和```STATISTICS```三个。
```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| erp_base           |
| erp_wms            |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
6 rows in set (0.01 sec)

mysql> use information_schema
Database changed
mysql> show tables;
+---------------------------------------+
| Tables_in_information_schema          |
+---------------------------------------+
| CHARACTER_SETS                        |
| COLLATIONS                            |
| COLLATION_CHARACTER_SET_APPLICABILITY |
| COLUMNS                               |
| COLUMN_PRIVILEGES                     |
| ENGINES                               |
| EVENTS                                |
| FILES                                 |
| GLOBAL_STATUS                         |
| GLOBAL_VARIABLES                      |
| KEY_COLUMN_USAGE                      |
| OPTIMIZER_TRACE                       |
| PARAMETERS                            |
| PARTITIONS                            |
| PLUGINS                               |
| PROCESSLIST                           |
| PROFILING                             |
| REFERENTIAL_CONSTRAINTS               |
| ROUTINES                              |
| SCHEMATA                              |
| SCHEMA_PRIVILEGES                     |
| SESSION_STATUS                        |
| SESSION_VARIABLES                     |
| STATISTICS                            |
| TABLES                                |
| TABLESPACES                           |
| TABLE_CONSTRAINTS                     |
| TABLE_PRIVILEGES                      |
| TRIGGERS                              |
| USER_PRIVILEGES                       |
| VIEWS                                 |
| INNODB_LOCKS                          |
| INNODB_TRX                            |
| INNODB_SYS_DATAFILES                  |
| INNODB_FT_CONFIG                      |
| INNODB_SYS_VIRTUAL                    |
| INNODB_CMP                            |
| INNODB_FT_BEING_DELETED               |
| INNODB_CMP_RESET                      |
| INNODB_CMP_PER_INDEX                  |
| INNODB_CMPMEM_RESET                   |
| INNODB_FT_DELETED                     |
| INNODB_BUFFER_PAGE_LRU                |
| INNODB_LOCK_WAITS                     |
| INNODB_TEMP_TABLE_INFO                |
| INNODB_SYS_INDEXES                    |
| INNODB_SYS_TABLES                     |
| INNODB_SYS_FIELDS                     |
| INNODB_CMP_PER_INDEX_RESET            |
| INNODB_BUFFER_PAGE                    |
| INNODB_FT_DEFAULT_STOPWORD            |
| INNODB_FT_INDEX_TABLE                 |
| INNODB_FT_INDEX_CACHE                 |
| INNODB_SYS_TABLESPACES                |
| INNODB_METRICS                        |
| INNODB_SYS_FOREIGN_COLS               |
| INNODB_CMPMEM                         |
| INNODB_BUFFER_POOL_STATS              |
| INNODB_SYS_COLUMNS                    |
| INNODB_SYS_FOREIGN                    |
| INNODB_SYS_TABLESTATS                 |
+---------------------------------------+
61 rows in set (0.00 sec)

```

- TABLES
  
  这个视图是用来存储表信息，包含了所属数据库、引擎类型、数据量、索引量、数据长度和表明等等信息。

  ```
  mysql> desc TABLES;
  +-----------------+---------------------+------+-----+---------+-------+
  | Field           | Type                | Null | Key | Default | Extra |
  +-----------------+---------------------+------+-----+---------+-------+
  | TABLE_CATALOG   | varchar(512)        | NO   |     |         |       |
  | TABLE_SCHEMA    | varchar(64)         | NO   |     |         |       |
  | TABLE_NAME      | varchar(64)         | NO   |     |         |       |
  | TABLE_TYPE      | varchar(64)         | NO   |     |         |       |
  | ENGINE          | varchar(64)         | YES  |     | NULL    |       |
  | VERSION         | bigint(21) unsigned | YES  |     | NULL    |       |
  | ROW_FORMAT      | varchar(10)         | YES  |     | NULL    |       |
  | TABLE_ROWS      | bigint(21) unsigned | YES  |     | NULL    |       |
  | AVG_ROW_LENGTH  | bigint(21) unsigned | YES  |     | NULL    |       |
  | DATA_LENGTH     | bigint(21) unsigned | YES  |     | NULL    |       |
  | MAX_DATA_LENGTH | bigint(21) unsigned | YES  |     | NULL    |       |
  | INDEX_LENGTH    | bigint(21) unsigned | YES  |     | NULL    |       |
  | DATA_FREE       | bigint(21) unsigned | YES  |     | NULL    |       |
  | AUTO_INCREMENT  | bigint(21) unsigned | YES  |     | NULL    |       |
  | CREATE_TIME     | datetime            | YES  |     | NULL    |       |
  | UPDATE_TIME     | datetime            | YES  |     | NULL    |       |
  | CHECK_TIME      | datetime            | YES  |     | NULL    |       |
  | TABLE_COLLATION | varchar(32)         | YES  |     | NULL    |       |
  | CHECKSUM        | bigint(21) unsigned | YES  |     | NULL    |       |
  | CREATE_OPTIONS  | varchar(255)        | YES  |     | NULL    |       |
  | TABLE_COMMENT   | varchar(2048)       | NO   |     |         |       |
  +-----------------+---------------------+------+-----+---------+-------+
  21 rows in set (0.00 sec)

  ```

  比如下面就是查询```erp_wms```库中表名为```wms_ac_code```的表，可以看到```TABLE_ROWS```为12，也就是有12行数据，字符编码```TABLE_COMMENT```为```utf8mb4_general_ci```。

  ```
  mysql> select * from TABLES where TABLE_SCHEMA = 'erp_wms' and TABLE_NAME = 'wms_ac_code' \G;
  *************************** 1. row ***************************
    TABLE_CATALOG: def
     TABLE_SCHEMA: erp_wms
       TABLE_NAME: wms_ac_code
       TABLE_TYPE: BASE TABLE
           ENGINE: InnoDB
          VERSION: 10
       ROW_FORMAT: Dynamic
       TABLE_ROWS: 12
   AVG_ROW_LENGTH: 1365
      DATA_LENGTH: 16384
  MAX_DATA_LENGTH: 0
     INDEX_LENGTH: 0
        DATA_FREE: 0
   AUTO_INCREMENT: 1296
      CREATE_TIME: 2020-08-29 16:33:11
      UPDATE_TIME: NULL
       CHECK_TIME: NULL
  TABLE_COLLATION: utf8mb4_general_ci
         CHECKSUM: NULL
   CREATE_OPTIONS:
    TABLE_COMMENT: 商品防伪码
  1 row in set (0.00 sec)
  ```

- COLUMNS

  这个存储了表字段的信息，包含了表明、字段名、字段类型等信息。

  ```
  mysql> desc COLUMNS;
  +--------------------------+---------------------+------+-----+---------+-------+
  | Field                    | Type                | Null | Key | Default | Extra |
  +--------------------------+---------------------+------+-----+---------+-------+
  | TABLE_CATALOG            | varchar(512)        | NO   |     |         |       |
  | TABLE_SCHEMA             | varchar(64)         | NO   |     |         |       |
  | TABLE_NAME               | varchar(64)         | NO   |     |         |       |
  | COLUMN_NAME              | varchar(64)         | NO   |     |         |       |
  | ORDINAL_POSITION         | bigint(21) unsigned | NO   |     | 0       |       |
  | COLUMN_DEFAULT           | longtext            | YES  |     | NULL    |       |
  | IS_NULLABLE              | varchar(3)          | NO   |     |         |       |
  | DATA_TYPE                | varchar(64)         | NO   |     |         |       |
  | CHARACTER_MAXIMUM_LENGTH | bigint(21) unsigned | YES  |     | NULL    |       |
  | CHARACTER_OCTET_LENGTH   | bigint(21) unsigned | YES  |     | NULL    |       |
  | NUMERIC_PRECISION        | bigint(21) unsigned | YES  |     | NULL    |       |
  | NUMERIC_SCALE            | bigint(21) unsigned | YES  |     | NULL    |       |
  | DATETIME_PRECISION       | bigint(21) unsigned | YES  |     | NULL    |       |
  | CHARACTER_SET_NAME       | varchar(32)         | YES  |     | NULL    |       |
  | COLLATION_NAME           | varchar(32)         | YES  |     | NULL    |       |
  | COLUMN_TYPE              | longtext            | NO   |     | NULL    |       |
  | COLUMN_KEY               | varchar(3)          | NO   |     |         |       |
  | EXTRA                    | varchar(30)         | NO   |     |         |       |
  | PRIVILEGES               | varchar(80)         | NO   |     |         |       |
  | COLUMN_COMMENT           | varchar(1024)       | NO   |     |         |       |
  | GENERATION_EXPRESSION    | longtext            | NO   |     | NULL    |       |
  +--------------------------+---------------------+------+-----+---------+-------+
  21 rows in set (0.00 sec)
  ```

  这条SQL查询了```erp_wms```库中```wms_ac_code_change```表中的```user_ip```字段，可以看到```IS_NULLABLE```表示字段是否为空，```COLLATION_NAME```是字符编码和```COLUMN_TYPE```是字段类型等信息。

  ```
  mysql> select * from COLUMNS where TABLE_SCHEMA = 'erp_wms' and TABLE_NAME = 'wms_ac_code_change' and COLUMN_NAME = 'user_ip' \G;
  *************************** 1. row ***************************
             TABLE_CATALOG: def
              TABLE_SCHEMA: erp_wms
                TABLE_NAME: wms_ac_code_change
               COLUMN_NAME: user_ip
          ORDINAL_POSITION: 15
            COLUMN_DEFAULT: NULL
               IS_NULLABLE: YES
                 DATA_TYPE: varchar
  CHARACTER_MAXIMUM_LENGTH: 24
    CHARACTER_OCTET_LENGTH: 96
         NUMERIC_PRECISION: NULL
             NUMERIC_SCALE: NULL
        DATETIME_PRECISION: NULL
        CHARACTER_SET_NAME: utf8mb4
            COLLATION_NAME: utf8mb4_general_ci
               COLUMN_TYPE: varchar(24)
                COLUMN_KEY:
                     EXTRA:
                PRIVILEGES: select,insert,update,references
            COLUMN_COMMENT: 操作人IP
     GENERATION_EXPRESSION:
  1 row in set (0.00 sec)
  ```

- STATISTICS
  
  这个表存储了索引相关信息，包含了索引类型、字符集类型、表名和字段名等信息。

  ```
  mysql> desc STATISTICS;
  +---------------+---------------+------+-----+---------+-------+
  | Field         | Type          | Null | Key | Default | Extra |
  +---------------+---------------+------+-----+---------+-------+
  | TABLE_CATALOG | varchar(512)  | NO   |     |         |       |
  | TABLE_SCHEMA  | varchar(64)   | NO   |     |         |       |
  | TABLE_NAME    | varchar(64)   | NO   |     |         |       |
  | NON_UNIQUE    | bigint(1)     | NO   |     | 0       |       |
  | INDEX_SCHEMA  | varchar(64)   | NO   |     |         |       |
  | INDEX_NAME    | varchar(64)   | NO   |     |         |       |
  | SEQ_IN_INDEX  | bigint(2)     | NO   |     | 0       |       |
  | COLUMN_NAME   | varchar(64)   | NO   |     |         |       |
  | COLLATION     | varchar(1)    | YES  |     | NULL    |       |
  | CARDINALITY   | bigint(21)    | YES  |     | NULL    |       |
  | SUB_PART      | bigint(3)     | YES  |     | NULL    |       |
  | PACKED        | varchar(10)   | YES  |     | NULL    |       |
  | NULLABLE      | varchar(3)    | NO   |     |         |       |
  | INDEX_TYPE    | varchar(16)   | NO   |     |         |       |
  | COMMENT       | varchar(16)   | YES  |     | NULL    |       |
  | INDEX_COMMENT | varchar(1024) | NO   |     |         |       |
  +---------------+---------------+------+-----+---------+-------+
  16 rows in set (0.00 sec)
  ```

  这条SQL则是查出```erp_wms```库中```wms_ac_code```表中所有的索引，可以看到有四条索引，第一条是主键索引，第二条则是唯一索引，后面两条都是非唯一索引，看到每个索引的类型，还可以看到索引名字以及索引对应的字段名。

  ```
  mysql> select * From STATISTICS where TABLE_SCHEMA = 'erp_wms'and TABLE_NAME = 'wms_ac_code' \G;
  *************************** 1. row ***************************
  TABLE_CATALOG: def
   TABLE_SCHEMA: erp_wms
     TABLE_NAME: wms_ac_code
     NON_UNIQUE: 0
   INDEX_SCHEMA: erp_wms
     INDEX_NAME: PRIMARY
   SEQ_IN_INDEX: 1
    COLUMN_NAME: ac_code_id
      COLLATION: A
    CARDINALITY: 12
       SUB_PART: NULL
         PACKED: NULL
       NULLABLE:
     INDEX_TYPE: BTREE
        COMMENT:
  INDEX_COMMENT:
  *************************** 2. row ***************************
  TABLE_CATALOG: def
   TABLE_SCHEMA: erp_wms
     TABLE_NAME: wms_ac_code
     NON_UNIQUE: 0
   INDEX_SCHEMA: erp_wms
     INDEX_NAME: code_uindex
   SEQ_IN_INDEX: 1
    COLUMN_NAME: ac_code_show
      COLLATION: A
    CARDINALITY: 12
       SUB_PART: NULL
         PACKED: NULL
       NULLABLE: YES
     INDEX_TYPE: BTREE
        COMMENT:
  INDEX_COMMENT:
  *************************** 3. row ***************************
  TABLE_CATALOG: def
   TABLE_SCHEMA: erp_wms
     TABLE_NAME: wms_ac_code
     NON_UNIQUE: 1
   INDEX_SCHEMA: erp_wms
     INDEX_NAME: id_index
   SEQ_IN_INDEX: 1
    COLUMN_NAME: matter_id
      COLLATION: A
    CARDINALITY: 3
       SUB_PART: NULL
         PACKED: NULL
       NULLABLE: YES
     INDEX_TYPE: BTREE
        COMMENT:
  INDEX_COMMENT:
  *************************** 4. row ***************************
  TABLE_CATALOG: def
   TABLE_SCHEMA: erp_wms
     TABLE_NAME: wms_ac_code
     NON_UNIQUE: 1
   INDEX_SCHEMA: erp_wms
     INDEX_NAME: id_index
   SEQ_IN_INDEX: 2
    COLUMN_NAME: warehouse_manage_id
      COLLATION: A
    CARDINALITY: 3
       SUB_PART: NULL
         PACKED: NULL
       NULLABLE: YES
     INDEX_TYPE: BTREE
        COMMENT:
  INDEX_COMMENT:
  4 rows in set (0.00 sec)
  ```

---

## 总结

MySQL的```information_schema```库保存很多数据相关的元数据，库里面都是视图，使用方式也是SQL语句查询的方法，通过这个库就可以很方便的查询分析整个库的表，字段名或者是索引的详细信息。