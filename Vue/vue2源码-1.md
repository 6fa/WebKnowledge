# Vue2源码学习笔记（1）

**参考：**

[Vue技术解密](https://ustbhuangyi.github.io/vue-analysis/v2/prepare/)

[面试官：Vue实例挂载的过程](https://vue3js.cn/interview/vue/new_vue.html)

[vue中Runtime-Compiler和Runtime-only的区别](https://www.cnblogs.com/lyt0207/p/11967141.html)

[对不同构建版本的解释](https://cn.vuejs.org/v2/guide/installation.html#%E5%AF%B9%E4%B8%8D%E5%90%8C%E6%9E%84%E5%BB%BA%E7%89%88%E6%9C%AC%E7%9A%84%E8%A7%A3%E9%87%8A)

**本文目录：**

[1. 源码核心目录](#1)

[2. 源码构建](#2)

[3. Runtime-Only 和 Runtime-Compiler](#3)

[4. 入口（runtime+compiler模式）](#4)
  - [4.1 Vue入口](#41)
    - [4.1.1 Vue的定义](#411)
    - [4.1.2 initGlobalAPI](#412)
  - [4.2 挂载](#42)
  - [4.3 总结](#43)



<span id="1"></span>
## 1.源码核心目录
Vue2源码可查看官方github：https://github.com/vuejs/vue

源码主要放在src目录：

```
src
├── compiler        # 编译相关 
├── core            # 核心代码 
├── platforms       # 不同平台的支持
├── server          # 服务端渲染
├── sfc             # .vue 文件解析
├── shared          # 共享工具代码
```
- compiler
  - 存放编译相关代码，包括
  - 将模板解析为AST树（抽象语法树）、AST树优化、代码生成
- core
  - 核心代码，包括
  - 内置组件、全局 API 封装、工具函数
  - Vue实例化、响应式、虚拟DOM
- platforms
  - vue.js的入口
  - 2 个目录代表 2 个主要入口，分别打包成运行在 web 上和 weex 上的 Vue.js
- server
  - 服务端渲染相关
- sfc
  - 把.vue文件解析成一个js对象
- shared
  - 一些工具，会被浏览器端Vue.js和服务端Vue.js共享

<span id="2"></span>
## 2.源码构建
在源码的package.json文件中可以看到，构建代码的入口是scripts/build.js：

```javascript
{
  //...
  "scripts": {
     //...
     "build": "node scripts/build.js",
     "build:ssr": "npm run build -- web-runtime-cjs,web-server-renderer",
     "build:weex": "npm run build -- weex",
     //...
   }
}
```

scripts/build.js的重点代码：
 - 先从配置文件（config）读取配置，再通过命令行参数对构建配置（builds）做过滤，以构建出不同用途的Vue.js
 - build()里面通过rollup（打包工具）来构建

```javascript
//scripts/build.js

let builds = require('./config').getAllBuilds()

// filter builds via command line arg
// 通过比对命令行参数对builds过滤
if (process.argv[2]) {
  const filters = process.argv[2].split(',')
  builds = builds.filter(b => {
    return filters.some(f => b.output.file.indexOf(f) > -1 || b._name.indexOf(f) > -1)
  })
} else {
  // filter out weex builds by default
  // 默认情况下过滤掉weex构建
  builds = builds.filter(b => {
    return b.output.file.indexOf('weex') === -1
  })
}

build(builds)
```

再来看一下config配置文件：

```javascript
// script/config.js
//配置是遵循 Rollup 的构建规则的

const builds = {
  // Runtime only (CommonJS). Used by bundlers e.g. Webpack & Browserify
  'web-runtime-cjs': {
    entry: resolve('web/entry-runtime.js'), //构建入口
    dest: resolve('dist/vue.runtime.common.js'), //构建后的文件位置
    format: 'cjs', //构建格式，cjs表示遵循CommonJS，es则ES Module，umd则UMD规范
    banner
  },
  // Runtime+compiler CommonJS build (CommonJS)
  'web-full-cjs': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.common.js'),
    format: 'cjs',
    alias: { he: './entity-decoder' },
    banner
  },
  //...
}
```

<span id="3"></span>
## 3.Runtime-Only 和 Runtime-Compiler

##### 编译器（compiler）：

  用来将模板字符串编译成为 JavaScript 渲染函数的代码

##### 运行时（runtime）：

  用来创建 Vue 实例、渲染并处理虚拟 DOM 等的代码。基本上就是除去编译器的其它一切

##### runtime + compiler（完整版）：

如果写了template，那么需要在运行时将template编译成render函数（生成虚拟DOM）。因为在 Vue2中，最终渲染都是通过 render 函数，如果写 template 属性，则需要编译成 render 函数,则需要compiler。

```javascript
// 需要编译器
new Vue({
  template: '<div>{{ hi }}</div>'
})

// 不需要编译器
new Vue({
  render (h) {
    return h('div', this.hi)
  }
})
```

##### runtime only（只有运行时）：

需要借助webpack的vue-loader将.vue文件编译成js文件：vue文件内部的模板会在构建时预编译成 JavaScript。（此时main.js里的new Vue选项依然不可以写template属性，只有.vue文件里可以写）

由于不需要编译器compiler了，所以体积减少了约30%，应该尽可能使用runtime only版本。

<span id="4"></span>
## 4.入口（runtime+compiler模式）
runtime+compiler模式入口文件位置：src/platforms/web/entry-runtime-with-compiler.js

代码的重点是：
 - 引入了Vue（vue的入口为./runtime/idex）
 - 在Vue的原型上定义了$mount方法
 - 导出Vue

```javascript
import config from 'core/config'
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'

import Vue from './runtime/index'
import { query } from './util/index'
import { compileToFunctions } from './compiler/index'
import { shouldDecodeNewlines, shouldDecodeNewlinesForHref } from './util/compat'

//...

//定义$mount方法
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (){......}


//...

Vue.compile = compileToFunctions

export default Vue
```

<span id="41"></span>
### 4.1 Vue入口
上面引入Vue的位置为：src/platforms/web/runtime/index.js

代码重点：
 - 从core/index 引入Vue
 - 对Vue对象进行一些拓展，比如定义了Vue原型上的基础$mount方法
 - 从 core/index.js 又可以看到
  - 从core/instance/index 导入了Vue对象，因此真正的Vue实例定义在core/instance/index
  - 初始化全局API ： initGlobalAPI(Vue)

```javascript
// src/platforms/web/runtime/index.js

import Vue from 'core/index'
import config from 'core/config'
import { extend, noop } from 'shared/util'
import { mountComponent } from 'core/instance/lifecycle'
import { devtools, inBrowser } from 'core/util/index'

import {
  query,
  //......
} from 'web/util/index'

//......

// install platform specific utils
Vue.config.mustUseProp = mustUseProp
//......

// public mount method
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}

//......

export default Vue
```

```javascript
// core/index

import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'

initGlobalAPI(Vue)

//......

export default Vue
```

<span id="411"></span>
#### 4.1.1 Vue的定义
位置core/instance/index，重点：
 - 定义了Vue构造函数（为什么不用class，是因为后面将Vue当参数传递，主要是给vue的原型拓展方法，这些拓展方法分散在各个模块，如果是class则难以实现）
 - xxMixin都是在Vue的prototype上拓展一些方法，比如_init方法就是在initMixin里添加的

```javascript
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

<span id="412"></span>
#### 4.1.2 initGlobalAPI
上面的是给Vue.prototype拓展方法，则initGlobalAPI则是给Vue本身添加静态方法。

位置：src/core/global-api/index.js

```javascript
//......

export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // Vue.util 暴露的方法最好不要依赖，因为它可能经常会发生变化，是不稳定的
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  // 2.6 explicit observable API
  Vue.observable = <T>(obj: T): T => {
    observe(obj)
    return obj
  }

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}
```

<span id="42"></span>
### 4.2 挂载

$mount方法重点：
 - 如果定义了template或el，则将template或el的innerHTML 转换为 render函数
 - 转换是通过 compileToFunctions 实现的
 - 最后调用Vue原型上的共享$mount方法挂载（这么做是为了复用，runtime模式直接用，省却了上面转换成render函数的步骤。位置在src/platform/web/runtime/index.js）
 - $mount方法实际会去调用mountComponent方法（定义在src/core/instance/lifecycle.js）

```javascript
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
          template = idToTemplate(template)  //idToTemplate是通过id来取得元素的innerHTML
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
// 可复用的共享$mount

Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

<span id="43"></span>
### 4.3 总结
runtime+compiler模式入口（src/platforms/web/entry-runtime-with-compiler.js）
- 从runtime/index引入vue（src/platforms/web/runtime/index.js)
  - 从core/index 引入Vue
    - 从core/instance/index 导入了Vue对象，因此真正的Vue实例定义在core/instance/index
      - 定义了Vue构造函数
      - 拓展Vue的prototype上一些方法
    - 初始化全局API ： initGlobalAPI(Vue)
  - 对Vue对象进行一些拓展，比如定义了Vue prototype上的基础$mount方法

- 在vue原型上定义$mount（注意和共享的$mount方法不同）
  - 将template转换为render函数，转换是通过 compileToFunctions 实现的
  - 调用基础$mount方法（src/platform/web/runtime/index.js）
  - 基础$mount方法实际会调用mountComponent方法（src/core/instance/lifecycle.js）