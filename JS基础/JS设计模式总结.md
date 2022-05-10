# JS设计模式

参考：
- [JS设计模式有哪些？有什么区别？](https://zhuanlan.zhihu.com/p/142304509)
- [深入理解JavaScript系列](https://www.cnblogs.com/TomXu/archive/2011/12/15/2288411.html)
- [掘金小册 - JavaScript 设计模式核⼼原理与应⽤实践](https://www.weisuoke.com/fe2020/Gof/juejin-GoF.html)
- [JavaScript设计模式es6（23种)](https://juejin.cn/post/6844904032826294286)
- [JavaScript设计模式与实践--工厂模式](https://juejin.cn/post/6844903653774458888)
- [写给前端的设计模式](https://www.yuque.com/wubinhp/uxiv5i)
- [从ES6重新认识JavaScript设计模式(三): 建造者模式](https://www.jianshu.com/p/be61fcc47a2f)
- [javascript设计模式-生成器模式（Builder）](https://www.cnblogs.com/webFrontDev/p/3471983.html)
- [JavaScript如何实现建造者模式？](https://www.cnblogs.com/yangxianyang/p/13675551.html)
- [JS设计模式七：装饰者模式](http://blog.chinaunix.net/uid-26672038-id-4364155.html)

## 1.设计原则
##### 单一职责原则（SRP）
一个对象或方法只做一件事情。如果一个方法承担了过多的职责，那么在需求的变迁过程中，需要改写这个方法的可能性就越大。应该把对象或方法划分成较小的粒度

##### 最少知识原则（LKP）
应当尽量减少对象之间的交互，如果两个对象之间不必彼此直接通信，那么这两个对象就不要发生直接的联系，可以转交给第三方进行处理

##### 开放-封闭原则（OCP）
类、模块、函数等应该是可以扩展的，但是不可修改。当需要改变一个程序的功能或者给这个程序增加新功能的时候，可以使用增加代码的方式，尽量避免改动程序的源代码，防止影响原系统的稳定

## 2.设计模式分类
##### 创建型设计模式
关注对象的创建方式，减少对象创建过程的复杂性。常见的创建型设计模式：
- 工厂模式
- 抽象工厂模式
- 构造函数模式
- 原型模式
- 单例模式
- 生成器模式（建造者模式）

##### 结构设计模式
关注增强对象功能、对象组成结构、不同对象之间的关系。有助于在系统某一部分发生改变时，不需要改变整个结构。
- 装饰者模式
- 外观模式
- 享元模式
- 适配器模式
- 代理模式

#### 行为设计模式
行为设计模式关注改善或精简在系统中不同对象间通信。
- 迭代器模式
- 中介者模式
- 观察者模式
- 访问者模式
- 发布-订阅模式
- 职责链模式
- 策略模式

## 3.工厂模式与抽象工厂模式
### 3.1工厂模式
工厂模式定义一个创建对象的通用接口，让实现这个接口的类来决定实例化哪个类，主要用来创建同一类对象。
比如根据用户的不同权限渲染展示不同的页面：
```javascript
    //简单工厂模式
    let UserFactory = function (role) {
      function SuperAdmin(){
        this.role = "超级管理员",
        this.view = ["首页","通讯录","发现页","应用数据","权限管理"]
      }
      function Admin(){
        this.role = "管理员",
        this.view = ["首页","通讯录","发现页","应用数据"]
      }
      function NormalUser(){
        this.role = "普通用户",
        this.view = ["首页","通讯录","发现页"]
      }

      switch(role){
        case "SuperAdmin":
          return new SuperAdmin();
          break;
        case "Admin":
          return new Admin();
          break;
        case "NormalUser":
          return new NormalUser();
          break;
        default:
          throw new Error('参数错误')
      }
    }
    
    
    

    //由于上面三个构造函数内部很相似，可以进行优化
    let UserFactory = function (role) {
      function User(option){
        this.role = option.role
        this.view = option.view
      }

      switch(role){
        case "SuperAdmin":
          return new User({role:"超级管理员",view:["首页","通讯录","发现页","应用数据","权限管理"]});
          break;
        case "Admin":
          return new User({role:"管理员",view:["首页","通讯录","发现页","应用数据"]});
          break;
        case "NormalUser":
          return new User({role:"普通用户",view:["首页","通讯录","发现页"]});
          break;
        default:
          throw new Error('参数错误')
      }
    }
```

在函数内包含了所有对象的创建逻辑（构造函数）和判断逻辑的代码，想添加新的构造函数还要修改判断逻辑。而且随着项目复杂度增加，函数会越来越庞大、难以打理。

可以将实际创建对象的工作推迟到子类，即工厂方法模式：
```javascript
    //工厂方法模式：将实际创建对象的工作推迟到子类中。
    //想添加新构造函数时就不用修改判断逻辑,而是直接添加在prototype对象里面
    function factory(role){
      if(this instanceof factory){
        return new this[type]()
      }else {
        return new factory(role)
      }
    }
    factory.prototype = {
      "SuperAdmin": function (){
        this.role = "超级管理员",
        this.view = ["首页","通讯录","发现页","应用数据","权限管理"]
      }
      //...
    }

    //工厂方法模式用es6的class写法
    class User {
      constructor(name = '', viewPage = []) {
        if(new.target === User) {
          throw new Error('抽象类不能实例化!');
        }
        this.name = name;
        this.viewPage = viewPage;
      }
    }

    class UserFactory extends User {
      constructor(name, viewPage) {
        super(name, viewPage)
      }
      create(role) {
        switch (role) {
          case 'superAdmin': 
            return new UserFactory( '超级管理员', ['首页', '通讯录', '发现页', '应用数据', '权限管理'] );
            break;
          case 'admin':
            return new UserFactory( '普通用户', ['首页', '通讯录', '发现页'] );
            break;
          case 'user':
            return new UserFactory( '普通用户', ['首页', '通讯录', '发现页'] );
            break;
          default:
            throw new Error('参数错误')
        }
      }
    }

    let userFactory = new UserFactory();
    let superAdmin = userFactory.create('superAdmin');
```

### 3.2抽象工厂模式
在实际项目中，通常不只是一个工厂、数个类就可以解决的，往往需要多个工厂。比如一个手机制造厂，有硬件系统HardWare和操作系统Operating System（OS）。

抽象工厂（AbstractFactory）约定手机流水线的通用功能：
```javascript
class MobilePhoneFactory {
    // 提供操作系统的接口
    createOS(){
        throw new Error("抽象工厂方法不允许直接调用，你需要将我重写！");
    }
    // 提供硬件的接口
    createHardWare(){
        throw new Error("抽象工厂方法不允许直接调用，你需要将我重写！");
    }
}
```

想要生产不同类型的手机，就要实现具体工厂（ConcreteFactory）类：
```javascript
//想要一个专门生产 Android 系统 + 高通硬件的手机的生产线
//手机型号起名叫 FakeStar，那我就可以为 FakeStar 定制一个具体工厂

// 具体工厂继承自抽象工厂
class FakeStarFactory extends MobilePhoneFactory {
    createOS() {
        // 提供安卓系统实例
        return new AndroidOS()
    }
    createHardWare() {
        // 提供高通硬件实例
        return new QualcommHardWare()
    }
}
```
createOS、createHardWare分别生成具体的操作系统和硬件，叫做具体产品类（ConcreteProduct）。不同的具体产品类（比如安卓系统、苹果系统）之间往往存在着共同的功能，因此可以用抽象产品类（AbstractProduct）来声明这一类产品的基本功能。

```javascript
// 定义操作系统这类产品的抽象产品类
class OS {
    controlHardWare() {
        throw new Error('抽象产品方法不允许直接调用，你需要将我重写！');
    }
}

// 定义具体操作系统的具体产品类
class AndroidOS extends OS {
    controlHardWare() {
        console.log('我会用安卓的方式去操作硬件')
    }
}

class AppleOS extends OS {
    controlHardWare() {
        console.log('我会用🍎的方式去操作硬件')
    }
}
...
//硬件同理
```

调用：
```javascript
const myPhone = new FakeStarFactory()
//安卓操作系统
const myOs = myPhone.createOs()
//硬件
const myHardWare = myPhone.createHardWare()
//启动操作系统(输出‘我会用安卓的方式去操作硬件’)
myOS.controlHardWare()
//...启动硬件系统
```
如果想创建另一款新手机，不需要对抽象工厂进行修改，只需要继承和拓展，满足“开放封闭原则”。

### 3.3工厂模式和抽象工厂模式区别
它们的共同点都是分离一个系统中变与不变的地方，不同之处在于场景的复杂度。

简单工厂模式适合处理逻辑比较简单、共性容易抽离的类，而对于复杂的类（能划分类别、等级，存在扩展可能性的），就必须对共性进行更特别的处理，使用抽象类去降低扩展成本。

### 3.4适应范围
- 对象的构建十分复杂
- 需要依赖具体环境创建不同实例
- 当处理很多共享相同属性的小型对象或组件时

### 3.5优缺点
优点：
- 只需要关心创建结果
- 构造函数和创建者分离, 符合“开闭原则”（工厂方法模式、抽象工厂模式）
- 扩展性高，如果想增加一个产品，只要扩展一个工厂类就可以（抽象工厂模式）

缺点：
- 考虑到系统的可扩展性，需要引入抽象层，增加了系统的抽象性和理解难度

## 4.构造函数模式
- 我们平时使用构造函数去初始化对象，就是应用了构造器模式。
```javascript
function User(name , age, career) {
    this.name = name
    this.age = age
    this.career = career 
}
const user = new User(name, age, career)
```
- 构造器模式的本质是抽象对象的 变 与 不变（实例对象都具有这些属性但是这些属性具体的值是变化的，既确保共性的不变，也确保了个性的灵活）
- 而工厂模式，是去抽象不同构造函数（类）之间的变与不变
- 它的问题是，其定义的方法会在每个实例上创建一遍（可以通过原型模式配合来解决）

## 5.单例模式
### 5.1单例的实现
单例模式即保证一个类只有一个实例，并提供一个访问它的全局访问点，通常会返回一个闭包。实现JS单例模式主要的思路是：先判断实例存在与否，如果存在直接返回，如果不存在就创建了再返回，且会缓存单例实例，用于下次判断实例是否存在，这就确保了一个类只有一个实例对象。

使用立即执行函数实现：
```javascript
var SingletonTester = (function(){
	// 构造器函数
	function Singleton(options){
		options = options || {};
		this.name = 'SingletonTester';
		this.pointX = options.pointX || 6;
		this.pointY = options.pointY || 10;
	}
	// 缓存单例的变量
	var instance;
	// 静态变量和方法
	var _static = {
		name: 'SingletonTester',
		getInstance: function(options) {
			if (instance === undefined) {
				instance = new Singleton(options);
			}
			return instance;
		}
	};
	return _static;
})();
 
var singletonTest = SingletonTester.getInstance({
	pointX: 5,
	pointY: 5
})
 
console.log(singletonTest.pointX); // 5
console.log(singletonTest.pointY); // 5
 
var singletonTest1 = SingletonTester.getInstance({
	pointX: 10,
	pointY: 10
})
console.log(singletonTest1.pointX) // 5
console.log(singletonTest1.pointY) // 5
```

使用class实现：
```javascript
class SingletonApple {
  constructor(name, creator, products) {
    //首次使用构造器实例
    if (!SingletonApple.instance) {
      this.name = name;
      this.creator = creator;
      this.products = products;
      //将this挂载到SingletonApple这个类的instance属性上
      SingletonApple.instance = this;
    }
    return SingletonApple.instance;
  }
}

let appleCompany = new SingletonApple('苹果公司', '乔布斯', ['iPhone', 'iMac', 'iPad', 'iPod']);
let copyApple = new SingletonApple('苹果公司', '阿辉', ['iPhone', 'iMac', 'iPad', 'iPod']);

console.log(appleCompany === copyApple);  //true

//可以使用class的static关键字，更加简洁
class SingletonApple {
  constructor(name, creator, products) {
      this.name = name;
      this.creator = creator;
      this.products = products;
  }
  //静态方法
  static getInstance(name, creator, products) {
    if(!this.instance) {
      this.instance = new SingletonApple(name, creator, products);
    }
    return this.instance;
  }
}
let appleCompany = SingletonApple.getInstance('苹果公司', '乔布斯', ['iPhone', 'iMac', 'iPad', 'iPod']);
let copyApple = SingletonApple.getInstance('苹果公司', '阿辉', ['iPhone', 'iMac', 'iPad', 'iPod'])
console.log(appleCompany === copyApple); //true
```

### 5.2应用
常见的例子有登录框、弹窗、vuex里的store。

vuex里的store：
- vue的组件通信方式最常用的是props、emit
- 当组件非常多，组件间关系十分复杂、嵌套层级很深时，上面的方式会使得逻辑复杂、难以维护
- 这时最好的做法是将共享的数据抽取出来，放在全局，让组件们以一定的规则存取数据
- 于是便有了vuex，用来存放共享数据的数据源，就是store
- vuex内部实现一个install方法，在插件被安装时被调用，从而把store注入vue实例。它判断传入的vue实例是否已经被install，如果没有再install

```javascript
//安装vuex插件
//vuex插件是一个对象，它在内部实现一个install方法，在插件被安装时被调用，从而把store注入vue实例
//也就是说，每install一次，都会尝试给Vue实例注入一个Store
Vue.use(Vuex)

//将store注入到vue中
new Vue({
  el:"#app",
  store
})
```

install方法里：
```javascript
let Vue;
...
export default install(_Vue){
  //判断传入的Vue实例对象是否已经被install过Vuex插件（是否有了唯一的state）
  if(Vue && _Vue === Vue){
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  
  //如果没有，为这个vue实例install唯一的Vuex
  Vue = _Vue
  
  //将vuex的初始化逻辑写进Vue的钩子函数
  applyMixin(Vue)
}
```

### 5.3优缺点
优点：
- 划分命名空间，减少全局变量
- 模块性强，便于维护
- 只实例化一次，简化了代码的调试和维护

缺点：
- 单例的存在，往往表明系统中的模块要么是系统紧密耦合，要么是其逻辑过于分散在代码库的多个部分。由于无法单独测试一个调用了来自单例的方法的类，而只能把它与那个单例作为一个单元一起测试，单例模式的测试会比较困难。
- 由于单利模式中没有抽象层，因此单例类的扩展有很大的困难
- 单例类的职责过重，在一定程度上违背了“单一职责原则”

## 6.原型模式
### 6.1原型模式的实现
原型模式就是创建一个共享的原型，通过拷贝原型对象来创建新的类，使其可以共享上面的属性和方法，提升性能。

使用class实现：
```javascript
class Person  {
  //...
}
class Student extends Person {
  //...
}

let student = new Student()
```

使用Object.create：
```javascript
let obj2 = Object.create(obj1)

obj2.__proto__ === obj1
//true
obj1.isPrototypeOf(obj2) 
//true
```

手动实现：
```javascript
const myPrototype = {}

function obj1(){
  function F(){};
  F.prototype = myPrototype
  
  let f = new F()
  return f
}

let myObj = obj1()
//myObject.__proto__ === myPrototype
```

### 6.2优缺点
优点是可以共享原型对象上的属性和方法，提升性能；而原型模式最主要的问题也源自共享特性，如果某一个实例不小心修改了某属性，会在其他实例中反映出来。

## 7.生成器模式（建造者模式）
### 7.1定义
建造者模式是将一个复杂对象的构建层、表示层分离，使得同样的构建过程可以创建不同的表示。它的特点就是分步骤构建复杂对象，并且可以控制不同的组合、顺序构建不同的对象（即根据不同的具体实现来得到不同的实例），而不需要知道建造的细节。

本质还是隔离变化与不变化的代码（分离整体构建算法和部件构造），复杂对象的各个部分经常变化，但是组合在一起的算法相对稳定。

### 7.2应用与实现
建造者模式经常在代码中用到，比如 Jquery中的ajax请求： 
```javascript
//1 用户发送一个请求
//2 $.ajax建造者模式（指挥者） 
//3 具体实现 （建造者）
$.ajax({
   url:'www.albertyy.com',
   success:function(argument){
      //...
    }
});
```

另一个例子：
```javascript
$('<div class= "foo"> bar </div>');
//只需要传入要生成的HTML字符，而不需要关心具体的HTML对象是如何生产的
```

建造者模式实现过程类似于：
```javascript
// 建造者，部件生产
class ProductBuilder {
    constructor(param) {
        this.param = param
    }
    
    /* 生产部件，part1 */
    buildPart1() {
        // ... Part1 生产过程
        this.part1 = 'part1'
        
    }
    
    /* 生产部件，part2 */
    buildPart2() {
        // ... Part2 生产过程
        this.part2 = 'part2'
    }
}
/* 指挥者，负责最终产品的装配 */
class Director {
    constructor(param) {
        const _product = new ProductBuilder(param)
        _product.buildPart1() //注意这里就是对建造步骤的控制
        _product.buildPart2()
        return _product
    }
}
// 获得产品实例
const product = new Director('param')
```

### 7.3和抽象工厂模式的区别
两者本质都是分离变化与不变，不需要关心创建的过程，只关心最终的调用。

工厂模式关注的是创建的结果；而建造者模式不仅得到了结果，同时也参与了创建的具体过程，适合用来创建一个复杂的复合对象  

如果将抽象工厂模式看成汽车配件生产工厂，生产一个产品族的产品，那么建造者模式就是一个汽车组装工厂，通过对部件的组装可以返回一辆完整的汽车  。

## 8.装饰者模式
### 8.1定义
不修改对象的原本结构，动态地给对象添加一些额外的功能，是一种实现继承的替代方案

假如自行车商店有4种类型自行车：
```javascript
var ABicycle = function(){ ... };
var BBicycle = function(){ ... };
var CBicycle = function(){ ... };
var DBicycle = function(){ ... };
```

需要为每类自行车推出附加功能或者配件，比如前灯，车篮，改色等的时候，其最基本的方式就是为每一种组合创建一个子类，如有铃铛的A类车、黄色的B类车、有车篮的C类车等等，有n种配置就需要生成4n个子类:
```javascript
var ABicycleWithBell = function(){ ... };
var BBicycleWithColorYellow = function(){ ... };
var CBicycleWithColorBule = function(){ ... };
...
```

如果将所有功能设置为实例属性，在类中由传递参数进行控制生成，这样在需要功能扩展时，或者功能较多时并不方便。
而用装饰模式，方便拓展，还可以用于给不同的对象添加新行为。

### 8.2例子
```javascript
class Cellphone {
    create() {
        console.log('生成一个手机')
    }
}
class Decorator {
    constructor(cellphone) {
        this.cellphone = cellphone
    }
    create() {
        this.cellphone.create()
        this.createShell(cellphone)
    }
    createShell() {
        console.log('生成手机壳')
    }
}
// 测试代码
let cellphone = new Cellphone()
cellphone.create()

let dec = new Decorator(cellphone)
dec.create()
```

不将对象再次包装成构造函数，仅仅是调整实例对象:
```javascript
//基础自行车A
function ABicycle(){ }
	ABicycle.prototype = {
    wash : function(){...},
    ride : function(){...},
    getPrice : function(){...}
}

//铃铛装饰
function bicycleBell( bicycle ){
    var price= bicycle.getPrice();

    bicycle.bell = function(){...};

    bicycle.getPrice = function(){...};
    return bicycle;
}

//使用
var bicycleA = new ABicycle();
bicycleA = bicycleBell( bicycleA );
```

### 8.3优缺点
优点：
- 装饰类和被装饰类都只关心自身的核心业务，实现了解耦
- 方便动态的扩展功能

缺点：
- 多层装饰比较复杂
- 使用装饰模式会产生比使用继承关系更多的对象，且这些对象看起来比较相似，不便于查错 

## 9.适配器模式
### 9.1定义
适配器模式（Adapter Pattern）将类（对象）的接口转换成用户所需的另一个接口，解决类之间接口不兼容问题。

适配器模式很常见，比如Axios网络请求库本质上就是对浏览器提供的XMLHTTPRequest的封装，使接口更容易使用。还有Vue的computed计算属性，不改变原有数据，只是将数据适配成需要的格式。
```html
<template>
    <div id="example">
        <p>Original message: "{{ message }}"</p>  <!-- Hello -->
        <p>Computed reversed message: "{{ reversedMessage }}"</p>  <!-- olleH -->
    </div>
</template>
<script type='text/javascript'>
    export default {
        name: 'demo',
        data() {
            return {
                message: 'Hello'
            }
        },
        computed: {
            reversedMessage: function() {
                return this.message.split('').reverse().join('')
            }
        }
    }
</script>
```

### 9.2例子
将日本插头适配成中国插头可用：
```javascript
var chinaPlug = {
    type: '中国插头',
    chinaInPlug() {
        console.log('开始供电')
    }
}
var japanPlug = {
    type: '日本插头',
    japanInPlug() {
        console.log('开始供电')
    }
}

/* 日本插头电源适配器 */
function japanPlugAdapter(plug) {
    return {
        chinaInPlug() {
            return plug.japanInPlug()
        }
				ukInPlug() { //其他国家电源
						return plug.japanInPlug()
				}
    }
}
japanPlugAdapter(japanPlug).chinaInPlug()
// 输出：开始供电
```

### 9.3适用场景
- 想使用一个已有的对象，但是它的接口不满足要求
- 想创建一个可复用对象，而且需要和一些（接口方法或属性）不兼容的对象协同工作

### 9.4优缺点
优点:
- 适配原有功能，使得类可用更好的复用
- 可拓展性良好，在适配时可以调用自己开发的功能
- 灵活性好，不会改变原对象，很容易去增加或删除适配器

缺点:
- 额外对象的创建会存在一定开销（且不像代理模式在某些功能点上可实现性能优化)
- 可能导致可阅读性不好
- 如果没必要使用适配器模式，可以考虑重构



## 10.代理模式
### 10.1定义
代理模式（Proxy Pattern）又称委托模式，为目标对象创建一个代理对象，以控制对目标对象的访问。

ES6提供了Proxy构造函数，可以很方便地创建代理：let proxy = new Proxy(target, handler)。
在ES6之前，一般用Object.defineProperty来完成相同功能。

### 10.2适用场景
代理模式很常见，比如HTML元素的事件代理、axios库的拦截器（类似的还有vue-router路由跳转的拦截器）、前端框架的数据响应式化（vue的数据双向绑定）等。

一般适用于以下场景：
- 保护代理：当一个对象可能会收到大量请求时，可以设置保护代理，通过一些条件判断对请求进行过滤
- 虚拟代理：在程序中可以能有一些代价昂贵的操作，可以设置虚拟代理，在适合的时候才执行操作。比如图片的懒加载（在图片加载前，用其他低质量图片占位，优化图片加载导致白屏的情况），还可以用“骨架屏”占位

### 10.3优缺点
优点：
- 将访问者和目标对象分离，降低了系统耦合度；代理对象可以在访问者和目标对象之间起到中介和保护作用。
- 可以拓展目标对象的功能，符合开闭原则

缺点：
- 处理请求速度可能有差别，非直接访问存在开销
- 增加了系统的复杂度，要斟酌当前场景是不是真的需要引入代理模式

### 10.4适配器与装饰者模式、代理模式对比
- 适配器：功能不变，只是将原有接口转换为其他格式
- 装饰者：方便地给目标对象添加拓展功能，也就是动态地添加功能
- 代理：提供访问目标对象的间接访问，以及对目标对象功能的扩展，一般会提供一样的接口和会设置访问限制
