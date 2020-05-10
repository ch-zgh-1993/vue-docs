/*
* @Author: Zhang Guohua
* @Date:   2020-05-09 18:33:15
* @Last Modified by:   zgh
* @Last Modified time: 2020-05-10 19:40:09
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






