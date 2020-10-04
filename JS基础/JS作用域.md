# JS作用域
内容参考：
1. 《你不知道的JavaScript》上卷
2. [现代JavaSCript教程](https://zh.javascript.info/)

### 1.作用域是什么
编程语言有一套设计良好的规则来储存变量，并且之后可以方便地找到这些变量，这套规则称为作用域。

对于var a =  2; 这句声明时，编译器会进行如下处理：
1. 遇到var a，首先编译器询问作用域是否已经有同名变量在同一个作用域集合中。有则忽略，没有则要求作用域 在当前作用域的集合中声明一个新变量，命名为a
2. 接下来编译器为引擎生成运行时所需代码，代码用来处理a = 2这个赋值操作。
3. 引擎运行时也会先问询作用域，当前作用域集合中是否已经有名为a的变量，有则使用，没有则继续查找。如果最终查找不到，会抛出异常。

总结：<br>
编译器：负责语法分析及代码生成等。<br>
引擎：从头到尾负责整个js程序的编译、执行过程。<br>
作用域：负责收集并维护由声明的标识符（变量）组成的一系列查询，并实施一套非常严格的规则，确定当前执行的代码对这些标识符的访问权限。

### 2.LHS 和 RHS
前面第3小步，引擎的查找会分为LHS查询和RHS查询。

LHS可以理解为查找 ‘赋值操作的目标是谁’，即容器，如a = 2（为=2这个赋值操作找到一个目标）。RHS可以理解为查找 ‘谁是赋值操作的源头’，即值，如console.log(a)（找到a的值）。

如果RHS查询在所有嵌套的作用域中都找不到所需变量，引擎会抛出ReferenceError异常。

如果LHS查询无法找到变量，在非严格模式下：全局作用域创建一个具有该名称的变量并返给引擎；在严格模式下：引擎会抛出ReferenceError异常。

如果RHS查找到了变量，但对这个变量的值进行不合理的操作（比如对一个非函数类型的值进行函数调用，或者引用null和undefined类型的值中的属性），引擎抛出TypeError异常

### 3.词法作用域 和 函数作用域 、''块作用域''

##### 词法作用域
就是定义在词法阶段的作用域。

词法作用域由你写代码时将变量和块作用域写在哪时决定的，当词法分析器处理代码时会保持作用域不变（大部分情况。无论函数在哪里被调用，它的词法作用域都只由函数被声明时所处位置决定。


##### 函数作用域
属于这个函数的全部变量都可以在整个函数的范围内使用及复用。（是最常见的作用域单元）。

函数声明和函数表达式之间最重要的区别是他们的名称标识符会绑定在何处。
```javascript
//函数声明：
var a = 2;
function foo(){
  var a = 3;
  console.log(3);   //3
}
foo();
console.log(a) //2


//函数表达式:foo被绑定在函数表达式(function foo(){...})的...中
var a = 2;
(function foo(){
  var a = 3;
  console.log(3);   //3
})()

console.log(a) //2
```


##### ''块作用域''
表面上js没有块作用域，即一般{}不会形成一个作用域。但with关键字、try/catch语句、let/const可以形成具有块作用域功能的代码块。

try/catch的catch分句会创建一个块作用域：
```javascript
try{
  ...
}catch(err){
  console.log(err);    //err只能在catch内部使用
}
```

let关键字可以将变量绑定到所在的任意作用域中（通常是{。。。}内部）,但使用let进行的声明不会在块作用域中提升:
```javascript
if(true){
  console.log(bar);//referenceError
  let bar = 2;
  console.log(bar)
}

console.log(bar);//referenceError
```

let形成作用域的几个优势：垃圾收集、let循环（每次迭代时进行重新绑定）.


### 3.提升
1）var a = 2; 会被看成2个声明：var a 和 a = 2;

第一个定义声明是在编译阶段进行的，第二个赋值声明会被留在原地等待执行阶段。

2）注意：函数声明会被提升，但函数表达式不会被提升。
```javascript
foo(); // -----> TypeError,因为对undefined进行函数调用

var foo = function bar(){
  ...
}
```

结果是 TypeError,因为对undefined进行函数调用


3）即使是具名函数表达式，名称标识符在赋值之前也无法在所在作用域中使用：
```javascript
foo(); //TypeError
bar(); //ReferenceError

var foo = function bar(){...}
```

4）函数优先

函数首先被提升，然后才是变量。
```javascript
foo();
var foo;
function foo(){
  console.log(1)
}
foo = function(){
  console.log(2)
}

//-------------> 会输出1

实际是下面这样：
function foo(){console.log(1)}
foo();
foo = function(){console.log(2)}
//var foo是重复的声明，被忽略了
//即使把两个声明对调，依然输出1
```

注意：尽管重复的var声明会被忽略，但出现在后面的函数声明还是可以覆盖前面的。


### 4.作用域闭包
定义：当函数可以记住并访问所在的词法作用域时，就产生了闭包，即使函数是在当前词法作用域之外执行。

```javascript
function foo(){
  var a = 2;
  function bar(){
    console.log(a)
  }
  return bar;
}
var baz = foo();
baz()
```

foo()执行后，由于闭包，foo内部作用域并没有被销毁，bar()本身在使用这个作用域。

bar()依然持有对所在作用域的引用，而这个引用就叫闭包。

另一个例子：
```javascript
function foo(msg){
  setTimeout(function timer(){
    console.log(msg);
  },1000)
}
foo('hello')
//timer(）具有涵盖wait（...）作用域的闭包
//在定时器、事件监听器、Ajax请求、跨窗口通信、web workers或任何其他的异步（或同步）任务中，
//只要使用了回调函数，实际上就是在使用闭包
```


### 5.循环和闭包
for循环是最常见的闭包：

虽然五个函数是在各个迭代中分别定义的，但是他们都被封闭在一个共享的全局作用域中，实际上只有一个i
```javascript
for(var i = 0; i<=5; i++){ //注意是var 不是let
  setTimeout(function timer(){
    console.log(i)
  },i*1000)
}

//实际会输出五次6，一秒一次，回调函数是在循环后执行的（但i*1000是已经计算了的
```

因此需要更多的闭包作用域。

解决方案1：立即执行函数
```javascript
for(var i = 0; i<=5; i++){ 
  (function(){
    var j = i; //如果作用域是空，那仅仅进行封闭是不够的。要有自己的变量j
     setTimeout(function timer(){
        console.log(j)
     },i*1000)
  })();
}
```

解决方案2：let
```javascript
for(let i = 0; i<=5; i++){ 
  setTimeout(function timer(){
    console.log(i)
  },i*1000)
}

//和下面相同

for(var i = 0; i<=5; i++){ 
  let j = i;
  setTimeout(function timer(){
    console.log(j)
  },i*1000)
}
```

### 6.模块与闭包
模块的两个主要特征：1.为创建内部作用域而调用了一个包装函数；2.包装函数的返回值必须至少包括一个对内部函数的引用，这样就会创建涵盖整个包装函数内部作用域的闭包
```javascript
function coolModule(){
  var something = 'cool';
  function dosomething(){
    console.log(something)
  }
  return {
    dosomething:dosomething
  }
}
var foo = coolModule();
foo.dosomething();//cool
```


### 7.词法作用域与动态作用域
js并不具有动态作用域，只有词法作用域。但是this机制某种程度上很像动态作用域。

主要区别：词法作用域是在写代码时或者说定义时确定，而动态作用域是在运行时确定的。词法作用域关注函数在何处声明，而动态作用域关注函数从何处调用。

```javascript
function foo(){
  console.log(a);
}
function bar(){
  var a = 3;
  foo();
}
var a = 2;
bar();

//js会输出2

//而动态作用域会输出3
```