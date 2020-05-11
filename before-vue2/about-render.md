/*
* @Author: Zhang Guohua
* @Date:   2020-05-09 18:33:15
* @Last Modified by:   zgh
* @Last Modified time: 2020-05-11 14:21:53
* @Description: create by zgh
* @GitHub: Savour Humor
*/


## 组件

- 模板引擎： string + data => html.
- 组件的本质： 模板引擎，一个函数，通过给定数据，渲染对应的 html 数据。
- 组件的产出： vue/react 产出的是 virtual DOM.
    + render 函数才是最重要的， data, computed props 等都是数据源。
    + render 本来可以直接产出 html, 但是却没有。他采用的不是完全替换，而是 patch. 进行 dom 对比，来进行数据更新。
    + virtual dom 带来了分层设计，对渲染过程进行抽象，使得框架可以渲染到 web 以外的平台，以及能够实现 ssr.
        * 至于 virtual dom 与 原声 dom 操作的性能差异，并不是 virtual dom 的最终目标。并且，要比较性能是要控制变量的，比如页面大小，数据变化量。
- VNode: 用对象，描述 DOM 节点。
    + vNode -> 真实的 DOM: 需要 render 函数。
        * 元素的 tag -> 元素， 组件的 tag -> 组件。
- 组件： 可以通过 function 和 class 来描述。
    + functional component: 纯函数，没有自身状态，只接受外部数据。产出 vNode 方式： 单纯的函数调用。
    + 有状态的组件(stateful component): 类，可以实例化；有状态；产出 vNode 方式，需要实例化，调用 render 函数。



## Vnode
组件的产出，渲染器的目标都是 vNode; 设计 vNode 本身就是在设计框架；

- vNode 描述：
    + tag, data, children, text, 
    + 设计什么样的属性来描述都没有问题，但要在尽可能保证语义能够说得通的情况下复用属性，会使 vNode 对象更加轻量。
- 使用 vNode 描述抽象内容：
    + 组件： 增加一个标示，来表示 vNode 到底是普通的 html, 还是组件。 可以用 tag 的类型，字符串为普通 html 标签。
    + Fragment: 抽象标示，渲染一个片段，根源素并不真实存在。 添加一个 flags 属性，表示类型。
    + Portal: 允许你把内容渲染到任何地方。即提供指定的目标，将子节点渲染到给定的目标下。
- vNode 的种类： 不同的 vNode 有不同的设计，我们完全可以将其划分为不同的类型。
    + html/svg, component, text, Fragment, portal.
    + component: function, stateful (普通， 需要被keepAlive, 已经被 keepAlive)
- flags 作为 vNode 标示： 既然有类别，就添加标示。添加 flags 也是 virtual DOM 算法优化手段之一。
    + Vue2 中区分 vNode 种类这么做： 先当组件处理，如果创建成功，就是组件，否则，检查 tag 是否有定义，如果有则当作普通标签，如果没有检查是否是注视节点，如果不是，则是文本节点。
    + 一个 vNode 描述什么，这些都是在 挂载/patch 阶段进行，带来两个问题： 无法从 AOT (预编译)层面进行优化，开发者无法手动优化。
    + 通过 flags 标明，避免很多耗性能的判断，通过位运算符再次提升运行时性能。 flags & VNodeFlags.ELEMENT
    + Vue3 采用的是 inferno 手段。
- 枚举 vNodeFlags: 通过枚举/对象来进行表示，例如：
    + 通过枚举属性，派生出额外三个标示。ELEMENT， COMPONENT_STATEFUL， COMPONENT
    + 判断 vNode 类型，通过按位与 & 运算。使用位运算的技巧。
```js
const VNodeFlags = {
  // html 标签
  ELEMENT_HTML: 1,
  // SVG 标签
  ELEMENT_SVG: 1 << 1,

  // 普通有状态组件
  COMPONENT_STATEFUL_NORMAL: 1 << 2,
  // 需要被keepAlive的有状态组件
  COMPONENT_STATEFUL_SHOULD_KEEP_ALIVE: 1 << 3,
  // 已经被keepAlive的有状态组件
  COMPONENT_STATEFUL_KEPT_ALIVE: 1 << 4,
  // 函数式组件
  COMPONENT_FUNCTIONAL: 1 << 5,

  // 纯文本
  TEXT: 1 << 6,
  // Fragment
  FRAGMENT: 1 << 7,
  // Portal
  PORTAL: 1 << 8
}
```
- children 和 childrenFlags: 以子节点的情况来设计 childrenFlags 枚举这些情形, 判断同样通过派生标示，位运算来进行。
    + children 需要标示，是为了进行优化；在胡须的 diff 算法的章节中。
    + 但是并不是所有的子节点都用来存储 vNode, 比如组件的 子 VNode 不是 children 而是 slots, 所以我们会通过定义 vNode.slots 来存储。
```js
const ChildrenFlags = {
  // 未知的 children 类型
  UNKNOWN_CHILDREN: 0,
  // 没有 children
  NO_CHILDREN: 1,
  // children 是单个 VNode
  SINGLE_VNODE: 1 << 1,

  // children 是多个拥有 key 的 VNode
  KEYED_VNODES: 1 << 2,
  // children 是多个没有 key 的 VNode
  NONE_KEYED_VNODES: 1 << 3
}
```
- vNodeData: 对 vNode 进行描述的内容，比如 class, style
    + 对于组件： event, props, 都可以放在 vNode Data 中。


- 目前为止的vNode 对象， Vue3 的源码中，vNode 还包括 handle, contextVNode, parentVNode, key, ref, slots等。
```js
export interface VNode {
  // _isVNode 属性在上文中没有提到，它是一个始终为 true 的值，有了它，我们就可以判断一个对象是否是 VNode 对象
  _isVNode: true
  // el 属性在上文中也没有提到，当一个 VNode 被渲染为真实 DOM 之后，el 属性的值会引用该真实DOM
  el: Element | null
  flags: VNodeFlags
  tag: string | FunctionalComponent | ComponentClass | null
  data: VNodeData | null
  children: VNodeChildren
  childFlags: ChildrenFlags
}
```

## 辅助创建 vNode 的 h 函数

有了 vNode, 我们开发时就要以 vNode 作为目标，但是实际开发中肯定不能直接写 vNode，这个很显然应该交给 complier 来做。 h 函数是一种封装，但只是一定程度的改善，其实本质上也是模板吧。我们实际解决问题，还是通过 模板和 jsx。但 h 函数依然重要， 无论是模板还是 jsx 都要通过编译，那么直接编译为 vNode,还是 h函数组成的调用集合呢？ 哪个好其实很难说，但是，将公用，灵活，复杂的逻辑封装成函数，并交给运行时，能大大降低编译器的书写难度，甚至编译后的代码也有一定的可读性，而 h 就是众多运行时中的一个。

- 在 VNode 创建时确定类型 - flags: 
    + 封装 h 函数，使其更加通用灵活，需要使 VNode 中的一些属性作为参数，提取 tag, data, children 即可。
    + 通过 tag 确定 flags, 通过 children 确认 childrenFlags
        * Fragment/Portal 的 tag 为 null, text 的 tag 也是 null, 我们通过 tag 作为其本身标识，来进行判断。
    + 要么是标签，要么是 Fragment/Portal, 要么是组件，纯文本节点单独创建。
    + vue 2 通过 tag.functional， vue3 通过 render 来判断状态组件和函数式组件。
    + 只有确定了 vNode 的 flags, h 函数就可以返回带有正确类型的 vNode;
- 确定 childrend 类型： 只对于非组件类型， 组件通过 slots 存储 children.
    + 通过 children 确认 childrenFlags：
    + children: 数组， 对象， 没有 children, 文本。
```js
function h(tag, data = null, children = null) {
  let flags = null
  if (typeof tag === 'string') {
    flags = tag === 'svg' ? VNodeFlags.ELEMENT_SVG : VNodeFlags.ELEMENT_HTML
  } else if (tag === Fragment) {
    flags = VNodeFlags.FRAGMENT
  } else if (tag === Portal) {
    flags = VNodeFlags.PORTAL
    tag = data && data.target
  } else {
    // 兼容 Vue2 的对象式组件
    if (tag !== null && typeof tag === 'object') {
      flags = tag.functional
        ? VNodeFlags.COMPONENT_FUNCTIONAL       // 函数式组件
        : VNodeFlags.COMPONENT_STATEFUL_NORMAL  // 有状态组件
    } else if (typeof tag === 'function') {
      // Vue3 的类组件
      flags = tag.prototype && tag.prototype.render
        ? VNodeFlags.COMPONENT_STATEFUL_NORMAL  // 有状态组件
        : VNodeFlags.COMPONENT_FUNCTIONAL       // 函数式组件
    }
  }

  let childFlags = null
  if (Array.isArray(children)) {
    const { length } = children
    if (length === 0) {
      // 没有 children
      childFlags = ChildrenFlags.NO_CHILDREN
    } else if (length === 1) {
      // 单个子节点
      childFlags = ChildrenFlags.SINGLE_VNODE
      children = children[0]
    } else {
      // 多个子节点，且子节点使用key
      childFlags = ChildrenFlags.KEYED_VNODES
      children = normalizeVNodes(children)
    }
  } else if (children == null) {
    // 没有子节点
    childFlags = ChildrenFlags.NO_CHILDREN
  } else if (children._isVNode) {
    // 单个子节点
    childFlags = ChildrenFlags.SINGLE_VNODE
  } else {
    // 其他情况都作为文本节点处理，即单个子节点，会调用 createTextVNode 创建纯文本类型的 VNode
    childFlags = ChildrenFlags.SINGLE_VNODE
    children = createTextVNode(children + '')
  }
}
```

- 使用 h 函数创建 VNode:
    + 框架内部设计时，会设计状态组件继承基础组件，基础组件内部默认由一个 render 函数，抛出 缺少render错误提示。一个组件没有 render 函数，则会调用基础组件的 render ，抛出错误。
    + 在设计由状态的组件时，我们会设计一个基础组件，所有组件都会继承基础组件，并且基础组件拥有用来报告错误信息的 render 函数。
    + render 函数用来将 VNode, 渲染到实例上。


## 渲染器之挂载

内容概要： 渲染器将各种类型的 VNode 挂载为真实 DOM 的原理。

- 渲染器： 将 Virtual DOM 渲染成特定平台下真实 DOM 的工具 (render 函数)。渲染器工作流程分为 mount, patch. 如果旧的 VNode 存在，就会进行对比，试图以最小的开销完成 DOM 的更新，这个过程叫 patch. 如果没有旧的 VNode, 则直接进行挂载 VNode, 叫做 mount.
    + render 通常接受两个参数： VNode, container.
    + 渲染器不仅包括渲染，还有： 
        * 控制部分组件生命周期钩子的调用： 在整个渲染周期中包含了大量的 DOM 操作、组件的挂载、卸载，控制着组件的生命周期钩子调用的时机。
        * 多端渲染的桥梁：自定义渲染器的本质就是把特定平台操作“DOM”的方法从核心算法中抽离，并提供可配置的方案。
        * 与异步渲染有直接关系： Vue3 的异步渲染是基于调度器的实现，若要实现异步渲染，组件的挂载就不能同步进行，DOM的变更就要在合适的时机，一些需要在真实DOM存在之后才能执行的操作(如 ref)也应该在合适的时机进行。对于时机的控制是由调度器来完成的，但类似于组件的挂载与卸载以及操作 DOM 等行为的入队还是由渲染器来完成的，这也是为什么 Vue2 无法轻易实现异步渲染的原因。
        * 包含最核心的 Diff 算法：Diff 算法是渲染器的核心特性之一，可以说正是 Diff 算法的存在才使得 Virtual DOM 如此成功。


- 挂载普通标签元素： 挂载方法根据 VNodeflags 采用不同的挂载方式。element, component, text, fragment, portal
    + 



























