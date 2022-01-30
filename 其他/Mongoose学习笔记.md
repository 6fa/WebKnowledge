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