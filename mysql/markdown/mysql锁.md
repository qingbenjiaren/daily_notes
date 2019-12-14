# MySQL锁

## MySQL锁介绍

![](mysql锁.assets/mysql锁.png)

## MySQL表级锁

### 表级锁介绍

**由MySQL SQL Layer层实现**

- MySQL的表级锁由两种：

> 一种是表锁。
>
> 一种是元数据锁（meta data lock，MDL）。

- MySQL实现的表级锁定的争用状态变量

> show status like 'table%';

### 表锁介绍

- 表锁有两种形式：

> 表共享读锁（Table Read Lock）
>
> 表独占写锁（Table Write Lock）

- 手动增加表锁

```mysql
lock table 表名称 read(write),表名称2 read(write),其他;
```

查看表锁情况

```mysql
show open tables;
```

删除表锁

```mysql
unlock tables;
```

### 表锁演示

#### 环境准备

```sql
--新建表
CREATE TABLE mylock (
id int(11) NOT NULL AUTO_INCREMENT,
NAME varchar(20) DEFAULT NULL,
PRIMARY KEY (id)
);
INSERT INTO mylock (id,NAME) VALUES (1, 'a');
INSERT INTO mylock (id,NAME) VALUES (2, 'b');
INSERT INTO mylock (id,NAME) VALUES (3, 'c');
INSERT INTO mylock (id,NAME) VALUES (4, 'd');
```

#### 读锁演示

表读锁

```sql
1、session1: lock table mylock read; -- 给mylock表加读锁
2、session1: select * from mylock; -- 可以查询
3、session1：select * from tdep; --不能访问非锁定表
4、session2：select * from mylock; -- 可以查询 没有锁
5、session2：update mylock set name='x' where id=2; -- 修改阻塞,自动加行写锁
6、session1：unlock tables; -- 释放表锁
7、session2：Rows matched: 1 Changed: 1 Warnings: 0 -- 修改执行完成
8、session1：select * from tdep; --可以访问
```

表写锁

```sql
1、session1: lock table mylock write; -- 给mylock表加写锁
2、session1: select * from mylock; -- 可以查询
3、session1：select * from tdep; --不能访问非锁定表
4、session1：update mylock set name='y' where id=2; --可以执行
5、session2：select * from mylock; -- 查询阻塞
6、session1：unlock tables; -- 释放表锁
7、session2：4 rows in set (22.57 sec) -- 查询执行完成
8、session1：select * from tdep; --可以访问
```

## 元数据锁

### 元数据锁介绍

MDL(metaDataLock)元数据：表结构

在MySQL5.5版本中引入了MDL，当对一个表做**增删改查操作**时，加MDL读锁；当对**表结构做变更操作**时，加MDL写锁

```sql
1、session1: begin;--开启事务
  select * from mylock;--加MDL读锁
2、session2: alter table mylock add f int; -- 修改阻塞
3、session1：commit; --提交事务 或者 rollback 释放读锁
4、session2：Query OK, 0 rows affected (38.67 sec)  --修改完成
```

## 行级锁

### 行级锁介绍

InnoDB存储引擎实现

InnoDB的行级锁，按照锁定范围来说，分三种：
记录锁（Record Locks）:锁定索引中一条记录。主键指定where id = 3

间隙锁(Gap Locks):锁定记录前，记录中、记录后的行RR隔离剂（可重复读）--MySQL默认隔离级别

Next-Key锁：记录锁+间隙锁

### 行级锁分类

按照功能来说，分为两种：

共享读锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。

```sql
select * from table_name where ... lock in share mode;  -- 共享读锁  手动添加
select * from table_name where ...; -- 无锁
```

排他写锁（X）：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。

1、自动加 DML

对于**UPDATE/DELETE/INSERT**语句，InnoDB会自动给涉及数据集加排他写锁（X）；

2、手动加

```sql
select * from table_name where id = ..... for update;
```

InnoDB也实现了表级锁，也就是易向锁，意向锁是mysql内部使用的，不需要用户干预

意向共享读（IS）：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前，必须先取得该表的IS锁。

意向排他写（IX）：事务打算给数据行加排它锁，事务在给一个数据行加排它锁前必须先取得该表的IX锁。

|                  | 共享锁（S） | 排他锁（X） | 意向共享锁（IS） | 意向排他锁（IX） |
| ---------------- | ----------- | ----------- | ---------------- | ---------------- |
| 共享锁（S）      | 兼容        | 冲突        | 兼容             | 冲突             |
| 排他锁（X）      | 冲突        | 冲突        | 冲突             | 冲突             |
| 意向共享锁（IS） | 兼容        | 冲突        | 兼容             | 兼容             |
| 意向排他锁（IX） | 冲突        | 冲突        | 兼容             | 兼容             |

### 两阶段锁（2PL）

![](mysql锁.assets/两阶段锁.png)



锁操作分为两个阶段：加锁阶段和解锁阶段；

加锁阶段与解锁阶段不相交

加锁阶段：只加锁，不放锁；

解锁阶段：只放锁，不加锁；

### 行锁演示



```mysql
#查看行锁状态
show status like 'innodb_row_lock'
```

