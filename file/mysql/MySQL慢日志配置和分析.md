### Mysql慢日志配置和分析

Mysql中可以开启慢日志记录，配合分析工具可以清晰的看到是哪条SQL执行慢，这样可以借助explain命令来分析索引的执行和语句的优化。


---


#### 1. 慢日志配置
可以通过下面两条SQL查看是否开启配置，设置时间和日志的存储位置。
```
mysql> show variables like 'slow_query%';
+---------------------+----------------------+
| Variable_name       | Value                |
+---------------------+----------------------+
| slow_query_log      | ON                   |
| slow_query_log_file | /data/query_slow.log |
+---------------------+----------------------+
mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set (0.00 sec)
```

默认是关闭的，日志目录也是一个随机字符串作为名字，可以通过这两条命令来更改设置。

```
set global slow_query_log='ON';
set global slow_query_log_file='/data/query_slow.log';
```

执行一条睡眠14秒的SQL，就可以看到满日志里面会有记录。
```
select sleep(14);
```
```
$ cat query_slow.log
/usr/sbin/mysqld, Version: 8.0.22 (MySQL Community Server - GPL). started with:
Tcp port: 3306  Unix socket: /var/run/mysqld/mysqld.sock
Time                 Id Command    Argument
# Time: 2021-01-29T09:40:50.195573Z
# User@Host: root[root] @  [172.17.0.1]  Id:     9
# Query_time: 14.000270  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 1
use testDatabase;
SET timestamp=1611913236;
/* ApplicationName=DataGrip 2019.3.3 */ select sleep(14);
```

---


#### 2. 使用pt-query-digest分析日志
慢日志的原生格式是Mysql命令行风格的，看起来不直观也很乱，日志数据多了查找也不方便，所以可以使用```pt-query-digest```来分析慢日志，它也是一个命令行工具。使用下列命令安装并且验证。
```
apt install percona-toolkit
pt-query-digest --help
```

使用pt-query-digest分析一下日志，就可以看到下面这样的统计数据。
```
half@Half-PC:~/opt/20210129183323$ pt-query-digest --report query_slow.log

# 90ms user time, 200ms system time, 27.12M rss, 44.71M vsz
# Current date: Fri Jan 29 18:50:11 2021
# Hostname: Half-PC
# Files: query_slow.log
# Overall: 1 total, 1 unique, 0 QPS, 0x concurrency ______________________
# Time range: all events occurred at 2021-01-29T09:40:50
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time            14s     14s     14s     14s     14s       0     14s
# Lock time              0       0       0       0       0       0       0
# Rows sent              1       1       1       1       1       0       1
# Rows examine           1       1       1       1       1       0       1
# Query size            56      56      56      56      56       0      56

# Profile
# Rank Query ID                           Response time  Calls R/Call  V/M
# ==== ================================== ============== ===== ======= ===
#    1 0x59A74D08D407B5EDF9A57DD5A41825CA 14.0003 100.0%     1 14.0003  0.00 SELECT

# Query 1: 0 QPS, 0x concurrency, ID 0x59A74D08D407B5EDF9A57DD5A41825CA at byte 0
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.00
# Time range: all events occurred at 2021-01-29T09:40:50
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count        100       1
# Exec time    100     14s     14s     14s     14s     14s       0     14s
# Lock time      0       0       0       0       0       0       0       0
# Rows sent    100       1       1       1       1       1       0       1
# Rows examine 100       1       1       1       1       1       0       1
# Query size   100      56      56      56      56      56       0      56
# String:
# Databases    testDatabase
# Hosts        172.17.0.1
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us
#   1ms
#  10ms
# 100ms
#    1s
#  10s+  ################################################################
# EXPLAIN /*!50100 PARTITIONS*/
/* ApplicationName=DataGrip 2019.3.3 */ select sleep(14)\G
```

还有很丰富的参数，比如limit限制查询数量。
```
half@Half-PC:~/opt/20210129183323$ pt-query-digest --report --limit 10 query_slow.log
```