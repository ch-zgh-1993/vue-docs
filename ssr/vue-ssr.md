/*
* @Author: Zhang Guohua
* @Date:   2020-06-01 14:28:45
* @Last Modified by:   zgh
* @Last Modified time: 2020-06-02 20:57:50
* @Description: create by zgh
* @GitHub: Savour Humor
*/
# Vue SSR

## 编写通用代码

- 响应数据： 
    + 应用程序实例： 在客户端，每个用户在他们各自的浏览器中使用新的应用程序实例。在服务端，我们也希望每个请求都是全新的，独立的应用程序实例。以便不会有交叉请求造成的状态感染。
    + 在服务器上，开始渲染时，应用程序已经解析完成其状态。响应式在服务器上是多余的，默认禁用。可以避免将数据转换为响应式对象的性能开销。


- vue 的生命周期钩子：  只有 beforeCreate 和 created 会在 SSR 过程中被调用。其他生命周期钩子只会在客户端执行。
- 访问特定平台 API: 不可接受特定平台的 API, 如果直接使用了 window, document 这种仅浏览器可用的全局变量，在 Node.js 中执行回抛出错误。反之同样。
    + 对于共享于客户端和服务器，但用于不同平台 API 的任务，建议将平台特定实现包含在通用 API 中，或者使用为你执行此操作的 library.
    + 对于仅适用于浏览器的 API， 通常在纯客户端的生命周期钩子中惰性访问他们。
    + 考虑到如果第三方 library 不是以上面的通用用法编写，则将其集成到服务器渲染的应用程序中，可能会很棘手。你可能要通过模拟 (mock) 一些全局变量来使其正常运行，但这只是 hack 的做法，并且可能会干扰到其他 library 的环境检测代码。

- 自定义指令： 大多数自定义指令直接操作 DOM, 在服务端渲染过程中会导致错误。有两种解决方法：
    + 推荐使用组件作为抽象机制，并运行在 虚拟 DOM 层级, 例如： 使用渲染函数。
    + 如果不容易替换为组件，则可以在出啊功能就爱你服务器 renderer 时，使用 directives 提供服务器端版本。



## 源码结构

- 避免状态单例。应该为每个请求创建一个新的 vue 实例，避免交叉请求状态感染，可以通过工厂函数，为每个请求创建新的应用程序实例。
    + 同样的，除了 Vue 根实例，对于 router, store, event bus 实例，都不应该直接从模块导出并导入到应用程序中，而是需要再 createApp 中创建一个新的实例，并从根 vue 注入。

- 构建步骤： 在服务器上使用 webpack 来打包应用程序。
    + 通常 vue 应用程序是由 webpack + vue-loader 构建，许多 webapck 特定功能不能直接在 Node.js 中运行。如 file-loader 导入文件，通过 css-loader 导入 css.
    + Node.js 最新版本能完全支持 ES2015 新特性，我们还是需要转译客户端代码以适应老版浏览器，这也会涉及到构建步骤。
    + 对于 SPA 和 SSR 都适用 webpack 打包，服务器需要使用服务器的 bundle 用于 SSR, 客户端 bundle 会发送给浏览器，用于混合静态标记。


- 使用 webpack 的源码结构： 
```sh
src
├── components
│   ├── Foo.vue
│   ├── Bar.vue
│   └── Baz.vue
├── App.vue
├── app.js # 通用 entry(universal entry)
├── entry-client.js # 仅运行于浏览器
└── entry-server.js # 仅运行于服务器
```
    + app.js 是应用程序的 通用 entry, SPA 中创建根 Vue 实例，并挂载到 DOM, 但是对于 SSR， 责任需要转译到纯客户端 entry 文件， app.js 简单的使用 export 导出一个 createApp 函数
```js
import Vue from 'vue'
import App from './App.vue'

// 导出一个工厂函数，用于创建新的
// 应用程序、router 和 store 实例
export function createApp () {
  const app = new Vue({
    // 根实例简单的渲染应用程序组件。
    render: h => h(App)
  })
  return { app }
}
```
    + entry-client.js: 创建应用程序，将其挂载到 DOM 中。
```js
import { createApp } from './app'

// 客户端特定引导逻辑……

const { app } = createApp()

// 这里假定 App.vue 模板中根元素具有 `id="app"`
app.$mount('#app')
```
    + entry-server.js: 在每次渲染中重复调用此函数，此时只有创建和返回应用程序实例，稍后会记那个服务器端路由匹配和数据预取逻辑。
```js
import { createApp } from './app'

export default context => {
  const { app } = createApp()
  return app
}
```


## 路由和代码分割

在服务端使用 * 处理路径，将所有的路由都传递到Vue应用程序中，客户端和服务器使用相同的路由配置。


- 使用工厂方法创建 vueRouter。
- 在 app.js 中引入。
- 在 entry-server.js 实现服务器端路由逻辑。
```js
// entry-server.js
import { createApp } from './app'

export default context => {
  // 因为有可能会是异步路由钩子函数或组件，所以我们将返回一个 Promise，
    // 以便服务器能够等待所有的内容在渲染前，
    // 就已经准备就绪。
  return new Promise((resolve, reject) => {
    const { app, router } = createApp()

    // 设置服务器端 router 的位置
    router.push(context.url)

    // 等到 router 将可能的异步组件和钩子函数解析完
    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      // 匹配不到的路由，执行 reject 函数，并返回 404
      if (!matchedComponents.length) {
        return reject({ code: 404 })
      }

      // Promise 应该 resolve 应用程序实例，以便它可以渲染
      resolve(app)
    }, reject)
  })
}
```


### 代码分割
代码分割/惰性加载有助于减少浏览器在初始渲染中下载的资源体积，可以极大的改善 bundle 的可交互时间，关键在于，首屏加载所需。

- 异步组件：
    + V2.5 以前，服务端渲染的异步组件只能用在路由组件上，而 2.5+ 版本中，得益于核心算法的升级，异步组件可以用在任何地方。
    + 在挂载 app 之前，调用 router.onReady, 路由器必须提前解析路由配置中的组件，才能很正确的调用组件中可能存在的路由钩子，现在只需要更新 client entry.
    + 异步路由配置示例:
```js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      { path: '/', component: () => import('./components/Home.vue') },
      { path: '/item/:id', component: () => import('./components/Item.vue') }
    ]
  })
}
```
## 数据预取和状态


### 数据预取存储器(Data Store)

SSR 中，我们在渲染应用程序的快照，如果应用程序依赖于一些异步数据，在开始渲染过程之前，需要先预取和解析好这些数据。另外，在客户端，在 mount 到客户端应用程序之前，需要获取到与服务器端应用程序完全相同的数据，否则客户端应用程序会因为使用与服务端应用程序不同的状态，导致混合失败。


- 使用状态管理工具 Vuex, 将预取数据条虫到 store 中。 在 HTML 中序列话和内联预置状态，这样，挂载到客户端应用程序之前，可以直接从 store 获取到内联预置状态。

- 带有逻辑配置的组件： 在路由组件上暴露一个自定义的静态函数 asyncData, 此方法会在组件实例化之前调用，无法访问 this,会将 store 和 路由信息传递进去。

- 服务端数据预取： 在 entry-server.js 中，我们可以通过路由获得相匹配的组件，如果组件暴露 asyncData , 我们就调用改方法，将解析完成的状态，附加到渲染上下文中。
    + 当使用 template 时，context.state 将作为 window.__INITIAL_STATE__ 状态，自动嵌入到最终的 HTML 中。而在客户端，在挂载到应用程序之前，store 就应该获取到状态：

- 客户端数据预取： 有两种不同的方式, 会产生不同的用户体验，因此要根据实际场景决定使用哪个。无论选择哪个，点那个路由组件重用时(同一路由， params 或 query 参数修改时，)也应该调用 asyncData 函数。
    + 在路由导航之前解析数据：等视图所有数据全部解析之后，再传入数据并处理当前视图。好处可以在数据准备就绪，传入视图渲染完整的内容。如果数据预取需要很长时间，需要增加一个数据加载指示器，来减少因数据存取时间过长，造成的卡顿。
        * 在初始路由数据准备就绪之后，我们应该注册此钩子，这样我们就不必再次获取服务器提取的数据。
    + 匹配要渲染的视图后，再获取数据: 将数据预取逻辑放在组件的 beforeMount 中，当路由导航被触发时，可以立即切换试图，应用程序具有更快的响应速度。然而，在传入视图时不会有完整的可用数据，因此，使用此策略的每个视图组件都需要有条件加载状态。
        * 可以通过全局 mixin 实现。
```js
Vue.mixin({
  beforeMount () {
    const { asyncData } = this.$options
    if (asyncData) {
      // 将获取数据操作分配给 promise
      // 以便在组件中，我们可以在数据准备就绪后
      // 通过运行 `this.dataPromise.then(...)` 来执行其他任务
      this.dataPromise = asyncData({
        store: this.$store,
        route: this.$route
      })
    }
  },
  // 路由参数修改时，更新数据。
    beforeRouteUpdate (to, from, next) {
    const { asyncData } = this.$options
    if (asyncData) {
      asyncData({
        store: this.$store,
        route: to
      }).then(next).catch(next)
    } else {
      next()
    }
  }
})
```

- Store 代码拆分：将 store 拆分为多个模块，当然，也可以将这些模块代码分割到相应的路由组件 chunk 中。
    + state 必须是一个函数，可以创建多个实例化模块。
    + 可以在路由组件的 asyncData 钩子中，调用 store.registerModule 惰性注册这个模块。
    + 

```html
// 在路由组件内
<template>
  <div>{{ fooCount }}</div>
</template>

<script>
// 在这里导入模块，而不是在 `store/index.js` 中
import fooStoreModule from '../store/modules/foo'

export default {
  asyncData ({ store }) {
    store.registerModule('foo', fooStoreModule)
    return store.dispatch('foo/inc')
  },

  // 重要信息：当多次访问路由时，
  // 避免在客户端重复注册模块。
  destroyed () {
    this.$store.unregisterModule('foo')
  },

  computed: {
    fooCount () {
      return this.$store.state.foo.count
    }
  }
}
</script>
```

## 客户端激活 (client-side hydration)

客户端激活：指的是 Vue 在浏览器中接管由服务端发送的静态 HTML, 使其变为由 Vue 管理的动态 DOM 的过程。


服务器渲染的输出结果，应用程序的更元素上添加了一个特殊的属性， data-server-rendered="true", 告诉客户端这部分的 HTML 由 vue 在服务器渲染，应该以激活模式进行挂载。你需要自行添加 id 或者能选取到应用程序跟元素的选择器。

在没有 data-server-rendered 的元素上，可以向 $mount 函数的 hydrating 参数位传入 true, 强制使用激活模式。

在开发模式下， vue 将推端客户端生成的虚拟 DOM， 与服务器渲染的 DOM 结构是否匹配，如果无法匹配，将退出混合模式，丢弃现有的 DOM 并从头开始渲染。 在生产模式下，此检测会被跳过，以免性能损耗。

### 一些需要注意的坑

点那个使用 SSR + 客户端混合 时，需要了解，浏览器可能会更改一些特殊的 HTML 结构。例如会在 table 内部自动注入 tbody， 然而由于 vue 生成 virtual DOM 不包含，所以导致无法匹配。所以请确保在模版中写入有效的 HTML.


## Bundle Renderer 指引

vue ssr 提供一个名为 createBundleRenderer API, 通过使用 webpack 自定义插件， server bundle 将生成为可传递到 bundle renderer 的特殊 JSON 文件， 所创建的 bundle render 用法和普通的 render 相同，但有以下几个优点：

- 内置 source map 支持。 (webpack 中配置 devtool: 'source-map')
- 在开发环境甚至部署过程中热重载 (通过读取更新后的 bundle, 创建 renderer 实例)
- 关键 CSS (critical CSS) 注入，在使用 *.vue 文件时： 自动内联在渲染过程中用到的组件所需要的 CSS。
- 使用 clientManiFest 进行资源注入： 自动推断出最佳的预加载 (preload) 和预取(prefetch) 指令，以及初始渲染所需的代码分割 chunk.


## 构建配置

### 服务器配置

SSR 配置与 SPA webpack 配置大致相似，我们将配置分为三个文件， base, client, server. base 共享， client 为客户端， server 为服务端。可以使用 webpack-merge 扩展基本配置。






















