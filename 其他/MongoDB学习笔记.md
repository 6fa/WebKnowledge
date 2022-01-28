# MongoDB学习笔记

参考：
- [mongodb中文手册](https://mongodb.net.cn/manual/introduction/)
- [菜鸟mongodb教程](https://www.runoob.com/mongodb/mongodb-tutorial.html)
- [因为 MongoDB 没入门，我丢了一份实习工作](https://juejin.cn/post/6844904182428729351)
- [mongoDB看这篇就够了](https://juejin.cn/post/6844903857315643406#heading-0)
- [关系型数据库 VS 非关系型数据库](https://zhuanlan.zhihu.com/p/78619241)
- [关系型与非关系型数据库的优缺点](https://blog.csdn.net/G_Q_L/article/details/77946483)
- [MongoDB索引原理](https://mongoing.com/archives/2797)

## 1.非关系型数据库
MongoDB是非关系型数据库。

关系型数据库与非关系型数据库概念：
  - 关系型数据库采用**关系模型**来组织数据库，通常是二维表格，每列表示特定类型的数据（比如身高、年龄等），每行表示一个对象或实体的相关值集合（如张三的身高、年龄等）
  - 非关系型数据库，通常以**键值对**的形式储存数据，每一个元组都可以有不一样的字段，不会局限于固定的结构，可以减少一些时间和空间的开销。使用这种方式，为了获取用户的不同信息，不需要像关系型数据库中，需要进行多表查询，仅仅需要根据key来取出对应的value值即可。

关系型数据库的优缺点、适用场景：
  - 优点：二维表格形式容易理解；保存的数据必须标准化，减少数据冗余提高可靠性
  - 缺点：
    - 数据读写必须经过sql解析，大量数据、高并发下读写性能不足
    - 扩展困难
  - 适用场景
    - 适合储存结构化、复杂的数据，比如用户的账号、地址
    - 这些数据的规模、增长的速度通常是可以预期的

非关系型数据库的优缺点、适用场景：
  - 优点：
    - 存储格式多，支持key-value形式、文档形式、图片形式
    - 处理高并发、大批量数据的能力强
    - 扩展性高
  - 缺点：只能储存简单的数据
  - 适用场景：
    - 储存非结构化数据，比如文章、评论
    - 这些数据是海量的，并且增长的速度是难以预期的
    - 按照key获取数据效率很高，但是对于join或其他结构化查询的支持就比较差

## 2.MongoDB核心

### 2.1 database
MongoDB的单个实例可以有多个独立的数据库，每一个都有自己的集合和权限，不同的数据库也放置在不同的文件中。

**a.查看数据库：show dbs**

在启动数据库和连接数据库后，终端输入 show dbs 可以查看当下的数据库，一般有以下几个：
- admin：权限数据库，储存mongodb的用户信息，admin库里的用户自动继承所有数据库的权限
- config：分片时（储存海量数据时，多台机器分割数据），用于保存分片的相关信息
- local：这个数据永远不会被复制，可以用来存储限于本地单台服务器的任意集合

**b.显示当前数据库：db**

会有一个默认的test数据库，如果没有创建新的数据库，则集合将被放在test数据库中

**c.创建/切换数据库：use 数据库名称**

如果数据库不存在，则创建；如果数据库为已经存在的，则是切换到该数据库

**d.删除数据库：db.dropDatebase()**

删除的是当前数据库，所以要切换到想要删除的库再执行命令

### 2.2 collection
集合即一组文档的组合
- 创建集合：db.createCollection( 集合名称 )
- 查看已有集合：show collections
- 删除集合：db.集合名称.drop()。删除成功则返回true

### 2.3 document
文档是指类似JSON对象的数据结构（BSON，JSON的超集），即一组键值对。

#### 2.3.1 插入文档
- db.集合名.insert( document )
  - 新建的数据会自动生成主键_id
  - 如果插入的document的主键已经存在，则会报错
  - 如果是多条数据，可以是传入文档数组
- db.集合名.insertOne()
- db.集合名.insertMany(arr)

#### 2.3.2 更新文档
可以使用 db.集合名.update( )
```javascript
   db.col.update(
     <query>,  //查询条件
     <update>, //update的对象和一些更新的操作符（如$,$inc...）等
     {
       upsert: <boolean>,//可选,如果不存在update的记录，是否插入objNew,true为插入，默认是false
     
       multi: <boolean>,//可选，mongodb 默认是false,只更新找到的第一条记录
     		//如果这个参数为true,就把按条件查出来多条记录全部更新
     
       writeConcern: <document>//可选，抛出异常的级别
     }
   )

```

例子：
```javascript
//假如user集合中有两条条数据：
{_id:...,name:"Jack", age:20}
{_id:...,name:"Rose", age:51}

//更新数据
db.user.update({age:{$gt: 50}},{$set:{role: "elderly"}}, true, true)
//WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })

//更新后
{_id:...,name:"Jack", age:20}
{_id:...,name:"Rose", age:51,role:"elderly"}
```

#### 2.3.3 常用的原子操作命令
比如文档的保存，修改，删除等，都是原子操作；所谓原子操作就是要么这个文档保存到Mongodb，要么没有保存到Mongodb，不会出现查询到的文档没有保存完整的情况。
常用操作：
- $set
- $unset
- $inc
- $push
- $pushAll
- $pull
- $addToSet
- $pop
- $rename

```javascript
// $set:
//指定一个键并更新键，如不存在则创建
{ $set : { field : value } }

// $unset: 
//删除指定键
{ $inc : { field : value } }

// $inc: 
//对文档的某个值为数字型（只能为满足要求的数字）的键进行增减的操作
{ $inc : { field : value } } //value为增加值，负数为减少

// $push: 
//把value追加到field里面去，field一定要是数组类型才行
// 如果field不存在，会新增一个数组类型加进去
{ $push : { field : value } }

// $pushAll
//同push，只是一次追加多个数组字段
{ $pushAll : { field : value_array } }

// $pull
// 从数组filed内删除一个等于value值
{ $pull : { field : _value } }

// $addToSet
// 增加一个值到数组内，而且只有当这个值不在数组内才增加

// $pop
// 删除数组的第一个或最后一个元素
{ $pop : { field : 1 } }

// $rename
// 修改field字段名称
{ $rename : { old_field_name : new_field_name } }
```

#### 2.3.4 常用的条件操作符
用于查询条件中
- $gt：大于
- $lt：小于
- $gte：大于等于
- $lte：小于等于
- $ne：不等于

```javascript
//等于
db.col.find({"key":"value"})

//大于
db.col.find({"key":{$gt:50}})
```

- $or

```javascript
//多个查询条件，类似于and
db.col.find({"key1":"value1","key2":"value2"})

// or条件,只需满足
db.col.find(
  {
    $or: [
      {"key1":"value1"}, {"key2":"value2"}
    ]
  }
)

//and or一起使用
//满足likes>50,且满足by或title其中一个
db.col.find({"likes": {$gt:50}, $or: [{"by": "菜鸟教程"},{"title": "MongoDB 教程"}]})
```

- $in：包含；$nin：不包含

```javascript
db.col.find({"age":{$in: [10,11]}})  //查找age等于10或11的数据
```

- $all：同时匹配所有

```javascript
db.col.find({"coures":{$all: ["js","mongodb"]}}) //course要同时包含js和mongodb

//如
{"name":"David",age:26,course:["js","Node","Mongodb"]},
{"name":"Tom",age:26,course:["js","Node","Mongoose"]})
```

- $size：数组元素个数
- $regex：正则表达式匹配

#### 2.3.5 删除文档
```javascript
db.col.remove(
   <query>, //（可选）删除的文档的条件
   {
     justOne: <boolean>, //（可选）如果设为 true 或 1，则只删除一个文档
  		//如果不设置该参数，或使用默认值 false，则删除所有匹配条件的文档
  
     writeConcern: <document> //（可选）抛出异常的级别
   }
)
```

#### 2.3.6 查询文档
- db.集合名.find( )
```javascript
db.col.find(
  query,  //可选，查询条件。不写则返回集合的全部文档
  projection //可选，使用投影操作符指定返回的键，比如{"key":1}为显示key
)
```

- db.集合名.find( ).pretty()：以格式化的方式来显示所有文档
- db.集合名.findOne( )：只返回符合查询条件的一个文档。find()返回的是数组，而findOne返回的是一个对象

### 2.4 limit( ) 与 skip( )
- limit( )接受一个数字参数，限制读取的文档条数

```javascript
db.COLLECTION_NAME.find().limit(NUMBER)
```

- skip( )接受一个数字参数，表示跳过指定数量的文档

```javascript
//下面例子中，limit和skip结合使用
//只返回符合条件的文档中第二条（limit限制为1，且skip了第一条）
db.col.find({},{"title":1,_id:0}).limit(1).skip(1)
```

### 2.5 排序
sort( )方法用于对数据进行排序：
```javascript
db.col.find().sort({"key":1})
// 按照键名为key的字段对数据进行排序
// 1为升序
// -1为降序
```

### 2.6 索引
索引使得mongodb可以快速地查询，如果没有索引，就要扫描集合中的每个文档。
- 插入文档后，会被存储引擎持久化存储，生成位置信息，可以通过位置信息，从存储引擎中读取文档。

| 位置信息 | 文档 |
| ---- | ---- |
| pos1 |	{“name” : “jack”, “age” : 19 } |
| pos2 |	{“name” : “rose”, “age” : 20 } |
| pos3 |	{“name” : “jack”, “age” : 18 } |
| pos4 |	{“name” : “tony”, “age” : 21} |
| pos5 |	{“name” : “adam”, “age” : 18} |

- 假如要查询年龄为50的用户：db.col.find({age: 50})，就要扫描所有文档对比age是否为50
- 当集合文档数量庞大时，对集合全面扫描开销会很大
- 可以给age字段建立一个索引：
```javascript
//给age建立升序索引
db.user.createIndex({age: 1})
```

- 建立索引后，会额外储存一份按照age升序排序的索引数据。索引通常采用类似btree的结构持久化存储，以保证从索引里快速（O(logN)的时间复杂度）找出某个age值对应的位置信息，然后根据位置信息就能读取出对应的文档

| age |	位置信息 |
| ---- | ---- |
| 18 |	pos3 |
| 18 |	pos5 |
| 19 |	pos1 |
| 20 |	pos2 |
| 21 |	pos4 |

#### 2.6.1 创建索引
默认索引是_id，且不可删除。

通过db.集合名.createIndex( )创建索引：

```javascript
db.collection.createIndex(keys, options)

keys：要创建索引的字段，如{age: 1}，1为升序，-1为降序

options:{
	background: 建索引过程会阻塞其它数据库操作，将它设置为true可以以后台方式创建索引,
  unique: 是否唯一，
  name: 索引名称，如果未指定，则以创建索引的字段+排序顺序为名称，如age_1,
  v: 索引版本号，默认的索引版本取决于mongod创建索引时运行的版本，
  
  //...省略其他
}
```

#### 2.6.2 查看索引
使用db.集合.getIndexes( ):

```javascript
[
  {
    "v" : 2,
    "key" : {
            "_id" : 1
    },
    "name" : "_id_"
  }
]
```

#### 2.6.3 复合索引
复合索引指使用多个字段联合创建索引，会先按第一个字段排序，在已经排序的基础上再按第二个字段排序，以此类推：

```javascript
db.col.createIndex({age:1,name:1})

//可以通过该索引加速查找的情况：
db.col.find( {age： 18， name: "jack"} )
db.person.find( {age： 18} )

//不可以通过该复合索引加速查找的情况：
db.person.find( {name: "jack"} )
```

#### 2.6.4 删除索引
- 删除所有索引：db.集合名.dropIndexes()
- 删除指定索引：db.集合名.dropIndex( 索引名称 )

### 2.7 聚合
聚合 aggregate( ) 通过**管道**处理数据(诸如统计平均值，求和等)，并返回计算后的数据结果。

```javascript
//基本语法
db.col.aggregate(
  [
    <stage1>, // 管道1
    <stage2>
  ]
)


//比如
db.col.aggregate(
  [
    //$match用于获取分数大于70小于或等于90记录
    { $match : { score : { $gt : 70, $lte : 90 } } },
    //将符合条件的记录送到下一阶段$group管道操作符进行处理
    { $group: { _id: null, count: { $sum: 1 } } }
  ]
)
//如果是单个管道操作，可以不用使用数组包裹
```

##### 管道
管道用于将当前命令的输出结果传递给下个命令：当前管道处理完毕后将结果传递下个管道处理。

聚合操作中的常用操作：
- $project：按照自定义的结构显示文档

```javascript
//结果中就只还有_id,tilte和author三个字段
db.article.aggregate(
    { $project : {
        title : 1 , //1或者true表示显示
        author : 1 ,
    }}
 );

//_id为默认显示的，如果不想显示_id，则设为0或false
db.article.aggregate(
  { $project : {
      _id : 0 ,
      title : 1 ,
      author : 1
 }});
```

- $group：将集合中的文档分组，可用于统计结果
- $match：用于过滤数据，只输出符合条件的文档
- $sort：排序
- $limit：限制聚合管道返回的文档数
- $skip：在聚合管道中跳过指定数量的文档，并返回余下的文档

##### 表达式操作符
- $sum：计算总和，$sum: 1表示每条记录累积加1，$sum: "$score"表示score字段的总和
- $avg：计算平均值，$avg: '$count' 计算count字段的平均值【注意这里count为字段的键名，所以要加上$引用】
- $min
- $max
- $push：将值加入一个数组中，不会判断是否有重复的值。scores: {$push: "$score"} 将每条记录的score字段值添加到scores数组
- $addToSet：将值加入一个数组中，会判断重复值，已存在则不添加
- $first：根据资源文档的排序获取第一个文档数据
- $last：根据资源文档的排序获取最后一个文档数据

##### 例子
```javascript
//根据name分组，将age添加到数组
> db.user.aggregate({$group:{_id:"$name",totalage:{$push:"$age"}}})
{ "_id" : "jack", "totalage" : [ 19 ] }
{ "_id" : "bobby", "totalage" : [ ] }
{ "_id" : "Rose", "totalage" : [ 51 ] }

//全部统计
> db.user.aggregate({$group:{_id:null,totalAge:{$push:"$age"}}})
{ "_id" : null, "totalAge" : [ 19, 51 ] }


// skip实例
db.article.aggregate(
    { $skip : 5 });

//
db.col.aggregate(
  [
    //$match用于获取分数大于70小于或等于90记录
    { $match : { score : { $gt : 70, $lte : 90 } } },
    //将符合条件的记录送到下一阶段$group管道操作符进行处理
    //这里表示共有几条传过来的文档
    { $group: { _id: null, count: { $sum: 1 } } }  //注意此处id为null
  ]
)
```

### 2.8 ObjectId
文档的默认索引_id值是ObjectId。

ObjectId 是一个12字节 BSON 类型数据，有以下格式：
- 前4个字节表示时间戳
- 接下来的3个字节是机器标识码
- 紧接的两个字节由进程id组成（PID）
- 最后三个字节是随机数。

创建新ObjectId：

```javascript
>newObjectId = ObjectId()

//返回唯一生成的id
>myObjectId = ObjectId("5349b4ddd2781d08c09890f4")
```

创建文档时间戳：

```javascript
//getTimestamp 函数来获取文档的创建时间
>ObjectId("5349b4ddd2781d08c09890f4").getTimestamp()
//返回 ISO 格式的文档创建时间
ISODate("2014-04-12T21:49:17Z")
```

转换为字符串：

```javascript
>new ObjectId().str
//返回Guid格式的字符串：
5349b4ddd2781d08c09890f3
```

### 2.9 引用
- 嵌入关系
可以在文档里可以包括其他文档，称为嵌入式关系，比如：

```javascript
{
   "_id":ObjectId("52ffc33cd85242f436000001"),
   "contact": "987654321",
   "dob": "01-01-1991",
   "name": "Tom Benzamin",
   "address": [
      {
         "building": "22 A, Indiana Apt",
         "pincode": 123456,
         "city": "Los Angeles",
         "state": "California"
      },
      {
         "building": "170 A, Acropolis Apt",
         "pincode": 456789,
         "city": "Chicago",
         "state": "Illinois"
      }]
} 

//可以这样查询用户地址
>db.users.findOne({"name":"Tom Benzamin"},{"address":1})
```
这样的缺陷是随着用户和地址不断增加，数据量变大影响读写性能。

- 引用关系
可以将用户文档和地址文档分开，用户文档通过引用地址文档的_id来建立联系：

```javascript
{
   "_id":ObjectId("52ffc33cd85242f436000001"),
   "contact": "987654321",
   "dob": "01-01-1991",
   "name": "Tom Benzamin",
   "address_ids": [
      ObjectId("52ffc4a5d85242602e000000"),
      ObjectId("52ffc4a5d85242602e000001")
   ]
}

//要读取到地址就需要两次查询
>var result = db.users.findOne({"name":"Tom Benzamin"},{"address_ids":1})
>var addresses = db.address.find({"_id":{"$in":result["address_ids"]}})
```
- DBRefs
假如地址文档不是在同一个集合，想要引用其他集合的文档，可使用DBRefs：

```javascript
//形式
{ $ref: 集合名称, $id: 引用的id, $db:数据库名称，可选参数}

//例子
	{
     "_id":ObjectId("53402597d852426020000002"),
     "address": {
     "$ref": "address_home",
     "$id": ObjectId("534009e4d852427820000002"),
     "$db": "runoob"
     },
     "contact": "987654321",
     "dob": "01-01-1991",
     "name": "Tom Benzamin"
  }


//查找
>var user = db.users.findOne({"name":"Tom Benzamin"})
>var dbRef = user.address
//切换到runoob库
>db[dbRef.$ref].findOne({"_id":(dbRef.$id)})
```

## 3.在NodeJs中使用MongoDB
### 安装驱动
需要使用MongoDB官方提供的node mongodb driver包，安装：
```
npm install mongodb

//如果使用ts，还需要Node.js类型定义来使用
npm install -D @types/node
```
安装完成后需要打开mongodb（在终端输入mongod），才能继续下面的连接

### 连接数据库

```javascript
const { MongoClient } = require('mongodb');
const url = 'mongodb://localhost:27017';
const client = new MongoClient(url);

const dbName = 'mydb';

async function main() {
  //使用connect()方法连接
  await client.connect();
  console.log('Connected successfully to server');
  const mydb = client.db(dbName);
  const collection = mydb.collection('colName'); //传入集合名称

  // 一些操作

  return 'done.';
}

main()
  .then(console.log)
  .catch(console.error)
	.finally(()=>{ client.close() })
```

### 操作例子

```javascript
const { MongoClient } = require('mongodb');
const url = 'mongodb://localhost:27017';
const client = new MongoClient(url);


async function main(){
	await client.connect()
	const collection = client.db('mydb').collection('colName');
	
  //插入文档
  const insertResult = await collection.insertMany([{ a: 1 }, { a: 2 }, { a: 3 }]);
	console.log('Inserted documents =>', insertResult);
  
  //更新文档
  const updateResult = await collection.updateOne({ a: 3 }, { $set: { b: 1 } });
	console.log('Updated documents =>', updateResult);
  
  //...
}

main()
  .then(console.log)
  .catch(console.error)
	.finally(()=>{ client.close() })
```