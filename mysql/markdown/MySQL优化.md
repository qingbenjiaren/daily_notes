# 性能分析和性能优化

## 性能分析的思路

1. **首先需要使用【慢查询日志】功能，去获取所有查询时间比较长的SQL语句**
2. **其次【查看执行计划】查看有问题的SQL的执行计划 explain**
3. **最后可以使用【show profile[s]】查看有问题的SQL的性能使用情况**

### 慢查询日志

开启

```shell
slow_query_log=ON
long_query_time=3
slow_query_log_file=/var/lib/mysql/slow-log.log
```

## 服务器层面优化

### 将数据保存在内存中，保证从内存读取数据

buffer pool默认128M

扩大buffer pool理论上内存的3/4或4/5

怎样确定innodb_buffer_pool_size足够大

```mysql
mysql> show global status like 'innodb_buffer_pool_pages_%';
+----------------------------------+-------+
| Variable_name          		   | Value |
+----------------------------------+-------+
| Innodb_buffer_pool_pages_data    | 8190  | 
| Innodb_buffer_pool_pages_dirty   | 0     |
| Innodb_buffer_pool_pages_flushed | 12646 |
| Innodb_buffer_pool_pages_free    | 0     |  0 表示已经被用光
| Innodb_buffer_pool_pages_misc    | 1     |
| Innodb_buffer_pool_pages_total   | 8191  |
+----------------------------------+-------+
```

修改my.cnf

```shell
innodb_buffer_pool_size=750M
```

### 内存预热

```mysql
mysql> select count(id) from tuser;
+-----------+
| count(id) |
+-----------+
|  10000000 |
+-----------+
1 row in set (5.03 sec)
mysql> select count(id) from tuser;
+-----------+
| count(id) |
+-----------+
|  10000000 |
+-----------+
1 row in set (2.85 sec)
```

### 降低磁盘写入次数

1. redolog大 落盘次数少
2. innodb_log_file_size 设置成innodb_buffer_pool_size * 0.25
3. 通用查询日志、慢查询日志 可以不开,bin-log要开
4. 写redolog策略innodb_flush_log_at_trx_commit 0 1 2

### 提高磁盘读写

SSD

## SQL设计层面优化

设计中间表，一般针对于统计分析功能，或者实时性不高的需求

为减少关联查询，创建合理的冗余字段

对于字段太多的大表，考虑拆表（比如一个表有100个字段以上）

对于表中经常不被使用的字段或者存储数据比较多的字段，考虑拆表（比如商品表中会存储商品介绍，此时可以将商品介绍字段单独拆分到另一个表中，使用ID关联）

![](MySQL优化.assets/垂直拆分.png)

每张表建议都要有一个主键（**主键索引**），而且主键类型最好是**int类型**，建议自增主键（不考虑分布式系统的情况下，在分布式系统中可以用雪花算法）

## SQL语句优化

索引优化

where字段、组合索引（最左前缀）、索引下推（非选择行，不加锁）、索引覆盖（不回表） on两边 排序

不要用*

LIMIT优化（有limit的，查询语句最好带索引）

小结果集关联大结果集



其他优化

count(*)找普通索引，找到最小那颗索引树来遍历，包含空值

count(字段)不包含空值

count(1)忽略字段，包含空值

不用MySQL内置函数，因为内置函数不会建立查询缓存

