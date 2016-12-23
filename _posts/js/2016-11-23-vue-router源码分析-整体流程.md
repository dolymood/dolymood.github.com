---
layout: post
category : js
tagline: ""
tags : [vue-router, vue, js]
---
{% include JB/setup %}

在现在单页应用这么火爆的年代，路由已经成为了我们开发应用必不可少的利器；而纵观各大框架，都会有对应的强大路由支持。Vue.js 因其性能、通用、易用、体积、学习成本低等特点已经成为了广大前端们的新宠，而其对应的路由 [vue-router](http://router.vuejs.org/) 也是设计的简单好用，功能强大。本文就从源码来分析下 Vue.js 官方路由 [vue-router](http://router.vuejs.org/) 的整体流程。

> 本文主要以 vue-router 的 [2.0.3](https://github.com/vuejs/vue-router/tree/v2.0.3) 版本来进行分析。

首先来张整体的图：

![vue-router.js流程图](http://static.galileo.xiaojukeji.com/static/tms/shield/vue-router%E5%89%AF%E6%9C%AC.png)

先对整体有个大概的印象，下边就以官方仓库下 `examples/basic` 基础例子来一点点具体分析整个流程。

### 目录结构

先来看看整体的目录结构：

![vue-router 目录结构图](http://static.galileo.xiaojukeji.com/static/tms/shield/vue-router-dir-tree-dolymood.png)

和流程相关的主要需要关注点的就是 `components`、`history` 目录以及 `create-matcher.js`、`create-route-map.js`、`index.js`、`install.js`。下面就从 basic 应用入口开始来分析 vue-router 的整个流程。

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
  // 遍历路由配置对象 增加 路由记录
  routes.forEach(route => {
    addRouteRecord(pathMap, nameMap, route)
  })

  return {
    pathMap,
    nameMap
  }
}

// 增加 路由记录 函数
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
  // 路由记录 对象
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

可以看出主要做的事情就是根据用户路由配置对象生成普通的根据 `path` 来对应的路由记录以及根据 `name` 来对应的路由记录的 map，方便后续匹配对应。

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
// ...
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

具体来说，首先判断实例化时 `options` 是否包含 `router`，如果包含也就意味着是一个带有路由配置的实例被创建了，此时才有必要继续初始化路由相关逻辑。然后给当前实例赋值 `_router`，这样在访问原型上的 `$router` 的时候就可以得到 `router` 了。

下边来看里边两个关键：`router.init` 和 定义响应式的 `_route` 对象。

#### router.init

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

可以看到初始化主要就是给 `app` 赋值，针对于 `HTML5History` 和 `HashHistory` 特殊处理，因为在这两种模式下才有可能存在进入时候的不是默认页，需要根据当前浏览器地址栏里的 `path` 或者 `hash` 来激活对应的路由，此时就是通过调用 `transitionTo` 来达到目的；而且此时还有个注意点是针对于 `HashHistory` 有特殊处理，为什么不直接在初始化 `HashHistory` 的时候监听 `hashchange` 事件呢？这个是为了修复[https://github.com/vuejs/vue-router/issues/725](https://github.com/vuejs/vue-router/issues/725)这个 bug 而这样做的，简要来说就是说如果在 `beforeEnter` 这样的钩子函数中是异步的话，`beforeEnter` 钩子就会被触发两次，原因是因为在初始化的时候如果此时的 `hash` 值不是以 `/` 开头的话就会补上 `#/`，这个过程会触发 `hashchange` 事件，所以会再走一次生命周期钩子，也就意味着会再次调用 `beforeEnter` 钩子函数。

来看看这个具体的 `transitionTo` 方法的大概逻辑，在 `src/history/base.js` 中：

```js
/* @flow */

import type VueRouter from '../index'
import { warn } from '../util/warn'
import { inBrowser } from '../util/dom'
import { runQueue } from '../util/async'
import { START, isSameRoute } from '../util/route'
import { _Vue } from '../install'

export class History {
// ...
  transitionTo (location: RawLocation, cb?: Function) {
  	// 调用 match 得到匹配的 route 对象
    const route = this.router.match(location, this.current)
    // 确认过渡
    this.confirmTransition(route, () => {
      // 更新当前 route 对象
      this.updateRoute(route)
      cb && cb(route)
      // 子类实现的更新url地址
      // 对于 hash 模式的话 就是更新 hash 的值
      // 对于 history 模式的话 就是利用 pushstate / replacestate 来更新
      // 浏览器地址
      this.ensureURL()
    })
  }
  // 确认过渡
  confirmTransition (route: Route, cb: Function) {
    const current = this.current
    // 如果是相同 直接返回
    if (isSameRoute(route, current)) {
      this.ensureURL()
      return
    }
    // 交叉比对当前路由的路由记录和现在的这个路由的路由记录
    // 以便能准确得到父子路由更新的情况下可以确切的知道
    // 哪些组件需要更新 哪些不需要更新
    const {
      deactivated,
      activated
    } = resolveQueue(this.current.matched, route.matched)
    
    // 整个切换周期的队列
    const queue: Array<?NavigationGuard> = [].concat(
      // leave 的钩子
      extractLeaveGuards(deactivated),
      // 全局 router before hooks
      this.router.beforeHooks,
      // 将要更新的路由的 beforeEnter 钩子
      activated.map(m => m.beforeEnter),
      // 异步组件
      resolveAsyncComponents(activated)
    )

    this.pending = route
    // 每一个队列执行的 iterator 函数
    const iterator = (hook: NavigationGuard, next) => {
      // 确保期间还是当前路由
      if (this.pending !== route) return
      hook(route, current, (to: any) => {
        if (to === false) {
          // next(false) -> abort navigation, ensure current URL
          this.ensureURL(true)
        } else if (typeof to === 'string' || typeof to === 'object') {
          // next('/') or next({ path: '/' }) -> redirect
          this.push(to)
        } else {
          // confirm transition and pass on the value
          next(to)
        }
      })
    }
    // 执行队列
    runQueue(queue, iterator, () => {
      const postEnterCbs = []
      // 组件内的钩子
      const enterGuards = extractEnterGuards(activated, postEnterCbs, () => {
        return this.current === route
      })
      // 在上次的队列执行完成后再执行组件内的钩子
      // 因为需要等异步组件以及是OK的情况下才能执行
      runQueue(enterGuards, iterator, () => {
      	// 确保期间还是当前路由
        if (this.pending === route) {
          this.pending = null
          cb(route)
          this.router.app.$nextTick(() => {
            postEnterCbs.forEach(cb => cb())
          })
        }
      })
    })
  }
  // 更新当前 route 对象
  updateRoute (route: Route) {
    const prev = this.current
    this.current = route
    // 注意 cb 的值 
    // 每次更新都会调用 下边需要用到！
    this.cb && this.cb(route)
    // 执行 after hooks 回调
    this.router.afterHooks.forEach(hook => {
      hook && hook(route, prev)
    })
  }
}
// ...
```

可以看到整个过程就是执行约定的各种钩子以及处理异步组件问题，这里有一些具体函数具体细节被忽略掉了（后续会具体分析）但是不影响具体理解这个流程。但是需要注意一个概念：路由记录，每一个路由 `route` 对象都对应有一个 `matched` 属性，它对应的就是路由记录，他的具体含义在调用 `match()` 中有处理；通过之前的分析可以知道这个 `match` 是在 `src/create-matcher.js` 中的：

```js
// ...
import { createRoute } from './util/route'
import { createRouteMap } from './create-route-map'
// ...
export function createMatcher (routes: Array<RouteConfig>): Matcher {
  const { pathMap, nameMap } = createRouteMap(routes)
  // 关键的 match
  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    const location = normalizeLocation(raw, currentRoute)
    const { name } = location

    // 命名路由处理
    if (name) {
      // nameMap[name] = 路由记录
      const record = nameMap[name]
      const paramNames = getParams(record.path)
// ...
      if (record) {
        location.path = fillParams(record.path, location.params, `named route "${name}"`)
        // 创建 route
        return _createRoute(record, location, redirectedFrom)
      }
    } else if (location.path) {
      // 普通路由处理
      location.params = {}
      for (const path in pathMap) {
        if (matchRoute(path, location.params, location.path)) {
          // 匹配成功 创建route
          // pathMap[path] = 路由记录
          return _createRoute(pathMap[path], location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }
// ...
  // 创建路由
  function _createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: Location
  ): Route {
    // 重定向和别名逻辑
    if (record && record.redirect) {
      return redirect(record, redirectedFrom || location)
    }
    if (record && record.matchAs) {
      return alias(record, location, record.matchAs)
    }
    // 创建路由对象
    return createRoute(record, location, redirectedFrom)
  }

  return match
}
// ...
```

路由记录在分析 `match` 匹配函数那里以及分析过了，这里还需要了解下创建路由对象的 `createRoute`，存在于 `src/util/route.js` 中：

```js
// ...
export function createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: Location
): Route {
  // 可以看到就是一个被冻结的普通对象
  const route: Route = {
    name: location.name || (record && record.name),
    meta: (record && record.meta) || {},
    path: location.path || '/',
    hash: location.hash || '',
    query: location.query || {},
    params: location.params || {},
    fullPath: getFullPath(location),
    // 根据记录层级的得到所有匹配的 路由记录
    matched: record ? formatMatch(record) : []
  }
  if (redirectedFrom) {
    route.redirectedFrom = getFullPath(redirectedFrom)
  }
  return Object.freeze(route)
}
// ...
function formatMatch (record: ?RouteRecord): Array<RouteRecord> {
  const res = []
  while (record) {
    res.unshift(record)
    record = record.parent
  }
  return res
}
// ...
```


回到之前看的 `init`，最后调用了 `history.listen` 方法：

```js
history.listen(route => {
  this.app._route = route
})
```

`listen` 方法很简单就是设置下当前历史对象的 `cb` 的值, 在之前分析 `transitionTo` 的时候已经知道在 `history` 更新完毕的时候调用下这个 `cb`。然后看这里设置的这个函数的作用就是更新下当前应用实例的 `_route` 的值，更新这个有什么用呢？请看下段落的分析。

#### defineReactive 定义 _route

继续回到 `beforeCreate` 钩子函数中，在最后通过 `Vue` 的工具方法给当前应用实例定义了一个响应式的 `_route` 属性，值就是获取的 `this._router.history.current`，也就是当前 `history` 实例的当前活动路由对象。给应用实例定义了这么一个响应式的属性值也就意味着如果该属性值发生了变化，就会触发更新机制，继而调用应用实例的 `render` 重新渲染。还记得上一段结尾留下的疑问，也就是 `history` 每次更新成功后都会去更新应用实例的 `_route` 的值，也就意味着一旦 `history` 发生改变就会触发更新机制调用应用实例的 `render` 方法进行重新渲染。

### router-link 和 router-view 组件

回到实例化应用实例的地方：

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

可以看到这个实例的 `template` 中包含了两个自定义组件：`router-link` 和 `router-view`。

#### router-view 组件

`router-view` 组件比较简单，所以这里就先来分析它，他是在源码的 `src/components/view.js` 中定义的：

```js
export default {
  name: 'router-view',
  functional: true, // 功能组件 纯粹渲染
  props: {
    name: {
      type: String,
      default: 'default' // 默认default 默认命名视图的name
    }
  },
  render (h, { props, children, parent, data }) {
    // 解决嵌套深度问题
    data.routerView = true
	// route 对象
    const route = parent.$route
    // 缓存
    const cache = parent._routerViewCache || (parent._routerViewCache = {})
    let depth = 0
    let inactive = false
    // 当前组件的深度
    while (parent) {
      if (parent.$vnode && parent.$vnode.data.routerView) {
        depth++
      }
      处理 keepalive 逻辑
      if (parent._inactive) {
        inactive = true
      }
      parent = parent.$parent
    }

    data.routerViewDepth = depth
    // 得到相匹配的当前组件层级的 路由记录
    const matched = route.matched[depth]
    if (!matched) {
      return h()
    }
    // 得到要渲染组件
    const name = props.name
    const component = inactive
      ? cache[name]
      : (cache[name] = matched.components[name])

    if (!inactive) {
      // 非 keepalive 模式下 每次都需要设置钩子
      // 进而更新（赋值&销毁）匹配了的实例元素
      const hooks = data.hook || (data.hook = {})
      hooks.init = vnode => {
        matched.instances[name] = vnode.child
      }
      hooks.prepatch = (oldVnode, vnode) => {
        matched.instances[name] = vnode.child
      }
      hooks.destroy = vnode => {
        if (matched.instances[name] === vnode.child) {
          matched.instances[name] = undefined
        }
      }
    }
    // 调用 createElement 函数 渲染匹配的组件
    return h(component, data, children)
  }
}
```

可以看到逻辑还是比较简单的，拿到匹配的组件进行渲染就可以了。

#### router-link 组件

再来看看导航链接组件，他在源码的 `src/components/link.js` 中定义的：

```js
// ...
import { createRoute, isSameRoute, isIncludedRoute } from '../util/route'
// ...
export default {
  name: 'router-link',
  props: {
    // 传入的组件属性们
    to: { // 目标路由的链接
      type: toTypes,
      required: true
    },
    // 创建的html标签
    tag: {
      type: String,
      default: 'a'
    },
    // 完整模式，如果为 true 那么也就意味着
    // 绝对相等的路由才会增加 activeClass
    // 否则是包含关系
    exact: Boolean,
    // 在当前（相对）路径附加路径
    append: Boolean,
    // 如果为 true 则调用 router.replace() 做替换历史操作
    replace: Boolean,
    // 链接激活时使用的 CSS 类名
    activeClass: String
  },
  render (h: Function) {
    // 得到 router 实例以及当前激活的 route 对象
    const router = this.$router
    const current = this.$route
    const to = normalizeLocation(this.to, current, this.append)
    // 根据当前目标链接和当前激活的 route匹配结果
    const resolved = router.match(to, current)
    const fullPath = resolved.redirectedFrom || resolved.fullPath
    const base = router.history.base
    // 创建的 href
    const href = createHref(base, fullPath, router.mode)
    const classes = {}
    // 激活class 优先当前组件上获取 要么就是 router 配置的 linkActiveClass
    // 默认 router-link-active
    const activeClass = this.activeClass || router.options.linkActiveClass || 'router-link-active'
    // 相比较目标
    // 因为有命名路由 所有不一定有path
    const compareTarget = to.path ? createRoute(null, to) : resolved
    // 如果严格模式的话 就判断是否是相同路由（path query params hash）
    // 否则就走包含逻辑（path包含，query包含 hash为空或者相同）
    classes[activeClass] = this.exact
      ? isSameRoute(current, compareTarget)
      : isIncludedRoute(current, compareTarget)
    
    // 事件绑定
    const on = {
      click: (e) => {
        // 忽略带有功能键的点击
        if (e.metaKey || e.ctrlKey || e.shiftKey) return
        // 已阻止的返回
        if (e.defaultPrevented) return
        // 右击
        if (e.button !== 0) return
        // `target="_blank"` 忽略
        const target = e.target.getAttribute('target')
        if (/\b_blank\b/i.test(target)) return
        // 阻止默认行为 防止跳转
        e.preventDefault()
        if (this.replace) {
          // replace 逻辑
          router.replace(to)
        } else {
          // push 逻辑
          router.push(to)
        }
      }
    }
    // 创建元素需要附加的数据们
    const data: any = {
      class: classes
    }

    if (this.tag === 'a') {
      data.on = on
      data.attrs = { href }
    } else {
      // 找到第一个 <a> 给予这个元素事件绑定和href属性
      const a = findAnchor(this.$slots.default)
      if (a) {
        // in case the <a> is a static node
        a.isStatic = false
        const extend = _Vue.util.extend
        const aData = a.data = extend({}, a.data)
        aData.on = on
        const aAttrs = a.data.attrs = extend({}, a.data.attrs)
        aAttrs.href = href
      } else {
        // 没有 <a> 的话就给当前元素自身绑定时间
        data.on = on
      }
    }
    // 创建元素
    return h(this.tag, data, this.$slots.default)
  }
}

function findAnchor (children) {
  if (children) {
    let child
    for (let i = 0; i < children.length; i++) {
      child = children[i]
      if (child.tag === 'a') {
        return child
      }
      if (child.children && (child = findAnchor(child.children))) {
        return child
      }
    }
  }
}

function createHref (base, fullPath, mode) {
  var path = mode === 'hash' ? '/#' + fullPath : fullPath
  return base ? cleanPath(base + path) : path
}
```

可以看出 `router-link` 组件就是在其点击的时候根据设置的 `to` 的值去调用 `router` 的 `push` 或者 `replace` 来更新路由的，同时呢，会检查自身是否和当前路由匹配（严格匹配和包含匹配）来决定自身的 `activeClass` 是否添加。

### 小结

整个流程的代码到这里已经分析的差不多了，再来回顾下：

![vue-router.js流程图](http://static.galileo.xiaojukeji.com/static/tms/shield/vue-router%E5%89%AF%E6%9C%AC.png)

相信整体看完后和最开始的时候看到这张图的感觉是不一样的，且对于 vue-router 的整体的流程了解的比较清楚了。当然由于篇幅有限，这里还有很多细节的地方没有细细分析，后续会根据模块来进行具体的分析。
