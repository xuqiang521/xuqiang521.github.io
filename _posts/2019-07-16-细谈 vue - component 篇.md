---
layout: post
title: "细谈 vue - component 篇"
date: 2019-07-16
description: "细谈 vue - component 篇"
tag: vue 源码
---

本篇文章是细谈 vue 系列的第六篇。看过我这个系列文章的小伙伴都知道：文章贼长，看不下去的建议先点个赞当收藏，然后等有时间静下心来慢慢看，前端交流群：731175396。以前的文章传送门如下

- [《细谈 vue 核心 - vdom 篇》](https://juejin.im/post/5cab347fe51d456e7a303b3d)
- [《细谈 vue - slot 篇》](https://juejin.im/post/5cced0096fb9a032426510ad)
- [《细谈 vue - transition 篇》](<https://juejin.im/post/5cf411d8e51d4550a629b222>)
- [《细谈 vue - transition-group 篇》](<https://juejin.im/post/5cf7a11f51882528394f22cd>)
- [《细谈 vue - 抽象组件实战篇》](https://juejin.im/post/5cfb8f3cf265da1bc14b1b07)

用过 `vue` 的小伙伴肯定知道，在 `vue` 的开发中，`component` 可谓是随处可见，项目中的那一个个 `.vue (SFC) ` 文件，可不就是一个个的组件么。

那么，既然 `component` 这么核心，这么重要，为何不好好来研究一波呢？

>  why not ？
>
>  ​           — 鲁迅


## 一、组件创建

之前我们分析 `vdom` 的时候分析过一个函数 `createElement`，与它相同的是 `createComponent`，两者都是用来创建 `vnode` 节点的，如果是普通的 `html` 标签，则直接实例化一个普通的 `vnode` 节点，否则通过 `createComponent` 来创建一个 `Component` 类型的 `vnode` 节点

### 1、createElement

这里仅列出不同情况下 `vnode` 节点创建的代码

```javascript
if (typeof tag === 'string') {
  let Ctor
  ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
  if (config.isReservedTag(tag)) {
    // platform built-in elements
    vnode = new VNode(
      config.parsePlatformTagName(tag), data, children,
      undefined, undefined, context
    )
  } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
    // component
    vnode = createComponent(Ctor, data, context, children, tag)
  } else {
    // unknown or unlisted namespaced elements
    // check at runtime because it may get assigned a namespace when its
    // parent normalizes children
    vnode = new VNode(
      tag, data, children,
      undefined, undefined, context
    )
  }
} else {
  // direct component options / constructor
  vnode = createComponent(tag, data, context, children)
}
```

### 2、createComponent

接下来，我们先看 `createComponent()` 的定义，具体如下

```javascript
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

  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // if at this stage it's not a constructor or an async component factory,
  // reject.
  if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
  }

  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
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
    // abstract components do not keep anything
    // other than props & listeners & slot

    // work around flow
    const slot = data.slot
    data = {}
    if (slot) {
      data.slot = slot
    }
  }

  // install component management hooks onto the placeholder node
  installComponentHooks(data)

  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  // Weex specific: invoke recycle-list optimized @render function for
  // extracting cell-slot template.
  // https://github.com/Hanks10100/weex-native-directive/tree/master/component
  /* istanbul ignore if */
  if (__WEEX__ && isRecyclableComponent(vnode)) {
    return renderRecyclableComponentTemplate(vnode)
  }

  return vnode
}
```

- 在其内部，第一件事情就是将构造函数 `Vue` 赋值给变量 `baseCtor` ，并通过 `extend` 将参数 `Ctor` 进行扩展

```javascript
const baseCtor = context.$options._base
if (isObject(Ctor)) {
  Ctor = baseCtor.extend(Ctor)
}
```

这里我们看到 `$options._base` ，其实就是构造函数 `Vue`

```javascript
// src/core/global-api/index.js
Vue.options._base = Vue

// src/core/instance/init.js
// 1. initMixin()
if (options && options._isComponent) {
  initInternalComponent(vm, options)
} else {
  vm.$options = mergeOptions(
    resolveConstructorOptions(vm.constructor),
    options || {},
    vm
  )
}
// 2. initInternalComponent()
const opts = vm.$options = Object.create(vm.constructor.options)
```

- 其次，紧接着，判定组件是否为异步组件、函数式组件或者抽象组件。具体每种情况的处理后面我再详细分析

```javascript
// 异步组件
let asyncFactory
if (isUndef(Ctor.cid)) {
  asyncFactory = Ctor
  Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
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

// 函数式组件
if (isTrue(Ctor.options.functional)) {
  return createFunctionalComponent(Ctor, propsData, data, context, children)
}

// 抽象组件
if (isTrue(Ctor.options.abstract)) {
  const slot = data.slot
  data = {}
  if (slot) {
    data.slot = slot
  }
}
```

- 对于组件上的事件也有相关处理，它会提取组件上的事件监听器。它需要作为子组件的监听器，而并非DOM监听器。所以需要将其替换为拥有 `.native` 修饰符的侦听器，让其能在父组件 patch 阶段能够得到处理

```javascript
const listeners = data.on
data.on = data.nativeOn
```

- 然后，安装组件的钩子函数。它将 `componentVNodeHooks` 的钩子函数合并到 `data.hook` 中，然后 `Component` 类型的 `vnode` 节点在 `patch` 过程中会执行相关的钩子函数，如果某个时机的钩子函数已经存在，则通过 `mergeHook` 将函数合并，即依次执行同一时机的这两个函数

```javascript
installComponentHooks(data)

function installComponentHooks (data: VNodeData) {
  const hooks = data.hook || (data.hook = {})
  for (let i = 0; i < hooksToMerge.length; i++) {
    const key = hooksToMerge[i]
    const existing = hooks[key]
    const toMerge = componentVNodeHooks[key]
    if (existing !== toMerge && !(existing && existing._merged)) {
      hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge
    }
  }
}

const hooksToMerge = Object.keys(componentVNodeHooks)

function mergeHook (f1: any, f2: any): Function {
  const merged = (a, b) => {
    f1(a, b)
    f2(a, b)
  }
  merged._merged = true
  return merged
}
```

### 3、componentVNodeHooks

上面的 `componentVNodeHooks` 则是组件初始化的时候实现的几个钩子函数，分别有 `init`、`prepatch`、`insert`、`destroy` 

```javascript
const componentVNodeHooks = {
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      const mountedNode: any = vnode
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },

  prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    const options = vnode.componentOptions
    const child = vnode.componentInstance = oldVnode.componentInstance
    updateChildComponent(
      child,
      options.propsData, // updated props
      options.listeners, // updated listeners
      vnode, // new parent vnode
      options.children // new children
    )
  },

  insert (vnode: MountedComponentVNode) {
    const { context, componentInstance } = vnode
    if (!componentInstance._isMounted) {
      componentInstance._isMounted = true
      callHook(componentInstance, 'mounted')
    }
    if (vnode.data.keepAlive) {
      if (context._isMounted) {
        queueActivatedComponent(componentInstance)
      } else {
        activateChildComponent(componentInstance, true /* direct */)
      }
    }
  },

  destroy (vnode: MountedComponentVNode) {
    const { componentInstance } = vnode
    if (!componentInstance._isDestroyed) {
      if (!vnode.data.keepAlive) {
        componentInstance.$destroy()
      } else {
        deactivateChildComponent(componentInstance, true /* direct */)
      }
    }
  }
}
```

接下来我们来仔细看看 `componentVNodeHooks` 里面的四个钩子函数都做了些什么

1. `init` ：当 `vnode` 为 `keep-alive` 组件时、存在实例且没被销毁，为了防止组件流动，直接执行了 `prepatch`。否则直接通过执行 `createComponentInstanceForVnode` 创建一个 `Component` 类型的 `vnode` 实例，并进行 `$mount` 操作
2. `prepatch`：将已有组件更新成最新的 `vnode` 上的数据，这里没啥好说的
3. `insert`：`insert` 钩子函数
   - 首先会判定组件实例是否已经被 `mounted`，若没被渲染，则直接将 `componentInstance` 作为参数执行 `mounted` 钩子函数。
   - 其次，则是组件为 `keep-alive` 内置组件的情况。这里有个操作有点骚，就是当它已经 `mounted` 了的时候，进入 `insert` 阶段的时候，为了防止 `keep-alive` 子组件更新触发 `activated` 钩子函数，直接就放弃了 `walking tree` 的更新机制，而是直接将组件实例 `componentInstance` 丢到 `activatedChildren` 这个数组中。当然没有 `mounted` 的情况则直接触发 `activated` 钩子函数进行 `mounted` 即可
4. `destroy`：组件销毁操作，这里同样对 `keep-alive` 组件做了兼容。如果不是 `keep-alive` 组件，直接执行 `$destory` 销毁组件实例，否则触发 `deactivated` 钩子函数进行销毁。

上面用的一些辅助函数如下

```javascript
export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}
```

- 最后实例化 `VNode`，然后返回

```javascript
const name = Ctor.options.name || tag
const vnode = new VNode(
  `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
  data, undefined, undefined, undefined, context,
  { Ctor, propsData, listeners, tag, children },
  asyncFactory
)
return vnode
```

`createComponent` 的全部过程就是：首先先构建 `Vue` 的子类构造函数，然后安装组件的钩子函数，最后实例化 `VNode`，然后返回。里面的很多操作都对 `keep-alive` 内置组件做了很多兼容。所以假如你用过 `keep-alive` 组件，并且恰巧看到这，相信你会有很多感悟。

## 二、配置合并

通常来说，设计一款插件或者组件，为了保证其可定制化、可扩展性，一般会在自身定义一些默认配置，然后在内部做好 `merge` 配置项的操作，让你能在其初始化阶段进行自定义的配置。

当然，`Vue` 在这块设计也是如此。`vue` 中对于  `options` 合并策略其实我上面也列出过代码，具体在 `src/core/instance/init.js` 中（这里我只保留相关代码）。

```javascript
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // ...
    // merge options
    if (options && options._isComponent) {
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    // ...
  }
}
```

能看出来，合并策略有两个。一种是为 `Component` 组件的情况下，执行 `initInternalComponent` 进行内部组件配置合并，一种是非组件的情况，直接通过 `mergeOptions` 做配置合并。

### 1、normal merge

这里直接将 `resolveConstructorOptions(vm.constructor)` 的返回值和 `options` 进行合并

```javascript
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

我们先来看下 `Vue.options` 的定义

```javascript
// src/core/global-api/index.js
export function initGlobalAPI (Vue: GlobalAPI) {
	// ...
  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })
  // ...
}

// src/shared/constants.js
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
```

接着，我们再来看看 `mergeOptions` 的逻辑：它是 `vue`  核心合并策略之一，它主要功能就是将 `parant` 和 `child` 进行策略合并，然后返回一个新的对象，代码在 `src/core/util/options.js` 中。

1. 首先会先对 `child` 上面的 `props`、`inject`、`directives` 进行 `object format` 操作（具体逻辑可自行研究，主要就是对其进行 `object` 转换操作）
2. 若 `child._base` 不存在，遍历 `child.extends` 和 `child.mixins` ，将其合并到 `parent` 上
3. 遍历 `parent`，调用 `mergeField` 合并到变量 `options` 上
4. 遍历 `child`，若 `child` 有 `parent` 不存在的属性，则调用 `mergeField` 将该属性合并到 `options` 上

```javascript
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)

  if (!child._base) {
    if (child.extends) {
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }

  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```

`vue` 中除了对 `options` 的合并外，还有很多合并策略，感兴趣的可以自己去 `src/core/util/options.js` 中查阅研究

### 2、component merge

在分析 `createComponent` 的时候我们了解到组件的构造函数是通过 `Vue.extend` 对 `Vue` 进行继承的，代码如下

```javascript
// src/core/global-api/index.js
Vue.options._base = Vue
// src/core/vdom/create-component.js
const baseCtor = context.$options._base
if (isObject(Ctor)) {
  Ctor = baseCtor.extend(Ctor)
}
```

我们再来 `Vue.extend`， 它的定义在 `src/core/global-api/extend.js` 中（仅保留关键逻辑），它通过执行 `mergeOptions()` 将 `Super.options`，即 `Vue.options` 合并到 `Sub.options` 中

```javascript
export function initExtend (Vue: GlobalAPI) {
	// ...
  Vue.extend = function (extendOptions: Object): Function {
    extendOptions = extendOptions || {}
    const Super = this
    // ...
    const Sub = function VueComponent (options) {
      this._init(options)
    }
    // ...
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
    // ...
    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({}, Sub.options)

    // ...
    return Sub
  }
}
```

然后在 `componentVNodeHooks` 的 `init` 钩子函数中，即子组件的初始化阶段，会执行 `createComponentInstanceForVnode` 进行组件实例的初始化。`createComponentInstanceForVnode` 函数中的 `vnode.componentOptions.Ctor` 指向的其实就是上面 `Vue.extend` 中返回的 `Sub`，所以执行 `new` 操作的时候会执行到 `this._init(options)`，即 `Vue._init(options)` 操作，又因为 `options._isComponent` 的定义是 `true`，所以直接进入了 `initInternalComponent` 操作

```javascript
// componentVNodeHooks init()
const child = vnode.componentInstance = createComponentInstanceForVnode(
  vnode,
  activeInstance
)
// createComponentInstanceForVnode()
export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }
  // ...
  return new vnode.componentOptions.Ctor(options)
}
```

`initInternalComponent` 只是做了一些简单的对象赋值，具体我就不分析了，代码如下：

```javascript
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```

讲到这，可能有些小伙伴会有些懵，我举个例子来说明下

```html
<template>
  <div class="hello">
    {{ msg }}
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  created () {
    console.log('this is child')
  }
}
</script>
```

然后在父组件进行调用

```html
<template>
  <div class="home">
    <HelloWorld msg="Welcome to Your Vue.js App"/>
  </div>
</template>

<script>
import HelloWorld from '@/components/HelloWorld.vue'

export default {
  name: 'home',
  components: {
    HelloWorld
  },
  created () {
    console.log('this is parent')
  }
}
</script>
```

走完上面的合并策略后，`vm.$options` 的值大致如下

```javascript
vm.$options = {
  parent: VueComponent, // 父组件实例
  propsData: {
    msg: 'Welcome to Your Vue.js App'
  },
  _componentTag: 'HelloWorld',
  _parentListeners: undefined,
  _parentVnode: VNode, // 父节点 vnode 实例
  _propKeys: ['msg'],
  _renderChildren: undefined,
  __proto__: {
    components: {
      HelloWorld: function VueComponent(options) {}
    },
    directives: {},
    filters: {},
    _base: function Vue(options) {},
    _Ctor: {},
    created: [
      function created() {
        console.log('this is parent')
      },
      function created() {
        console.log('this is child')
      }
    ]
  }
}
```

## 三、异步组件

在上面分析 `createComponent` 的时候，我们留下几种特殊情况没有分析，其中一种就是异步组件的情况。它的场景是，当 `Ctor.cid` 未定义的情况下，则直接走异步组件创建的流程，具体代码如下

```javascript
let asyncFactory
if (isUndef(Ctor.cid)) {
  asyncFactory = Ctor
  Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
  if (Ctor === undefined) {
    // return a placeholder node for async component, which is rendered
    // as a comment node but preserves all the raw information for the node.
    // the information will be used for async server-rendering and hydration.
    return createAsyncPlaceholder(
      asyncFactory,
      data,
      context,
      children,
      tag
    )
  }
}
```

在做具体分析前，我们先通过官方示例来看下异步组件的用法和普通组件的用法有何不同

```javascript
// 普通组件
Vue.component('my-component-name', {
  // ... options ...
})

// 异步组件
Vue.component('async-webpack-example', function (resolve, reject) {
  // 这个特殊的 require 语法
  // 将指示 webpack 自动将构建后的代码，
  // 拆分到不同的 bundle 中，然后通过 Ajax 请求加载。
  require(['./my-async-component'], resolve)
})
```

在例子中，`Vue` 普通组件是一个对象，而异步组件则是一个工厂函数，它接收2个参数，一个 `resolve` 回调函数用来从服务器获取到组件定义的对象，另外一个 `reject` 回调函数来表明加载失败。除了上面的写法外，异步组件还支持以下两种写法

```javascript
// Promise 异步组件
Vue.component(
  'async-webpack-example',
  // `import` 函数返回一个 Promise.
  () => import('./my-async-component')
)

// 高级异步组件
const AsyncComponent = () => ({
  // 加载组件（最终应该返回一个 Promise）
  component: import('./MyComponent.vue'),
  // 异步组件加载中(loading)，展示为此组件
  loading: LoadingComponent,
  // 加载失败，展示为此组件
  error: ErrorComponent,
  // 展示 loading 组件之前的延迟时间。默认：200ms。
  delay: 200,
  // 如果提供 timeout，并且加载用时超过此 timeout，
  // 则展示错误组件。默认：Infinity。
  timeout: 3000
})
Vue.component('async-component', AsyncComponent)
```

### 1、resolveAsyncComponent

`resolveAsyncComponent` 主要功能就是对上面提及的 3 种异步组件创建方式进行支持，具体代码如下

```javascript
export function resolveAsyncComponent (
  factory: Function,
  baseCtor: Class<Component>
): Class<Component> | void {
  if (isTrue(factory.error) && isDef(factory.errorComp)) {
    return factory.errorComp
  }

  if (isDef(factory.resolved)) {
    return factory.resolved
  }

  const owner = currentRenderingInstance
  if (owner && isDef(factory.owners) && factory.owners.indexOf(owner) === -1) {
    // already pending
    factory.owners.push(owner)
  }

  if (isTrue(factory.loading) && isDef(factory.loadingComp)) {
    return factory.loadingComp
  }

  if (owner && !isDef(factory.owners)) {
    const owners = factory.owners = [owner]
    let sync = true
    let timerLoading = null
    let timerTimeout = null

    ;(owner: any).$on('hook:destroyed', () => remove(owners, owner))

    const forceRender = (renderCompleted: boolean) => {
      for (let i = 0, l = owners.length; i < l; i++) {
        (owners[i]: any).$forceUpdate()
      }

      if (renderCompleted) {
        owners.length = 0
        if (timerLoading !== null) {
          clearTimeout(timerLoading)
          timerLoading = null
        }
        if (timerTimeout !== null) {
          clearTimeout(timerTimeout)
          timerTimeout = null
        }
      }
    }

    const resolve = once((res: Object | Class<Component>) => {
      // cache resolved
      factory.resolved = ensureCtor(res, baseCtor)
      // invoke callbacks only if this is not a synchronous resolve
      // (async resolves are shimmed as synchronous during SSR)
      if (!sync) {
        forceRender(true)
      } else {
        owners.length = 0
      }
    })

    const reject = once(reason => {
      process.env.NODE_ENV !== 'production' && warn(
        `Failed to resolve async component: ${String(factory)}` +
        (reason ? `\nReason: ${reason}` : '')
      )
      if (isDef(factory.errorComp)) {
        factory.error = true
        forceRender(true)
      }
    })

    const res = factory(resolve, reject)

    if (isObject(res)) {
      if (isPromise(res)) {
        // () => Promise
        if (isUndef(factory.resolved)) {
          res.then(resolve, reject)
        }
      } else if (isPromise(res.component)) {
        res.component.then(resolve, reject)

        if (isDef(res.error)) {
          factory.errorComp = ensureCtor(res.error, baseCtor)
        }

        if (isDef(res.loading)) {
          factory.loadingComp = ensureCtor(res.loading, baseCtor)
          if (res.delay === 0) {
            factory.loading = true
          } else {
            timerLoading = setTimeout(() => {
              timerLoading = null
              if (isUndef(factory.resolved) && isUndef(factory.error)) {
                factory.loading = true
                forceRender(false)
              }
            }, res.delay || 200)
          }
        }

        if (isDef(res.timeout)) {
          timerTimeout = setTimeout(() => {
            timerTimeout = null
            if (isUndef(factory.resolved)) {
              reject(
                process.env.NODE_ENV !== 'production'
                  ? `timeout (${res.timeout}ms)`
                  : null
              )
            }
          }, res.timeout)
        }
      }
    }

    sync = false
    // return in case resolved synchronously
    return factory.loading
      ? factory.loadingComp
      : factory.resolved
  }
}
```

首先我们先来看看对于异步组件是如何加载的。这里我们先跳过 `resolveAsyncComponent` 一开始就对我们前面提及的高级异步组件做的处理。

在分析 `resolveAsyncComponent` 异步组件创建逻辑前，我们先过看看其中会用到的一些核心的方法

- `forceRender`：对组件强制进行重新渲染，然后在 `render` 完成的时候清掉工厂函数中当前的渲染实例 `owners`，顺带把 `timerLoading` 和 `timerTimeout` 清除掉。

   `$forceUpdate`：调用 `watcher` 的 `update` 方法，即组件的重新渲染。对 `vue` 中一般只有数据变更才会触发视图的重新渲染，而异步组件在加载过程中数据是不会发生变化的，那么这个时候是不会触发组件重新渲染的，所以需要通过执行 `$forceUpdate` 强制对组件进行重新渲染

```javascript
const forceRender = (renderCompleted: boolean) => {
  for (let i = 0, l = owners.length; i < l; i++) {
    (owners[i]: any).$forceUpdate()
  }

  if (renderCompleted) {
    owners.length = 0
    if (timerLoading !== null) {
      clearTimeout(timerLoading)
      timerLoading = null
    }
    if (timerTimeout !== null) {
      clearTimeout(timerTimeout)
      timerTimeout = null
    }
  }
}

Vue.prototype.$forceUpdate = function () {
  const vm: Component = this
  if (vm._watcher) {
    vm._watcher.update()
  }
}
```

- `once`：利用闭包以及一个标识变量 `called` 保证其包装的函数只会执行一次

```javascript
export function once (fn: Function): Function {
  let called = false
  return function () {
    if (!called) {
      called = true
      fn.apply(this, arguments)
    }
  }
}
```

- `resolve`：内部 `resolve` 函数，首先会执行 `ensureCtor` 并将其返回值作为 `factory` 的 `resolved` 值。紧接着若  `sync` 异步变量为 `false` ，则直接执行 `forceRender` 强制让组件重新渲染，否则则清空 `owners`

  `ensureCtor` 则是为了保证能找到异步组件上定义的组件对象而定义的函数。如果发现它是普通对象，则直接通过 `Vue.extend` 将其转换成组件的构造函数

```javascript
const resolve = once((res: Object | Class<Component>) => {
  factory.resolved = ensureCtor(res, baseCtor)
  if (!sync) {
    forceRender(true)
  } else {
    owners.length = 0
  }
})
function ensureCtor (comp: any, base) {
  if (
    comp.__esModule ||
    (hasSymbol && comp[Symbol.toStringTag] === 'Module')
  ) {
    comp = comp.default
  }
  return isObject(comp)
    ? base.extend(comp)
    : comp
}
```

- `reject`：内部 `reject` 函数，异步组件加载失败时执行

```javascript
const reject = once(reason => {
  process.env.NODE_ENV !== 'production' && warn(
    `Failed to resolve async component: ${String(factory)}` +
    (reason ? `\nReason: ${reason}` : '')
  )
  if (isDef(factory.errorComp)) {
    factory.error = true
    forceRender(true)
  }
})
```

看完其中的核心方法后，接下来我们具体异步组件是如何创建的。

1. 我们从 `resolveAsyncComponent` 的定义中知道该方法接收 2 个参数，一个是 `factory` 工厂函数，一个是 `baseCtor` ，即 `Vue`。
2. 然后在当前渲染实例存在、且在 `factory.owners` 中存在的情况下，即组件进入 `pending` 阶段，则直接将当前实例丢到 `factory.owners` 中。
3. 然而，初始化异步组件的时候 `factory` 是不会有 `owners` 滴，那这个时候又该怎么办呢？很简单呗，直接执行 `factory` 工厂函数，并把内部定义的 `resolve` 和 `reject` 函数作为其参数，这样我们就能直接通过 `resolve` 和 `reject` 做点事了，这些逻辑也正是对普通异步组件支持的逻辑，相关代码如下

```javascript
const owner = currentRenderingInstance
if (owner && isDef(factory.owners) && factory.owners.indexOf(owner) === -1) {
  // already pending
  factory.owners.push(owner)
}
if (owner && !isDef(factory.owners)) {
  const owners = factory.owners = [owner]
  let sync = true
  ;(owner: any).$on('hook:destroyed', () => remove(owners, owner))
  const forceRender = (renderCompleted: boolean) => {
    // ...
  }
  const resolve = once((res: Object | Class<Component>) => {
    // ...
  })
  const reject = once(reason => {
    // ...
  })
  const res = factory(resolve, reject)
  // ...
}
```

- `Promise` 异步组件

在 `vue` 中，你可以使用 `webpack2+` + ES6 的方式来异步加载组件，如下

```javascript
Vue.component(
  'async-webpack-example',
  // `import` 函数返回一个 Promise.
  () => import('./my-async-component')
)
```

即执行完 `res = factory(resolve, reject)` 时，`res` 的值为 `import('./my-async-component')` 的返回值，是一个 `Promise` 对象。之后进入 `Promise` 异步组件的处理逻辑，异步组件加载成功后执行 `resolve`，加载失败则执行 `reject`

```javascript
const res = factory(resolve, reject)
if (isPromise(res)) {
  // () => Promise
  if (isUndef(factory.resolved)) {
    res.then(resolve, reject)
  }
}
```

- 高级异步组件

其实这里所谓的高级，其实就是 `vue` 在 2.3.0+ 版本新增了加载状态处理的功能，即抛出了一些可配置的字段给用户，其中有 `component`、 `loading`、`error`、`delay`、`timeout`，其中  `component` 支持 `Promise` 异步组件加载的形式，具体案例代码如下

```javascript
const AsyncComponent = () => ({
  // 加载组件（最终应该返回一个 Promise）
  component: import('./MyComponent.vue'),
  // 异步组件加载中(loading)，展示为此组件
  loading: LoadingComponent,
  // 加载失败，展示为此组件
  error: ErrorComponent,
  // 展示 loading 组件之前的延迟时间。默认：200ms。
  delay: 200,
  // 如果提供 timeout，并且加载用时超过此 timeout，
  // 则展示错误组件。默认：Infinity。
  timeout: 3000
})
Vue.component('async-component', AsyncComponent)
```

和刚分析 `Promise` 异步组件加载逻辑一样，若执行完 `res = factory(resolve, reject)` ，`res.component` 的返回值是 `Promise` 的话，直接执行 `then` 方法，代码如下

```javascript
else if (isPromise(res.component)) {
  res.component.then(resolve, reject)
}
```

紧接着就是对其它 4 个可配置字段的处理

1. 首先判定是否自定义了 `error` 组件，如果有，执行 `ensureCtor(res.error, baseCtor)` 并将返回值直接赋值给 `factory.errorComp`
2. 同理若传入了 `loading` 组件，则执行 `ensureCtor(res.loading, baseCtor)` 并将返回值直接赋值给 `factory.loadingComp`
3. 紧接着，在定义了 `loading` 组件的逻辑中，若设置了 `delay` 值为 0，则直接将 `factory.loading` 值设为 `true`，否则延时 `delay` 执行，`delay` 未设置，延时默认为 200ms
4. 最后，若设置了组件加载的 `timeout` 加载时长的话，若组件在 `res.timeout` 时间后还未加载成功，则直接执行 `reject` 进行抛错

```javascript
if (isDef(res.error)) {
  factory.errorComp = ensureCtor(res.error, baseCtor)
}

if (isDef(res.loading)) {
  factory.loadingComp = ensureCtor(res.loading, baseCtor)
  if (res.delay === 0) {
    factory.loading = true
  } else {
    timerLoading = setTimeout(() => {
      timerLoading = null
      if (isUndef(factory.resolved) && isUndef(factory.error)) {
        factory.loading = true
        forceRender(false)
      }
    }, res.delay || 200)
  }
}

if (isDef(res.timeout)) {
  timerTimeout = setTimeout(() => {
    timerTimeout = null
    if (isUndef(factory.resolved)) {
      reject(
        process.env.NODE_ENV !== 'production'
        ? `timeout (${res.timeout}ms)`
        : null
      )
    }
  }, res.timeout)
}
```

然后最后通过判定 `factory.loading` 进行不同值的返回，从上面自定义字段 `loading` 的处理我们得知，若自定义字段 `delay` 设为 0，则说明这次直接渲染 `loading` 组件，否则会直接延时并执行到 `forceRender` 方法，这样就会触发组件的重新渲染，从而再次执行 `resolveAsyncComponent`

```javascript
sync = false
return factory.loading
  ? factory.loadingComp
	: factory.resolved
```

然后我们再次回到 `resolveAsyncComponent` 开篇被我们跳过的一些操作

```javascript
if (isTrue(factory.error) && isDef(factory.errorComp)) {
  return factory.errorComp
}

if (isDef(factory.resolved)) {
  return factory.resolved
}

if (isTrue(factory.loading) && isDef(factory.loadingComp)) {
  return factory.loadingComp
}
```

### 2、createAsyncPlaceholder

从上面的对 `resolveAsyncComponent` 的分析中，我们得知，如果是第一次执行 `resolveAsyncComponent` ，返回值会是 `undefined`，当然，你将 `delay` 值设置为 0 的时候除外。为了避免 `Ctor` 为 `undefined` 时，导致节点信息无法捕获的情况，会直接通过 `createAsyncPlaceholder` 创建一个注释的 `vnode` 节点，作为异步组件的占位符，同时用来保留该 `vnode` 节点所有的原始信息。具体代码如下

```javascript
export function createAsyncPlaceholder (
  factory: Function,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag: ?string
): VNode {
  const node = createEmptyVNode()
  node.asyncFactory = factory
  node.asyncMeta = { data, context, children, tag }
  return node
}
```

## 四、函数式组件

分析 `createComponent` 组件创建的时候我们还留下一个问题没讲，那就是 `functional component`（函数式组件），具体场景如下

```javascript
// functional component
if (isTrue(Ctor.options.functional)) {
  return createFunctionalComponent(Ctor, propsData, data, context, children)
}
```

分析前，防止有小伙伴不是很了解函数式组件是啥，我先列举两个官方支持的函数式组件写法

```javascript
// 方式一 render function
Vue.component('my-component', {
  functional: true,
  // Props 是可选项
  props: {
    // ...
  },
  // 为了弥补缺少的实例
  // 我们提供了第二个参数 context 作为上下文
  render: function (createElement, context) {
    // ...
  }
})

// 方式二 template functional
<template functional></template>
```

想了解更多的，可以直接去[官方文档](https://vuejs.org/v2/guide/render-function.html#Functional-Components)先仔细阅读。

函数式组件官方定义：组件被标记成 `functional`，它无状态，无响应式 `data`，无实例，即没有 `this` 上下文。

下面我们就来揭开函数式组件的面纱。

### 1、createFunctionalComponent

`createFunctionalComponent` 主要核心分为三步

1. 将 `Ctor.options` 中的 `props` 合并到新对象 `props` 中。若 `Ctor.options` 存在 `props`，直接遍历其 `props`，执行 `validateProp` 对 `Ctor.options.props` 当前属性进行校验并将当前属性复制给 `props[key]`。若 `Ctor.options.props` 未定义，则将 `data` 上定义好的 `attrs` 和 `props` 通过执行 `mergeProps` 函数合并到新对象 `props` 上。
2. 执行 `new FunctionalRenderContext` 实例化 `functional` 组件的上下文，并执行 `options` 上的 `render` 函数实例化 `vnode` 节点
3. 对实例化的 `vnode` 进行特殊的克隆操作并进行返回

```javascript
export function createFunctionalComponent (
  Ctor: Class<Component>,
  propsData: ?Object,
  data: VNodeData,
  contextVm: Component,
  children: ?Array<VNode>
): VNode | Array<VNode> | void {
  const options = Ctor.options
  const props = {}
  const propOptions = options.props
  if (isDef(propOptions)) {
    for (const key in propOptions) {
      props[key] = validateProp(key, propOptions, propsData || emptyObject)
    }
  } else {
    if (isDef(data.attrs)) mergeProps(props, data.attrs)
    if (isDef(data.props)) mergeProps(props, data.props)
  }

  const renderContext = new FunctionalRenderContext(
    data,
    props,
    children,
    contextVm,
    Ctor
  )

  const vnode = options.render.call(null, renderContext._c, renderContext)

  if (vnode instanceof VNode) {
    return cloneAndMarkFunctionalResult(vnode, data, renderContext.parent, options, renderContext)
  } else if (Array.isArray(vnode)) {
    const vnodes = normalizeChildren(vnode) || []
    const res = new Array(vnodes.length)
    for (let i = 0; i < vnodes.length; i++) {
      res[i] = cloneAndMarkFunctionalResult(vnodes[i], data, renderContext.parent, options, renderContext)
    }
    return res
  }
}
```

上面提及的两个辅助函数如下

1. `cloneAndMarkFunctionalResult` ：为了避免复用节点，`fnContext` 导致命名槽点不匹配的情况，直接在设置 `fnContext` 之前克隆节点，最后返回克隆好的 `vnode`
2. `mergeProps`： `props` 合并策略

```javascript
function cloneAndMarkFunctionalResult (vnode, data, contextVm, options, renderContext) {
  const clone = cloneVNode(vnode)
  clone.fnContext = contextVm
  clone.fnOptions = options
  if (process.env.NODE_ENV !== 'production') {
    (clone.devtoolsMeta = clone.devtoolsMeta || {}).renderContext = renderContext
  }
  if (data.slot) {
    (clone.data || (clone.data = {})).slot = data.slot
  }
  return clone
}

function mergeProps (to, from) {
  for (const key in from) {
    to[camelize(key)] = from[key]
  }
}
```

### 2、FunctionalRenderContext

从文档中我们知道函数式组件支持两种书写方式，第一种是 `render function` 的方式，另外一种则是 `<template functional>` 单文件组件的方式。`render function` 的方式在 `createFunctionalComponent` 的处理中已经做了支持，它会直接执行 `Ctor.options` 上的 `render` 方法。在函数式组件渲染上下文构造函数 `FunctionalRenderContext` 中则是对 `<template functional>` 单文件组件的方式也进行了支持。

1. 首先，它为了确保函数式组件的 `createElement` 函数能够获得唯一的上下文，将克隆的 `parent` 对象赋值给上下文 `vm` 变量 `contextVm`，`contextVm._original` 则赋值为 `parent`，当做其上下文来源的标记。其中有种比较临界的情况就是，若传入的上下文 `vm` 也是函数式上下文，这该怎么办呢？其实只要按照 `_uid` 存在的情况来逆向推动下逻辑即可，`contextVm` 接收 `parent`，`parent` 接收 `parent._original` 即可，因为往上继续找，总能找着存在 `_uid` 的 `parent` 不是。
2. 接下来就是对函数式组件中 `data`、`props`、`listeners`、`injections` 等进行支持处理，这里对于 `slots` 做了一层转换处理，即将 `normal slots` 对象转换成 `scoped slots`
3. 最后对 `options._scopeId` 存在与否的场景进行不同的 `createElement` 节点创建操作

```javascript
export function FunctionalRenderContext (
  data: VNodeData,
  props: Object,
  children: ?Array<VNode>,
  parent: Component,
  Ctor: Class<Component>
) {
  const options = Ctor.options
  let contextVm
  if (hasOwn(parent, '_uid')) {
    contextVm = Object.create(parent)
    contextVm._original = parent
  } else {
    contextVm = parent
    parent = parent._original
  }
  const isCompiled = isTrue(options._compiled)
  const needNormalization = !isCompiled

  this.data = data
  this.props = props
  this.children = children
  this.parent = parent
  this.listeners = data.on || emptyObject
  this.injections = resolveInject(options.inject, parent)
  this.slots = () => {
    if (!this.$slots) {
      normalizeScopedSlots(
        data.scopedSlots,
        this.$slots = resolveSlots(children, parent)
      )
    }
    return this.$slots
  }

  Object.defineProperty(this, 'scopedSlots', ({
    enumerable: true,
    get () {
      return normalizeScopedSlots(data.scopedSlots, this.slots())
    }
  }: any))

  // support for compiled functional template
  if (isCompiled) {
    // exposing $options for renderStatic()
    this.$options = options
    // pre-resolve slots for renderSlot()
    this.$slots = this.slots()
    this.$scopedSlots = normalizeScopedSlots(data.scopedSlots, this.$slots)
  }

  if (options._scopeId) {
    this._c = (a, b, c, d) => {
      const vnode = createElement(contextVm, a, b, c, d, needNormalization)
      if (vnode && !Array.isArray(vnode)) {
        vnode.fnScopeId = options._scopeId
        vnode.fnContext = parent
      }
      return vnode
    }
  } else {
    this._c = (a, b, c, d) => createElement(contextVm, a, b, c, d, needNormalization)
  }
}
```

## 五、抽象组件

对于抽象组件，我曾经写过几篇文章对其进行分析，所以这里就不再赘述了，想看的可以点击传送门自行去阅读。

- [《细谈 vue - transition 篇》](<https://juejin.im/post/5cf411d8e51d4550a629b222>)
- [《细谈 vue - transition-group 篇》](<https://juejin.im/post/5cf7a11f51882528394f22cd>)
- [《细谈 vue - 抽象组件实战篇》](https://juejin.im/post/5cfb8f3cf265da1bc14b1b07)

## 总结

文章到这，经过了大篇幅的文字分析，我们对 `vue` 中组件的创建（其中包括异步组件的创建、函数式组件的创建以及抽象组件的创建）、组件的钩子函数、组件配置合并等都有了一个较为全面的了解。

这里我也希望各位小伙伴能在了解组件的这些原理以后，在自身业务开发中，可以结合业务进行最佳的组件开发实践，比如我个人曾因业务中的权限操作的统一管理而采用了个人认为的最佳方案 - 抽象组件，它很好的解决权限管理这一业务痛点

前端交流群：731175396，欢迎大家加入
