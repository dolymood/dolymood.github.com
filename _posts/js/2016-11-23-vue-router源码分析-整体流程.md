---
layout: post
category : js
tagline: ""
tags : [vue-router, vue, js]
---
{% include JB/setup %}

在现在单页应用这么火爆的年代，路由已经成为了我们开发应用必不可少的利器；而纵观各大框架，都会有对应的强大路由支持。Vue.js 因其性能、通用、易用、体积、学习成本低等特点已经成为了广大前端们的新宠，而其对应的路由 [vue-router](http://router.vuejs.org/) 也是设计的简单好用，功能强大。本文就从源码来分析下 Vue.js 官方路由 [vue-router](http://router.vuejs.org/) 的整体流程。

> 本文主要以 vue-router 当前最新的 2.0.3 版本来进行分析。

首先来张整体的图：

![vue-router.js流程图](http://static.galileo.xiaojukeji.com/static/tms/shield/vue-router%E5%89%AF%E6%9C%AC.png)

有图在心中，下边就以官方仓库下 `examples/basic` 基础例子来一点点分析整个流程。

### 入口

首先看应用入口的代码部分：

```js
import Vue from 'vue'
import VueRouter from 'vue-router'

// 1. 插件
// 安装 <router-view> and <router-link> 组件
// 且给当前应用下所有的组件都注入 $router and $route 对象
Vue.use(VueRouter)

// 2. 定义各个路由下使用的组件，简称路由组件
const Home = { template: '<div>home</div>' }
const Foo = { template: '<div>foo</div>' }
const Bar = { template: '<div>bar</div>' }

// 3. 创建 VueRouter 实例 router
const router = new VueRouter({
  mode: 'history',
  base: __dirname,
  routes: [
    { path: '/', component: Home },
    { path: '/foo', component: Foo },
    { path: '/bar', component: Bar }
  ]
})

// 4. 创建 启动应用
// 一定要确认注入了 router 
// 在 <router-view> 中将会渲染路由组件
new Vue({
  router,
  template: `
    <div id="app">
      <h1>Basic</h1>
      <ul>
        <li><router-link to="/">/</router-link></li>
        <li><router-link to="/foo">/foo</router-link></li>
        <li><router-link to="/bar">/bar</router-link></li>
        <router-link tag="li" to="/bar">/bar</router-link>
      </ul>
      <router-view class="view"></router-view>
    </div>
  `
}).$mount('#app')
```

### 作为插件

上边代码中关键的第 1 步，利用 Vue.js 提供的插件机制 `.use(plugin)` 来安装 `VueRouter`，而这个插件机制则会调用该 `plugin` 对象的 `install` 方法（当然如果该 `plugin` 没有该方法的话会把 `plugin` 自身作为函数来调用）；下边来看下 vue-router 这个插件具体的实现部分。

<!--more-->

`VueRouter` 对象是在 `src/index.js` 中暴露出来的，这个对象有一个静态的 `install` 方法：

```js
/* @flow */
// 导入 install 模块
import { install } from './install'
// ...
import { inBrowser, supportsHistory } from './util/dom'
// ...

export default class VueRouter {
// ...
}

// 赋值 install
VueRouter.install = install

// 自动使用插件
if (inBrowser && window.Vue) {
  window.Vue.use(VueRouter)
}
```

可以看到这是一个 Vue.js 插件的经典写法，给插件对象增加 `install` 方法用来安装插件具体逻辑，同时在最后判断下如果是在浏览器环境且存在 `window.Vue` 的话就会自动使用插件。

`install` 在这里是一个单独的模块，继续来看同级下的 `src/install.js` 的主要逻辑：

```js
// router-view router-link 组件
import View from './components/view'
import Link from './components/link'

// export 一个 Vue 引用
export let _Vue

// 安装函数
export function install (Vue) {
  if (install.installed) return
  install.installed = true
	
  // 赋值私有 Vue 引用
  _Vue = Vue

  // 注入 $router $route
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this.$root._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this.$root._route }
  })
  // beforeCreate mixin
  Vue.mixin({
    beforeCreate () {
      // 判断是否有 router
      if (this.$options.router) {
      	// 赋值 _router
        this._router = this.$options.router
        // 初始化 init
        this._router.init(this)
        // 定义响应式的 _route 对象
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      }
    }
  })

  // 注册组件
  Vue.component('router-view', View)
  Vue.component('router-link', Link)
// ...
}
```

这里就会有一些疑问了？

* 为啥要 export 一个 Vue 引用？

> 插件在打包的时候是肯定不希望把 vue 作为一个依赖包打进去的，但是呢又希望使用 `Vue` 对象本身的一些方法，此时就可以采用上边类似的做法，在 `install` 的时候把这个变量赋值 `Vue` ，这样就可以在其他地方使用 `Vue` 的一些方法而不必引入 vue 依赖包（前提是保证 `install` 后才会使用）。

* 通过给 `Vue.prototype` 定义 `$router`、`$route` 属性就可以把他们注入到所有组件中吗？

> 在 Vue.js 中所有的组件都是被扩展的 Vue 实例，也就意味着所有的组件都可以访问到这个实例原型上定义的属性。

`beforeCreate mixin` 这个在后边创建 Vue 实例的时候再细说。

### 实例化 `VueRouter`

在入口文件中，首先要实例化一个 `VueRouter` ，然后将其传入 Vue 实例的 `options` 中。现在继续来看在 `src/index.js` 中暴露出来的 `VueRouter` 类：

```js
// ...
import { createMatcher } from './create-matcher'
// ...
export default class VueRouter {
// ...
  constructor (options: RouterOptions = {}) {
    this.app = null
    this.options = options
    this.beforeHooks = []
    this.afterHooks = []
    // 创建 match 匹配函数
    this.match = createMatcher(options.routes || [])
    // 根据 mode 实例化具体的 History
    let mode = options.mode || 'hash'
    this.fallback = mode === 'history' && !supportsHistory
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this)
        break
      default:
        assert(false, `invalid mode: ${mode}`)
    }
  }
// ...
}
```

里边包含了重要的一步：创建 `match` 匹配函数。

#### `match` 匹配函数

匹配函数是由 `src/create-matcher.js` 中的 `createMatcher` 创建的：

```js
/* @flow */

import Regexp from 'path-to-regexp'
// ...
import { createRouteMap } from './create-route-map'
// ...

export function createMatcher (routes: Array<RouteConfig>): Matcher {
  // 创建路由 map
  const { pathMap, nameMap } = createRouteMap(routes)
  // 匹配函数
  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
// ...
  }

  function redirect (
    record: RouteRecord,
    location: Location
  ): Route {
// ...
  }

  function alias (
    record: RouteRecord,
    location: Location,
    matchAs: string
  ): Route {
// ...
  }

  function _createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: Location
  ): Route {
    if (record && record.redirect) {
      return redirect(record, redirectedFrom || location)
    }
    if (record && record.matchAs) {
      return alias(record, location, record.matchAs)
    }
    return createRoute(record, location, redirectedFrom)
  }
  // 返回
  return match
}
// ...
```

具体逻辑后续再具体分析，现在只需要理解为根据传入的 `routes` 配置生成对应的路由 map，然后直接返回了 `match` 匹配函数。

继续来看 `src/create-route-map.js` 中的 `createRouteMap` 函数：

```js
/* @flow */

import { assert, warn } from './util/warn'
import { cleanPath } from './util/path'

// 创建路由 map
export function createRouteMap (routes: Array<RouteConfig>): {
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>
} {
  // path 路由 map
  const pathMap: Dictionary<RouteRecord> = Object.create(null)
  // name 路由 map
  const nameMap: Dictionary<RouteRecord> = Object.create(null)
  // 遍历路由配置对象 增加 route 记录
  routes.forEach(route => {
    addRouteRecord(pathMap, nameMap, route)
  })

  return {
    pathMap,
    nameMap
  }
}

// 增加 route 记录 函数
function addRouteRecord (
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>,
  route: RouteConfig,
  parent?: RouteRecord,
  matchAs?: string
) {
  // 获取 path 、name
  const { path, name } = route
  assert(path != null, `"path" is required in a route configuration.`)
  // 记录对象
  const record: RouteRecord = {
    path: normalizePath(path, parent),
    components: route.components || { default: route.component },
    instances: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {}
  }
  // 嵌套子路由 则递归增加 记录
  if (route.children) {
// ...
    route.children.forEach(child => {
      addRouteRecord(pathMap, nameMap, child, record)
    })
  }
  // 处理别名 alias 逻辑 增加对应的 记录
  if (route.alias !== undefined) {
    if (Array.isArray(route.alias)) {
      route.alias.forEach(alias => {
        addRouteRecord(pathMap, nameMap, { path: alias }, parent, record.path)
      })
    } else {
      addRouteRecord(pathMap, nameMap, { path: route.alias }, parent, record.path)
    }
  }
  // 更新 path map
  pathMap[record.path] = record
  // 更新 name map
  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else {
      warn(false, `Duplicate named routes definition: { name: "${name}", path: "${record.path}" }`)
    }
  }
}

function normalizePath (path: string, parent?: RouteRecord): string {
  path = path.replace(/\/$/, '')
  if (path[0] === '/') return path
  if (parent == null) return path
  return cleanPath(`${parent.path}/${path}`)
}
```

#### 实例化 History

这也是很重要的一步，所有的 `History` 类都是在 `src/history/` 目录下，现在呢不需要关心具体的每种 `History` 的具体实现上差异，只需要知道他们都是继承自 `src/history/base.js` 中的 `History` 类的：

```js
/* @flow */

// ...
import { inBrowser } from '../util/dom'
import { runQueue } from '../util/async'
import { START, isSameRoute } from '../util/route'
// 这里从之前分析过的 install.js 中 export _Vue
import { _Vue } from '../install'

export class History {
// ...
  constructor (router: VueRouter, base: ?string) {
    this.router = router
    this.base = normalizeBase(base)
    // start with a route object that stands for "nowhere"
    this.current = START
    this.pending = null
  }
// ...
}

// 得到 base 值
function normalizeBase (base: ?string): string {
  if (!base) {
    if (inBrowser) {
      // respect <base> tag
      const baseEl = document.querySelector('base')
      base = baseEl ? baseEl.getAttribute('href') : '/'
    } else {
      base = '/'
    }
  }
  // make sure there's the starting slash
  if (base.charAt(0) !== '/') {
    base = '/' + base
  }
  // remove trailing slash
  return base.replace(/\/$/, '')
}
// ...
```

实例化完了 `VueRouter`，下边就该看看 `Vue` 实例了。

### 实例化 `Vue`

实例化很简单：

```js
new Vue({
  router,
  template: `
    <div id="app">
      <h1>Basic</h1>
      <ul>
        <li><router-link to="/">/</router-link></li>
        <li><router-link to="/foo">/foo</router-link></li>
        <li><router-link to="/bar">/bar</router-link></li>
        <router-link tag="li" to="/bar">/bar</router-link>
      </ul>
      <router-view class="view"></router-view>
    </div>
  `
}).$mount('#app')
```

`options` 中传入了 `router`，以及模板；还记得上边没具体分析的 `beforeCreate mixin` 吗，此时创建一个 Vue 实例，对应的 `beforeCreate` 钩子就会被调用：

```js
  // 注入 $router $route
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this.$root._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this.$root._route }
  })
  // beforeCreate mixin
  Vue.mixin({
    beforeCreate () {
      // 判断是否有 router
      if (this.$options.router) {
      	// 赋值 _router
        this._router = this.$options.router
        // 初始化 init
        this._router.init(this)
        // 定义响应式的 _route 对象
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      }
    }
  })
```

具体来说，首先判断实例化时 `options` 是否包含 `router`，然后给当前实例赋值 `_router`，这样在访问原型上的 `$router` 的时候就可以得到 `router` 了。

然后来看 `router` 的 `init` 方法就干了哪些事情，依旧是在 `src/index.js` 中：

```js
/* @flow */

import { install } from './install'
import { createMatcher } from './create-matcher'
import { HashHistory, getHash } from './history/hash'
import { HTML5History, getLocation } from './history/html5'
import { AbstractHistory } from './history/abstract'
import { inBrowser, supportsHistory } from './util/dom'
import { assert } from './util/warn'

export default class VueRouter {
// ...
  init (app: any /* Vue component instance */) {
// ...
    this.app = app

    const history = this.history

    if (history instanceof HTML5History) {
      history.transitionTo(getLocation(history.base))
    } else if (history instanceof HashHistory) {
      history.transitionTo(getHash(), () => {
        window.addEventListener('hashchange', () => {
          history.onHashChange()
        })
      })
    }

    history.listen(route => {
      this.app._route = route
    })
  }
// ...
}
// ...
```

_未完待续..._
