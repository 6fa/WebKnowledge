# Vue2源码学习笔记（3）

**相关文章：**

[vue2源码学习笔记-1](./vue2源码-1.md)
[vue2源码学习笔记-2](./vue2源码-2.md)

**参考：**

[Vue技术解密](https://ustbhuangyi.github.io/vue-analysis/v2/prepare/)

[从initProps开始理解vue2的响应式原理](https://juejin.cn/post/6997053372507521055)

[Watcher](https://blog.windstone.cc/vue/source-study/observer/watcher.html)

## 8. 响应式原理

在vue初始化的阶段就把数据变成了响应式。

### 8.1 initState
初始化_init方法里面会执行initState(vm)方法，主要是对**props、methods、data、computed 和 wathcer 等属性**做了初始化操作：
- initProps：初始化props
- initMethod：初始化方法
- initData：初始化data
- initComputed：初始化computed
- initWatch：初始化watch

```javascript
// src/core/instance/state.js
// initState

export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    // 如果没有定义data，则创建一个空对象，并设置为响应式
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

#### 8.1.1 initProps
initprops主要工作是遍历options中的props：
- 通过validateProp方法返回正确的prop值（因为prop可以指定类型或者默认值）（validateProp定义于src/core/util/props.js）
- 调用 **defineReactive** 方法把每个 prop 对应的值变成响应式
- 通过 **proxy** 把 vm._props.xxx 的访问代理到 vm.xxx 上，即通过vm.xxx可以直接访问到某props属性

```javascript
// src/core/instance/state.js
// initprops

function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) { 
    toggleObserving(false)  //toggleObserving表示是否深度观察Object和array
  }
  
  // 遍历options中的props，并将prop key缓存起来
  for (const key in propsOptions) {
    keys.push(key)
    // 通过validateProp函数获取props值
    // 因为props可以指定类型或者默认值，validateProp的作用是返回正确的props值
    const value = validateProp(key, propsOptions, propsData, vm)
    
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      const hyphenatedKey = hyphenate(key)
      if (isReservedAttribute(hyphenatedKey) ||
          config.isReservedAttr(hyphenatedKey)) {
        warn(
          `"${hyphenatedKey}" is a reserved attribute and cannot be used as component prop.`,
          vm
        )
      }
      defineReactive(props, key, value, () => {
        if (!isRoot && !isUpdatingChildComponent) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

#### 8.1.2 initData
初始化data时的主要工作：
- 对data函数返回的对象进行遍历
- 通过 proxy 把每一个值 vm._data.xxx 都代理到 vm.xxx 上
- 调用observe观测整个data

可以看到初始化各种响应式数据的过程中，都会涉及到observe等方法。

```javascript
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```

### 8.2 observe
observe的作用是监测对象数据，主要给非VNode对象数据实例化一个Observer类，如果添加过则返回：

```javascript
// src/core/observer/index.js
// observe

export function observe (value: any, asRootData: ?boolean): Observer | void {
  // 如果value不是对象则退出
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

#### 8.2.1 Observer类
Observer类的作用是给对象的属性添加getter、setter，用于依赖收集和派发更新：
- 实例化**Dep**对象
  - Dep是依赖收集的核心，相当于调度中心，维护一个subs数组，在响应式数据的getter里将订阅者watcher添加到subs，在setter里调用dep.notify()通知watcher执行更新函数
- 判断传入的value是什么类型
  - 如果是数组，**劫持数组上的原生方法、遍历数组再次进行observe**
  - 如果是对象，则**遍历属性**，调用**defineReactive**将对象变为响应式对象

```javascript
// src/core/observer/index.js
// Observer

export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    
    // def将this挂载到value的__ob__属性，
    def(value, '__ob__', this)
    
    if (Array.isArray(value)) {
      // 把 数组上的原生方法进行了一次劫持
      // 所以调用一些原生方法，vue会通知watcher
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)   // 对数组来说，对每一项都进行递归 observe
    } else {
      this.walk(value)  // 不是数组，调用defineReactive对每个属性定义为响应式
    }
  }

  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

 
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

#### 8.2.2 Dep
Dep是依赖收集的核心，每个dep实例对应着一个响应式数据。实际上Dep是对Watcher的管理：
- Dep类定义静态属性target，调用pushTarget方法时会将Dep.target赋值为Watcher
- Dep类维护着subs数组，用来存放watcher：
  - 定义了addSub、removeSub来往subs中添加、移除watcher
  - 定义了depend方法来调用watcher的addDep，addDep即依赖收集，最后还是调用dep.addSub()
  - 定义了notify方法用于通知subs里的watcher更新

```javascript
// src/core/observer/dep.js

import type Watcher from './watcher'
import { remove } from '../util/index'
import config from '../config'

let uid = 0

/**
 * A dep is an observable that can have multiple directives subscribing to it.
 */
export default class Dep {
  // 定义了静态属性 target，指向全局唯一 Watcher
  // 因为在同一时间只能有一个全局的 Watcher 被计算
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      // Dep.target即watcher
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

#### 8.2.3 Watcher
Watcher的重点：
- 定义了两个dep数组：this.deps、this.newDeps（存放新增的dep）
- 定义了依赖收集相关的有 get、addDep 和 cleanupDeps 等方法
- 当**new Watcher**时会首先触发它的**get方法**：
  - 将Dep.target指向这个watcher实例（同一时间只能有一个watcher在计算）
  - 调用传入的更新函数：this.getter.call(vm, vm)。this.getter即new Watcher时传入的更新函数updateComponent，实际是执行vm._update(vm._render(), hydrating)
  - 执行_render()生成vnode的过程会涉及到对vm数据的访问，这个时候就触发了数据对象的 getter
  - 数据的getter触发dep.depend()，相当于执行watcher的addDep，最后还是调用dep.addSub()，完成了依赖收集
  - finally块中：
    - 递归去访问 value，触发它所有子项的 getter
    - 把 Dep.target 恢复成上一个状态（会执行Dep.target = targetStack.pop()）
    - cleanupDeps清空依赖（清除不在newDeps中但在deps中的dep，即已不需要的dep，与watcher的订阅关系）

```javascript
// src/core/observer/watcher.js

let uid = 0

export default class Watcher {
  //......

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
      
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn     // 注意这里定义了this.getter，是传入的updateComponent
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    
     // 注意pushTarget从dep.js中引入的
    // 它的作用是将这个Dep.target指向这个watcher实例
    pushTarget(this) 
    
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm) // this.getter是new Watcher时传入的updateComponent
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // 递归去访问 value，触发它所有子项的 getter
      if (this.deep) {
        traverse(value)
      }
      // 把 Dep.target 恢复成上一个状态
      // 因为当前 vm 的数据依赖收集已经完成，那么对应的渲染Dep.target 也需要改变
      // popTarget会执行：Dep.target = targetStack.pop()
      popTarget()
      // 清空依赖
      this.cleanupDeps()
    }
    return value
  }

  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {  // 保证同一数据不会被添加多次
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          const info = `callback for watcher "${this.expression}"`
          invokeWithErrorHandling(this.cb, this.vm, [value, oldValue], this.vm, info)
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }

  /**
   * Depend on all deps collected by this watcher.
   */
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }

  /**
   * Remove self from all dependencies' subscriber list.
   */
  teardown () {
    if (this.active) {
      // remove self from vm's watcher list
      // this is a somewhat expensive operation so we skip it
      // if the vm is being destroyed.
      if (!this.vm._isBeingDestroyed) {
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}
```

cleanupDeps的处理步骤：
- 遍历deps，找出不在newDeps中的dep。因为newDeps存放的是新增的dep，所以不在newDeps中而在deps中的dep是已经不需要的dep
- 将当前watcher从该dep中移除
- 将newDeps赋给deps，newDeps清空

相关例子：
- 通过vm.$watch新建了一个watcher，当this.a为1时会依赖this.a、this.b两个dep，不会依赖this.c
- 当this.a变为2时，重新依赖收集，那么this.b变为不需要的依赖，需要将这个dep与该watcher解除依赖关系

```javascript
vm.$watch(function () {
  return this.a === 1 ? this.b : this.c
}, function () {
  console.log('watcher 改变啦！')
})
```

#### 8.2.4 defineReactive
defineReactive的作用就是利用Object.defineProperty来定义响应式数据：
- 初始化Dep对象
- 拿到传入对象obj的属性描述符
- 如果开启了深度，对val递归监听
- 使用Object.defineProperty劫持数据的getter、setter：
  - 依赖收集：当数据被访问时，触发getter，调用dep.depend()将watcher添加到该dep的subs数组，完成依赖收集
  - 派发更新：当数据被修改时，触发setter，会再次进行依赖收集，且调用dep.notify()通知subs数组里的watcher更新

```javascript
// src/core/observer/index.js
// defineReactive

export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // 如果开启了深度，对val递归监听
  let childOb = !shallow && observe(val)
  
  // 定义getter、setter
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      // Dep.target 相当于 watcher
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```

### 8.3 总结

![响应式原理.png](img/响应式原理.png)