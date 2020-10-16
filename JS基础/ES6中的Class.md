# ES6中的class类

### 0.参考
1. [现代JavaScript教程](https://zh.javascript.info/)
2. 《你不知道的JavaScript》上卷
3. 《你不知道的JavaScript》下卷

### 1.什么是class
在面向对象的编程中，class 是用于创建对象的可扩展的程序代码模版，它为对象提供了状态（成员变量）的初始值和行为（成员函数或方法）的实现。

传统的面向类的语言中，父类和子类、子类和实例之间其实是复制操作。但是JS中没有复制，而是通过原型链来实现class。

在 JavaScript 中，类是一种函数。用 typeof  验证为 function。
声明class 时所做的事：

```javascript
Class User {
  constructor(name) { this.name = name; }
  sayHi() { alert(this.name); }
}
// class 是函数 
alert(typeof User); // function

// ...或者，更确切地说，是构造器方法
alert(User === User.prototype.constructor); // true

// 方法在 User.prototype 中，例如：
alert(User.prototype.sayHi); // alert(this.name);

// 在原型中实际上有两个方法
alert(Object.getOwnPropertyNames(User.prototype)); // constructor, sayH
```

第一步：

创建一个名为 User 的函数，该函数成为类声明的结果。该函数的代码来自于 constructor 方法（如果我们不编写这种方法，那么它就被假定为空）。

第二步：

存储类中的方法，例如sayHi储存到User.prototype 中。

<em>和function Foo() 的区别：</em>

1)class必须通过new 调用，function Foo可以用Foo.call(obj)

通过 class 创建的函数具有特殊的内部属性标记 [[FunctionKind]]:"classConstructor"。因此，它与手动创建并不完全相同。不像普通函数，调用类构造器时必须要用 new 关键词

2) function foo是可以提升的，class不行

3）全局作用域中的class Foo创建了这个作用域的一个词法标识符Foo，但是和function foo不一样，并没有创建一个同名的全局对象属性

4) 类方法不可枚举。 类定义将 "prototype" 中的所有方法的 enumerable 标志设置为 false。

5) 类总是使用 use strict。 在类构造中的所有代码都将自动进入严格模式。


### 2.class基本语法
class可以声明形式，也可以是表达式形式：let User = class {  }

或者：
```javascript
// “命名类表达式（Named Class Expression）”
// (规范中没有这样的术语，但是它和命名函数表达式类似)
let User = class MyClass {
  sayHi() {
    alert(MyClass); // MyClass 这个名字仅在类内部可见
  }
};

new User().sayHi(); // 正常运行，显示 MyClass 中定义的内容

alert(MyClass); // error，MyClass 在外部不可见
```


class字段：
之前，类仅具有方法。“类字段”是一种允许添加任何属性的语法。

例如，让我们在 class User 中添加一个 name 属性：
```javascript
class User {
  name = "Anonymous";

  sayHi() {
    alert(`Hello, ${this.name}!`);
  }
}

new User().sayHi();// Hello, Anonymous

alert(User.prototype.sayHi); // 被放在 User.prototype 中
alert(User.prototype.name); // undefined，没有被放在 User.prototype 中
```

class字段是很有用的，可以使用class字段解决this丢失问题。

this丢失问题：
```javascript
class Button {
  constructor(value) {
    this.value = value;
  }

  click() {
    alert(this.value);
  }
}

let button = new Button("hello");

setTimeout(button.click, 1000); // undefined
```

有两种可以修复它的方式：

1. 传递一个包装函数，例如 setTimeout(() => button.click(), 1000)。
2. 将方法绑定到对象，例如在 constructor 中：
```javascript
class Button {
  constructor(value) {
    this.value = value;
    this.click = this.click.bind(this);
  }

  click() {
    alert(this.value);
  }
}

let button = new Button("hello");

setTimeout(button.click, 1000); // hello
```

3. 类字段为后一种解决方案提供了更优雅的语法：
```javascript
class Button {
  constructor(value) {
    this.value = value;
  }
  click = () => {           //------->注意这个click不在Button的prototype中
    alert(this.value);
  }
}

let button = new Button("hello");

setTimeout(button.click, 1000); // hello

console.log(button)
//-------把结果打印出来，可以看到有click方法
```

类字段 click = () => {...} 在每个 Button 对象上创建一个独立的函数，并将 this 绑定到该对象上。然后，我们可以将 button.click 传递到任何地方，并且它会被以正确的 this 进行调用。

基本语法总结：
```javascript
class MyClass {
  prop = value; // 属性

  constructor(...) { // 构造器
    // ...
  }

  method(...) {} // method

  get something(...) {} // getter 方法
  set something(...) {} // setter 方法

  [Symbol.iterator]() {} // 有计算名称（computed name）的方法（此处为 symbol）
  // ...
}
```

技术上来说，MyClass 是一个函数（我们提供作为 constructor 的那个），而 methods、getters 和 settors 都被写入了 MyClass.prototype。


### 3.使用new调用时
```javascript
class Animal {
  constructor(name) {
    this.speed = 0;
    this.name = name;
  }
  run(speed) {
    this.speed = speed;
    alert(`${this.name} runs with speed ${this.speed}.`);
  }
  stop() {
    this.speed = 0;
    alert(`${this.name} stands still.`);
  }
}

let animal = new Animal("My animal");
```

animal的[[prototype]]是Animal的prototype

Animal的prototype里面有constructor(指向自身)、run函数、stop函数


### 4.类继承 extends/super
##### 4.1 类的继承
接上一个例子：
```javascript
class Rabbit extends Animal {
  hide() {
    alert(`${this.name} hides!`);
  }
}

let rabbit = new Rabbit("White Rabbit");

rabbit.run(5); // White Rabbit runs with speed 5.
rabbit.hide(); // White Rabbit hides!
```
在内部，关键字 extends 使用了很好的旧的原型机制进行工作。它将 Rabbit.prototype.[[Prototype]] 设置为 Animal.prototype。所以，如果在 Rabbit.prototype 中找不到一个方法，JavaScript 就会从 Animal.prototype 中获取该方法。

##### 4.2在 extends 后允许任意表达式。

类语法不仅允许指定一个类，在 extends 后可以指定任意表达式。例如，一个生成父类的函数调用：
```javascript
function f(phrase) {
  return class {
    sayHi() { alert(phrase) }
  }
}
class User extends f("Hello") {}
new User().sayHi(); // Hello
```

##### 4.3子类重写父类方法
子类可以在父类方法的基础上进行调整或扩展其功能。

Class 为此提供了 "super" 关键字。
- 执行 super.method(...) 来调用一个父类方法。
- 执行 super(...) 来调用一个父类 constructor（只能在我们的 constructor 中）

来看个例子：
```javascript
class Foo {
  constructor(a,b){
    this.x = a;
    this.y = b;
  }
  givemeXY() {
    return this.x * this.y
  }
}
class Bar extends Foo {
  constructor(a,b,c){
    super(a,b);
    this.z = c
  }
  givemeXYZ(){
    return super.givemeXY() * this.z
  }
}

let test = new Bar(2,3,4)
test.z  //4
test.givemeXY()  //6
test.givemeXYZ() //24
```

即super在givemeXYZ()方法中具体指Foo.prototype,而在Bar的constructor构造器中指的是Foo。

##### 4.3箭头函数没有 super
如果被访问，它会从外部函数获取。例如：
```javascript
class Animal {
  stop(){
    console.log('test')
  }
}
class Rabbit extends Animal {
  stop() {
    console.log(super)
    setTimeout(() => super.stop(), 1000); // 1 秒后调用父类的 stop
  }
}
```
箭头函数中的 super 与 stop() 中的是一样的，所以它能按预期工作。如果我们在这里指定一个“普通”函数，那么将会抛出错误：
```javascript
// 意料之外的 super
setTimeout(function() { super.stop() }, 1000);
```

总结一下箭头函数的特殊地方：
- 箭头函数没有this（不具有 this 自然也就意味着另一个限制：箭头函数不能用作构造器（constructor）。不能用 new 调用它们。）
- 箭头函数没有arguments
- 不能使用 new 进行调用
- 没有 super
这是因为，箭头函数是针对那些没有自己的“上下文”，但在当前上下文中起作用的短代码的


##### 4.4重写constructor
子类不写constructor则将生成下面这样的“空” constructor：
```javascript
class Rabbit extends Animal {
  // 为没有自己的 constructor 的扩展类生成的
  constructor(...args) {
    super(...args);
  }
}
```
它自动调用了父类的 constructor，并传递了所有的参数。如果我们没有写自己的 constructor，就会出现这种情况。

注意，重写父类的 constructor 必须调用 super(...)，并且 (!) 一定要在使用 this 之前调用。

原因：

在 JavaScript 中，继承类的构造函数（所谓的“派生构造器”，英文为 “derived constructor”）与其他函数之间是有区别的。派生构造器具有特殊的内部属性 [[ConstructorKind]]:"derived"。这是一个特殊的内部标签。

该标签会影响它的 new 行为：
- 当通过 new 执行一个常规函数时，它将创建一个空对象，并将这个空对象赋值给 this。
- 但是当继承的 constructor 执行时，它不会执行此操作。它期望父类的 constructor 来完成这项工作。

因此，派生的 constructor 必须调用 super 才能执行其父类（非派生的）的 constructor，否则 this 指向的那个对象将不会被创建。并且我们会收到一个报错。

JavaScript 为函数添加了一个特殊的内部属性：[[HomeObject]]。当一个函数被定义为类或者对象方法时，它的 [[HomeObject]] 属性就成为了该对象。

然后 super 使用它来解析（resolve）父原型和它自己的方法。[[HomeObject]] 不能被更改，所以这个绑定是永久的。在 JavaScript 语言中 [[HomeObject]] 仅被用于 super。所以，如果一个方法不使用 super，那么我们仍然可以视它为自由的并且可在对象之间复制。但是用了 super 再这样做可能就会出错。

[[HomeObject]] 是为类和普通对象中的方法定义的。但是对于对象而言，方法必须确切指定为 method()，而不是 "method: function()"。

在下面的例子中，使用非方法（non-method）语法进行了比较。未设置 [[HomeObject]] 属性，并且继承无效：
```javascript
let animal = {
  eat: function() { // 这里是故意这样写的，而不是 eat() {...
    // ...
  }
};
let rabbit = {
  __proto__: animal,
  eat: function() {
    super.eat();
  }
};
rabbit.eat();  // 错误调用 super（因为这里没有 [[HomeObject]]）
```


### 5.class xx extends Object 与 class xx的区别
所有的对象通常都继承自 Object.prototype，并且可以访问“通用”对象方法。

但是，如果我们像这样 "class Rabbit extends Object" 把它明确地写出来，那么结果会与简单的 "class Rabbit" 有所不同么？
```javascript
class Rabbit extends Object {
  constructor(name) {
    super(); // 需要在继承时调用父类的 constructor
    this.name = name;
  }
}
let rabbit = new Rabbit("Rab");
alert( rabbit.hasOwnProperty('name') ); //------------------------- true
```

“extends” 语法会设置两个原型：
- 在构造函数的 "prototype" 之间设置原型（为了获取实例方法）。
- 在构造函数之间会设置原型（为了获取静态方法）。

```javascript
class Rabbit extends Object {}
alert( Rabbit.prototype.__proto__ === Object.prototype ); // (1) true
alert( Rabbit.__proto__ === Object ); // (2) true
```
现在 Rabbit 可以通过 Rabbit 访问 Object 的静态方法，像这样：

```javascript
class Rabbit extends Object {}
// 通常我们调用 Object.getOwnPropertyNames
alert ( Rabbit.getOwnPropertyNames({a: 1, b: 2})); // a,b
```

但是如果我们没有 extends Object，那么 Rabbit.__proto__ 将不会被设置为 Object。
```javascript
class Rabbit {}
alert( Rabbit.prototype.__proto__ === Object.prototype ); // (1) true

alert( Rabbit.__proto__ === Object ); // (2) false 

alert( Rabbit.__proto__ === Function.prototype ); // true，所有函数都是默认如此

// error，Rabbit 中没有这样的函数
alert ( Rabbit.getOwnPropertyNames({a: 1, b: 2})); // Error
```


### 6.静态属性、静态方法
把一个方法赋值给类的函数本身，而不是赋给它的 "prototype"。这样的方法被称为 静态的（static）。
```javascript
class User {
  static staticMethod() {
    alert(this === User);
  }
}

User.staticMethod(); // true
```

等同于下面：
```javascript
class User { }

User.staticMethod = function() {
  alert(this === User);
};

User.staticMethod(); // true
```

通常，静态方法用于实现属于该类但不属于该类任何特定对象的函数.

继承静态属性、静态方法：<br>
静态属性和方法是可被继承的，比如 class Rabbit extends Animal 、 Rabbit._ proto_ === Animal


### 7.扩展内建类
内建的类，例如 Array，Map 等也都是可以扩展的（extendable）。

例如，这里有一个继承自原生 Array 的类 PowerArray：
```javascript
// 给 PowerArray 新增了一个方法
class PowerArray extends Array {
  isEmpty() {
    return this.length === 0;
  }
}

let arr = new PowerArray(1, 2, 5, 10, 50);
---------------arr instanceof Array   //true
---------------arr instanceof PowerArray // true
alert(arr.isEmpty()); // false

let filteredArr = arr.filter(item => item >= 10);
---------------filteredArr instanceof Array  //true
------------------------------filterdArr instanceof PowerArray // true
alert(filteredArr); // 10, 50
alert(filteredArr.isEmpty()); // false
```

注意到arr.filter返回的也是PowerArray的子类（可以用isEmpty），因为内部使用了对象的 constructor 属性来实现这一功能
```
arr.__proto__.constructor === PowerArray
```
当 arr.filter() 被调用时，它的内部使用的是 arr.constructor (也即arr.__proto__.constructor)来创建新的结果数组，而不是使用原生的 Array


静态getter Symbol.species：

如果当任何父类方法需要构造一个新实例，但不使用子类的构造器本身时，这个功能使得子类可以通知父类应该使用哪个构造器。
```javascript
class PowerArray extends Array {
  isEmpty() {
    return this.length === 0;
  }
  // 内建方法将使用这个作为 constructor
  static get [Symbol.species]() {
    return Array;
  }
}
let arr = new PowerArray(1, 2, 5, 10, 50);

alert(arr.isEmpty()); // false

// filter 使用 arr.constructor[Symbol.species] 作为 constructor 创建新数组
let filteredArr = arr.filter(item => item >= 10);

// filteredArr 不是 PowerArray，而是 Array
alert(filteredArr.isEmpty()); // Error: filteredArr.isEmpty is not a function
```

需要注意：内建类没有静态方法继承。

内建对象有它们自己的静态方法，例如 Object.keys，Array.isArray 等。如我们所知道的，原生的类互相扩展。例如，Array 扩展自 Object。

通常，当一个类扩展另一个类时，静态方法和非静态方法都会被继承。但内建类却是一个例外。它们相互间不继承静态方法。

例如，Array 和 Data 都继承自 Object，所以它们的实例都有来自 Object.prototype 的方法。但 Array.[[Prototype]] 并不指向 Object，所以它们没有例如 Array.keys()（或 Data.keys()）这些静态方法。

与我们所了解的通过 extends 获得的继承相比，<em>这是内建对象之间继承的一个重要区别</em>。


### 8.instanceof
##### 8.1
通常，instanceof 在检查中会将原型链考虑在内。此外，我们还可以在静态方法 Symbol.hasInstance 中设置自定义逻辑。

obj instanceof Class 算法的执行过程大致如下：

1）如果这儿有静态方法 Symbol.hasInstance，那就直接调用这个方法：
```javascript
// 设置 instanceOf 检查
// 并假设具有 canEat 属性的都是 animal
class Animal {
  static [Symbol.hasInstance](obj) {
    if (obj.canEat) return true;
  }
}
let obj = { canEat: true };
alert(obj instanceof Animal); // true：Animal[Symbol.hasInstance](obj) 被调用
```

2）大多数 class 没有 Symbol.hasInstance。在这种情况下，标准的逻辑是：使用 obj instanceOf Class 检查 Class.prototype 是否等于 obj 的原型链中的原型之一。
```javascript
obj.__proto__ === Class.prototype?
obj.__proto__.__proto__ === Class.prototype?
obj.__proto__.__proto__.__proto__ === Class.prototype?
...
// 如果任意一个的答案为 true，则返回 true
// 否则，如果我们已经检查到了原型链的尾端，则返回 false
```
##### 8.2
isPrototypeOf()方法:

objA.isPrototypeOf(objB)，如果 objA 处在 objB 的原型链中，则返回 true。所以，可以将 obj instanceof Class 检查改为 Class.prototype.isPrototypeOf(obj)

##### 8.3
使用 Object.prototype.toString 方法来揭示类型：

```javascript
// 方便起见，将 toString 方法复制到一个变量中
let objectToString = Object.prototype.toString;

// 它是什么类型的？
let arr = [];

alert( objectToString.call(arr) ); // [object Array]
```

##### 8.4
Symbol.toStringTag:

可以使用特殊的对象属性 Symbol.toStringTag 自定义对象的 toString 方法的行为
```javascript
let user = {
  [Symbol.toStringTag]: "User"
};

alert( {}.toString.call(user) ); // [object User]
```

所以，如果我们想要获取内建对象的类型，并希望把该信息以字符串的形式返回，而不只是检查类型的话，我们可以用 {}.toString.call 替代 instanceof。


### 9.Mixin模式
在 JavaScript 中，我们只能继承单个对象。每个对象只能有一个 [[Prototype]]。并且每个类只可以扩展另外一个类。

根据维基百科的定义，mixin 是一个包含可被其他类使用而无需继承的方法的类。换句话说，mixin 提供了实现特定行为的方法，但是我们不单独使用它，而是使用它来将这些行为添加到其他类中。

```javascript
// mixin
let sayHiMixin = {
  sayHi() {
    alert(`Hello ${this.name}`);
  },
  sayBye() {
    alert(`Bye ${this.name}`);
  }
};
// 用法：
class User {
  constructor(name) {
    this.name = name;
  }
}

// 拷贝方法
Object.assign(User.prototype, sayHiMixin);  //--------------主要是这行


// 现在 User 可以打招呼了
new User("Dude").sayHi(); // Hello Dude!
```