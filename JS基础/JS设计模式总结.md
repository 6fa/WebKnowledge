# JS设计模式

参考：
- [JS设计模式有哪些？有什么区别？](https://zhuanlan.zhihu.com/p/142304509)
- [深入理解JavaScript系列](https://www.cnblogs.com/TomXu/archive/2011/12/15/2288411.html)
- [掘金小册 - JavaScript 设计模式核⼼原理与应⽤实践](https://www.weisuoke.com/fe2020/Gof/juejin-GoF.html)
- [JavaScript设计模式es6（23种)](https://juejin.cn/post/6844904032826294286)
- [JavaScript设计模式与实践--工厂模式](https://juejin.cn/post/6844903653774458888)
- [写给前端的设计模式](https://www.yuque.com/wubinhp/uxiv5i)

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