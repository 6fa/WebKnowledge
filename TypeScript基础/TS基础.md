# TypeScript基础

## TypeScript安装与使用
__安装TypeScript:__

```
//全局安装
npm install -g typescript

//项目中安装
npm install -D typescript
```

__使用TypeScript:__
新建文件demo1.ts

```
function test(){
  let str: string = "learn TypeScript"
  console.log(str)
}
test();
```

在终端运行tsc demo1.ts，将ts文件转换成js文件demo1.js(因为node不能直接运行ts文件). <br>
终端运行node demo1.ts，即可看到运行结果

`"learn TypeScript"`

__使用ts-node:__
每次都要先用tsc转换成js文件比较麻烦，可以安装ts-node插件: 

```
//全局安装
npm install -g ts-node

//在项目中安装
npm install -D ts-node

//还需安装一个运行库
npm install -D tslib @types/node
```

这样可以直接运行ts文件：ts-node demo.js


## 类型注解
类型注解用来约束变量或函数的类型，注解方式为变量后跟冒号和TS类型。

如将num规定为数字类型：

```
let num: number;

//如果将num赋值为字符串，则ts报错
num = '123'

//错误信息
//error TS2322: Type 'string' is not assignable to type 'number'.
```

## 基础类型
TypeScript的类型和JavaScript类型有很多相似的，比如boolean、number、string等类型，此外还有其他独特的类型如枚举类型。

#### 3.1布尔、数字、字符串
```
//布尔类型
let flag: boolean = true;

//数字类型
let scores: number = 10;

//字符串
let name: string = 'John'
```

#### 3.2数组、元组
有两种方式定义数组类型：

```
//在元素类型后接上[]
//表示数组内的元素都是某类型
let list1: number[] = [1,2,3]
let list2: string[] = ["1","2","3"]

//使用数组泛型 Array<元素类型>
let list3: Array<number> = [1,2,3]
let list2: Array<string> = ["1","2","3"]
```

元组类型指数组的元素的类型可以不同，比如数组的元素可以同时有string和number类型，但是对应位置的类型必须相同：

```
let arr: [string, number];
arr = ['1', 1] 
arr = [1, '1'] //会报错，对应位置的类型不匹配
```

#### 3.3枚举类型
枚举类型用于为一组数值定义名称，来方便使用这组数值。关键字为enum：

```
//默认元素从0开始编号, 所以这里数值0的名称为0
enum Color {red, green, blue}
//等同于
enum Color {red = 0, green = 1, blue = 2}

//也可以设置为其他值
enum Color {red = 1, green = 3, blue = 5}

//只设置起始值，后面的编号默认依次加1
//如red=2，则green为3，blue为4
enum Color {red = 2, green, blue}
```

枚举类型的使用：

```
//由枚举的数值取得对应的名称
enum Color {red, green, blue}
let color = Color[0]; 
console.log(color); //1

//由名称取得对应的数值
let colorNum = Color[2];
console.log(colorNum); //"blue"
```

#### 3.4Any 类型
any 表示任意类型，用于给还不确定类型的变量指定一个类型，以跳过编译阶段的类型检查。

any类型的适用情况：
  - 变量值动态变化时，比如来自用户输入
  ```
  let x: any = 10;
  x = '10'; //正常编译，ts不会报错
  ```
  - 改写现有代码时，可选择地包含或者移除类型检查
  - 仅知道一部分数据类型时，或者定义包含各种类型元素的数组时
  ```
  let x: any[] = [1,"2",true]
  x[0] = false; //正常编译
  ```

#### 3.5Void类型
void类型表示没有类型，常用于没有返回值的函数中，表示函数没有返回值，如果之后加入返回值则报错：

```
//在函数名后加类型，表示函数返回值的类型
function sayHello(): void(){
  console.log("hello");
  return "hello";
}
//报错：Type 'string' is not assignable to type 'void'
```

#### 3.6Null 和 Undefined 类型
null和undefiend类型的值只能是它们本身：

```
let u: undefined = undefined;
let n: null = null;
```

但是null和undefined是其他类型（包括void）的子类型，所以可以把它们赋值给其他任意类型：

```
//注意是没有启用严格空校验（--strictNullChecks标记）的情况下
let x: number = 10;
x = null; 
//运行正确



//指定了--strictNullChecks标记，null和undefined只能赋值给void和它们各自
let y: number = 10;
y = null; 
//报错：Type 'null' is not assignable to type 'number'
```

#### 3.7Never类型
never类型是其他类型（包括undefined和null）的子类型，表示从不会出现值的类型。由于没有类型是never类型的子类型，所以never类型的值只能是never本身，any也不可以赋值给never。

出现never类型的场景：

  - 函数无法执行到终点，如无限循环，则函数返回值为never类型
  ```
  function loop():never{
    while(true){}
  }
  ```
  - 函数抛出异常
  ```
  function error():never{
    throw new Error("errMessage")
  }
  ```

#### 3.8对象类型
js中数据类型可分为基本类型和引用类型，基本类型为boolean、number、string等，引用类型即object，而数组、函数等为object的子类型。

ts中对象类型的形式可以是：

  - 普通对象类型
  - 数组类型
  - 类类型
  - 函数类型

  ```
  //将函数的参数设置为object类型
  function createPerson(obj: object){}

  let p1 = createPerson({
    name: "John",
    age: 18
  })

  //传入非对象ts将报错
  let p2 = createPerson("John")
  //Argument of type 'string' is not assignable to parameter of type 'object'

  //传入数组、函数、类也不会报错
  let p3 = createPerson([])
  let p4 = createPerson(()=>{})
  class Person {}
  let p5 = createPerson(Person)
  ```

但是上面这种直接指定为object类型用处不大，可以分别指定对象里面的属性类型：

```
function createPerson(obj: {name: string, age: number}){}

let p1 = createPerson({
  name: "John",
  age: 18
})
```

为了方便阅读，一般将它提取出来，这时候就用到了ts中的接口（interface）。