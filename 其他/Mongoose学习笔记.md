# Mongoose学习笔记

参考：
- [Mongoose实战开发-基础篇](https://juejin.cn/post/6844903509356183560)
- [mongoose官网](https://mongoosejs.com/)

mongoose是对在node中使用mongodb的操作封装，可以更方便地在node环境中使用mongodb。

## 1.安装与使用
安装：
```
npm install mongoose
```
注意使用Mongoose前需要先安装MongoDB。安装完成后，要通过终端输入mongod先打开mongodb。

基本用法：
- 连接数据库
- 创建schema
- 生成modal

```javascript
const mongoose = require('mongoose')

async function main(){
  await mongoose.connect('mongodb://数据库地址/数据库名称')
  
  //创建schema
  const kittySchema = new mongoose.Schema({
    name: String
  });
  //schema也可以有方法，注意要在生成model之前定义
  kittySchem.methods.meow = function(){
    console.log(`${this.name} meow meow ~`)
  }
  
  //安装schema来生成模型，会自动生成kittens集合
  //不需手动去创建Collection，当你操作时，如Collection不存在，会自动创建
  const Kitten = mongoose.model('Kitten', kittySchema);
  
  
  //生成model实例
  const mimi = new Kitten({name:"mimi"})
  console.log(mimi.name)
  //保存到数据库，相当于插入文档
  await mini.save()
}



main().catch(err=>console.log(err))
```

## 2.Schema

### 2.1 new Schema( )

每个Schema都映射到一个集合，并定义集合中文档的格式。

```javascript
const mongoose = require('mongoose')
const {Schema} = mongoose

const BookSchema = new Schema({
  title:  String, //或者写作{type: String}，String为简写方式
  author: String,
  body:   String,
  comments: [{ body: String, date: Date }], //值可以其他对象
  date: { type: Date, default: Date.now },  //可以指定多个类型
  //...
})
```

属性的类型会被转换为 定义对应的SchemaType，比如title属性被转换为SchemaType String。
允许的SchemaType有：
- String
- Number
- Date
- Buffer
- Boolean
- Mixed
- ObjectId
- Array
- Decimal128
- Map

### 2.2 subdocuments（子文档）
对于嵌套的类型，比如上面的例子中的comments，可以给它的子元素定义类型，但是它自己无法定义。

可以使用子文档，在一个文档中嵌套另一个文档：

```javascript
const childSchema = new Schema({ name: 'string' });

const parentSchema = new Schema({
  //子文档有两种类型
  //1.子文档数组
  children: [childSchema],
  
  // 单个嵌套子文档
  child: childSchema
});
```

子文档可以有自己的验证（validation）和中间件（middleware），但是无法单独保存（save），父文档保存时会一并保存子文档。

```javascript
//save中间件，调用父文档的 save() 会触发其所有子文档的 save() 中间件， validate() 中间件同理
childSchema.pre('save', function (next) {
  if ('invalid' == this.name) {
    return next(new Error('#sadpanda'));
  }
  next();
});

var parent = new Parent({ children: [{ name: 'invalid' }] });
parent.save(function (err) {
  console.log(err.message) // #sadpanda
});
```

### 2.3 Schema配置选项
new Schema时，第一个参数是格式，第二个参数是配置选项。可以在构造时传入，也可以通过set方法：

```javascript
new Schema({..}, {_id: false, autoIndex: false});

// or

const schema = new Schema({..});
schema.set(option, value);
```

具体选项查看官方文档

### 2.4 实例方法与静态方法
Schema可以定义方法，实例方法：

```javascript
const mongoose = require('mongoose')
const {Schema} = mongoose

const animalSchema = new Schema({ name: String, type: String });
//定义实例方法
animalSchema.methods.findSimilarTypes = function(cb) {
  return mongoose.model('Animal').find({ type: this.type }, cb); //注意不要用箭头函数，因为this
};

//则所有animalSchema的model实例（即每个文档），都有这个方法
const Animal = mongoose.model('Animal', animalSchema);
const dog = new Animal({ type: 'dog' });

dog.findSimilarTypes((err, dogs) => {
  console.log(dogs); // woof
});
```

静态方法：

```javascript
animalSchema.statics.findByName = function(name) {
  return this.find({ name: new RegExp(name, 'i') });
};

//或者直接调用static方法
animalSchema.static('findByBreed', function(breed) { return this.find({ breed }); });

//static方法不用生成model实例，model本身就可以使用
const Animal = mongoose.model('Animal', animalSchema);
let res = Animal.findByName('fido')
```

### 2.5 Schema.prototype.add( )
如果想在定义Schema之后增加其他属性，则使用add( )方法：

```javascript
const mongoose = require('mongoose')
const {Schema} = mongoose

const BookSchema = new Schema({...})
                         
BookSchema.add({
  price: Number                             
})
```

### 2.6 查询助手（query helpers）
作用于query实例，方便自定义拓展**链式查询**，语法为schema.query.xx：

```javascript
animalSchema.query.byName = function(name) {
  return this.where({ name: new RegExp(name, 'i') })
};

const Animal = mongoose.model('Animal', animalSchema);

Animal.find().byName('fido').exec((err, animals) => {
  console.log(animals);
});
```

### 2.7 索引（index）
在mongoose中定义索引，应用启动时，会自动调用createIndex初始化索引。

索引分**字段级别**和**schema级别**，复合索引需要在schema级别定义。

```javascript
const animalSchema = new Schema({
    name: String,
    type: String,
    tags: { type: [String], index: true } //字段级别
  });

animalSchema.index({ name: 1, type: -1 }); // schema级别
```

## 3.SchemaTypes

### 3.1 配置选项
SchemaType可以配置选项（全部类型可用的）：

```javascript
const schema = new Schema({
  test: {
    type: String,
    required: true,	//布尔值或函数 如果值为真，为此属性添加 required 验证器
		default:  ,	//任何值或函数 设置此路径默认值。如果是函数，函数返回值为默认值
		select: , //布尔值 指定 query 的默认 projections
		validate: ,//函数 添加验证器
		get: ,//函数 使用 Object.defineProperty() 定义自定义 getter
		set: ,//函数 使用 Object.defineProperty() 定义自定义 setter
		alias: ,//字符串,为该字段路径定义虚拟值 gets/sets
  }
});
```

如果属性是索引：

```javascript
const schema = new Schema({
  test: {
    type: String,
    index: true,
    unique: true, // 是否为唯一索引
    sparse: true, //布尔值 是否对这个属性创建稀疏索引
  }
});
```

属性是字符串类型：
- lowercase: 布尔值，是否在保存前对此值调用 .toLowerCase()
- uppercase: 布尔值，是否在保存前对此值调用 .toUpperCase()
- trim: 布尔值，是否在保存前对此值调用 .trim()
- match: 正则表达式，创建验证器检查这个值是否匹配给定正则表达式
- enum: 数组，创建验证器检查这个值是否包含于给定数组

属性是数字类型：
- min: 数值，创建验证器检查属性是否大于或等于该值
- max: 数值，创建验证器检查属性是否小于或等于该值

属性是日期类型：
- min: Date
- max: Date

### 3.2 注意事项
- String、Number、Boolean类型，会自动将属性值转换为相应类型

```javascript
const schema1 = new Schema({ name: String }); //或者"String"
const Person = mongoose.model('Person', schema1);
new Person({ name: 42 }).name; // "42" as a string
new Person({ name: { toString: () => 42 } }).name; // "42" as a string


const schema2 = new Schema({ age: Number }); //或者"Number"
const Car = mongoose.model('Car', schema2);
new Car({ age: '15' }).age; // 15 as a Number
new Car({ age: true }).age; // 1 as a Number
new Car({ age: false }).age; // 0 as a Number
new Car({ age: { valueOf: () => 83 } }).age; // 83 as a Number

//下面会视为true
true
'true'
1
'1'
'yes'

//下面会视为false
false
'false'
0
'0'
'no'
```

- Date类型，使用内建的Date方法（如setMonth（））后使用save保存，mongoose是不会识别到变化的，因此保存不到数据库。如果你一定要用内建 Date 方法， 请手动调用 doc.markModified('pathToYourDate') 告诉 mongoose 你修改了数据：

```javascript
const Assignment = mongoose.model('Assignment', { dueDate: Date });
Assignment.findOne(function (err, doc) {
  doc.dueDate.setMonth(3);
  doc.save(callback); // THIS DOES NOT SAVE YOUR CHANGE

  doc.markModified('dueDate');
  doc.save(callback); // works
})
```

### 3.3 schema.path( )
返回给定字段的SchemaType实例：

```javascript
const sampleSchema = new Schema({ name: { type: String, required: true } });
console.log(sampleSchema.path('name'));
// Output looks like:
/**
 * SchemaString {
 *   enumValues: [],
  *   regExp: null,
  *   path: 'name',
  *   instance: 'String',
  *   validators: ...
  */
```

## 4.Connection
mongoose.connect( )可以接受一个回调，或返回一个promise：

```javascript
mongoose.connect(uri, options, function(error) {
  //只有error一个参数
});

// 或者使用promise
mongoose.connect(uri, options).then(() => { }, err => { });
```

也可以在async函数里await mongoose.connect():

```javascript
async function main(){
  await mongoose.connect('mongodb://数据库地址/数据库名称'，options)
  //...其他操作
}
main().catch(err => {})
```

连接的配置选项可查看官方文档

## 5.Model
Model是Schema编译来的构造函数，model的实例即文档。从数据库创建和读取 document 的所有操作都是通过 model 进行的。

编译mongoose.model时，会自动创建collection：

```javascript
const schema = new mongoose.Schema({ name: 'string', size: 'string' });
const Tank = mongoose.model('Tank', schema);

//collection名称为tanks
```

##### 添加文档的方法：
- create（）：MyModel.create(docs) 等同于 new MyModel(doc).save()
- save（）

##### 查询文档常用方法：
- find
- findById
- findOne
- where

如：
```javascript
Tank.find({ size: 'small' }).where('createdDate').gt(oneYearAgo).exec(callback);
```

##### 删除文档：
- deletOne( filter, [options], [callback])
- deletMany( filter, [options], [callback])

##### 更新文档：
model的 **update** 系列方法不会返回结果，如果想返回结果，可以使用 **findOneAndUpdate** 系列方法。
- model.update()
- model.updateOne()
- model.updateMany()
- model.findOneAndUpdate()
- model.findByIdAndUpdate()

##### 聚合：
Model.aggregate( )，参数：
- [pipeline]：可选，聚合管道数组
- [options]
- [callback]

```javascript
const res = await Users.aggregate([
  { $group: { _id: null, maxBalance: { $max: '$balance' }}},
  { $project: { _id: 0, maxBalance: 1 }}
]);

console.log(res); // [ { maxBalance: 98000 } ]

//或者使用管道聚合构造器
const res = await Users.aggregate().
  group({ _id: null, maxBalance: { $max: '$balance' } }).
  project('-id maxBalance').
  exec();
console.log(res); // [ { maxBalance: 98 } ]
```

## 6.document
##### 修改document
修改document，可以直接赋值修改也可以用set()方法：

```javascript
//Tanl为model
//tank为查找到的文档
Tank.findById(id, function (err, tank) {
  if (err) return handleError(err);

  tank.set({ size: 'large' }); //等同于 tank.size = 'large'
  tank.save(function (err, updatedTank) {
    if (err) return handleError(err);
    res.send(updatedTank);
  });
});
```
##### 保存document
使用save(options， cb(err, res) )方法

##### 更新document
- document.prototype.update( doc, options, cb )
- document.prototype.updateOne( doc, options, cb )

## 7.查询（query）

#### 查询方式
Model中很多方法都可以查询文档，包含查询条件的查询方法（如find、findById、update等）都可以按两种方式执行：
- 传入cb，操作执行后会将错误信息err、结果result传给cb
- 不传入cb，返回一个query对象实例，这个 query 提供了构建查询器的特殊接口，有then函数，用法类似promise

第一种方式例子：

```javascript
const Person = mongoose.model('Person', yourSchema);


// 只返回name、 occupation字段
Person.findOne({ 'name.last': 'Ghost' }, 'name occupation', function (err, person) {
  if (err) return handleError(err);

  console.log('%s %s is a %s.', person.name.first, person.name.last,
    person.occupation);
});
```

第二种方式例子：需要自己调用exec()执行查询:

```javascript
// 查询每个 last name 是 'Ghost' 的 person
const query = Person.findOne({ 'name.last': 'Ghost' });

// 选择返回 `name` 和 `occupation` 字段
query.select('name occupation');

// 然后执行查询
query.exec(function (err, person) {
  if (err) return handleError(err);

  console.log('%s %s is a %s.', person.name.first, person.name.last,
    person.occupation);
});
```

query实例可以链式操作：

```javascript
// With a JSON doc
Person.
  find({
    occupation: /host/,
    'name.last': 'Ghost',
    age: { $gt: 17, $lt: 66 },
    likes: { $in: ['vaporizing', 'talking'] }
  }).
  limit(10).
  sort({ occupation: -1 }).
  select({ name: 1, occupation: 1 }).
  exec(callback);

// Using query builder
Person.
  find({ occupation: /host/ }).
  where('name.last').equals('Ghost').
  where('age').gt(17).lt(66).
  where('likes').in(['vaporizing', 'talking']).
  limit(10).
  sort('-occupation').
  select('name occupation').
  exec(callback);
```

#### query常用方法
- where（path）：指定path为后续链接操作的对象

```javascript
//User为model
User.where('age').gte(21).lte(65).exec(callback);

//等同于
User.find({age: {$gte: 21, $lte: 65}}, callback);

//find返回的也是query对象，所以可以链式操作
User.find().where({ name: 'vonderful' })
```

- exec（可选的操作名称，可选回调）

```javascript
query.exec()
query.exec(callback);
query.exec('find', callback);
```

- all（[path]，val）：即$all查询

```javascript
MyModel.find().where('pets').all(['dog', 'cat', 'ferret']);
// 等同于:
MyModel.find().all('pets', ['dog', 'cat', 'ferret']);
```

- 类似的还有 
  - gt（[path]，val）
  - gte（[path]，val）
  - lt（[path]，val）
  - lte（[path]，val）、
  - in（[path]，val）
  - nin（[path]，val）
  - ne（[path]，val）等

- 删除操作：deleteMany（）、deleteOne（）【注意remove操作被弃用】

```javascript
//参数为filter、options、callabck
const res = await Character.deleteMany({ name: /Stark/, age: { $gte: 18 } });
//res有3个属性：
{
  ok: 1,//如果没有错误
  deletecount: ,//被删除的个数
  n: ,//等于deletecount
}

// 使用回调:
Character.deleteMany({name: /Stark/, age: { $gte: 18 }}, callback);
```

- 查找操作：
  - find（[filter], [callback]）
  - findOne（[filter], [projection], [options], [callback]）：projection为想要返回的字段
  - findOneAndDelete（）
  - findOneAndRemove（）
  - findOneAndReplace（）
  - findOneAndUpdate（）
- 更新操作：
  - update（）
  - updateOne（）
  - updateMany（）


- limit( val )
- post( fn )：增加post中间件

```javascript
const q1 = Question.find({ answer: 42 });
q1.post(function middleware() {
  console.log(this.getFilter());
});
await q1.exec(); // Prints "{ answer: 42 }"

// Doesn't print anything, because `middleware()` is only
// registered on `q1`.
await Question.find({ answer: 42 });
```

- pre（fn）：增加pre中间件

```javascript
const q1 = Question.find({ answer: 42 });
q1.pre(function middleware() {
  console.log(this.getFilter());
});
await q1.exec(); // Prints "{ answer: 42 }"

// Doesn't print anything, because `middleware()` is only
// registered on `q1`.
await Question.find({ answer: 42 });
```

- projection（）：获取/设置返回的文档结构

```javascript
const q = Model.find();
q.projection(); // null

q.select('a b');
q.projection(); // { a: 1, b: 1 }

q.projection({ c: 1 });
q.projection(); // { c: 1 }

q.projection(null);
q.projection(); // null
```

- select（）：选择想要返回的字段

```javascript
query.select('a b');
// 等同于
query.select(['a', 'b']);
query.select({ a: 1, b: 1 });

// 排除c、d字段
query.select('-c -d');
```

- size（[path], val）
- skip( val )
- sort( )

```javascript
// sort by "field" ascending and "test" descending
query.sort({ field: 'asc', test: -1 });

// equivalent
query.sort('field -test');
```