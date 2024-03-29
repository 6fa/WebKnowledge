# TypeScript基础

[1.TypeScript安装与使用](#1)

[2.类型注解](#2)

[3.基础类型](#3)

[4.接口](#4)

[5.类](#5)

[6.函数](#6)

[7.联合类型与类型保护](#7)

[8.泛型](#8)

[9.模块与命名空间](#9)

[10.配置文件](#10)

<span id="1"></span>
## 1.TypeScript安装与使用

__安装TypeScript:__

```
//全局安装
npm install -g typescript

//项目中安装
npm install -D typescript
```

__使用TypeScript:__

新建文件demo1.ts

```TypeScript
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

<span id="2"></span>
## 2.类型注解
类型注解用来约束变量或函数的类型，注解方式为变量后跟冒号和TS类型。

如将num规定为数字类型：

```TypeScript
let num: number;

//如果将num赋值为字符串，则ts报错
num = '123'

//错误信息
//error TS2322: Type 'string' is not assignable to type 'number'.
```

<span id="3"></span>
## 3.基础类型
TypeScript的类型和JavaScript类型有很多相似的，比如boolean、number、string等类型，此外还有其他独特的类型如枚举类型。

#### 3.1布尔、数字、字符串
```TypeScript
//布尔类型
let flag: boolean = true;

//数字类型
let scores: number = 10;

//字符串
let name: string = 'John'
```

#### 3.2数组、元组
有两种方式定义数组类型：

```TypeScript
//在元素类型后接上[]
//表示数组内的元素都是某类型
let list1: number[] = [1,2,3]
let list2: string[] = ["1","2","3"]

//使用数组泛型 Array<元素类型>
let list3: Array<number> = [1,2,3]
let list2: Array<string> = ["1","2","3"]
```

元组类型指数组的元素的类型可以不同，比如数组的元素可以同时有string和number类型，但是对应位置的类型必须相同：

```TypeScript
let arr: [string, number];
arr = ['1', 1] 
arr = [1, '1'] //会报错，对应位置的类型不匹配
```

#### 3.3枚举类型
枚举类型用于为一组数值定义名称，来方便使用这组数值。关键字为enum：

```TypeScript
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

```TypeScript
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

```TypeScript
//在函数名后加类型，表示函数返回值的类型
function sayHello(): void(){
  console.log("hello");
  return "hello";
}
//报错：Type 'string' is not assignable to type 'void'
```

#### 3.6Null 和 Undefined 类型
null和undefiend类型的值只能是它们本身：

```TypeScript
let u: undefined = undefined;
let n: null = null;
```

但是null和undefined是其他类型（包括void）的子类型，所以可以把它们赋值给其他任意类型：

```TypeScript
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
  ```TypeScript
  function loop():never{
    while(true){}
  }
  ```
  - 函数抛出异常
  ```TypeScript
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

  ```TypeScript
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

```TypeScript
function createPerson(obj: {name: string, age: number}){}

let p1 = createPerson({
  name: "John",
  age: 18
})
```

为了方便阅读，一般将它提取出来，这时候就用到了ts中的接口（interface）。

<span id="4"></span>
## 4.接口
#### 4.1接口特性
使用接口重写上面的例子：

```TypeScript
interface person {
  name: string;		//注意不是逗号是分号
  age: number;
}

function createPerson(o: person){}

//如果不符合接口描述则报错
let p1 = createPerson({
  name: "John",
  age: "18", //报错
})
```

接口里面的属性是必须包括的，如果想让属性可选，可以设置“option bags”模式，即在可选属性后面加上？：

```TypeScript
interface person {
  name: string;
  age: number;
  career?: string; //可选
}

function createPerson(o: person){}
  
//报错，属性没有包括age
 let p1 = createPerson({
  name: "John"
})
 
 //没有包括可选属性，正常运行
let p2 = createPerson({
  name: "John",
  age:18
})
```

也可以给属性设置成只读模式：

```TypeScript
interface person {
  name: string;
  age: number;
  readonly gender: string; //只读属性，不能被修改
}
```

如果想添加接口里面没定义的属性，而且绕过类型检查，可以添加一个字符串索引签名：

```TypeScript
interface person {
  name: string;
  age: number;
  [propName: string]: any; //这样person可以有其他任意数量的额外属性，且可以是任意类型
}

function createPerson(o: person){}
let p1 = createPerson({
  name: "John",
  age: 18,
  stature: 180,  //正常运行
  gender: "male"
})
```

接口也可以定义方法：

```TypeScript
interface person {
  name: string;
  age: number;
  sayHi(): string; //表示必须要有sayHi方法，且返回值是字符串
  exercise(time: number); //表示必须要有exercise方法，且参数是number类型
}
```

#### 4.2可索引类型与索引签名
前面提到了索引签名，索引签名是用来描述可索引类型的。
可索引类型是指 可以通过索引取得值 的类型，比如 a[10] 或 ageMap["daniel"]。

例子：
接口StringArray中有一个索引签名，它表示通过number去索引StringArray时，可以取得string类型的返回值

```TypeScript
interface StringArray {
  [index: number]: string;
}

let myArray: StringArray = ["Bob", "Fred"];

//通过number去索引，返回值是string类型
let myStr: string = myArray[0];
```

ts只支持字符串索引和数字索引。可以将索引设置为只读，防止赋值：

```TypeScript
interface ReadonlyStringArray {
    readonly [index: number]: string;
}
let myArray: ReadonlyStringArray = ["Alice", "Bob"];
myArray[2] = "Mallory"; // 报错
```

#### 4.3接口与函数类型
接口除了能描述带属性的普通对象外，也能描述函数和类。

描述函数类型的接口要有一个调用签名，表示函数的参数列表和返回值类型：

```TypeScript
interface person {
  (name: string, age: number): string;
}

let createPerson: person;
createPerson = function(name: string, age: number){
  return `my name is ${name}`
}

//函数参数不一定要和接口定义的一样，只需要同样的位置是匹配的类型
createPerson = function(n: string, a: number){
  return `my name is ${n}`
}

//或者函数不用再指定参数类型，ts会推断出其类型
createPerson = function(name, age){
  return `my name is ${name}`
}
```

#### 4.4接口与类类型
需要用到implements关键字：

```TypeScript
interface Person {
  name: string;
  age: number;
}

class Student implements Person {
  name = "student";
	age = 18;
}
```

#### 4.5接口继承
接口可以相互继承:

```TypeScript
interface Person {
  name: string;
  age: number;
}

interface Student extends Person {
  study(): string;
}

let student: Student = {
  name: "John",
  age:18,
  study:function(){
    return "study TypeScript"
  }
}

//也可以继承多个接口
interface Student extends Person1, Person2{
  //...
}
```

<span id="5"></span>
## 5.类
#### 5.1类字段
ts中的类的使用和js差不多, 但是类里面的变量要声明成类字段, 一个简单的例子:
```TypeScript
class Person {
  greeting: string;  //ts的类字段,即类里面声明的变量
	constructor(name: string){
    this.greeting = `Welcome, ${name}`
  }
}


let person1 = new Person("John")

Person.greeting; //报错,greeting不存在于Person上
person1.greeting; //正常运行
```

如果变量不事先声明,则报错:

```TypeScript
class Person {
	constructor(name: string){
    this.greeting = `Welcome, ${name}` //错误,类型Person上不存在greeting
  }
}
```

声明类字段时顺便赋值,  效果和在constructor里赋值一样:

```TypeScript
class Person {
  name: string = "John"//类字段会绑定this,所以实例会有该属性
}

let p1 = new Person()
console.log(p1.name); //"John"


//和下面一样的效果
class Person {
  name:string; //还是要声明否则报错
  constructor(){
    this.name = "John"
  }
}
let p1 = new Person()
console.log(p1.name); //"John"
```

#### 5.2访问控制修饰符
ts用public、private、protected修饰符来限制对类成员( 类、变量、方法和构造方法 )的访问。

  - public

  在ts里，类的成员默认为public状态，即允许在类内部和外部被访问

  ```TypeScript
  class Person {
    public name: string = "John";  
    sayHi(){
      console.log(`Hi, ${this.name}`) //内部访问
    }
  }

  let person1 = new Person()
  console.log(person1.name); //外部访问
  ```

  - private

  有private修饰符的类成员为私有, 只允许在类的内部被访问, 不允许外部访问. private常用于getter,setter存取器中

  ```TypeScript
  class Person {
    private name: string = "John"; 
    sayHi(){
      console.log(`Hi, ${this.name}`) 
    }
  }

  let person1 = new Person()
  console.log(person1.name); //报错,私有属性不允许外部访问
  person1.sayHi(); //"Hi,John"
  ```
  ```TypeScript
  //private在getter,setter中的使用
  class Person {
    private _age: number;
    constructor(a:number){
      this._age = a
    }

    get age(){
      return this._age - 10
    }
    set age(age:number){
      this._age = age + 10
    }
  }
  ```

  - protected

  protected成员与private类似,  但是protected可以在派生类中被访问. 即protected只能在类中和子类中被访问

  ```TypeScript
  class Person {
    public name: string = "Rose"; 
    private age: number = 18;
    protected gender: string = "female";
  }

  class Girl extends Person {
    public getName(){
      console.log(this.name) 
    }
    public getAge(){
      console.log(this.age)//报错,私有属性只能在Person中被访问
    }
    public getGender(){
      console.log(this.gender) //正常运行
    }
  }
  ```

#### 5.3构造器
构造器的参数属性, 可以定义的同时并初始化一个变量:

```TypeScript
//参数属性必须要在前面加上一个访问限定符
//比如public,private,protected , 或者只读readonly修饰符
class Book {
  type: string = "Art";
	constructor(public title: string){} //等同于this.title = title
}

let book = new Book("艺术与美")
console.log(book.type) //"Art"
console.log(book.title) //"艺术与美"
```

#### 5.4抽象类
抽象类只用作其他类的基类, 不能实例化,  关键字为abstract

```TypeScript
abstract class BaseBook {}
class Art extends BaseBook{}

let book = new baseBook() //错误,不能实例化抽象类
```

抽象类中可以有抽象方法, 抽象方法只能在派生类中被实现:

```TypeScript
abstract class BaseBook {
  abstract getType(): string;
}
class Art extends BaseBook{
  getType():string{
    return "art"
  }
}
```

<span id="6"></span>
## 6.函数
#### 6.1为函数定义类型
函数类型由函数参数和返回值组成

函数声明形式:

```TypeScript
//函数的返回值是number,参数也是number
function add(a:number,b:number): number{
  return a + b
}
```

函数表达式形式:

```TypeScript
let addFn = function (a:number, b:number): number{
  return a + b
}
//完整的类型应该这样写
let addFn: (aVal: number,bVal: number) => {
  number = function(a: number,b: number): number{return a + b}
}
```

如果函数没有返回值则为void类型, 不建议留空:

```TypeScript
function add(a:number, b:number):void{
  a + b
}
```

#### 6.2函数参数为对象时
当函数参数是对象时,定义参数类型:

```TypeScript
//错误写法
function add({x: number, y: number, z: number}): number{
  return	x + y + z
}

//正确写法
function add({x,y,z}: {x: number, y: number, z: number}){
  return x + y + z
}
//或者
function add(obj: {x: number, y: number, z: number}){
  return obj.x + obj.y + obj.z
}
```

#### 6.3函数重载
函数重载指的是根据不同的参数返回不同类型的数据:

```TypeScript
function getData(type){
  if(typeof type === "number"){
    return [1,2,3]
  }else if(typeof type === "string"){
    return {a:"1",b:"2",c:"3"}
  }
}
```

在ts里这样约束函数重载的类型:

```TypeScript
function getDate(x:string):number[];
function getData(x:number):{a:string,b:string,c:string};

function getData(x:any):any{
  if(typeof type === "number"){
    return [1,2,3]
  }else if(typeof type === "string"){
    return {a:"1",b:"2",c:"3"}
  }
}
```

<span id="7"></span>
## 7.联合类型和类型保护
#### 7.1联合类型
联合类型使用管道( | )将变量设置成多种类型, 变量可以根据类型来赋值:

```TypeScript
//变量value可以是number或者string类型
let value: string | number; 

//联合类型作为函数参数
function getInfo(name: string | string[]){}

//接口作为联合类型
interface teacher {
  teach():void
}
interface student {
  study():void
}
function createPerson(person: teacher | student){
  //...
}
```

#### 7.2 类型保护
在使用联合类型时, 需要明确是哪个类型才能使用它的属性/方法,可以使用类型保护.

类型保护有很多方式:
- 类型断言
- in语法
- typeof 语法
- instanceof 语法

##### (a)类型断言
类型断言即通过断言的方式确定传递过来的准确值, 如下面的例子, 根据共同的属性确定类型, 断言 person as teacher 

```TypeScript
interface teacher {
  flag:true;
  teach():void;
}
interface student {
  flag:false;
  study():void
}
function createPerson(person: teacher | student){
  if(person.flag){
    (person as teacher).teach()
  }else {
    (person as student).study()
  }
}
```

##### (b)in语法
```TypeScript
interface teacher {
  teach():void;
}
interface student {
  study():void
}
function createPerson(person: teacher | student){
  if("teach" in person){
    person.teach()
  }else if("study" in person) { //这里可以不用else if 而用else, ts会自动判断
    person.study()
  }
}
```

##### (c)typeof语法
如果函数的两个参数可以是number或string类型,如果直接相加则ts会报错

```TypeScript
function add(a: number | string, b: number | string){
  return a + b //错误
}

//用typeof解决
function add(a: number | string, b: number | string){
  if(typeof a === "string" || typeof b === "string"){
    return `${a}${b}`
  }else {
    return a + b
  }
}
```

##### (d)instanceof语法
instanceof只能用于类上:

```TypeScript
class Teacher{
  teach(){}
}
class Student{
  study(){}
}

function person(x: Teacher | Student){
  if(x instanceof Teacher){
    x.teach()
  }
}
```

<span id="8"></span>
## 8.泛型
#### 8.1泛型的使用
泛型即声明的时候没定义类型，使用的时候再指定类型. 使用方式是用尖括号<>包裹自定义的类型名称,  一般用尖括号包裹T

假如希望函数有两个参数,均可以是字符串或者数字类型, 但是第一个参数是string的话, 第二个参数也应为string, number同理. 这时可以用泛型:

```TypeScript
//不用泛型
function joint(x: number | string, y: number | string){
  return `${x} ${y}`
}

//使用泛型
function joint<T>(x: T, y: T){
  return `${x} ${y}`
}

//第一种使用方式,指定类型
joint< number >(1,2)

//第二种使用方式,利用类型推荐,编译器根据传入的数据类型自动确定T
joint("1", "2")
```

还可以定义多个泛型:

```TypeScript
function joint<T, P>(x: T, y: P){
  return `${x} ${y}`
}
joint<number, string>(1, "2")
```

#### 8.2泛型约束/继承
假如想把泛型约束成某几个类型:

```TypeScript
function joint<T extends string | number>(x: T, y: T){
  return `${x} ${y}`
}
```

或者约束成自定义接口类型:

```TypeScript
interface myType {
  name: string
}
function joint<T extends myType>(x: T, y: T){
  return `${x.name} ${y.name}`
}

joint({name: "a"}, {name: "b"}) //"a b"
```

#### 8.3泛型中数组的使用
```TypeScript
//第一种写法
function add<T>(params: T[]){
  return params.join('-')
}
add<string>(["1","2","3"]) //"1-2-3"

//第二种写法
function add<T>(params: Array<T>){
  return params.join('-')
}
```

#### 8.4类中的泛型
泛型类与泛型函数差不多, 使用<T>跟在类名后:

```TypeScript
class handler<T> {
  constructor(private data: T[]){};

  getData(key: number):T{
    return this.data[key]
  }
}

let handleData = new handler<string>(["a","b","c"])
console.log(handleData.getData(1)) //"b"
```

<span id="9"></span>
## 9.模块与命名空间
#### 9.1 模块
包含顶级import或者export的文件会被当作一个模块, 如果文件顶级不带import或者export,则文件对全局可见, 意味着里面的变量, 函数, 类等等对外可见.

模块在执行之前, 通过模块加载器加载其他模块, 形成模块所有依赖关系.  常见的模块加载器有CommonJs( 服务于Node.js ),  Require.js ( 服务于Web应用 ),  ES6的模块加载系统.

#### 9.2 TS中的模块使用
ts中的模块使用和es6自带的模块系统很类似.

导出：

```TypeScript
//通过export关键字导出
export const name = "TypeScript"
export interface person {}

//默认导出
export default function(){}

//导出重命名
class Person {}
export {Human as Person}

//重新导出
export person from './TSModule.ts'
```

导入：

```TypeScript
import {name} from "./name.ts"

//重命名
import {nameVal as name} from "./name.js"

//将整个模块导入一个变量
import * as data from "./data.ts"
```

#### 9.3 生成模块代码
通过指定模块目标参数, 可以将ts模块代码生成想要的模块系统的代码:

```TypeScript
//'--module' 选项必须是以下之一
//'none', 'commonjs', 'amd', 'system', 'umd', 'es6', 'es2015', 'es2020', 'esnext'

//比如将demo.ts代码生成es6模块代码
tsc --module es6 demo.ts
```

#### 9.4 命名空间
命名空间和模块化思想很像, 都是为了有自己的环境, 不产生全局变量污染环境.

命名空间定义了标识符的可见范围, 同样的标识符在不同的命名空间互不干扰. 使用namespace定义:

```TypeScript
namespace space1 {
  let name = "Jack"
}
namespace space2{
  let name = "Rose"
}
```

如果要调用命名空间里面的类和接口,  需要在命名空间里用export导出:

```TypeScript
namespace space1 {
  export class someClass{}
  export function getName(){
    return "Jack"
  }
}
space1.getName()
```

一般项目中会单独写一个components文件, 进行组件化:

```TypeScript
//src/components.ts
namespace Components {
  export class Header {}

  export class Content {}

  export class Footer {}
}
```

命名空间可以嵌套:

```TypeScript

namespace Components {
  export namespace SubComponents {
    export class Test {}
  }

  //...
}
```

对于多文件中的命名空间, 需要使用三个斜杠引用文件:

```TypeScript
//下面三个文件中的命名空间有相同的名称animal,其实是同一个命名空间,export的东西相同则报错
//space1.ts
namespace animal {
  export function eat(){}
}

//space2.ts
namespace animal {
  export function fly(){}
}

//space3.ts
namespace animal {
  export function sleep(){}
}

//index.ts
//引用其他三个文件
///<reference path = "space1.ts" /> 
///<reference path = "space2.ts" /> 
///<reference path = "space3.ts" /> 
function init(){
  animal.eat();
  animal.fly();
  animal.sleep();
}
```

在index.html里使用index.js之前, 也需要引用其他文件的js:

```TypeScript
<script src="space1.js"></script>
<script src="space2.js"></script>
<script src="space3.js"></script>
 <script src="index.js"></script>
```

可以将它们编译到一个文件(sample.js):

- 使用 tsc --outFile sample.js index.ts
- 或者已有tsconfig.json文件, 将 outFile 配置项设置为 "./sample.js" . 需要注意的是配置了这个选项, 就不再支持"module" : "commonjs", 需要改成 "module" : "amd"

<span id="10"></span>
## 10.配置文件
#### tsconfig.json文件的生成与使用
tsconfig.json配置文件是用来指定项目的根文件和一些编译选项, 其实就是用来规定如何对ts文件进行编译的.

生成tsconfig.json:

```
//在项目根目录下
tsc --init
```

想让tsconfig.json文件生效, 在终端输入tsc xxx.ts是无效的, 它的编译并没有经过tsconfig.json.
这是因为当命令行上指定了输入文件时，tsconfig.json文件会被忽略.  应该直接运行tsc(不带任何输入文件).  