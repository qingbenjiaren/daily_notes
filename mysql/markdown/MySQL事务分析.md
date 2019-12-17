# InnoDB的事务分析

![](MySQL事务分析.assets/InnoDB事务.png)

数据库事务具有ACID四大特性。ACID是以下4个词的缩写

- 原子性（Atomicity）:事务最小工作单元，要么全部成功，要么全部失败。
- 一致性（Consistency）：事务开始和结束后，数据库的完整性不会被破坏。
- 隔离性（Isolation）：不同事务之间互不影响，四种隔离级别为RU(读未提交)、RC（读已提交）、RR(可重复度)、SERIALIZABLE(串行化)。
- 持久性（durability）：事务提交后，对数据的修改是永久性的，即使系统故障也不会丢失。

**总结来说，事务的隔离性由多版本控制机制和锁实现，而原子性、一致性和持久性通过InnoDB的redo log、undo log 和Force log at commit机制来实现**

## 原子性，持久性和一致性

**原子性，持久性和一致性主要是通过redo log、undo log和Force log at commit机制来完成的。redo log用于在崩溃时恢复数据，undo log用于对事务的影响进行撤销，也可以用于多版本控制。而Force log at commit机制保证事务提交后redo log日志都已经持久化。**

### RedoLog

数据库日志和数据落盘机制，如下图所示：

![](MySQL事务分析.assets/mysql数据落盘流程.png)

redo log写入磁盘时，必须进行一次操作系统的fsync操作，防止redo log知识写了操作系统的磁盘缓存中。参数innodb_flush_log_at_trx_commit可以控制redo log日志刷新到磁盘的策略。

### UndoLog

**数据库崩溃重启后需要从redo log中把未落盘的脏页数据恢复出来，重新写入磁盘，保证用户的数据不丢失。当然，在崩溃恢复中还需要回滚没有提交的事务。由于回滚操作需要undo日志的支持，undo日志的完整性和可靠性需要redo日志来保证，所以崩溃恢复先做redo恢复数据，然后做undo回滚**

![](MySQL事务分析.assets/undo log.png)

在事务执行的过程中，除了记录redo log，还会记录一定量的undo log。undo log记录了数据在每隔操作前的状态，如果事务执行过程中需要回滚，就可以根据undo log进行回滚操作。

![](MySQL事务分析.assets/数据和回滚日志的逻辑存储结构.png)

数据和回滚日志的逻辑存储结构

| rowid | trxid  | Roll Pointer                 | id   | name |
| ----- | ------ | ---------------------------- | ---- | ---- |
| 行id  | 事务id | 回滚指针：只想上一个历史版本 | 数据 | 数据 |

undo log的存储不同于redo log，它存放在数据库内部的一个特殊的段（segment）中，这个段称为回滚段。回滚段位于共享表空间中。undo段中的以undo page为更小组织单位。**undo page和存储数据库数据和索引页类似。因为redo log是物理日志，记录的是数据库页的物理修改操作。所以undolog（也看成数据库数据）的写入也会产生redolog，也就是undo log的产生会伴随着redo log的产生，着是因为undo log也需要持久性的保护。**如上图所示，表空间中有回滚段和页节点段和非页节点段，而三者都有对应的页结构。

总结一下数据库事务的整个流程，如下图所示：

![](MySQL事务分析.assets/事务流程.png)

事务进行过程中，commit之后系统崩溃（未刷盘）用redolog恢复数据

commit之前崩溃，用redolog里的undolog进行回滚，commit后，redolog会区分出来已经提交的undolog，只要redolog里有未提交的undolog，会根据undolog进行回滚

undolog的完整性是靠redolog保证的





