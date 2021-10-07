# Vue3新特性梳理

[1.新特性总结](#1)

[2.创建应用实例](#2)

[3.应用挂载](#3)

[4.生命周期](#4)


<span id="1"></span>
## 新特性总结
vue3和vue2对比：
 - 重写了响应式系统、重写了虚拟DOM的实现，性能上得到了提升
 - 新推出组合式API（composition API），使维护组件代码变得更简单
 - 新增了一些功能，如Teleport、Suspense、片段
 - 修改和优化了一些API（生命周期、动画类名），同时移除了一些API（如$children、$listeners、filter）
 - 有更好的TypeScript支持

同时Vue3是向下兼容的，可以使用大部分的Vue2特性。

<span id="2"></span>
## 创建应用实例
从如何创建一个Vue实例说起，Vue2通过new Vue( )创建实例，而Vue3通过Vue.createApp( )创建应用实例：

```javascript
<div id="app">counter: {{counter}}</div>

<script>
  const app = Vue.createApp({
    data(){				
      return {
        counter: 0
      }
    }
  })
  const vm = app.mount("#app") 

  //createApp():创建一个Vue应用实例
  //mount(): 挂载到`id=app`的DOM上
</script>
```

注意应用实例和组件实例的区别：

1.应用实例(app)：
  - Vue.createApp( )返回一个应用实例
  - 应用实例上有很多方法，比如app.component( )注册全局组件，app.directive( )注册全局指令等。而Vue2中它们是全局API，如Vue.component( )
  - 应用实例上的方法大多返回同一应用实例 (mount除外)，所以可以链式操作

  ```javascript
  Vue.createApp({})
    .component('SearchInput', SearchInputComponent)
    .directive('focus', FocusDirective)
    .use(LocalePlugin)
  ```

2.组件实例(vm)：
  - 传递给createApp( )的选项用于配置根组件
  - 将应用实例挂载到一个DOM元素上，返回根组件实例（vm）


<span id="3"></span>
## 应用挂载
将应用挂载到DOM元素上时，Vue3有一点小改变：

  - 有设置template时，Vue2将模板内容替换目标元素，而Vue3将模板内容作为子元素插入
  ```javascript
  <!--Vue2-->
  <body>
    <div id="app"></div>
  </body>
  <script>
    const vm = new Vue({
      template:`<h1>Vue2</h1>`
    })
    vm.$mount("#app")
  </script>

  <!--渲染结果，this.$el指向<h1>-->
  <body>
    <h1>Vue2</h1>
  </body>
  ```

  ```javascript
  <!--Vue3-->
  <body>
    <div id="app"></div>
  </body>
  <script>
    const app = Vue.createApp({
      template:`<h1>Vue3</h1>`
    })
    app.mount("#app")
  </script>

  <!--渲染结果，this.$el指向<h1>-->
  <body>
    <div id="app">
      <h1>Vue3</h1>
    </div>
  </body>
  ```

  - 无template时，Vue2将目标元素的outerHTML当作template，Vue3将目标元素的innerHTML当作template
  ```javascript
  <div id="app">
    <h1>{{msg}}</h1>
  </div>

  //vue3
  Vue.createApp({}).mount('#app')
  //则模板是id为app的元素的innerHTML: <h1>{{msg}}</h1>
  //即this.$el指向h1这个DOM元素



  //vue2
  new Vue({
    el:"#app"
  })
  //模板是id为app的元素的outerHTML: <div id="app"><h1>{{msg}}</h1></div>
  //即this.$el指向div这个DOM元素
  ```

<span id="4"></span>
## 生命周期

### 钩子函数
Vue3的钩子函数与Vue2相比，将beforeDestory、destoryed修改为beforeUnmount、unmounted，增加了renderTracked、renderTriggered、errorCaptured：
 - beforeCreate
 - created
 - beforeMount
 - mounted
 - beforeUpdate
 - updated
 - beforeUnmount
 - unmounted
 - renderTracked
 - renderTriggered
 - errorCaptured

##### errorCaptured （新增）
 - 当捕获到一个后代组件的错误时，会调用errorCaptured
  ```javascript
  errorCaptured(err, errComponent, errInfo){ 
    //err:错误对象
    //errComponent:发生错误的后代组件实例
    //errInfo:包含错误来源的信息
    return true/false 
  }
  ```
 - 函数返回false可以阻止错误继续向上传递（但还是会传递给app.config.errorHandler）

##### renderTracked (新增，状态跟踪，调试用)
 - 此事件告诉你哪个操作跟踪了组件以及该操作的目标对象和键
 - 跟踪虚拟 DOM 重新渲染时调用，可以看作收集依赖时触发，就是渲染依赖了那些数据，当数据变化时会触发，接收一个event对象
 - 状态跟踪在一开始渲染页面时就会触发，触发顺序：beforeMounted - renderTracked - mounted
 - 页面更新也会触发，触发顺序：beforeUpdate - renderTracked - updated
  ```javascript
  //event对象
  {
    effect: {...},
    key: "girls",               //操作的键
    target: {girls:{...},...},  //操作的目标对象
    type: "get"                 //哪个操作跟踪了组件
  }
  ```

##### renderTriggered (新增，状态触发，调试用)
 - 当虚拟DOM重新渲染时触发。和renderTracked类似，接收event对象，此事件告诉你是什么操作触发了重新渲染，以及该操作的目标对象和键
 - 操作页面的显示的数据（引起虚拟DOM重渲染）就会触发，触发顺序：renderTriggered - beforeUpdate


### setup生命周期
setup函数里面设置生命周期函数，是为了使组合式API的功能和选项式API一样完整。
 - setup函数内部的钩子函数基本和选项式一样，只是没有beforeCreate、created，并且在函数前面加上on。
 - 因为setup函数会在beforeCreate之前就执行，即组件创建之前执行。所有beforeCreate、created的代码应该写在setup中
 - 因为还没组件实例，setup中避免使用this，也不能访问以下选项：
   - data
   - computed
   - methods
 - 只能访问以下属性（通过第一个参数props，第二个参数context）：
   - props
   - attrs
   - slots
   - emit

  ![lifeCycle](img/lifeCycle.png)