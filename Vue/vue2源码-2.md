# Vue2源码学习笔记（2）

**相关文章：**

[vue2源码学习笔记-1](./vue2源码-1.md)

**参考：**

[Vue技术解密](https://ustbhuangyi.github.io/vue-analysis/v2/prepare/)

[面试官：你了解vue的diff算法吗？说说看](https://github.com/lihongxun945/myblog/issues/33)

[Vue2.x源码解析系列十：Patch和Diff 算法](https://vue3js.cn/interview/vue/diff.html)


**本文目录：**

[5. new Vue()](#5)

[6. Vue实例挂载](#6)

[7. 渲染](#7)

<span id="5"></span>
## 5. new Vue()

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

<span id="7"></span>

## 7. 渲染

#### 7.1 创建vnode

##### 7.1.1 _render
vue最后都是通过render函数来生成虚拟DOM，写了template也会被转换成render函数。

vue实例挂载时，mountComponent会调用私有_render方法，而私有_render方法最主要的是调用了render方法。私有_render定义在src/core/instance/render.js，主要做了两件事
  - 调用render函数，传入$createElement，拿到vnode节点
  - 设置vnode节点的父节点

```javascript
// src/core/instance/render.js
// 在new Vue时，会引入这个文件的 renderMixin 并调用，_render因此挂载到Vue原型上

export function renderMixin (Vue: Class<Component>) {
  //......
  
  Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    if (_parentVnode) {
      vm.$scopedSlots = normalizeScopedSlots(
        _parentVnode.data.scopedSlots,
        vm.$slots,
        vm.$scopedSlots
      )
    }

    vm.$vnode = _parentVnode

    let vnode
    try {
      // 不需要维护一个堆栈
      // 因为所有的渲染函数都是单独调用的。当父组件被打补丁时，子组件的渲染函数会被调用
      
      //调用render函数，传入$createElement，即平时我们写reander时的createElement
      currentRenderingInstance = vm
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      //...处理错误
    } finally {
      currentRenderingInstance = null
    }
    //如果节点数组只包含一个节点，返回这个节点
    if (Array.isArray(vnode) && vnode.length === 1) {
      vnode = vnode[0]
    }
    // 当渲染函数出错时，返回空的vnode
    if (!(vnode instanceof VNode)) {
      if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
        warn(
          'Multiple root nodes returned from render function. Render function ' +
          'should return a single root node.',
          vm
        )
      }
      vnode = createEmptyVNode()
    }
    // 设置vnode的父节点
    vnode.parent = _parentVnode
    return vnode
  }
}
```

##### 7.1.2 $createElement
当Vue初始化时，先执行了各种mixin（作用是在vue原型上拓展各种方法，比如上面的_render），然后调用vm._init，vm._init会从src/core/instance/render.js引入initRender并调用（initRender和上面的renderMixin在同一个文件），initRender里定义了vm.$createElement。

从代码中可以看到：
- vm.$createElement 和 vm._c  都返回 createElement方法（定义于src/core/vdom/create-element）
- vm.$createElement 用于用户手写render函数的情况；vm._c 用于template转换为render函数的情况

```javascript
// src/core/instance/render.js

import { createElement } from '../vdom/create-element'
//......

export function initRender (vm: Component) {
  //......
  
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

  //......
}
```

而createElement的作用是创建返回虚拟DOM节点（VNode，是对DOM节点的抽象描述，用js对象来实现，定义在 src/core/vdom/vnode.js ）。

从代码中可知createElement是对_createElement的封装，_createElement位于同一个文件中。createElement传入5个参数：
- context：表示VNode的上下文环境
- tag：表示标签，可以是字符串也可以是component
- data：表示VNode的数据
- children：表示VNode的子节点
- normalizationType：表示子节点规范的类型，即 render 函数是编译生成的还是用户手写的

```javascript
//src/core/vdom/create-element
// createElement

export function createElement (
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}
```

而_createElement的两个主要流程是：
- 对子节点进行规范化（调用normalizeChildren或simpleNormalizeChildren）， 因为传入的children 是任意类型的，因此我们需要把它们规范成类型为VNode 的数组
- 根据不同类型的tag创建不同类型的VNode：
  - tag为string
    - 如果是内置节点，则直接通过new VNode()创建普通VNode
    - 如果是已经注册的组件名，则通过createComponent创建组件类型VNode
    - 否则创建一个未知标签的VNode
  - tag为Component，直接通过createComponent创建组件类型VNode

```javascript
//src/core/vdom/create-element
// _createElement

export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  
  //......
  
	// 对子子节点进行规范化
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }

	// 创建节点
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      //......
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }

	//......
}
```

##### 7.1.3 new Vnode()
VNode定义于src/core/vdom/vnode.js，new VNode时主要设置vnode对象的标签名、子对象、父节点、文本、对应的真实节点、option选项等属性：

```javascript
// src/core/vdom/vnode.js
// VNode

export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  functionalContext: Component | void; // only for functional component root nodes
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions
  ) {
    /*当前节点的标签名*/
    this.tag = tag
    /*当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型，可以参考VNodeData类型中的数据信息*/
    this.data = data
    /*当前节点的子节点，是一个数组*/
    this.children = children
    /*当前节点的文本*/
    this.text = text
    /*当前虚拟节点对应的真实dom节点*/
    this.elm = elm
    /*当前节点的名字空间*/
    this.ns = undefined
    /*编译作用域*/
    this.context = context
    /*函数化组件作用域*/
    this.functionalContext = undefined
    /*节点的key属性，被当作节点的标志，用以优化*/
    this.key = data && data.key
    /*组件的option选项*/
    this.componentOptions = componentOptions
    /*当前节点对应的组件的实例*/
    this.componentInstance = undefined
    /*当前节点的父节点*/
    this.parent = undefined
    /*简而言之就是是否为原生HTML或只是普通文本，innerHTML的时候为true，textContent的时候为false*/
    this.raw = false
    /*静态节点标志*/
    this.isStatic = false
    /*是否作为跟节点插入*/
    this.isRootInsert = true
    /*是否为注释节点*/
    this.isComment = false
    /*是否为克隆节点*/
    this.isCloned = false
    /*是否有v-once指令*/
    this.isOnce = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next https://github.com/answershuto/learnVue*/
  get child (): Component | void {
    return this.componentInstance
  }
}
```

##### 7.1.4 createComponent
通过createComponent创建组件类型的VNode，定义于src/core/vdom/create-component.js:
- 构建子类构造函数
  - 如果组件是对象，则调用Vue.extend()将组件对象转换为继承Vue的构造器Sub（即Vue的子类）
  - Vue.extned()会Sub对象本身扩展了一些属性，如扩展 options、添加全局 API 等；并且对配置中的 props 和 computed 做了初始化工作；最后对于这个 Sub 构造函数做了缓存，避免多次执行 Vue.extend 的时候对同一个子组件重复构造
  - 实例化Sub时，依然会调用Vue的_init方法
- 安装组件钩子函数
- 实例化VNode：new VNode（）

```javascript
//src/core/vdom/create-component.js
// createComponent

export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }
 // 构建子类构造函数 
 // 这里的baseCtor实际上就是Vue
  const baseCtor = context.$options._base

  // 组件为对象
  if (isObject(Ctor)) {
    // Vue.extend 的作用就是构造一个 Vue 的子类
    // 把一个纯对象转换一个继承于 Vue 的构造器 并返回
    Ctor = baseCtor.extend(Ctor)
  }

  // 如果它不是一个构造函数或异步组件函数
  if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
  }

  // 异步组件
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor, context)
    if (Ctor === undefined) {
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }

  data = data || {}

  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  resolveConstructorOptions(Ctor)

  // transform component v-model data into props & events
  if (isDef(data.model)) {
    transformModel(Ctor.options, data)
  }

  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)

  // functional component
  if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children)
  }

  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  const listeners = data.on
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn

  if (isTrue(Ctor.options.abstract)) {
    const slot = data.slot
    data = {}
    if (slot) {
      data.slot = slot
    }
  }

  // 安装组件钩子函数，把钩子函数合并到data.hook中
  installComponentHooks(data)

  //实例化一个VNode返回。组件的VNode是没有children的
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )
  if (__WEEX__ && isRecyclableComponent(vnode)) {
    return renderRecyclableComponentTemplate(vnode)
  }

  return vnode
}
```

#### 7.2 将Vnode渲染为真实DOM

##### 7.2.1 _update
调用_update的时机有两个，一是初次渲染，二是数据更新的时候。

前面知道**vm._render创建了vnode，而vm._update将vnode渲染为真实的DOM。**_update方法定义于：src/core/instance/lifecycle.js：
- _update方法核心是调用**vm.__patch__**方法，会区分是初次渲染还是数据更新，传入不同的参数
  - vm.__patch__核心是通过比对虚拟 DOM ，局部更新 DOM 

- 注意区分代码中的vm._vnode 和 vm.$vnode
  - vm.$vnode 指组件的虚拟父节点，也即占位符节点，比如例子下面代码中，child为vm.$vnode，vm.$vnode不存在则表示为vue根实例
  - vm._vnode表示执行_render后返回的VNode节点，即子组件里面的div

```
// vm.$vnode 与 vm._vnode例子

//父组件
<child></child>

//子组件
<template>
  <div>text</div>
</template>
```
```javascript
// src/core/instance/lifecycle
// _update的定义

Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const restoreActiveInstance = setActiveInstance(vm)
  vm._vnode = vnode
  if (!prevVnode) {
    // 初次渲染
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // 数据更新时
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  restoreActiveInstance()
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```

##### 7.2.2 __patch__
_update的核心是调用__patch__，而__patch__在不同平台定义是不一样的，web平台的定义在src/platforms/web/runtime/index.js，代码可知其指向patch：

```javascript
Vue.prototype.__patch__ = inBrowser ? patch : noop
```

而patch定义在src/platforms/web/runtime/patch.js中：
- 通过createPatchFunction创建patch函数
- 传入nodeOps和modules为参数，nodeOps的作用是封装一系列DOM操作的方法，modules定义了模块的相应钩子函数的回调（用于patch.js中的createPatchFunction方法：添加hooks执行时对应的回调。具体可看patchVnode这一小节）

```javascript
export const patch: Function = createPatchFunction({ nodeOps, modules })
```

createPatchFunction位于src/core/vdom/patch.js：
- createPatchFunction在内部定义了一系列辅助方法，然后返回**patch**方法，patch方法的参数：
  - oldVnode：旧的Vnode节点
  - vnode：由_render创建返回的Vnode节点
  - hydrating：是否服务端渲染
  - removeOnly：与transition-group相关的参数
- **path方法通过判断新旧vnode的不同，来移除、创建、更新dom节点**：
  - 如果vnode不存在，oldVnode存在，表示移除，调用invokeDestroyHook销毁旧节点
  - 如果vnode存在：
    - oldNode不存在，表示新增，调用**createElm**创建新节点
    - oldNode存在：
      - 如果oldNode不是真实元素，且oldNode与vnode是同一节点，表示需要修改。通过**patchVnode**方法来比对更新
      - 如果oldVnode是真实元素，表示是初次渲染，oldVnode即传入的el，则将oldVnode转换为Vnode对象，然后调用**createElm**创建新节点
      - 如果oldVnode不是真实元素，且oldNode与Vnode不是同一节点，直接调用**createElm**创建新节点

```javascript
// src/core/vdom/patch.js
// createPatchFunction、patch

//...

	export const emptyNode = new VNode('', {}, [])

	//......


  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      // vnode不存在，oldVnode存在，表示移除，销毁旧节点（旧DOM元素）
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // vnode存在，oldNode不存在，表示新增，创建新的DOM元素
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      // oldNode与vnode都存在
      // 再判断oldNode是否真实元素
      const isRealElement = isDef(oldVnode.nodeType)
      
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // oldNode不是真实元素，且oldNode与vnode是同一节点，表示需要修改
        // 同一节点：key相同；元素标签相同；
        // patchVnode作用：对children进行diff以决定该如何更新
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        // oldNode与vnode不是同一节点，创建新的DOM元素
        
        // oldNode是真实元素（一般是初始渲染，oldVnode的真实dom即传入的vm.$el）
        if (isRealElement) {
          //...
          // oldVnode是真实元素表示是初次渲染时传入的el
          // 则将oldVnode转换为Vnode对象
          oldVnode = emptyNodeAt(oldVnode)
        }

        // 获取oldVnode的真实元素、父元素
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // createElm创建真实的元素，并插入到它的父节点DOM中
        createElm(
          vnode,
          insertedVnodeQueue,
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        //oldNode与vnode不是同一节点且vnode.parent 存在，表示替换（更新），即异步组件
        // 递归地更新父节点元素
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // 销毁父元素（占位符节点）
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
}
```

##### 7.2.3 createElm
createElm方法的作用是创建真实DOM并插入到它的父节点中，主要逻辑：
- 如果是一个组价，通过 createComponent 创建组件型vnode，最后createComponent会执行挂载( $mount )操作
- 如果不是，则通过createChildren递归创建子节点，最后添加到页面：
  - createChildren 的逻辑很简单，实际上是遍历子虚拟节点，递归调用 createElm

```javascript
// src/core/vdom/patch.js
// createElm

function createElm (
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  //......
  
  // 如果是一个组价，那么 createComponent 会返回 true。不用往下执行
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }

  // 如果不是组件，则通过createChildren递归创建节点（createChildren里会调用createElm）
  const data = vnode.data
  const children = vnode.children
  const tag = vnode.tag
  if (isDef(tag)) {
    //......

    vnode.elm = vnode.ns
      ? nodeOps.createElementNS(vnode.ns, tag)
      : nodeOps.createElement(tag, vnode)
    setScope(vnode)

    if (__WEEX__) {
      // ...
    } else {
      createChildren(vnode, children, insertedVnodeQueue)
      if (isDef(data)) {
        invokeCreateHooks(vnode, insertedVnodeQueue)
      }
      // 添加到页面
      insert(parentElm, vnode.elm, refElm)
    }

    if (process.env.NODE_ENV !== 'production' && data && data.pre) {
      creatingElmInVPre--
    }
  } else if (isTrue(vnode.isComment)) {
    vnode.elm = nodeOps.createComment(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  } else {
    vnode.elm = nodeOps.createTextNode(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  }
}
```

##### 7.2.4 patchVnode
patchVnode方法的作用是比较并更新元素的差异（注意进入patchVnode部分的vnode均不是真实元素）
- 如果新vnode是文本节点，则直接更新dom的文本内容
- 如果新vnode非文本节点，分为几种情况：
  - 新旧都有子节点且不完全一致，则调用**updateChildren**对比更新子节点
  - 新vnode有子节点而旧无，则不用比较，直接通过**addVnodes**来创建出新dom，插入父节点中
  - 旧vnode有子节点而新无，则直接通过**removeVnodes**来删除旧dom节点

```javascript
// src/core/vdom/patch.js
// createPatchFunction、patchVnode


const hooks = ['create', 'activate', 'update', 'remove', 'destroy']

//...

// 添加hooks执行时对应的回调
export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  const { modules, nodeOps } = backend

  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = []
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
 
//......

function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    if (oldVnode === vnode) {
      return
    }

    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }
      
		// 让vnode.el引用到oldVnode的真实dom，当el修改时，vnode.el会同步变化
    const elm = vnode.elm = oldVnode.elm

   //......


    const oldCh = oldVnode.children
    const ch = vnode.children
    
    // cbs.update 用于更新attributes
    // cbs定义于上面的createPatchFunction，为每个cbs[hooks]添加相应的回调
    // cbs.update 的回调有下面这些，可以看到都是更新属性相关 
    //  updateAttributes
    //  updateClass
    //  updateDOMListeners
    //  updateDOMProps
    //  updateStyle
    //  update
    //  updateDirectives
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
      
      
    // 新vnode非文本节点
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {
        
        // 新旧vnode都有子节点且不一致，则调用updateChildren对比更新子节点
        
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        
        // 如果新vnode有子节点、旧vnode无子节点，则创建新节点
        
        // 如果旧vnode为文本节点，去掉文本
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        
        // elm已经引用了老的dom节点，在老的dom节点上添加子节点
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        
        // 如果新vnode无子节点、旧vnode有子节点，则删除旧子节点
        
        removeVnodes(oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        
        // 如果老节点是文本节点
        
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      
      // 新旧vnode都是文本节点且不相等，则直接更新DOM的text
      
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```

##### 7.2.5 updateChildren【diff算法的核心】
patchVnode中调用updateChildren来对比更新新旧vnode的子节点，对比的核心算法即diff算法。diff算法整体思路是**从两边到中间开始比较，深度优先、同级比较。**

对比从新旧vnode的头尾开始，核心思路：
- 情况一：子节点相同且位置一样时，调用patchNode继续迭代比较这个子节点
- 情况二：子节点相同但位置不一样，调用patchNode继续迭代比较这个子节点，且相应移动oldVnode中的子节点
- 情况三：循环结束，如果这个子节点不存在与oldVnode中，不可以复用节点，则调用createElm新建节点
- 情况四：循环结束，如果这个子节点不存在与newVnode中，则移除该节点

```javascript
// src/core/vdom/patch.js
// updateChildren

function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0] // oldVnode的第一个child
    let oldEndVnode = oldCh[oldEndIdx] // oldVnode的最后一个child
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]	// newVnode的第一个child
    let newEndVnode = newCh[newEndIdx]	// newVnode的最后一个child
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(newCh)
    }

  // 如果oldEndIdx > oldStartIdx重合，或者newEndIdx > newStarIdx，证明diff完了，循环结束
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        // 如果oldVnode的第一个child不存在，oldStart索引右移
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        // 如果oldVnode的最后一个child不存在，oldEnd索引左移
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        // 新旧第一个子节点相同，调用patchVnode迭代比较这个子节点
        // 头部索引继续右移
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        // 新旧最后一个子节点相同，调用patchVnode迭代比较这个子节点
        // 尾部索引继续左移
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // 节点右移
        // oldStartVnode和newEndVnode是同一个节点, 调用patchVnode迭代比较这个子节点
        // 同时将该子节点插入到oldEndVnode后面
        // oldStart索引右移，newEnd索引左移
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // 节点左移
        // oldEndVnode和newStartVnode是同一个节点, 调用patchVnode迭代比较这个子节点
        // 同时将该子节点插入到oldStartVnode前面
        // oldEnd索引左移，newStart索引右移
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
       // 如果都不匹配
       // 尝试在oldChildren中寻找和newStartVnode的具有相同的key的Vnode
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        
        // 如果都不存在，说明newStartVnode是一个新的节点，则新建
        if (isUndef(idxInOld)) { // New element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          
        // 如果存在，定义为vnodeToMove
        // 比较vnodeToMove与newStartVnode是否相同
       	// 相同则调用patchVnode继续迭代，且将vnodeToMove节点插入到oldNode最前面
        // （只有key相同）不相同则新建
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        // newStarIdx右移
        newStartVnode = newCh[++newStartIdx]
      }
    }
  
  // 循环结束时，若oldStartIdx > oldEndIdx，表示oldVnode中找不到newVnode的子节点，需要新建
  // 若newStarIdx > newEndidx，表示newVnode中找不到oldVnode的子节点，需要删除
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      // addVnodes内部同样调用了createElm新建节点
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
  }
```

#### 7.3 总结
- 初次渲染会调用更新函数updateComponent来完成视图渲染，数据更新会触发watcher，而watcher也是调用更新函数updateComponent来完成视图更新
- updateComponent核心是调用_update(），且传入_render()当作参数
- _render()的作用是生成vnode（虚拟dom）
- _update()的作用是将vnode渲染为真实DOM，核心是调用__patch__，通过判断新旧vnode的不同，来决定移除、创建、更新dom节点:
  - 如果是初次渲染，通过createElm创建新节点
  - 如果是数据更新，通过patchVnode对比更新新旧vnode的差异，比对算法即Diff算法，如果有非文本差异且不能复用的节点，最后还是通过createElm创建新节点