# InnoDB与MyISAM的区别

create table XXXX() engine=InnoDB/Memory/MyISAM

MySQL的存储引擎是针对表进行指定的。

| 存储引擎   | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| **MyISAM** | 高速引擎，拥有较高的插入，查询速度，<font color='red'>**但不支持事务、不支持行锁**、</font>支持3中不同的存储格式。包括静态型、动态型和压缩型 |
| **InnoDB** | **5.5版本后MySQL的默认存储引擎，<font color='red'>支持事务和行级锁定，事务处理、回滚、崩溃修复能力和多版本并发控制的事务安全</font>**，比MyISAM处理速度稍慢，支持<font color='red'>**外键（FOREIGN KEY）**</font> |
| MEMORY     | **内存存储引擎，拥有极高的插入、更新和查询效率。但是会占用和数据量成正比**的内存空间。只在内存上保存数据，意味着数据可能会丢失。 |

xtraDB储存引擎是由Percona公司提供的存储引擎，该公司还出品了Percona Server这个产品，它是基于MySQL开源代码进行修改之后的产品。

阿里对于Percona Server服务器进行修改，衍生了自己的数据库（alisql）

- InnoDB和MyISAM存储引擎的区别


|              | InnoDB                                     | MyISAM                                               |
| ------------ | :----------------------------------------- | ---------------------------------------------------- |
| **存储文件** | .frm表定义文件<br />.ibd数据文件和索引文件 | .frm表定义文件<br />.myd数据文件<br />.myi索引为文件 |
| **锁**       | 表锁，行锁                                 | 表锁                                                 |
| **事务**     | 支持                                       | 不支持                                               |
| **CRUD**     | 读、写                                     | 读多                                                 |
| **count**    | 扫表                                       | 专门存储的地方（加where也扫表）                      |
| **索引结构** | B+Tree                                     | B+Tree                                               |
| **外键**     | 支持                                       | 不支持                                               |

存储引擎的选型：

**InnoDB：**支持事务处理，支持外键，支持崩溃修复能力和并发控制。如果需要**对事务的完整性要求比**
**较高**（比如银行），**要求实现并发控制**（比如售票），那选择InnoDB有很大的优势。如果需要**频繁的
更新、删除**操作的数据库，也可以选择InnoDB，因为支持事务的提交（commit）和回滚
（rollback）。

**MyISAM:**插入数据快，空间和内存使用比较低。如果表主要是**用于插入新记录和读出记录**，那么选择
MyISAM能实现处理高效率。如果应用的完整性、并发性要求比 较低，也可以使用。

**MEMORY:**所有的数据都在内存中，数据的处理速度快，但是安全性不高。如果需要**很快的读写速度**，
对数据的安全性要求较低，不需要持久保存，可以选择MEMOEY。它对表的大小有要求，不能建立太大的表。所以，这类数据库只使用在相对较小的数据库表。

同一个数据库也可以使用多种存储引擎的表，如果一个表要求比较高的事务处理，可以选择InnoDB。这个数据库中可以将查询要求比较高的表选择MyISAM存储。如果数据库需要一个用于查询的临时表，可以选择MEMORY存储引擎。



# 以下内容从CSDN摘抄并做修改

InnoDB和MyISAM是许多人在使用MySQL时最常用的两个表类型，这两个表类型各有优劣，视具体应用而定。基本的差别为：MyISAM类型不支持事务处理等高级处理，而InnoDB类型支持。MyISAM类型的表强调的是性能，其执行数度比InnoDB类型更快，但是不提供事务支持，而InnoDB提供事务支持以及外部键等高级数据库功能。

**InnoDB 中不保存表的具体行数，也就是说，执行select count(*) from table时，InnoDB要扫描一遍整个表来计算有多少行，但是MyISAM只要简单的读出保存好的行数即可。注意的是，当count(*)语句包含 where条件时，两种表的操作是一样的。**

**DELETE FROM table时，InnoDB不会重新建立表，而是一行一行的删除。**

两种类型最主要的差别就是Innodb 支持事务处理与外键和行级锁.而MyISAM不支持.所以MyISAM往往就容易被人认为只适合在小项目中使用。

我作为使用MySQL的用户角度出发，Innodb和MyISAM都是比较喜欢的，但是从我目前运维的数据库平台要达到需求：99.9%的稳定性，方便的扩展性和高可用性来说的话，MyISAM绝对是我的首选。



原因如下：

首先我目前平台上承载的大部分项目是读多写少的项目，而MyISAM的读性能是比Innodb强不少的。

**MyISAM的索引和数据是分开的，并且索引是有压缩的，内存使用率就对应提高了不少**。能加载更多索引，而Innodb是索引和数据是紧密捆绑的，没有使用压缩从而会造成**Innodb比MyISAM体积庞大不小。**

从平台角度来说，经常隔1，2个月就会发生应用开发人员不小心update一个表where写的范围不对，导致这个表没法正常用了，这个时候MyISAM的优越性就体现出来了，随便从当天拷贝的压缩包取出对应表的文件，随便放到一个数据库目录下，然后dump成sql再导回到主库，并把对应的binlog补上。如果是Innodb，恐怕不可能有这么快速度，别和我说让Innodb定期用导出xxx.sql机制备份，因为我平台上最小的一个数据库实例的数据量基本都是几十G大小。

从我接触的应用逻辑来说，select count(*) 和order by 是最频繁的，大概能占了整个sql总语句的60%以上的操作，而这种操作Innodb其实也是会锁表的，很多人以为Innodb是行级锁，那个只是where对它主键是有效，非主键的都会锁全表的。

还有就是经常有很多应用部门需要我给他们定期某些表的数据，MyISAM的话很方便，只要发给他们对应那表的frm.MYD,MYI的文件，让他们自己在对应版本的数据库启动就行，而Innodb就需要导出xxx.sql了，因为光给别人文件，受字典数据文件的影响，对方是无法使用的。

如果和MyISAM比insert写操作的话，Innodb还达不到MyISAM的写性能，如果是针对基于索引的update操作，虽然MyISAM可能会逊色Innodb,但是那么高并发的写，从库能否追的上也是一个问题，还不如通过多实例分库分表架构来解决。

当然Innodb也不是绝对不用，用事务的项目如模拟炒股项目，我就是用Innodb的，活跃用户20多万时候，也是很轻松应付了，因此我个人也是很喜欢Innodb的，只是如果从数据库平台应用出发，我还是会首选MyISAM。

另外，可能有人会说你MyISAM无法抗太多写操作，但是我可以通过架构来弥补，说个我现有用的数据库平台容量：主从数据总量在几百T以上，每天十多亿 pv的动态页面，还有几个大项目是通过数据接口方式调用未算进pv总数，(其中包括一个大项目因为初期memcached没部署,导致单台数据库每天处理 9千万的查询)。而我的整体数据库服务器平均负载都在0.5-1左右。
