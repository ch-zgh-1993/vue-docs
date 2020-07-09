/*
* @Author: Zhang Guohua
* @Date:   2020-06-01 14:28:45
* @Last Modified by:   zgh
* @Last Modified time: 2020-07-09 21:02:57
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

SSR 配置与 SPA webpack 配置大致相似，我们将配置分为三个文件， base, client, server. base 共享， client 为客户端， server 为服务端。可以使用 webpack-merge 扩展基本配置。

### 服务器配置

用于生成传递给 createBundleRenderer 的 server bundle.

```js
const merge = require('webpack-merge')
const nodeExternals = require('webpack-node-externals')
const baseConfig = require('./webpack.base.config.js')
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')

module.exports = merge(baseConfig, {
  // 将 entry 指向应用程序的 server entry 文件
  entry: '/path/to/entry-server.js',

  // 这允许 webpack 以 Node 适用方式(Node-appropriate fashion)处理动态导入(dynamic import)，
  // 并且还会在编译 Vue 组件时，
  // 告知 `vue-loader` 输送面向服务器代码(server-oriented code)。
  target: 'node',

  // 对 bundle renderer 提供 source map 支持
  devtool: 'source-map',

  // 此处告知 server bundle 使用 Node 风格导出模块(Node-style exports)
  output: {
    libraryTarget: 'commonjs2'
  },

  // https://webpack.js.org/configuration/externals/#function
  // https://github.com/liady/webpack-node-externals
  // 外置化应用程序依赖模块。可以使服务器构建速度更快，
  // 并生成较小的 bundle 文件。
  externals: nodeExternals({
    // 不要外置化 webpack 需要处理的依赖模块。
    // 你可以在这里添加更多的文件类型。例如，未处理 *.vue 原始文件，
    // 你还应该将修改 `global`（例如 polyfill）的依赖模块列入白名单
    whitelist: /\.css$/
  }),

  // 这是将服务器的整个输出
  // 构建为单个 JSON 文件的插件。
  // 默认文件名为 `vue-ssr-server-bundle.json`
  plugins: [
    new VueSSRServerPlugin()
  ]
})
```
生成 vue-ssr-server-bundle.json 后，将文件路径传递给 createBundleRenderer

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer('/path/to/vue-ssr-server-bundle.json', {
  // ……renderer 的其他选项
})
```

可以将 bundle 作为对象传递给 createBundleRenderer, 这对热重载非常有用。

### 扩展说明

请注意，在 externals 选项中，我们将 CSS 文件列入白名单。这是因为从依赖模块导入的 CSS 还应该由 webpack 处理。如果你导入依赖于 webpack 的任何其他类型的文件（例如 *.vue, *.sass），那么你也应该将它们添加到白名单中。


如果你使用 runInNewContext: 'once' 或 runInNewContext: true，那么你还应该将修改 global 的 polyfill 列入白名单，例如 babel-polyfill。这是因为当使用新的上下文模式时，**server bundle 中的代码具有自己的 global 对象。**由于在使用 Node 7.6+ 时，在服务器并不真正需要它，所以实际上只需在客户端 entry 导入它。


### 客户端配置


client config 和 base config 大体上相同。显然你需要把 entry 指向你的客户端入口文件。除此之外，如果你使用 CommonsChunkPlugin，请确保仅在客户端配置 client config 中使用，因为服务器包需要单独的入口 chunk。


生成 clientManifest: 客户端构建清淡，使用 client mainfest 和 server bundle, renderer 有了这些构建信息，可以自动推断和注入 资源预加载/数据预取指令，以及 css, scrpit 到所渲染的 html.

- 在生成的文件名中有哈希时，可以取代 html-webpack-plugin 来注入正确的资源 URL。
- 在通过 webpack 的按需代码分割特性渲染 bundle 时，我们可以确保对 chunk 进行最优化的资源预加载/数据预取，并且还可以将所需的异步 chunk 智能地注入为 script 标签，以避免客户端的瀑布式请求 (waterfall request)，以及改善可交互时间 (TTI - time-to-interactive)。

客户端配置:

```js
const webpack = require('webpack')
const merge = require('webpack-merge')
const baseConfig = require('./webpack.base.config.js')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')

module.exports = merge(baseConfig, {
  entry: '/path/to/entry-client.js',
  plugins: [
    // 重要信息：这将 webpack 运行时分离到一个引导 chunk 中，
    // 以便可以在之后正确注入异步 chunk。
    // 这也为你的 应用程序/vendor 代码提供了更好的缓存。
    new webpack.optimize.CommonsChunkPlugin({
      name: "manifest",
      minChunks: Infinity
    }),
    // 此插件在输出目录中
    // 生成 `vue-ssr-client-manifest.json`。
    new VueSSRClientPlugin()
  ]
})
```


使用客户端清单以及页面模版：

```js
const { createBundleRenderer } = require('vue-server-renderer')

const template = require('fs').readFileSync('/path/to/template.html', 'utf-8')
const serverBundle = require('/path/to/vue-ssr-server-bundle.json')
const clientManifest = require('/path/to/vue-ssr-client-manifest.json')

const renderer = createBundleRenderer(serverBundle, {
  template,
  clientManifest
})
```

通过以上设置，使用代码分割特性构建后的服务端渲染的 HTML 代码，看起来如下: (所有的都是自动注入)

```html
<html>
  <head>
    <!-- 用于当前渲染的 chunk 会被资源预加载(preload) -->
    <link rel="preload" href="/manifest.js" as="script">
    <link rel="preload" href="/main.js" as="script">
    <link rel="preload" href="/0.js" as="script">
    <!-- 未用到的异步 chunk 会被数据预取(prefetch)（次要优先级） -->
    <link rel="prefetch" href="/1.js" as="script">
  </head>
  <body>
    <!-- 应用程序内容 -->
    <div data-server-rendered="true"><div>async</div></div>
    <!-- manifest chunk 优先 -->
    <script src="/manifest.js"></script>
    <!-- 在主 chunk 之前注入异步 chunk -->
    <script src="/0.js"></script>
    <script src="/main.js"></script>
  </body>
</html>
```


### 手动资源注入


当提供 template 选项时，资源是自动注入的。当你想要对资源注入进行细粒度的控制，或者根本不使用模版，可以创建 renderer 并手动执行资源注入，传入 inject: false.


在 renderToString 回调中， context 会暴露一下方法：

- context.renderStyles(): 返回内联 style 包含的 关键 CSS, 关键 CSS 是在要用到 .vue 组件中收集的。
    + 如果提供了 clientManifest, 返回 CSS 中也将包含 link 由 webpack 输入的 css 文件。
- context.renderState(options?: Object): 此方法序列化 context.state, 并返回一个内联的 script, 状态被嵌入在 window.__INITIAL_STATE__, 上下文状态键 (context state key) 和 window 状态键 (window state key)， 都可以通过传递选项对象进行自定义
- context.renderScripts(): 需要 clientManifest; 此方法返回应用程序所需的 script 标签，当在应用程序中使用一步代码分割(async code-spliting) 时，将智能地正确的推断需要引入的那些异步 chunk。
- context.renderResourceHints():  需要 clientManifest, 返回当前要渲染的页面，所需要的 link rel="preload/prefetch" 资源提示，默认情况下会: 
    + 预加载页面所需的 js 和 css 文件
    + 预取一步 js chunk， 之后可能用于渲染。
- 使用 shouldPreload 选项可以进一步自定义要预加载的文件。

- context.getPreloadFiles()： 需要 clientManifest, 此方法不返回字符串，而是返回一个数组，由要预加载的资源文件对象所组成，这可以用在以编程方式执行 HTTP/2 服务器推送 (HTTP/2 server push)


由于传递给 createBundleRender 的 template 将会使用 context 对象进行差值，你可以通过传入 inject: false 在模版中使用这些方法：

```html
<html>
  <head>
    <!-- 使用三花括号(triple-mustache)进行 HTML 不转义插值(non-HTML-escaped interpolation) -->
    {{{ renderResourceHints() }}}
    {{{ renderStyles() }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
    {{{ renderState() }}}
    {{{ renderScripts() }}}
  </body>
</html>
```


如果你根本没有使用 tempalte , 可以自己拼接字符串。


## CSS 管理

同 SPA 项目一样，推荐使用 .vue 中的 style, 提供如下功能：

- 组件作用域
- 能够使用预处理器 pre-processor / postCss
- 开发中 hot-reload

vue-loader 中的 vue-style-loader 具备一些服务端渲染的特殊功能：

- SPA/SSR 通用编程体验
- 在使用 bundleRenderer 时，自动注入关键 CSS(critical CSS)： 
  + 如果在服务器端渲染期间使用，可以在 HTML 中收集和内联（使用 template 选项时自动处理）组件的 CSS。在客户端，当第一次使用该组件时，vue-style-loader 会检查这个组件是否已经具有服务器内联(server-inlined)的 CSS - 如果没有，CSS 将通过 style 标签动态注入。
- 通用 css 提取
  + 此设置支持使用 extract-text-webpack-plugin 将主 chunk(main chunk) 中的 CSS 提取到单独的 CSS 文件中（使用 template 自动注入），这样可以将文件分开缓存。建议用于存在很多公用 CSS 时。
- 内部异步组件中的 CSS 将内联为 JavaScript 字符串，并由 vue-style-loader 处理。

### 启用 css 提取

使用 vue-loader 中的 extractCSS 选项，版本 vue-loader 12.0.0+

```js
// webpack.config.js
const ExtractTextPlugin = require('extract-text-webpack-plugin')

// CSS 提取应该只用于生产环境
// 这样我们在开发过程中仍然可以热重载
const isProduction = process.env.NODE_ENV === 'production'

module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          // enable CSS extraction
          extractCSS: isProduction
        }
      },
      // ...
    ]
  },
  plugins: isProduction
    // 确保添加了此插件！
    ? [new ExtractTextPlugin({ filename: 'common.[chunkhash].css' })]
    : []
}


```
请注意，上述配置仅适用于 *.vue 文件中的样式，然而你也可以使用 style src="./foo.css"将外部 CSS 导入 Vue 组件。


如果你想从 JavaScript 中导入 CSS，例如，import 'foo.css'，你需要配置合适的 loader：

```js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.css$/,
        // 重要：使用 vue-style-loader 替代 style-loader
        use: isProduction
          ? ExtractTextPlugin.extract({
              use: 'css-loader',
              fallback: 'vue-style-loader'
            })
          : ['vue-style-loader', 'css-loader']
      }
    ]
  },
  // ...
}
```

### 从依赖模块导入样式


从 NPM 依赖模块导入 CSS 时需要注意的几点：

- 在 SSR 构建过程中，不应该外置化提取。
- 在使用 CSS 提取 + 使用 CommonsChunkPlugin 插件提取 vendor 时，如果提取的 CSS 位于提取的 vendor chunk 之中，extract-text-webpack-plugin 会遇到问题。为了解决这个问题，请避免在 vendor chunk 中包含 CSS 文件。客户端 webpack 配置示例如下：

```js
module.exports = {
  // ...
  plugins: [
    // 将依赖模块提取到 vendor chunk 以获得更好的缓存，是很常见的做法。
    new webpack.optimize.CommonsChunkPlugin({
      name: 'vendor',
      minChunks: function (module) {
        // 一个模块被提取到 vendor chunk 时……
        return (
          // 如果它在 node_modules 中
          /node_modules/.test(module.context) &&
          // 如果 request 是一个 CSS 文件，则无需外置化提取
          !/\.css$/.test(module.request)
        )
      }
    }),
    // 提取 webpack 运行时和 manifest
    new webpack.optimize.CommonsChunkPlugin({
      name: 'manifest'
    }),
    // ...
  ]
}
```


## Head 管理

类似于资源注入，Head管理遵循相同的理念：我们可以在组件的生命周期中，将数据动态地追加到渲染上下文 (render context)，然后在模板中的占位符替换为这些数据

- 在 2.3.2 + , 可以通过 this.$ssrContext 来直接访问组件中的服务器端渲染上下文。旧版本中hack, 传递给 createApp, 并暴露在根实例的 options 上，才能手动注入 ssr context, 子组件通过 this.$root.$options.ssrContext 访问。

标题管理 mixin
```js
// title-mixin.js

function getTitle (vm) {
  // 组件可以提供一个 `title` 选项
  // 此选项可以是一个字符串或函数
  const { title } = vm.$options
  if (title) {
    return typeof title === 'function'
      ? title.call(vm)
      : title
  }
}

const serverTitleMixin = {
  created () {
    const title = getTitle(this)
    if (title) {
      this.$ssrContext.title = title
    }
  }
}

const clientTitleMixin = {
  mounted () {
    const title = getTitle(this)
    if (title) {
      document.title = title
    }
  }
}

// 可以通过 `webpack.DefinePlugin` 注入 `VUE_ENV`
export default process.env.VUE_ENV === 'server'
  ? serverTitleMixin
  : clientTitleMixin

// 在路由组件上，利用 mixin , 控制 document title
// Item.vue
export default {
  mixins: [titleMixin],
  title () {
    return this.item.title
  },

  asyncData ({ store, route }) {
    return store.dispatch('fetchItem', route.params.id)
  },

  computed: {
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}

```

```html
// 然后模板中的内容将会传递给 bundle renderer：

<html>
  <head>
    <title>{{ title }}</title>
  </head>
  <body>
    ...
  </body>
</html>
```

注意内容：

- 使用双花括号(double-mustache)进行 HTML 转义插值(HTML-escaped interpolation)，以避免 XSS 攻击。
- 你应该在创建 context 对象时提供一个默认标题，以防在渲染过程中组件没有设置标题。

使用相同的策略，你可以轻松地将此 mixin 扩展为通用的头部管理工具 (generic head management utility)。





## 缓存

SSR 项目是快，但是创建组件实例，虚拟 DOM 节点的开销，无法与基于字符串拼接(pure string-base) 的模版性能相当，我们要利用缓存，提升性能，减少服务器负载。

### Page level caching

相同的地址，对于不同的用户，展示相同的内容，可以使用 micro-caching 策略，来提高程序处理高流量的能力。 通常在 nginx 层完成。 我们也可以在 Node 中实现。那就在 ng 做。


## component level caching

vue-server-renderer 内支持。要启用，在创建 renderer 时提供 cache implementaton, 典型做法是传入 lru-cache.

```js
const LRU = require('lru-cache')

const renderer = createRenderer({
  cache: LRU({
    max: 10000,
    maxAge: ...
  })
})
```

怎么实现呢？ 通过 serverCacheKey, name 唯一。serverCacheKey 返回的 key 应该包含足够的信息，来表示渲染结果的具体情况。 返回常量将导致组件始终被缓存，这对纯静态组件是有好处的。

```js
export default {
  name: 'item', // 必填选项
  props: ['item'],
  serverCacheKey: props => props.item.id,
  render (h) {
    return h('div', this.item.id)
  }
}
```

### 什么时候用 component cache

如果 renderer 在渲染过程中进行缓存命中，那么将直接重新使用整个子树的缓存结果。那么一下情况 **不**该使用：

- 可能依赖全局状态的子组件。 不依赖外部状态。
- 对渲染上下文产生副作用的子组件。 不对外部产生影响。

小心使用它来解决性能瓶颈问题，多数情况下，不需要缓存单一实例组件。 常见可用，在大的 v-for 中重复出现的组件，由于这些组件通常由数据库集合(database collection)中的对象驱动，它们可以使用简单的缓存策略：使用其唯一 id，再加上最后更新的时间戳，来生成其缓存键(cache key)。



## 流式渲染

vue ssr 对于 renderer & bundle renderer 提供开箱即用的流式渲染功能。 使用 renderToStream 代替 renderToString, 返回的是 Node.js stream

在流式渲染模式下，当 renderer 遍历虚拟 DOM 树 (virtual DOM tree) 时，会尽快发送数据。这意味着我们可以尽快获得"第一个 chunk"，并开始更快地将其发送给客户端。

然而，当第一个数据 chunk 被发出时，子组件甚至可能不被实例化，它们的生命周期钩子也不会被调用。这意味着，如果子组件需要在其生命周期钩子函数中，将数据附加到渲染上下文 (render context)，当流 (stream) 启动时，这些数据将不可用。这是因为，大量上下文信息 (context information)（如头信息 (head information) 或内联关键 CSS(inline critical CSS)）需要在应用程序标记 (markup) 之前出现，我们基本上必须等待流(stream)完成后，才能开始使用这些上下文数据。

因此，如果你依赖由组件生命周期钩子函数填充的上下文数据，则不建议使用流式传输模式。







































