# this

参考：
1. [《现代Javascript教程》](https://zh.javascript.info/)
2. 《你不知道的javascript 上卷》

### 为什么要用this
this提供了一种更优雅的方式来隐式‘传递’一个对象引用，因此可以将API设计得更加简洁并且容易复用。
例子：

```javascript
function identify(){
  return this.name.toUpperCase();
}
function speak(){
  var greeting = 'hello,i'm '+identify.call(this);
  console.log(greeting)
}

var me = {name:'kyle'}
identify.call(me);//KYLE
speak.call(me);//hello,i'm kyle
```

如果不使用this，就要显式地传入一个上下文对象：

```javascript
function identify(context){
  return context.name.toUpperCase()
}
function speak(context){
  var greeting = 'hello,i'm '+identify.call(context);
  console.log(greeting)
}

var me = {name:'kyle'}
identify(me);//KYLE
speak(me);//hello,i'm kyle
```

### this并非指向自身
```javascript
function foo(num){
  console.log('foo:',num)
  this.count++;
  console.log('this',this) //------------------------> window
  console.log('this.count',this.count) //------------------> NaN,因为count为undefined,不能++
}
foo.count = 0;
var i;
for(i = 0;i<10;i++){
  if(i>5){
    foo(i)
  }
}
//foo:6
//foo:7
//foo:8
//foo:9

//foo被调用多少次？
console.log(foo.count)
//-------------------> 0
//可以证明this不是指向自身
```

如果需要从函数内部引用它自身，使用this是不够的，一般需要通过一个指向函数对象的词法标识符（变量）来引用它。
```javascript
//方法1：------------具名函数
function foo(){
  foo.count = 4; //foo指向自身
}
//如果是匿名函数，可以用arguments.callee来引用当前正在运行的函数对象（已经弃用


//方法2：-------------强制this指向foo函数对象
function foo(num){
  console.log('foo:',num)
  this.count++;
}
foo.count = 0;
var i;
for(i = 0;i<10;i++){
  if(i>5){
    foo.call(foo,i);
  }
}
//foo被调用多少次？
console.log(foo.count)//4
```

### this也不指向函数的词法作用域
```javascript
function foo(){
  var a = 2;
  this.bar(); //引用bar最自然的方式是去掉this, bar里的this为window
}
function bar(){
  console.log(this);
  console.log(this.a);
}
foo();    //ReferenceError: a is not defined
          //foo在这里调用，则this为window
```

因为!!!!!!!!!!!!!!!!!!!!!

```javacript
function foo(){
  console.log(this);    
}
foo(); //-------------------------> window
```

### this是什么
this是运行时绑定的，它的上下文取决于函数调用的各种条件。this的绑定和函数声明的位置没有任何关系，只取决于函数的调用方式。

当一个函数被调用，会创建一个活动记录（执行上下文），这个记录会包含函数在哪里被调用（调用栈）、函数的调用方式、传入的参数等信息。this就是这个记录的一个属性，会在执行过程中用到。

### 绑定规则
1. 默认绑定
直接使用不带任何修饰的函数引用进行调用，则应用默认规则。

非严格模式下，this绑定到全局。

严格模式下，注意是函数本身内部使用严格模式，this为undefiend。
```javascript
function foo(){
  'use strict';
  console.log(this.a);
}
var a = 2;
foo(); //----------------->TypeError:this is undefined
```

另外，在严格模式<em>调用</em>那个函数则无影响：
```javascript
function foo(){
  console.log(this.a);
}
var a = 2;
(function(){
  'use strict';
  foo(); //---------------->2 this还是绑定到了全局
})()
```

2. 隐式绑定
调用位置有上下文对象，则this绑定到那个上下文对象。
```javascript
function foo(){
  console.log(this.a);
}
var obj = {
  a:2,
  foo:foo
}
obj.foo();//-------------->2
```

注意1，对象属性引用链中只有最后一层起作用：
```javascript
function foo(){
  console.log(this.a);
}
var obj2 = {
  a:2,
  foo:foo
}
var obj1 = {
  a:1,
  obj2:obj2
}
obj1.obj2.foo();//-------------->2  只有obj2起作用
```

注意2, 隐式丢失：被隐式绑定的函数可能会丢失绑定对象，从而把this绑定到全局对象或者undefined上（取决于严格模式）

  情况1：使用函数别名var bar = obj.foo
  ```javascript
  function foo(){
    console.log(this.a);
  }
  var obj = {
    a:2,
    foo:foo
  }
  var a = 100;
  var bar = obj.foo; //--------------------------> 注意这行,引用的是foo本身
  bar(); //------------------->100
  ```

  情况2：传参doFoo(obj.foo)
  ```javascript
  function foo(){
    console.log(this.a);
  }
  function doFoo(fn){
    fn();
  }
  var obj = {
    a:2,
    foo:foo
  }
  var a = 'oops,global'
  doFoo(obj.foo); //------------->实际上是 令fn = obj.foo，与情况一类似
  ```

  情况3：使用回调函数
  ```javascript
  function foo(){
    console.log(this.a);
  }
  var obj = {
    a:2,
    foo:foo
  }
  var a = 'oops,global'
  setTimeout(obj.foo,1000);//-------------->'oops,global'

  //------------------原因
  //回调函数和下面代码类似：
  function setTimeout(fn,delay){  //------------>fn = obj.foo
    //等待delay秒
    fn();
  }
  ```

3. 显示绑定
使用call(...)、apply(...)、bind(...)、API调用的上下文。

如果在call、apply里传入一个原始值（字符串类型、布尔类型、数字类型）来当作this的绑定对象，这个原始值会被转换成它的对象形式（new String()、new Boolean()、new Numebr()）,称为装箱。

var bar = foo.bind(obj):   bind(..)是内置方法Function.prototype.bind()，会返回一个硬编码的新函数，把参数设置为this的上下文并调用原始函数。

API调用的‘上下文’：即内置函数里面的可选参数，如[1,2,3].forEach(foo,obj), 调用foo时把this绑定到obj

//-----

装箱：

定义：把基本数据类型转换为对应的引用类型的操作称为装箱。

//-----

拆箱:

定义：把引用类型转换为基本的数据类型称为拆箱。

4. new绑定
使用new操作符时，会执行以下操作：

1.创建（或者说构造）一个全新的对象；

2.这个新对象会被执行[[prototype]]连接；

3.这个新对象会绑定到函数调用的this；

4.如果函数没有返回其他对象，那么new表达式中的函数调用会自动返回这个新对象。

### 绑定的优先级
高 -----> 低

new  -----> call/apply/bind --------------> 上下文 ------------> 默认

### 一些例外
1. 将null或者undefined传入call、apply、bind
这些值调用时会被忽略，实际应用默认规则。

什么时候会传入null？bind对参数进行柯里化时（预先设置一些参数）、用apply展开数组。此时null的作用相当于占位符
```javascript
function foo(a,b){
  console.log('a:'+a+',b:'+b);
}
//展开数组
foo.apply(null,[2,3]); //a:2,b:3

//柯里化
var bar = foo.bind(null,2);
bar(3);
//a:2,b:3
```

但这种方法会有副作用（如果函数里面有this，容易修改全局对象），更安全的方法是创建一个空的委托对象。

let dmz = Object.create(null); 

//和{}很像，但没有Object.pototype这个委托，所以比{}更空；this会被限制在这个空对象

foo.apply(dmz,[2,3])

2. 间接引用
```javascript
function foo(){
  console.log(this.a)
}
var a = 2;
var o = {a:3,foo:foo}
var p = {a:4}

o.foo(); //--------------->3
(p.foo = o.foo)(); //------------->2
//因为：
console.log(p.foo = o.foo) //-----------> function foo()
```

3. 箭头函数
箭头函数不使用this的四种标准规则，而是根据外层（函数或全局）作用域来决定this。

箭头函数的绑定无法修改（new也不行）

常用于回调函数中，例如事件处理器或者定时器：
```javascript
function foo(){
  let a = 1;
  setTimeout(()=>{
    console.log(this.a) //------------> 这里的this在词法上继承自foo
  },1000)
}
```