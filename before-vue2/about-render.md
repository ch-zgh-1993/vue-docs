/*
* @Author: Zhang Guohua
* @Date:   2020-05-09 18:33:15
* @Last Modified by:   zgh
* @Last Modified time: 2020-05-12 14:17:48
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
    + mount: 根据类型采用不同的挂载方法：element, component, text, fragment, portal
    + mountElement: (VNode, container, isSVG)
        * 处理 VNodeData:  switch style/class/on/props 
            - 对于 class: 底层设计应该是 class 的值，体现为 字符串，可以直接挂载。但是对于应用层，可以是数组，可以是对象。 可以通过一个函数进行转换。
            - attr: 标签上存在的属性。 标准属性，通过 document.element.id/class 等能访问的。
            - dom prop: 存在于 dom 对象上的属性。 setAttribute 可以为 dom 元素设置标准/非标准属性，但是设置前，会将属性值转换为字符串加到元素上。一些特殊的 attribute 如 checked/disabled/value/selected/muted, 只要出现， property 就会设置为 true, 只有 removeAttribute 才会变为 false.
            - 此时我们只需要将这些属性额外的拿出来，还有携带大写的如: innerHTML, text Content, 直接作为 prop 对待。其他的当作 attr 处理。
            - 事件的处理： 主要在于设计 VNodeData. 如果直接以 click 作为属性，会无法与 attr 进行区别，而如果要规定所有的方法，显然是比较笨的。那么采取原声的 DOM 对象设计，将所有的事情，采用 on+'click' 进行标示。 当然，从模版到 VNode 是编译器来做的。 确定后，通过 addEvent/attachevent
        * 挂载子节点： 根据类型挂载，再次对每个子节点调用 mount 函数。
        * 对于 SVG： 创建使用： document.createElementNS， 对于 circle, react 等，通过父元素是否为 svg, 来判断接下来，是否创建 svg 标签。传递负元素的 svg 标签。
        * 对其他挂载函数，也需要视情形增加第三个参数。

- 挂载纯文本，Fragment,portal:
    + 挂载纯文本： document.createTextNode
    + 挂载 Fragment: 类似于 VNode 的 children, 只是对于 Fragment 标签不进行渲染。
        * el 属性指向： 一个节点指向当前；多个节点指向第一个节点；没有创建文本节点，引用空文本节点。
        * 意义： 在于 patch 阶段， DOM 元素移动时，确保被放置到争取的位置。合理使用 appendChild, removeChild, insertBefore.
    + 挂载 Portal: 类似于挂载，是将其挂载到 tag 指向的元素。那么 Portal 的 el 应该指向谁？他需要站位庸俗，因为他的事件机制仍然是按照 DOM 结构实施，需要一个占位元素来承接事件。我们创建一个空的文本节点，并挂载到 container 下。让 el 引用该文本节点。
- 有状态组件的挂载和原理： 组件产出 VNode, 将 VNode 挂载到 container 中。
    + 由于组件类型： 将组件内部再次划分为 有状态组件/函数式组件 进行处理。
    + 挂载有状态的组件：像 data, props, ref, slots 属于基本原理基础上，再次依据组件实例设计的产物，为 render 函数生成 VNOde 的过程中提供数据来源服务，而组件产出 VNode 才是核心。
        * 创建组件实例： new vnode.tag()
        * 获取组件产出的 VNode: 调用组件的 render 函数，获取 VNode.
        * mount 挂载： 挂载 vnode 到 container.
        * 让组件实例 $el 和 vnode.el 引用当前组件的根 DOM 元素。如果组件返回的是一个 Fragment, 那么 $el 和 el 应该是该片段的第一个 DOM 元素。
- 函数式组件的挂载和原理： 函数式组件是一个返回 VNode 的函数：
    + 比有状态的组件少了一个实例化的过程。
    + 有状态组件实例化过程中，会产生 data, state, computed, watch 声明周期等内容。而函数式组件，只有 props 和 slots, 性能会更好。



























