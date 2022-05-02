# JSON

### JSON概念
- JSON是一种通用的数据格式，很多语言都有解析和序列号JSON的内置功能
- JSON可以直接被解析成可用的js对象，与解析为DOM文档的XML相比，优势更明显

### JSON语法
- JSON支持3种类型的值：简单值（字符串、数字、布尔值、null）、对象、数组。undefined不行
- JSON中没有变量、函数、对象实例的概念，所有记号都是为了表示结构化数据
- 字符串必须使用 双引号
- 对象中的属性名也必须使用 双引号
- 没有分号
- 对象中不允许出现相同的属性（同一层级中）

- JS的JSON对象有两种方法：stringify（）、parse（）

- 不安全的JSON值：undefined、function、symbol、包含循环引用的对象（对象之间相互引用，形成无限循环）。JSON.stringify()遇到undefined、function、symbol时会将其忽略，在数组中则返回null（以保证单元位置不变）

### JSON序列化
JSON.stringify( )接收三个参数：
  - 要序列化的对象（只传第一个参数返回无空格和缩进的字符串）
  - 过滤器（可以是数组、函数，会应用到对象所包含的所有对象）
  - 缩进（可以是数字、字符串）

```javascript
let person = {
  name: "jack",
  age: 24,
  job:undefined
}

JSON.stringify(person)
//"{"name":"jack","age":24}"

JSON.stringify(person, ["name"])	//注意数组中属性是字符串
//"{"name":"jack"}"






let obj= {
  name:"name",
  age:"24",
  friend:{
    name:"name2",
    age:24,
    job:undefined
  }
}


JSON.stringify(obj, (key,val)=>{
  switch(key){
    case "name":
      return "Bob";
    case "job":
      return "writer";
    default:
      return val
  }
})
//"{"name":"Bob","age":"24","friend":{"name":"Bob","age":24,"job":"writer"}}"

//可以看到嵌套的name、age、job也改变了
```

自定义JSON序列化：在要序列化的对象中添加toJSON（）方法:
```javascript
let obj= {
  name:"name",
  age:"24",
  friend:{
    name:"name2",
    age:24,
    job:undefined
  },
  toJSON:function(){
    return "这是测试"
  }
}
let jsonText= JSON.stringify(obj) //	""这是测试""
```

### 解析JSON
- JSON.parse接收两个参数：要解析成js对象的JSON对象、还原函数
- 还原函数常用于将日期字符串转换为Date对象
```javascript
let obj= {
  name:"name",
  age:"24",
  birthday:new Date(1997,10,10)
};

//序列化
let jsonText = JSON.stringify(obj);
//"{"name":"name","age":"24","birthday":"1997-11-09T16:00:00.000Z"}"

//解析
let obj2 = JSON.parse(jsonText, (key,val)=>{
  return key == "birthday" ? new Date(val) : val
});
//{name: "name", age: "24", birthday: Mon Nov 10 1997 00:00:00 GMT+0800 (中国标准时间)}
```