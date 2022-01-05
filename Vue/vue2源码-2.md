# Vue2源码学习笔记（1）

**相关文章：**

[vue2源码学习笔记-1](Vue/vue2源码-1.md)

**参考：**

[Vue技术解密](https://ustbhuangyi.github.io/vue-analysis/v2/prepare/)

[面试官：你了解vue的diff算法吗？说说看](https://github.com/lihongxun945/myblog/issues/33)

[Vue2.x源码解析系列十：Patch和Diff 算法](https://vue3js.cn/interview/vue/diff.html)


**本文目录：**

[5. new Vue()](#5)

[6. Vue实例挂载](#6)

[7. 渲染](#7)

<span id="5"></span>
## 5.new Vue()

#### _init
前面一节知道定义vue对象的地方是：src/core/instance/index
- new Vue时会执行 _init 方法
- _init方法是由 initMixin方法添加到 Vue的原型上的，init方法位置在：src/core/instance/init.js。主要做的事：
  - 初始化生命周期、事件侦听、渲染方法
  - 调用beforeCreate钩子函数
  - 初始化注入内容
  - 初始化state：即props/data/watch/methods
  - 初始化依赖内容
  - 调用created钩子函数
  - 调用$mount进行挂载

```javascript
// src/core/instance/index
// vue的定义

import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```

```javascript
// src/core/instance/init.js
// vue原型上的_init方法

import config from '../config'
import { initProxy } from './proxy'
import { initState } from './state'
import { initRender } from './render'
import { initEvents } from './events'
import { mark, measure } from '../util/perf'
import { initLifecycle, callHook } from './lifecycle'
import { initProvide, initInjections } from './inject'
import { extend, mergeOptions, formatComponentName } from '../util/index'

let uid = 0

export function initMixin (Vue: Class<Component>) {
  
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    
    //......
    
    // 初始化生命周期、事件侦听、渲染方法
    // 调用beforeCreate钩子函数
    // 初始化注入内容
    // 初始化state：即props/data/watch/methods
    // 初始化依赖内容
    // 调用created钩子函数
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

    //......

    // 调用$mount进行挂载
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

<span id="6"></span>
## 6.Vue实例挂载

#### $mount
new Vue最后会调用$mount方法挂载，根据平台（浏览器或原生客户端）、模式（runtime only或runtime+compiler）的不同，$mount方法在多个文件中有不同的定义：
- src/platform/web/entry-runtime-with-compiler.js
- src/platform/web/runtime/index.js
- src/platform/weex/runtime/index.js

runtime+compiler模式会调用src/platforms/web/entry-runtime-with-compiler.js里的$mount：
- 将template或el的innerHTML 转换为 render函数（**通过compileToFunctions编译**）
- 调用原先Vue原型上的$mount（即src/platform/web/runtime/index.js下的$mount，这个$mount是共享的，方便复用）
- 而共享$mount实际会调用**mountComponent**。

```javascript
// src/platforms/web/entry-runtime-with-compiler.js
// runtime+compiler模式下的$mount

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

	//...

  const options = this.$options
  
  // 将template 或者 el的outerHTML 转换为render函数
  if (!options.render) {                  
    let template = options.template
    if (template) {
      if (typeof template === 'string') { //template为字符串
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)  //idToTemplate：通过id来取得元素的innerHTML
					//...
        }
      } else if (template.nodeType) {	//template为节点
        template = template.innerHTML
      } else { 	//无效template
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {		//如果不存在template但有el
      template = getOuterHTML(el)
    }
    if (template) {
      //...
			// compileToFunctions 将template转为render函数
      const { render, staticRenderFns } = compileToFunctions(template, {
        //...传递的选项
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns

      //...
    }
  }
  return mount.call(this, el, hydrating)
}
```

```javascript
// src/platform/web/runtime/index.js
// 共享的$mount

Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

#### mountComponent
mountComponent方法定义在src/core/instance/lifecycle.js：
- 调用beforeMount钩子函数
- 定义更新函数updateComponent
- 实例化Watcher来监听当前组件状态，传入更新函数updateComponent，且**马上**调用来完成初次渲染。当状态有变化，依然调用更新函数。而更新函数通过调用vm._update更新视图：
  - _update 方法接收 vm._render方法结果为参数
  - _render 方法的作用主要是生成 vnode
  - 即 **_update 负责将 _render 生成的 vnode渲染为真实DOM**
  - 同时值得注意的是，**_render方法里会调用render方法**，如果是runtime+compiler模式（写了template属性），则前面的**compileToFunctions会将template转为render方法**
- 挂载实例，调用mounted钩子函数

```javascript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  // 注意这里将el赋值给了vm.$el
  vm.$el = el
  
  //......(如果没有获取解析的render函数，则会抛出警告)
  
  // 调用beforeMount钩子函数
  callHook(vm, 'beforeMount')

  // 定义更新函数
  let updateComponent
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
      //......
      vm._update(vnode, hydrating) 
      //......
    }
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating) 
    }
  }

  // 实例化Watcher：监听当前组件状态，当有数据变化时，更新组件
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        // 更新前调用 beforeupdate钩子函数
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // 实例已挂载，调用mounted钩子函数
  // vm.$vnode 表示 Vue 实例的父虚拟 Node，所以它为 Null 则表示当前是根 Vue 的实例
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```