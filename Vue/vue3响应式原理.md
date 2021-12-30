# vue3响应式原理

[1.响应式核心](#1)

[2.Proxy & Reflect的基本使用](#2)

[3.Reactive函数的实现](#3)

[4.副作用函数](#4)

[5.Track & Trigger的实现](#5)

[6.整合&使用](#6)


<span id="1"></span>
## 1.响应式核心
假如下面的例子中，想让sum变为响应式变量：

```javascript
let num1 = 1;
let num2 = 2;
let sum = num1 + num2;

num1 = 10
console.log(sum) //sum依旧是3，非响应式
```

则要实现的部分有：
  - 数据劫持：要知道num1、num2何时发生变化
  - 依赖收集：知道sum依赖哪些数据，例子中sum依赖了num1、num2，则要建立它们的依赖关系
  - 派发更新：当依赖的数据num1、num2发生改变时，要通知响应对象sum重新运算

vue3通过Proxy拦截数据的读取和设置（数据劫持），当数据读取时，通过track函数触发依赖的收集；当数据被设置时，通过trigger函数去派发更新。

那么vue3如何使用响应式呢？
  - vue3既可以通过data函数返回一个响应式对象，也可以通过ref、reactive来创建响应式变量。使用reactive等时，即在内部对数据用Proxy进行了包装。
  - 使用computed、watch、视图渲染函数等时，可以看作声明了一个依赖响应式数据的回调，这个回调会被传入effect（副作用函数），当依赖的数据改变时，回调被重新调用，从而computed等得到更新。

要实现简单版的响应式，其大致结构为：

```javascript
//创建响应式变量，拦截数据的get和set
function reactive(obj){}

//effect函数包裹那些 依赖响应式数据的函数cb
//cb依赖的数据更新时，重新执行effect
function effect(cb){}

//依赖收集，建立响应式数据和effect的映射关系
function track(target, property){}
//触发更新，根据依赖关系，执行effect函数
function trigger(target, property){}
```

使用：

```javascript
let obj = reactive({
  num1: 10,
  num2: 20
})
let sum = 0

effect(()=>{
  sum = obj.num1 + obj.num2
})

console.log(sum) //30

obj.num1 = 100
console.log(sum)  //应该为120
```

<span id="2"></span>
## 2.Proxy & Reflect的基本使用

实现响应式变量的创建前，需要知道Proxy和Reflect的基本使用。

JS很难对单个局部变量进行跟踪，但是可以跟踪对象的属性变化：vue3使用的ES6的Proxy和Reflect。
  - Proxy拦截对象的读取、设置等操作，然后进行操作处理。但是不会直接操作源对象，而是通过对象的代理对象。
  ```javascript
  //Proxy用法
  //Proxy对象由target（目标对象）、handler（指定代理对象行为的对象）组成

  let target = {
    a:1,
    b:2
  }
  let handler = {
    //receiver指调用该行为的对象，通常是Proxy实例本身
    get(target, propKey, receiver){
      return target[propKey]	//getter甚至可以不返回数据
    },
    
    set(target, propKey, value, receiver){
      target[propKey] = value
    }
  }

  let proxy = new Proxy(target, handler)
  console.log(proxy.a) 				//1

  proxy.a = 3
  console.log(proxy.a)        //3
  ```

  - Reflect能直接调用对象的内部方法，和Proxy一样有获取、设置等操作。
  ```javascript
  //Reflect用法

  let target = {
    get a(){return this.num1 + this.num2},
    set a(val){return this.num1 = val}
  }
  //receiver为可选参数
  let receiver = {
    num1:10,
    num2:20
  }

  //Reflect.get(target, propKey, receiver)
  //相当于直接操作target的get a(){}
  Reflect.get(target, 'a', receiver)  //30  this绑定到了receiver

  //Reflect.set(target, propKey, value, receiver)
  Reflect.set(target, 'a', 100, receiver) //100
  ```

  - Reflect的作用主要是解决this的绑定问题，将this绑定到proxy对象而不是目标对象：比如Reflect.get（target，property，receiver）获取属性时，如果property指定了getter，getter的this将绑定到receiver对象。
  ```javascript
  //Proxy的问题
  const obj = {
    a: 10,
    get double(){
      return this.a*2
    }
  }

  const proxyobj = new Proxy(obj,{
    get(target, propKey, receiver){
      return target[propKey]
    }
  })
  let obj2 = {
    __proto__: proxyobj
  }
  obj2.a = 20
  obj2.double //期望值为40，实际是20，因为double的getter里的this绑定到了obj




  //使用Reflect解决this绑定问题
  const obj = {
    a: 10,
    get double(){
      return this.a*2
    }
  }

  const proxyobj = new Proxy(obj,{
    get(target, propKey, receiver){								//这里的receiver是obj2
      return Reflect.get(target, propKey, receiver)
    }
  })
  let obj2 = {
    __proto__: proxyobj
  }
  obj2.a = 20
  obj2.double  //40, 通过Refelct的receiver，get double()里的this绑定到了obj2
  ```

<span id="3"></span>
## 3.Reactive函数的实现
创建响应式变量reactive函数的实现，主要靠内部实例化一个Proxy对象：

```javascript
function reactive(obj){
  const handler = { //拦截数据的get、set进行处理
    get(){},
    set(){}
  }
  const proxyObj = new Proxy(obj,handler)
  return proxyObj  //返回代理对象实例
}
```

在获取数据时，就要开始进行数据的依赖收集（交给track函数去实现）；在设置数据时，要触发更新（交给trigger函数去实现）：

```javascript
function reactive(obj){
  const handler = {
    get(target, propKey, receiver){
      const val = Reflect.get(...arguments)  //读取数据
      track(target, propKey)	//依赖收集
      return val
    },
    set(target, propKey, newVal, receiver){
      const success = Reflect.set(...arguments)	//设置数据，返回true or false
      trigger(target, propKey)	//触发更新
      return success
    }
  }
  const proxyObj = new Proxy(obj,handler)
  return proxyObj  //返回代理对象实例
}
```

但是上面仅对一层的对象起作用，对于属性还是对象的多层嵌套对象不起作用，需要手动递归实现响应：

```javascript
function reactive(obj){
  const handler = {
    get(target, propKey, receiver){
      const val = Reflect.get(...arguments)
      track(target, propKey)	
      if(typeof val === 'object'){
        return reactive(val)   //新增
      }
      return val
    }
  }
  ...
}
```

<span id="4"></span>
## 4.副作用函数
在实现上面的track和trigger前，还要了解副作用函数。副作用函数effect用来跟踪正在运行的函数，比如watch、computed，里面的代码会被传入effect，当watch、computed里面依赖的其他数据变化时，重新运行里面的代码。

以vue3的computed为例子：

```javascript
const num = ref(10) //num是响应式
const double = computed(()=>num*2)
```

可以把computed里面的内容看作依赖了响应式数据的更新函数（下面简称更新函数），且computed返回一个ref引用，则在computed函数内部，会有大概类似于下面的操作：

```javascript
computed(cb){
  const result = ref()
  effect(()=>result.value = cb())
  return result
}
```

更新函数被当作effect函数的回调：

```javascript
//当前运行的副作用函数
let activeEffect = null

const effect = (cb)=>{
  activeEffect = cb
  //运行响应式函数
  cb()
  activeEffect = null
}
```

effect函数执行了更新函数，则会读取它依赖的数据，前面我们已经为这些数据设置了proxy代理，就在此时完成了依赖收集（建立更新函数与依赖的数据的映射关系，当数据发生变化，会通过该映射关系找到依赖该数据的更新函数，再次执行）。

以下面例子来说明：

```javascript
let num = reactive({value: 10})
let double = 0
effect(()=>{double = num.value*2}) //double为响应式
let triple = num.value*3  //triple不是响应式

//1. num被reactive包装成响应式变量，会对它属性的获取、设置进行拦截
//2. 运行副作用函数effect，将当前运行的副作用函数activeEffect指向effect的回调，即更新函数
//3. 执行activeEffect（更新函数）
//4. 更新函数内部会读取num.value, 触发proxy的依赖收集track函数

//6. track里面会将 num.value与更新函数建立映射关系
//	 要建立映射关系是因为，当num.value改变时，trigger需要查找出全部依赖num.value的更新函数
//   然后全部重新运行，从而double被重新赋值

//7. double被赋值
//8. 把activeEffect重新指向null

//8. 运行到triple，即使读取了num.value，但是此时activeEffect为null
//	 不会建立num.value与activeEffect的映射关系，所以num.value改变时不会更新到triple
```

<span id="5"></span>
## 5.Track & Trigger的实现
track(target, property)：

  主要将target.property与更新函数记录在一起，形成映射关系，这样就知道依赖target.property都有哪些更新函数

trigger(target, property)：

  从映射关系中找到依赖target.property的更新函数，重新运行它们

依赖一个target.property的更新函数可以有很多，用Set结构去储存它们：

```
property1: Set [cb1,cb2,cb3...]
```

将这个set结构称为dep，每个响应式属性都要有一个dep，可以用Map结构储存：

```
Map {
  property1: Set [cb1,cb2,cb3...],
  property2: Set [cb1,cb2,cb3...],
  property3: Set [cb1,cb2,cb3...],
  ......
}
```

将这个Map结构称为depsMap。但是这只是同一个对象里的属性，如果有多个对象呢？

因此需要又包裹一层，用另一个Map（称为targetMap结构）包裹每个对象的Map，但是targetMap用WeakMap结构，WeakMap的属性刚好只能是对象：

```
WeakMap {
  obj1: Map { 
    property1:Set [cb1,cb2,cb3...],
    ...
  },
  obj2: Map { 
    property1:Set [cb1,cb2,cb3...],
    ...
  },
  ...
}
```

当数据被读取时，track函数正是通过这个结构实现响应式属性和依赖它的更新函数的映射：

```javascript
const targetMap = new WeakMap()

function track(target, property){
  if(!activeEffect)return 
  //假如activeEffect为null则返回
  //只有运行effect()时activeEffect才有值
  
  let depsMap = targetMap.get(target)
  //如果target对象还没对应的depsMap则新建
  if(!depsMap){
    targetMap.set(target, depsMap = new Map())
  }

  let dep = depsMap.get(property)
  //如果属性还没对应的dep则新建
  if(!dep){
    depsMap.set(property, dep = new Set())
  }

  dep.add(activeEffect) //添加属性对应的effect进映射结构
}
```

当数据被设置时，trigger函数通过映射结构取出数据对应的所有更新函数并执行：

```javascript
function trigger(target, property){
  const depsMap = targetMap.get(target)
  if(!depsMap) return
  
  const dep = depsMap.get(property)
  if(!dep) return
  //dep是Set结构，有forEach方法
  dep.forEach((effect)=>{
    effect()
  })
}
```

<span id="6"></span>
## 6.整合&使用
整合上面的代码：

```javascript
//reactive.js

//effect函数的实现
let activeEffect = null
function effect(cb){
  activeEffect = cb
  cb()
  activeEffect = null
}

//创建响应式变量函数
function reactive(obj){
  const handler = {
    get(target, propKey, receiver){
      const val = Reflect.get(...arguments)//读取数据
      track(target, propKey)  //依赖收集
      if(typeof val === 'object'){
        return reactive(val)
      }
      return val
    },
    set(target, propKey, newVal, receiver){
      const success = Reflect.set(...arguments)//设置数据,返回true/false
      trigger(target, propKey)  //触发更新 
      return success
    }
  }
  const proxyObj = new Proxy(obj,handler)
  return proxyObj
}

//依赖收集函数
const targetMap = new WeakMap() //储存映射关系的结构
function track(target, property){
  if(!activeEffect)return 

  let depsMap = targetMap.get(target)
  //如果target对象还没对应的depsMap则新建
  if(!depsMap){
    targetMap.set(target, depsMap = new Map())
  }

  let dep = depsMap.get(property)
  //如果属性还没对应的dep则新建
  if(!dep){
    depsMap.set(property, dep = new Set())
  }

  dep.add(activeEffect) //添加属性对应的effect进映射结构
}

//派发更新函数
function trigger(target, property){
  const depsMap = targetMap.get(target)
  if(!depsMap) return
  
  const dep = depsMap.get(property)
  if(!dep) return
  dep.forEach((effect)=>{
    effect()
  })
}
```

测试使用：

```javascript
let obj = reactive({
  num1: 10,
  num2: 20,
  son:{
    num3:20
  },
})
let sum = 0

effect(()=>{
  sum = obj.num1 + obj.num2 + obj.son.num3
})

console.log(sum) //50

obj.num1 = 100
console.log(sum)  //130 可知sum为响应式
```


参考：

[Vue3响应式原理及实现](https://segmentfault.com/a/1190000022871354?utm_source=sf-similar-article)

[Vue3响应式原理+手写reactive](https://segmentfault.com/a/1190000023465134)

[Vue3响应式原理与reactive、effect、computed实现](https://segmentfault.com/a/1190000023380448)

[ES6 Reflect 与 Proxy](https://www.runoob.com/w3cnote/es6-reflect-proxy.html)