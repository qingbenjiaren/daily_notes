# mongo shell

## 使用mongo shell

> use kkb;
>
> db.myCollection.insert( { x: 1 } );

如果集合中包含空格，连字符"-"，或者以数字开始，可以使用代替语法来指代集合

>  db["3test"].find()
>
> db.getCollection("3test").find()

格式化打印

```
db.myCollection.find().pretty()
```

退出shell输入“quit()”

## 配置mongo shell

### 自定义提示符

可以通过在mongo shell中设置变量prompt的值来修改提示符的内容。prompt变量可以存储字符串以及JavaScript代码。也可以在.mongorc.js文件中增加提示符的逻辑操作来设置每次启动mongo shell时的提示符。

#### 自定义提示符展示操作数

例如，为了创建一个显示当前会话中操作数的mongo shell，在mongo shell中定义一下变量：

>  cmdCount = 1;
>
> prompt = function(){
>
> ​				return (cmdCount++) + "> ";	
>
> ​	}

提示符将会类似下面：

>  1>
>
>  2>
>
>  3>

#### 自定义提示符显示数据库和主机名

为创建形式为<database>@<hostname> $的mongo shell提示符，定义下列变量：

>  hots = db.serverStatus().host;
>
> prompt = function(){
>
> ​			return db+"@"+host+"$";
>
> }

提示符将会类似下面

> kkb@192.168.1.106$

#### 自定义提示符展示服务器的启动时间以及文档数

创建一个包含系统启动时间以及当前数据库种文档数的mongo shell提示符：

> prompt = function{
>
> ​		return "Uptime:"+db.serverStatus().uptime+" Documents:"+db.stats().objects+“ >”;
>
> }

提示符将会类似下面：

> Uptime:5896 Documents:6 >

### 在mongo shell种使用外部编辑器

可以通过在启动mongo shell 之前设置EDITOR环境变量来在mongo shell种使用自己的编辑器。

> export EDITOR = vim
>
> mongo

一旦进入mongo shell中，你可以通过输入edit  <variable>或者edit <function>使用特定的编辑器进行编辑

定义一个函数

> function myFunction(){}

使用编辑器编辑函数

> edit myFunction

命令应该打开vim编辑会话

### 修改mongo Shell批处理大小

> DBQuery.shellBatchSize = 10



# MongoDB CRUD操作

## 插入文档

### 插入方法

- db.collection.insertOne()
- db.collection.insertMany()
- db.collection.insert()

### 插入操作的行为表现

插入的时候如果集合不存在，那么插入操作会创建集合

#### _id字段

在MongoDB中，存储于集合中的每一个文档都需要一个唯一的_id字典作为primary_key。如果一个插入文档操作遗漏了"_id"字段，MongoDB驱动会自动为"_id"字段生成一个ObjectId

#### 原子性

MongoDB中所有的写操作在单一文档层级上是原子的。

#### insertOne

```
db.users.insertOne(
   {
      name: "sue",
      age: 19,
      status: "P"
   }
)
```

insertOne()返回一个结果文档，该结果文档中列举了插入文档的“_id”字段值。

```
{
        "acknowledged" : true,
        "insertedId" : ObjectId("5e11df35b94752ad750b3b6a")
}
```

#### insertMany()

```
db.users.insertMany(
   [
     { name: "bob", age: 42, status: "A", },
     { name: "ahn", age: 22, status: "A", },
     { name: "xi", age: 34, status: "D", }
   ]
)
```

#### insert()



## 查询文档

### 查询方法

MongoDB提供了`db.collection.find()`方法从集合中读取文档。

> db.collection.find(<query filter>,<projection>)

你可以随意增加一个游标修饰符来进行限制、跳过以及排序。除非你声明一个方法 [`sort()`](http://www.mongoing.com/docs/reference/method/cursor.sort.html#cursor.sort) ，否则不会定义查询返回的文档顺序。

#### 示例集合

```
db.users.insertMany(
  [
     {
       _id: 1,
       name: "sue",
       age: 19,
       type: 1,
       status: "P",
       favorites: { artist: "Picasso", food: "pizza" },
       finished: [ 17, 3 ],
       badges: [ "blue", "black" ],
       points: [
          { points: 85, bonus: 20 },
          { points: 85, bonus: 10 }
       ]
     },
     {
       _id: 2,
       name: "bob",
       age: 42,
       type: 1,
       status: "A",
       favorites: { artist: "Miro", food: "meringue" },
       finished: [ 11, 25 ],
       badges: [ "green" ],
       points: [
          { points: 85, bonus: 20 },
          { points: 64, bonus: 12 }
       ]
     },
     {
       _id: 3,
       name: "ahn",
       age: 22,
       type: 2,
       status: "A",
       favorites: { artist: "Cassatt", food: "cake" },
       finished: [ 6 ],
       badges: [ "blue", "red" ],
       points: [
          { points: 81, bonus: 8 },
          { points: 55, bonus: 20 }
       ]
     },
     {
       _id: 4,
       name: "xi",
       age: 34,
       type: 2,
       status: "D",
       favorites: { artist: "Chagall", food: "chocolate" },
       finished: [ 5, 11 ],
       badges: [ "red", "black" ],
       points: [
          { points: 53, bonus: 15 },
          { points: 51, bonus: 15 }
       ]
     },
     {
       _id: 5,
       name: "xyz",
       age: 23,
       type: 2,
       status: "D",
       favorites: { artist: "Noguchi", food: "nougat" },
       finished: [ 14, 6 ],
       badges: [ "orange" ],
       points: [
          { points: 71, bonus: 20 }
       ]
     },
     {
       _id: 6,
       name: "abc",
       age: 43,
       type: 1,
       status: "A",
       favorites: { food: "pizza", artist: "Picasso" },
       finished: [ 18, 12 ],
       badges: [ "black", "blue" ],
       points: [
          { points: 78, bonus: 8 },
          { points: 57, bonus: 7 }
       ]
     }
  ]
)
```

#### 选择集合中所有文档

> ```
> db.users.find( {} )
> db.users.find()
> ```

#### 指定等于条件

> db.users.find({status:"A"})

#### 使用查询操作符指定条件

> ```
> db.users.find( {status :{$in:["P","D"]}} )
> ```

也可以使用$or 操作符表示这个查询，但是在相同字段执行等于检查时，建议使用$in而不是$or

#### 指定AND条件

复合查询可以在集合文档的多个字段上指定条件。隐含的，一个逻辑的AND连接词会连接符合查询的子句，使得查询选出集合中匹配所有条件的文档

> ```
> db.users.find( {status:"A",age:{$lt:30}} )
> ```

#### 指定OR条件

通过使用$OR操作符，你可以指定一个使用逻辑OR连接词连接各子句的符合查询选择集合中匹配至少一个条件的文档。

> db.users.find(
>
> ​	{
>
> ​		$or:[{status:"a"},{age:{$lt:30}}]
>
> ​	}
>
> )

#### 指定AND和OR条件

> db.users.find(
>
> ​	{
>
> ​		type:1,
>
> ​		$or:[{status:"a"},{age:{$lt:30}}]
>
> ​	}
>
> )

#### 嵌入文档的查询

当字段中包含嵌入文档时，查询可以指定嵌入文档中的精确匹配或者使用 “[*dot notation*](http://www.mongoing.com/docs/reference/glossary.html#term-dot-notation) 对嵌入文档中的单个字段指定匹配。

##### 嵌入文档上的精确匹配

> db.users.find({favorites:{artist:"Picasso",food:"pizza"}})

##### 嵌入文档中字段上的等于匹配

使用 [*dot notation*](http://www.mongoing.com/docs/reference/glossary.html#term-dot-notation) 匹配内嵌文档中的特定的字段。内嵌文档中特定字段的相等匹配将筛选出集合中内嵌文档包含该指定字段并等于指定的值的文档。内嵌文档可以包含其他的字段

> db.users.find({"favorites.artist":"Picasso"})

#### 数组上的查询

当字段包含数组，你可查询精确的匹配数组或数组中特定的值。如果数组包含嵌入文档，你可以使用 [*dot notation*](http://www.mongoing.com/docs/reference/glossary.html#term-dot-notation) 查询内嵌文档中特定的字段。

##### 数组上的精确匹配

> db.users.find({badges:["blue","black"]})

#### 匹配一个元素

等于匹配可以指定匹配数组中的单一元素。如果数组中至少一个元素包含特定的值，就可以匹配这些声明

> db.users.find({badges:"black"})

##### 匹配数组中的指定元素

等于匹配可以指定匹配数组某一特定所有或位置的元素，使用 [*dot notation*](http://www.mongoing.com/docs/reference/glossary.html#term-dot-notation) 。

> db.users.find({"badges.0":"black"})

##### 指定数组元素的多个查询条件

使用 [`$elemMatch`](http://www.mongoing.com/docs/reference/operator/query/elemMatch.html#op._S_elemMatch) 操作符为数组元素指定复合条件，以查询数组中至少一个元素满足所有指定条件的文档。

下面的例子查询 `finished` 数组至少包含一个大于 ([`$gt`](http://www.mongoing.com/docs/reference/operator/query/gt.html#op._S_gt)) `15` 并且小于 ([`$lt`](http://www.mongoing.com/docs/reference/operator/query/lt.html#op._S_lt)) `20` 的元素的文档：

```
db.users.find( { finished: { $elemMatch: { $gt: 15, $lt: 20 } } } )
```

##### 元素组合满足查询条件

下面的例子查询 `finished` 数组包含以某种组合满足查询条件的元素的文档;例如,一个元素满足大于 `15`的条件并且有另一个元素满足小于 `20` 的条件,或者有一个元素满足了这两个条件：

```
db.users.find( { finished: { $gt: 15, $lt: 20 } } )
```

#### 其他方法

下面的方法也可以从集合中读取文档：

- [`db.collection.findOne`](http://www.mongoing.com/docs/reference/method/db.collection.findOne.html#db.collection.findOne) [[1\]](http://www.mongoing.com/docs/tutorial/query-documents.html#findone)
- 在 [*aggregation pipeline*](http://www.mongoing.com/docs/core/aggregation-pipeline.html) 中, [`$match`](http://www.mongoing.com/docs/reference/operator/aggregation/match.html#pipe._S_match) 管道阶段提供对MongoDB查询的访问.

##### 返回嵌入文档中的指定字段

```
db.users.find(
   { status: "A" },
   { name: 1, status: 1, "favorites.food": 1 }
)
```

##### 映射数组中的嵌入文档

```
db.users.find( { status: "A" }, { name: 1, status: 1, "points.bonus": 1 } )
```

#### 查询为NULL或不存在的字段

```
db.users.insert(
   [
      { "_id" : 900, "name" : null },
      { "_id" : 901 }
   ]
)
```



##### 相等过滤

```
db.users.find({name : null})
```

该查询返回这两个文档:

```
{ "_id" : 900, "name" : null }
{ "_id" : 901 }
```



##### 类型筛选

`{ name : { $type: 10 } }` 查询 *仅仅* 匹配那些包含值是 `null` 的 `name` 字段的文档,亦即 `条目` 字段的值是BSON类型中的 `Null` (即 `10` ):

```
db.users.find({name:{$type:10}})
```

该查询只返回文 `条目` 字段是 `null` 值的文档:

```
{ "_id" : 900, "name" : null }
```



##### 存在性筛查

```
db.users.find( { name : { $exists: false } } )
```

该查询只返回那些 *没有* 包含 `条目` 字段的文档:

```
{ "_id" : 901 }
```

### 在mongo命令里迭代游标

#### 手动迭代游标

在mongo里，当你使用var关键字把find()方法返回的游标赋值给一个变量时，它将不会自动迭代

在命令行里，可以调用游标变量迭代最多20次并且打印匹配的文档

```
var myCursor = db.users.find({type:2});
myCursor

```

也可以使用游标的Next（）方法来访问文档

```
var myCursor = db.user.find({type:2})
while (myCursor.hasNext()){
	print(tojson(myCursor.next()))
}
```

作为一种替代的打印操作，考虑使用 `printjson()` 助手方法来替代 `print(tojson())` ：

```
var myCursor = db.users.find( { type: 2 } );

while (myCursor.hasNext()) {
   printjson(myCursor.next());
}
```

你可以使用游标方法 [`forEach()`](http://www.mongoing.com/docs/reference/method/cursor.forEach.html#cursor.forEach) 来迭代游标并且访问文档，如下例所示：

```
var myCursor =  db.users.find( { type: 2 } );

myCursor.forEach(printjson);
```

#### 迭代器索引

在 `mongo`命令行里，你可以使用 :method:`~cursor.toArray()` 方法来迭代游标，并且以数组的形式来返回文档，如下例所示：

```
var myCursor = db.inventory.find( { type: 2 } );
var documentArray = myCursor.toArray();
var myDocument = documentArray[3];
```

```
var myCursor = db.users.find( { type: 2 } );
var myDocument = myCursor[1];
```



## 更新文档

### 更新

MongoDB提供如下方法更新集合中的文档

| 方法                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`db.collection.updateOne()`](http://www.mongoing.com/docs/reference/method/db.collection.updateOne.html#db.collection.updateOne) | 即使可能有多个文档通过过滤条件匹配到，但是也最多也只更新一个文档。 |
| [`db.collection.updateMany()`](http://www.mongoing.com/docs/reference/method/db.collection.updateMany.html#db.collection.updateMany) | 更新所有通过过滤条件匹配到的文档.                            |
| [`db.collection.replaceOne()`](http://www.mongoing.com/docs/reference/method/db.collection.replaceOne.html#db.collection.replaceOne) | 即使可能有多个文档通过过滤条件匹配到，但是也最多也只替换一个文档。 |
| [`db.collection.update()`](http://www.mongoing.com/docs/reference/method/db.collection.update.html#db.collection.update) | 即使可能有多个文档通过过滤条件匹配到，但是也最多也只更新或者替换一个文档。默认情况下, [`db.collection.update()`](http://www.mongoing.com/docs/reference/method/db.collection.update.html#db.collection.update) 只更新 **一个** 文档。要更新多个文档，请使用 [*multi*](http://www.mongoing.com/docs/reference/method/db.collection.update.html#multi-parameter) 选项。 |

### 行为表现

#### 原子性

MongoDB中所有的写操作在单一文档层级上都是原子的。

#### _id字段

一旦设定，你不能更新**_id**字段的值，也不能用有不同**_id**字段值的替换文档来替换已经存在的文档

#### 文档大小

当执行更新操作增加的文档大小超过了为该文档分配的空间时。更新操作会在磁盘上重定位该文档。

#### 字段顺序

MongoDB按照文档写入的顺序整理文档字段，*除了* 如下的情况：

- `_id` 字段始终是文档中的第一个字段。
- 包括字段名称的 [`renaming`](http://www.mongoing.com/docs/reference/operator/update/rename.html#up._S_rename) 操作可能会导致文档中的字段重新排序。

#### Upsert选项

如果 [`db.collection.update()`](http://www.mongoing.com/docs/reference/method/db.collection.update.html#db.collection.update)，[`db.collection.updateOne()`](http://www.mongoing.com/docs/reference/method/db.collection.updateOne.html#db.collection.updateOne)， [`db.collection.updateMany()`](http://www.mongoing.com/docs/reference/method/db.collection.updateMany.html#db.collection.updateMany) 或者 [`db.collection.replaceOne()`](http://www.mongoing.com/docs/reference/method/db.collection.replaceOne.html#db.collection.replaceOne) 包含 `upsert : true` **并且** 没有文档匹配指定的过滤器，那么此操作会创建一个新文档并插入它。如果有匹配的文档，那么此操作修改或替换匹配的单个或多个文档。

## 删除文档

### 删除方法

| 方法                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`db.collection.remove()`](http://www.mongoing.com/docs/reference/method/db.collection.remove.html#db.collection.remove) | Delete a single document or all documents that match a specified filter. |
| [`db.collection.deleteOne()`](http://www.mongoing.com/docs/reference/method/db.collection.deleteOne.html#db.collection.deleteOne) | Delete at most a single document that match a specified filter even though multiple documents may match the specified filter. |
| [`db.collection.deleteMany()`](http://www.mongoing.com/docs/reference/method/db.collection.deleteMany.html#db.collection.deleteMany) | 删除所有匹配指定过滤条件的文档.                              |

### 删除的行为表现

#### 索引

Delete operations do not drop indexes, even if deleting all documents from a collection.

#### 原子性

MongoDB中所有的写操作在单一文档层级上是原子的

## Read Isolation(Read Concern)

### Read Concern Levels

The following read concern levels are available:

| level          | Description                                                  |
| -------------- | ------------------------------------------------------------ |
| **"local"**    | Default. The query returns the instance’s most recent data. Provides no guarantee that the data has been written to a majority of the replica set members (i.e. may be rolled back). |
| **"majority"** | The query returns the instance’s most recent data acknowledged as having been written to a majority of members in the replica set.<br />To use read concern level of **"majority"**<br />you must start the mongod instances with the --enableMajorityReadConcern command line option(or the replication.enableMajorityReadConcern set to true if using a configuration file)<br />replica sets must use WitrdTiger storage engine and election protocol version 1. |
|                |                                                              |

### 来自阿里巴巴张东友的解读

#### 作者简介

张东友，阿里巴巴技术专家，主要关注分布式存储、Nosql数据库等技术领域，先后参与[TFS（淘宝分布式文件系统)](https://github.com/alibaba/tfs)、[Redis云数据库](https://www.aliyun.com/product/kvstore)等项目，目前主要从事[MongoDB云数据库](https://www.aliyun.com/product/mongodb)的研发工作，致力于让开发者用上最好的MongoDB云服务。

#### MongoDB readConcern原理解析

MongoDB 可以通过 [writeConcern](https://yq.aliyun.com/articles/54367?spm=5176.8091938.0.0.peSsIL) 来定制写策略，3.2版本后又引入了 `readConcern` 来灵活的定制读策略。

##### readConcern vs readPreference

MongoDB控制读策略，还有一个readPreference的设置，为了避免混淆，先简单说明下二者的区别

- **readPreference**主要控制客户端Driver从复制集的哪个节点读取数据，这个特性可方便实现读写分离，就近读取等策略
  - **primary** 只从primary节点读取数据，这个是默认设置
  - **primaryPreferred** 优先从primary读取，primary不可服务，从secondary读
  - **secondary** 只从secondary节点读数据
  - **secondaryPreferred** 优先从secondary读取，没有secondary成员时，从primary读取
  - **nearest** 根据网络距离就近读取
- **readConcern**决定读取数据时，能读到什么样的数据
  - local 能读取任意数据，这个是默认设置
  - majority 只能读取到【成功写入大多数节点的数据】

readPreference 和 readConcern可以配合使用。

##### readConcern解决什么问题

readConcern的初衷在于解决**脏读**的问题，比如用户从MongoDB的primary上读取了某一条数据，但这条数据并没有同步到大多数节点，然后primary就故障了，重新恢复后这个primary节点会将未同步到大多数节点的数据回滚掉，导致用户读到了**脏数据**

当指定readConcern级别为majority是，能保证用户读到的数据**已经写入到大多数节点**，而这样的数据肯定不会发生回滚，避免了脏读的问题。

需要注意的是：readConcern能保证读到的数据**不会发生回滚**，但并不能保证读的数据是最新的。

有用户误以为，`readConcern` 指定为 majority 时，客户端会从大多数的节点读取数据，然后返回最新的数据。

##### readConcern实现原理

MongoDB 要支持 majority 的 readConcern 级别，必须设置`replication.enableMajorityReadConcern`参数，加上这个参数后，MongoDB 会起一个单独的snapshot 线程，会周期性的对当前的数据集进行 snapshot，并记录 snapshot 时最新 oplog的时间戳，得到一个映射表。

| 最新OPLOG时间戳 | SNAPSHOT  | 状态        |
| --------------- | --------- | ----------- |
| t0              | snapshot0 | committed   |
| t1              | snapshot1 | uncommitted |
| t2              | snapshot2 | uncommitted |
| t3              | snapshot3 | uncommitted |

只有确保 oplog 已经同步到大多数节点时，对应的 snapshot 才会标记为 commmited，用户读取时，从最新的 commited 状态的 snapshot 读取数据，就能保证读到的数据一定已经同步到的大多数节点。

关键的问题就是如何确定『oplog 已经同步到大多数节点』？

###### primary 节点

secondary 节点在 自身oplog发生变化时，会通过 replSetUpdatePosition 命令来将 oplog 进度立即通知给 primary，另外心跳的消息里也会包含最新 oplog 的信息；通过上述方式，primary 节点能很快知道 oplog 同步情况，知道『最新一条已经同步到大多数节点的 oplog』，并更新 snapshot 的状态。比如当t2已经写入到大多数据节点时，snapshot1、snapshot2都可以更新为 commited 状态。（不必要的 snapshot也会定期被清理掉）

###### secondary节点

secondary 节点拉取 oplog 时，primary 节点会将『最新一条已经同步到大多数节点的 oplog』的信息返回给 secondary 节点，secondary 节点通过这个oplog时间戳来更新自身的 snapshot 状态。

##### 注意事项

- 目前 `readConcern` 主要用于跟 mongos 与 config server 的交互上，参考[MongoDB Sharded Cluster 路由策略](https://yq.aliyun.com/articles/58689?spm=5176.8091938.0.0.UemLKg)
- 使用 `readConcern` 需要配置`replication.enableMajorityReadConcern`选项
- 只有支持 readCommited 隔离级别的存储引擎才能支持 `readConcern`，比如 wiredtiger 引擎，而 mmapv1引擎则不能支持。

## Write Concern

### Write Concern Specification

Write concern can include the following fields:

```
{ w: <value>, j: <boolean>, wtimeout: <number> }
```

- the [*w*](http://www.mongoing.com/docs/reference/write-concern.html#wc-w) option to request acknowledgment that the write operation has propagated to a specified number of [`mongod`](http://www.mongoing.com/docs/reference/program/mongod.html#bin.mongod) instances or to [`mongod`](http://www.mongoing.com/docs/reference/program/mongod.html#bin.mongod) instances with specified tags.
- the [*j*](http://www.mongoing.com/docs/reference/write-concern.html#wc-j) option to request acknowledgement that the write operation has been written to the journal, and
- the [*wtimeout*](http://www.mongoing.com/docs/reference/write-concern.html#wc-wtimeout) option to specify a time limit to prevent write operations from blocking indefinitely.

### 来自阿里张东有的解读

MongoDB支持客户端灵活配置写入策略（[writeConcern](https://docs.mongodb.com/manual/reference/write-concern/)），以满足不同场景的需求。

```
db.collection.insert({x: 1}, {writeConcern: {w: 1}})
```

#### writeConcern选项

MongoDB支持的WriteConncern选项如下

1. w: 数据写入到number个节点才向用客户端确认
   - {w: 0} 对客户端的写入不需要发送任何确认，适用于性能要求高，但不关注正确性的场景
   - {w: 1} 默认的writeConcern，数据写入到Primary就向客户端发送确认
   - {w: “majority”} 数据写入到副本集大多数成员后向客户端发送确认，适用于对数据安全性要求比较高的场景，该选项会降低写入性能
2. j: 写入操作的journal持久化后才向客户端确认
   - 默认为”{j: false}，如果要求Primary写入持久化了才向客户端确认，则指定该选项为true
3. wtimeout: 写入超时时间，仅w的值大于1时有效。
   - 当指定{w: }时，数据需要成功写入number个节点才算成功，如果写入过程中有节点故障，可能导致这个条件一直不能满足，从而一直不能向客户端发送确认结果，针对这种情况，客户端可设置wtimeout选项来指定超时时间，当写入过程持续超过该时间仍未结束，则认为写入失败。

#### {w:"majority"}解析

{w: 1}、{j: true}等writeConcern选项很好理解，Primary等待条件满足发送确认；但{w: “majority”}则相对复杂些，需要确认数据成功写入到大多数节点才算成功，而MongoDB的复制是通过Secondary不断拉取oplog并重放来实现的，并不是Primary主动将写入同步给Secondary，那么Primary是如何确认数据已成功写入到大多数节点的？

1. Client向Primary发起请求，指定writeConcern为{w: “majority”}，Primary收到请求，本地写入并记录写请求到oplog，然后等待大多数节点都同步了这条/批oplog（Secondary应用完oplog会向主报告最新进度)。
2. Secondary拉取到Primary上新写入的oplog，本地重放并记录oplog。为了让Secondary能在第一时间内拉取到主上的oplog，find命令支持一个[awaitData的选项](https://docs.mongodb.com/manual/reference/command/find/#dbcmd.find)，当find没有任何符合条件的文档时，并不立即返回，而是等待最多maxTimeMS(默认为2s)时间看是否有新的符合条件的数据，如果有就返回；所以当新写入oplog时，备立马能获取到新的oplog。
3. Secondary上有单独的线程，当oplog的最新时间戳发生更新时，就会向Primary发送replSetUpdatePosition命令更新自己的oplog时间戳。
4. 当Primary发现有足够多的节点oplog时间戳已经满足条件了，向客户端发送确认。

## MongoDB CRUD概念

### 原子性和事务处理

#### $isolate操作

##### Definition

> Prevents a write operation that affects multiple documents from yielding to other reads or writes once the first document is written. By using the `$isolated` option, you can ensure that no client sees the changes until the operation completes or errors out.
>
> This behavior can significantly affect the concurrency of the system as the operation holds the write lock much longer than normal for storage engines that take a write lock (e.g. MMAPv1), or for document-level locking storage engine that normally do not take a write lock (e.g. WiredTiger), `$isolated` operator will make WiredTiger single-threaded for the duration of the operation.

##### Behavior

> The `$isolated` isolation operator does **not** provide “all-or-nothing” atomicity for write operations.
>
> `$isolated` does **not** work with [*sharded clusters*](http://www.mongoing.com/docs/reference/glossary.html#term-sharded-cluster).

##### example

```
db.foo.update(
    { status : "A" , $isolated : 1 },
    { $inc : { count : 1 } },
    { multi: true }
)
```

Without the `$isolated` operator, the `multi`-update operation will allow other operations to interleave with its update of the matched documents.

#### 类事务处理语句执行两个阶段的提交

##### 未完待续

#### 并行处理控制

并行处理的控制允许多个应用同时运行而不会造成数据的不一致或者冲突。

一个方法是在字段上创建一个唯一性的:ref: unique index <index-type-unique> 。这样就可以阻止插入或者更新重复的数据。在多个字段上创建唯一性索引将保证多个字段组合的唯一性。

另外一种方法是通过在写操作中使用查询断言来指定期望的字段当前值。两阶段提交模式除了提供查询断言以外还额外可以指定期望的数据写的状态

### 读隔离、一致性和时近性

#### 隔离的保证

##### 读的无限制性

在MongoDB中客户端可以在写操作持久化之前看到些的结果

## 评估当前操作的性能

```
db.users.find({name:"sue"}).explain()
```



## 优化查询性能

### 创建索引优化查询

对于一般场景下发布的查询，创建 [*indexes*](http://www.mongoing.com/docs/indexes.html)。如果查询搜索了多个字段，创建 [*compound index*](http://www.mongoing.com/docs/core/index-compound.html#index-type-compound)。扫描索引要比扫描一个集合快得多。**索引结构比文档引用要小，并且有序地存储着文档引用**。

```
db.posts.createIndex( { author_name : 1 } )
```

索引还能提升在指定字段上进行常规排序的查询的效率。

```
db.posts.createIndex( { timestamp : 1 } )
```

被优化的查询：

```
db.posts.find().sort( { timestamp : -1 } )
```

由于MongoDB能以升序和降序两种顺序读取索引，所以单一键的索引的方向并不重要。

索引支持查询、更新操作，以及 [*aggregation pipeline*](http://www.mongoing.com/docs/core/aggregation-pipeline.html#aggregation-pipeline-operators-and-performance) 的某些阶段。

### 限制查询结果的数目以减少网络需求

MongoDB [*cursors*](http://www.mongoing.com/docs/reference/glossary.html#term-cursor) 以多个文档为一组返回结果。如果你知道你想要的结果数目，你就可以使用 [`limit()`](http://www.mongoing.com/docs/reference/method/cursor.limit.html#cursor.limit)方法来减少对网络资源的需求。

该方法通常与排序操作结合使用。例如，如果你只需要 `posts` 集合上的查询的10条结果，你可以使用如下命令：

```
db.posts.find().sort( { timestamp : -1 } ).limit(10)
```

### 使用映射只返回需要的数据

当你仅仅需要文档字段的子集，你可以通过只返回你需要的字段来获取更好的性能。

例如，如果对于 `posts` 集合的查询，你只需要 `timestamp`， `title`， `author` 和 `abstract` 字段，你可以使用如下命令：

```
db.posts.find( {}, { timestamp : 1 , title : 1 , author : 1 , abstract : 1} ).sort( { timestamp : -1 } )
```

### 使用$hint选择一个特定的索引

在大多数情况下， [*query optimizer*](http://www.mongoing.com/docs/core/query-plans.html#read-operations-query-optimization) 能对指定的操作选择最优的索引；不过，你可以使用 [`hint()`](http://www.mongoing.com/docs/reference/method/cursor.hint.html#cursor.hint) 方法强制MongoDB使用指定的索引。你可以使用 [`hint()`](http://www.mongoing.com/docs/reference/method/cursor.hint.html#cursor.hint) 来帮助性能测试，或者在某些你必须选择一个字段或字段被多个索引包含的查询上使用它。

### 使用增量操作符在服务端执行操作

Use MongoDB’s [`$inc`](http://www.mongoing.com/docs/reference/operator/update/inc.html#up._S_inc) operator to increment or decrement values in documents. The operator increments the value of the field on the server side, as an alternative to selecting a document, making simple modifications in the client and then writing the entire document to the server. The [`$inc`](http://www.mongoing.com/docs/reference/operator/update/inc.html#up._S_inc) operator can also help avoid race conditions, which would result when two application instances queried for a document, manually incremented a field, and saved the entire document back at the same time.

## 写操作性能

### 索引

在每次插入、更新及删除操作之后，除了数据本身之外，MongoDB还必须更新与该集合相关的 *每一个* 索引。因此，一个集合中的每个索引都会对写操作的性能加大一定数量的开销。

一般说来，索引为 *读操作* 提供的性能提升相较于插入的惩罚是值得的。然而，为了尽可能地优化写入的性能，在创建新索引及评估现有索引时，一定要谨慎，以保证您的查询确实使用了这些索引。

### 存储性能

#### 硬件

存储系统的容量给MongoDB写操作的性能带来了一些重要的、物理方面的限制。许多与驱动器存储系统相关的单值因子都会影响写操作的性能，包括：随机存取模式、磁盘缓存、磁盘预加载库文件以及独立磁盘冗余阵列配置等。

#### 日志

To provide durability in the event of a crash, MongoDB uses ***write ahead logging*** to an on-disk [*journal*](http://www.mongoing.com/docs/reference/glossary.html#term-journal). MongoDB writes the in-memory changes first to the on-disk journal files. If MongoDB should terminate or encounter an error before committing the changes to the data files, MongoDB can use the journal files to apply the write operation to the data files.

While the durability assurance provided by the journal typically outweigh the performance costs of the additional write operations, consider the following interactions between the journal and performance:

- If the journal and the data file reside on the same block device, the data files and the journal may have to **contend for** a **finite** number of available I/O resources. Moving the journal to a separate device may increase the capacity for write operations.
- If applications specify [*write concerns*](http://www.mongoing.com/docs/reference/write-concern.html) that include the [`j option`](http://www.mongoing.com/docs/reference/write-concern.html#writeconcern.j), [`mongod`](http://www.mongoing.com/docs/reference/program/mongod.html#bin.mongod) will decrease the duration between journal writes, which can increase the overall write load.
- The duration between journal writes is configurable using the [`commitIntervalMs`](http://www.mongoing.com/docs/reference/configuration-options.html#storage.journal.commitIntervalMs) run-time option. Decreasing the period between journal commits will increase the number of write operations, which can limit MongoDB’s capacity for write operations. Increasing the amount of time between journal commits may decrease the total number of write operation, but also increases the chance that the journal will not record a write operation in the event of a failure.