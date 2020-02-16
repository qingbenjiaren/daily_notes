# MongoDB介绍

MongoDB 是一个基于【分布式文件存储】的数据库，它属于NoSQL数据库。由 C++ 语言编写。旨在为 WEB 应用提供【**可扩展**】的【**高性能**】数据存储解决方案。

MongoDB是一个<font color='red'>**介于关系数据库和非关系数据库之间的产品**</font>，是非关系数据库当中功能最丰富，最像关系数据库的。它支持的数据结构非常松散，是类似json的bson格式，因此可以存储比较复杂的数据类型。Mongo最大的特点是它支持的**查询语言非常强大**，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库**<font color='red'>单表查询</font>**的绝大部分功能，而且还支持对数据建立索引。

> NoSQL分类：键值型（key-value）、文档型（document）
>
> MongoDB就是文档型NoSQL数据库，它文档中的数据是以类似JSON的BSON格式进行存储的。我们拿JSON去理解，JSON中的数据，都是key-value，key一般都是String类型的，而value就多种多样了。只有value的类型，后续有专门的讲解。记住value中可以再存储一个文档。

## MongoDB概念解析

在mongodb中基本的概念是**文档、集合、数据库**。

| SQL术语/概念 | MongoDB术语/概念 | 解释/说明                        |
| ------------ | ---------------- | -------------------------------- |
| database     | database         | 数据库                           |
| table        | collection       | 数据库表/集合                    |
| row          | document         | 数据记录行/文档                  |
| column       | field            | 数据字段/域                      |
| index        | index            | 索引                             |
| table joins  |                  | 表连接，MongoDB不支持            |
| primary key  | primary key      | 主键，MongoDB自动将_id设置为主键 |



### 数据库

一个mongodb中可以建立多个数据库。

MongoDB的默认数据库为"db"，该数据库存储在data目录中（**安装时，可以默认，可以指定，但是必须该目录是存在的**）。

MongoDB的单个实例可以容纳多个独立的数据库，每一个都有自己的集合和权限，不同的数据库也放置在不同的文件中。

"**show dbs**" 命令可以显示所有数据的列表。

```shell
$ ./mongo
MongoDB shell version: 3.0.6
connecting to: test
> show dbs
local 0.078GB
test  0.078GB
>
```

执行 "**db**" 命令可以显示当前数据库对象或集合。

```shell
$ ./mongo
MongoDB shell version: 3.0.6
connecting to: test
> db
test
>
```

运行use命令，可以连接一个指定的数据库

```shell
> use local
switched to db local
```

### 文档

文档是一组键值(key-value)对(即 BSON)。**MongoDB 的文档不需要设置相同的字段，并且相同的字段不需要相同的数据类型，这与关系型数据库有很大的区别，也是 MongoDB 非常突出的特点。**

需要注意的是：

1. 文档中的键/值对是有序的。
2. 文档中的值不仅可以是在双引号里面的字符串，还可以是其他几种数据类型（甚至可以是整个嵌入的文档)。
3. MongoDB区分类型和大小写。
4. MongoDB的文档不能有重复的键。
5. 文档的键是字符串。除了少数例外情况，键可以使用任意UTF-8字符。

文档命名规范：

- 键不能含有\0 (空字符)。这个字符用来表示键的结尾。**（C++）**
- .和$有特别的意义，只有在特定环境下才能使用。
- 以下划线"_"开头的键是保留的(不是严格要求的)。

### 集合

集合就是 MongoDB 文档组，类似于 RDBMS （关系数据库管理系统：Relational Database Management System)中的表格。

**集合存在于数据库中，集合没有固定的结构，这意味着你在对集合可以插入不同格式和类型的数据，但通常情况下我们插入集合的数据都会有一定的关联性。**

比如，我们可以将以下不同数据结构的文档插入到集合中：

```sql
{"site":"www.baidu.com"}
{"site":"www.google.com","name":"Google"}
{"site":"www.kaikeba.com","name":"开课吧","num":5}
```

当第一个文档插入时，集合就会被创建。

> 一个collection（集合）中的所有field（域）是collection（集合）中所有document（文档）中包含的field（域）的并集。

合法集合名：

- 集合名不能是空字符串""。
- 集合名不能含有\0字符（空字符)，这个字符表示集合名的结尾。（C++）
- 集合名不能以"system."开头，这是为系统集合保留的前缀。
- 用户创建的集合名字不能含有保留字符。有些驱动程序的确支持在集合名里面包含，这是因为某些系统生成的集合中包含该字符。除非你要访问这种系统创建的集合，否则千万不要在名字里出现$。

### capped collections

Capped collections 就是固定大小的collection。

它有很高的性能以及队列过期的特性(过期按照插入的顺序). 有点和 "RRD" 概念类似。

Capped collections 是高性能自动的维护对象的插入顺序。它非常适合类似记录日志的功能和标准的 collection 不同，你必须要显式的创建一个capped collection，指定一个 collection 的大小，单位是字节。collection 的数据存储空间值提前分配的。

**<font color='red'>Capped collections 可以按照文档的插入顺序保存到集合中，而且这些文档在磁盘上存放位置也是按照插入顺序来保存的，所以当我们更新Capped collections 中文档的时候，更新后的文档不可以超过之前文档的大小，这样话就可以确保所有文档在磁盘上的位置一直保持不变。</font>**

由于 Capped collection 是按照文档的插入顺序而不是使用索引确定插入位置，这样的话可以提高增添数据的效率MongoDB 的操作日志文件 oplog.rs 就是利用 Capped Collection 来实现的。

要注意的是指定的存储大小包含了数据库的头信息。

```sql
db.createCollection("mycoll", {capped:true, size:100000})
```

在 capped collection 中，你能添加新的对象。

- 能进行更新，然而，对象不会增加存储空间。如果增加，更新就会失败 。
- 使用 Capped Collection 不能删除一个文档，可以使用 drop() 方法删除 collection 所有的行。
- 删除之后，你必须显式的重新创建这个 collection。
- 在32bit机器中，capped collection 最大存储为 1e9( 1X10的9次方)个字节。

### ObjectId

ObjectId 类似唯一主键，可以很快的去生成和排序，包含 12 bytes，含义是：

- 前 4 个字节表示创建 unix 时间戳,格林尼治时间 UTC 时间，比北京时间晚了 8 个小时
- 接下来的 3 个字节是机器标识码
- 紧接的两个字节由进程 id 组成 PID
- 最后三个字节是随机数

MongoDB 中存储的文档必须有一个 _id 键。这个键的值可以是任何类型的，默认是个 ObjectId 对象

由于 ObjectId 中保存了创建的时间戳，所以你不需要为你的文档保存时间戳字段，你可以通过 getTimestamp 函数来获取文档的创建时间:

```sql
> var newObject = ObjectId()
> newObject.getTimestamp()
ISODate("2017-11-25T07:21:10Z")
```

ObjectId 转为字符串

```sql
5a1919e63df83ce79df8b38f
```



## MongoDB底层原理

**MongoDB的部署方案由单机部署、主从部署、副本集（主备）部署、分片部署、副本集与分片混合部署。**

### 副本集集群

**对于副本集集群，又有主和从两种角色，写数据和读数据也是不同，写数据的过程是只写到主结点中，由主 结点以异步的方式同步到从结点中：**

**而读数据则只要从任一结点中读取，具体到哪个结点读取是可以指定的：**



![](mongodb.assets/副本集集群.png)

### 副本集与分片混合部署

Mongodb的集群部署方案有三类角色：**实际数据存储节点，配置文件存储节点和路由接入节点。**

- **实际数据存储节点**的作用就是存储数据，
- **路由接入节点**的作用是在分片的情况下起到负载均衡的作用。
- **存储配置存储节点**的作用其实存储的是片键与chunk 以及chunk 与server 的映射关系，用上面的数据表 示的配置结点存储的数据模型如下表：

**map1**

| key range | chunk  |
| --------- | ------ |
| [0,10}    | chunk1 |
| [10,20}   | chunk2 |
| [20,30}   | chunk3 |
| [30,40}   | chunk4 |
| [40,50}   | chunk5 |

**map2**

| chunk  | shard  |
| ------ | ------ |
| chunk1 | shard1 |
| chunk2 | shard2 |
| chunk3 | shard3 |
| chunk4 | shard4 |
| chunk5 | shard5 |

MongoDB的客户端直接与**路由节点**相连，从配置节点上查询数据，根据查询结果到实际的存储节点上查询和存储数据。

**副本集与分片混合部署方式如图：**

![](mongodb.assets/副本集与分片混合部署.png)

> 相同的副本集中的节点存储的数据是一样的，副本集中的节点是分为主节点、从节点、仲裁节点（非必须）三种角色。【这种设计方案的目的，主要是为了高性能、高可用、数据备份。】
>
> 不同的副本集中的节点存储的数据是不一样，【这种设计方案，主要是为了解决高扩展问题，理论上是可以无限扩展的。】
>
> 每一个副本集可以看成一个shard（分片），多个副本集共同组成一个逻辑上的大数据节点。通过对shard上面进行逻辑分块chunk（块），每个块都有自己存储的数据范围，所以说客户端请求存储数据的时候，会去读取config server中的映射信息，找到对应的chunk（块）存储数据。

# MongoDB的应用场景和不适用场景

