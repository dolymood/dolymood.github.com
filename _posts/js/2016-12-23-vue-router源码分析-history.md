---
layout: post
category : js
tagline: ""
tags : [vue-router, vue, js]
---
{% include JB/setup %}

在[上篇](http://blog.aijc.net/js/2016/11/23/vue-router%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E6%95%B4%E4%BD%93%E6%B5%81%E7%A8%8B)中介绍了 [vue-router](http://router.vuejs.org/) 的整体流程，但是具体的 `history` 部分没有具体分析，本文就具体分析下和 `history` 相关的细节。

### 初始化 Router

通过整体流程可以知道在路由实例化的时候会根据当前 `mode` 模式来选择实例化对应的`History`类，这里再来回顾下，在 `src/index.js` 中：

```js
// ...
import { HashHistory, getHash } from './history/hash'
import { HTML5History, getLocation } from './history/html5'
import { AbstractHistory } from './history/abstract'
// ...
export default class VueRouter {
// ...
  constructor (options: RouterOptions = {}) {
// ...
    // 默认模式是 hash
    let mode = options.mode || 'hash'
    // 如果设置的是 history 但是如果浏览器不支持的话 
    // 强制退回到 hash
    this.fallback = mode === 'history' && !supportsHistory
    if (this.fallback) {
      mode = 'hash'
    }
    // 不在浏览器中 强制 abstract 模式
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode
    // 根据不同模式选择实例化对应的 History 类
    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        // 细节 传入了 fallback
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
```

可以看到 vue-router 提供了三种模式：`hash`（默认）、`history` 以及 `abstract` 模式，还不了解具体区别的可以在[文档](https://router.vuejs.org/zh-cn/api/options.html#mode) 中查看，有很详细的解释。下面就这三种模式初始化一一来进行分析。

<!--more-->

#### HashHistory

首先就看默认的 `hash` 模式，也应该是用的最多的模式，对应的源码在 `src/history/hash.js` 中：

```js
// ...
import { History } from './base'
import { getLocation } from './html5'
import { cleanPath } from '../util/path'

// 继承 History 基类
export class HashHistory extends History {
  constructor (router: VueRouter, base: ?string, fallback: boolean) {
    // 调用基类构造器
    super(router, base)

    // 如果说是从 history 模式降级来的
    // 需要做降级检查
    if (fallback && this.checkFallback()) {
      // 如果降级 且 做了降级处理 则什么也不需要做
      return
    }
    // 保证 hash 是以 / 开头
    ensureSlash()
  }

  checkFallback () {
    // 得到除去 base 的真正的 location 值
    const location = getLocation(this.base)
    if (!/^\/#/.test(location)) {
      // 如果说此时的地址不是以 /# 开头的
      // 需要做一次降级处理 降级为 hash 模式下应有的 /# 开头
      window.location.replace(
        cleanPath(this.base + '/#' + location)
      )
      return true
    }
  }
// ...
}

// 保证 hash 以 / 开头
function ensureSlash (): boolean {
  // 得到 hash 值
  const path = getHash()
  // 如果说是以 / 开头的 直接返回即可
  if (path.charAt(0) === '/') {
    return true
  }
  // 不是的话 需要手工保证一次 替换 hash 值
  replaceHash('/' + path)
  return false
}

export function getHash (): string {
  // 因为兼容性问题 这里没有直接使用 window.location.hash
  // 因为 Firefox decode hash 值
  const href = window.location.href
  const index = href.indexOf('#')
  // 如果此时没有 # 则返回 ''
  // 否则 取得 # 后的所有内容
  return index === -1 ? '' : href.slice(index + 1)
}
```

可以看到在实例化过程中主要做两件事情：针对于不支持 history api 的降级处理，以及保证默认进入的时候对应的 hash 值是以 `/` 开头的，如果不是则替换。值得注意的是这里并没有监听 `hashchange` 事件来响应对应的逻辑，这部分逻辑在[上篇](http://blog.aijc.net/js/2016/11/23/vue-router%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E6%95%B4%E4%BD%93%E6%B5%81%E7%A8%8B)的 `router.init` 中包含的，主要是为了解决 [https://github.com/vuejs/vue-router/issues/725](https://github.com/vuejs/vue-router/issues/725)，在对应的回调中则调用了 `onHashChange` 方法，后边具体分析。

#### 友善高级的 HTML5History

`HTML5History` 则是利用 history.pushState/repaceState API 来完成 URL 跳转而无须重新加载页面，页面地址和正常地址无异；源码在 `src/history/html5.js` 中：

```js
// ...
import { cleanPath } from '../util/path'
import { History } from './base'
// 记录滚动位置工具函数
import {
  saveScrollPosition,
  getScrollPosition,
  isValidPosition,
  normalizePosition,
  getElementPosition
} from '../util/scroll-position'

// 生成唯一 key 作为位置相关缓存 key
const genKey = () => String(Date.now())
let _key: string = genKey()

export class HTML5History extends History {
  constructor (router: VueRouter, base: ?string) {
    // 基类构造函数
    super(router, base)
    
    // 定义滚动行为 option
    const expectScroll = router.options.scrollBehavior
    // 监听 popstate 事件 也就是
    // 浏览器历史记录发生改变的时候（点击浏览器前进后退 或者调用 history api ）
    window.addEventListener('popstate', e => {
// ...
    })

    if (expectScroll) {
      // 需要记录滚动行为 监听滚动事件 记录位置
      window.addEventListener('scroll', () => {
        saveScrollPosition(_key)
      })
    }
  }
// ...
}
// ...
```

可以看到在这种模式下，初始化作的工作相比 hash 模式少了很多，只是调用基类构造函数以及初始化监听事件，不需要再做额外的工作。

#### AbstractHistory

理论上来说这种模式是用于 Node.js 环境的，一般场景也就是在做测试的时候。但是在实际项目中其实还可以使用的，利用这种特性还是可以很方便的做很多事情的。由于它和浏览器无关，所以代码上来说也是最简单的，在 `src/history/abstract.js` 中：

```js
// ...
import { History } from './base'

export class AbstractHistory extends History {
  index: number;
  stack: Array<Route>;

  constructor (router: VueRouter) {
    super(router)
    // 初始化模拟记录栈
    this.stack = []
    // 当前活动的栈的位置
    this.index = -1
  }
// ...
}
```

可以看出在抽象模式下，所做的仅仅是用一个数组当做栈来模拟浏览器历史记录，拿一个变量来标示当前处于哪个位置。

三种模式的初始化的部分已经完成了，但是这只是刚刚开始，继续往后看。

### history 改变

history 改变可以有两种，一种是用户点击链接元素，一种是更新浏览器本身的前进后退导航来更新。

先来说浏览器导航发生变化的时候会触发对应的事件：对于 hash 模式而言触发 `window` 的 `hashchange` 事件，对于 history 模式而言则触发 `window` 的 `popstate` 事件。

先说 hash 模式，当触发改变的时候会调用 `HashHistory` 实例的 `onHashChange`：

```js
  onHashChange () {
    // 不是 / 开头
    if (!ensureSlash()) {
      return
    }
    // 调用 transitionTo
    this.transitionTo(getHash(), route => {
      // 替换 hash
      replaceHash(route.fullPath)
    })
  }
```

对于 history 模式则是：

```js
window.addEventListener('popstate', e => {
  // 取得 state 中保存的 key
  _key = e.state && e.state.key
  // 保存当前的先
  const current = this.current
  // 调用 transitionTo
  this.transitionTo(getLocation(this.base), next => {
    if (expectScroll) {
      // 处理滚动
      this.handleScroll(next, current, true)
    }
  })
})
```

上边的 `transitionTo` 以及 `replaceHash`、`getLocation`、`handleScroll` 后边统一分析。

再看用户点击链接交互，即点击了 `<router-link>`，回顾下这个组件在渲染的时候做的事情：

```js
// ...
  render (h: Function) {
// ...

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
// ...
```

这里一个关键就是绑定了元素的 `click` 事件，当用户触发后，会调用 `router` 的 `push` 或 `replace` 方法来更新路由。下边就来看看这两个方法定义，在 `src/index.js` 中：

```js
  push (location: RawLocation) {
    this.history.push(location)
  }

  replace (location: RawLocation) {
    this.history.replace(location)
  }
```

可以看到其实他们只是代理而已，真正做事情的还是 `history` 来做，下面就分别把 history 的三种模式下的这两个方法进行分析。

#### HashHistory

直接看代码：

```js
// ...
  push (location: RawLocation) {
    // 调用 transitionTo
    this.transitionTo(location, route => {
// ...
    })
  }

  replace (location: RawLocation) {
    // 调用 transitionTo
    this.transitionTo(location, route => {
// ...
    })
  }
// ...
```

操作是类似的，主要就是调用基类的 `transitionTo` 方法来过渡这次历史的变化，在完成后更新当前浏览器的 hash 值。[上篇](http://blog.aijc.net/js/2016/11/23/vue-router%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E6%95%B4%E4%BD%93%E6%B5%81%E7%A8%8B)中大概分析了 `transitionTo` 方法，但是一些细节并没细说，这里来看下遗漏的细节：

```js
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
// ...
    }
    // 执行队列 leave 和 beforeEnter 相关钩子
    runQueue(queue, iterator, () => {
//...
    })
  }
```

这里有一个很关键的路由对象的 `matched` 实例，从上次的分析中可以知道它就是匹配到的路由记录的合集；这里从执行顺序上来看有这些 `resolveQueue`、`extractLeaveGuards`、`resolveAsyncComponents`、`runQueue` 关键方法。

首先来看 `resolveQueue`：

```js
function resolveQueue (
  current: Array<RouteRecord>,
  next: Array<RouteRecord>
): {
  activated: Array<RouteRecord>,
  deactivated: Array<RouteRecord>
} {
  let i
  // 取得最大深度
  const max = Math.max(current.length, next.length)
  // 从根开始对比 一旦不一样的话 就可以停止了
  for (i = 0; i < max; i++) {
    if (current[i] !== next[i]) {
      break
    }
  }
  // 舍掉相同的部分 只保留不同的
  return {
    activated: next.slice(i),
    deactivated: current.slice(i)
  }
}
```

可以看出 `resolveQueue` 就是交叉比对当前路由的路由记录和现在的这个路由的路由记录来决定调用哪些路由记录的钩子函数。

继续来看 `extractLeaveGuards`：

```js
// 取得 leave 的组件的 beforeRouteLeave 钩子函数们
function extractLeaveGuards (matched: Array<RouteRecord>): Array<?Function> {
  // 打平组件的 beforeRouteLeave 钩子函数们 按照顺序得到 然后再 reverse
  // 因为 leave 的过程是从内层组件到外层组件的过程
  return flatten(flatMapComponents(matched, (def, instance) => {
    const guard = extractGuard(def, 'beforeRouteLeave')
    if (guard) {
      return Array.isArray(guard)
        ? guard.map(guard => wrapLeaveGuard(guard, instance))
        : wrapLeaveGuard(guard, instance)
    }
  }).reverse())
}
// ...
// 将一个二维数组（伪）转换成按顺序转换成一维数组
// [[1], [2, 3], 4] -> [1, 2, 3, 4]
function flatten (arr) {
  return Array.prototype.concat.apply([], arr)
}
```

可以看到在执行 `extractLeaveGuards` 的时候首先需要调用 `flatMapComponents` 函数，下面来看看这个函数具体定义：

```js
// 将匹配到的组件们根据fn得到的钩子函数们打平
function flatMapComponents (
  matched: Array<RouteRecord>,
  fn: Function
): Array<?Function> {
  // 遍历匹配到的路由记录
  return flatten(matched.map(m => {
    // 遍历 components 配置的组件们
    //// 对于默认视图模式下，会包含 default （也就是实例化路由的时候传入的 component 的值）
    //// 如果说多个命名视图的话 就是配置的对应的 components 的值
    // 调用 fn 得到 guard 钩子函数的值
    // 注意此时传入的值分别是：视图对应的组件类，对应的组件实例，路由记录，当前 key 值 （命名视图 name 值）
    return Object.keys(m.components).map(key => fn(
      m.components[key],
      m.instances[key],
      m, key
    ))
  }))
}
```

此时需要仔细看下调用 `flatMapComponents` 时传入的 `fn`：

```js
flatMapComponents(matched, (def, instance) => {
  // 组件配置的 beforeRouteLeave 钩子
  const guard = extractGuard(def, 'beforeRouteLeave')
  // 存在的话 返回
  if (guard) {
    // 每一个钩子函数需要再包裹一次
    return Array.isArray(guard)
      ? guard.map(guard => wrapLeaveGuard(guard, instance))
      : wrapLeaveGuard(guard, instance)
  }
  // 这里没有返回值 默认调用的结果是 undefined
})
```

先来看 `extractGuard` 的定义：

```js
// 取得指定组件的 key 值
function extractGuard (
  def: Object | Function,
  key: string
): NavigationGuard | Array<NavigationGuard> {
  if (typeof def !== 'function') {
    // 对象的话 为了应用上全局的 mixins 这里 extend 下
    // 赋值 def 为 Vue “子类”
    def = _Vue.extend(def)
  }
  // 取得 options 上的 key 值
  return def.options[key]
}
```

很简答就是取得组件定义时的 `key` 配置项的值。

再来看看具体的 `wrapLeaveGuard` 是干啥用的：

```js
function wrapLeaveGuard (
  guard: NavigationGuard,
  instance: _Vue
): NavigationGuard {
  // 返回函数 执行的时候 用于保证上下文 是当前的组件实例 instance
  return function routeLeaveGuard () {
    return guard.apply(instance, arguments)
  }
}
```

其实这个函数还可以这样写：

```js
function wrapLeaveGuard (
  guard: NavigationGuard,
  instance: _Vue
): NavigationGuard {
  return _Vue.util.bind(guard, instance)
}
```

这样整个的 `extractLeaveGuards` 就分析完了，这部分还是比较绕的，需要好好理解下。但是目的是明确的就是得到将要离开的组件们按照由深到浅的顺序组合的 `beforeRouteLeave` 钩子函数们。

再来看一个关键的函数 `resolveAsyncComponents`，一看名字就知道这个是用来解决异步组件问题的：

```js
function resolveAsyncComponents (matched: Array<RouteRecord>): Array<?Function> {
  // 依旧调用 flatMapComponents 只是此时传入的 fn 是这样的：
  return flatMapComponents(matched, (def, _, match, key) => {
    // 这里假定说路由上定义的组件 是函数 但是没有 options
    // 就认为他是一个异步组件。
    // 这里并没有使用 Vue 默认的异步机制的原因是我们希望在得到真正的异步组件之前
    // 整个的路由导航是一直处于挂起状态
    if (typeof def === 'function' && !def.options) {
      // 返回“异步”钩子函数
      return (to, from, next) => {
// ...
      }
    }
  })
}
```

下面继续看，最后一个关键的 `runQueue` 函数，它的定义在 `src/util/async.js` 中：

```js
// 执行队列
export function runQueue (queue: Array<?NavigationGuard>, fn: Function, cb: Function) {
  // 内部迭代函数
  const step = index => {
    // 如果说当前的 index 值和整个队列的长度值齐平了 说明队列已经执行完成
    if (index >= queue.length) {
      // 执行队列执行完成的回调函数
      cb()
    } else {
      if (queue[index]) {
        // 如果存在的话 调用传入的迭代函数执行
        fn(queue[index], () => {
          // 第二个参数是一个函数 当调用的时候才继续处理队列的下一个位置
          step(index + 1)
        })
      } else {
        // 当前队列位置的值为假 继续队列下一个位置
        step(index + 1)
      }
    }
  }
  // 从队列起始位置开始迭代
  step(0)
}
```

可以看出就是一个执行一个函数队列中的每一项，但是考虑了异步场景，只有上一个队列中的项显式调用回调的时候才会继续调用队列的下一个函数。

在切换路由过程中调用的逻辑是这样的：

```js
// 每一个队列执行的 iterator 函数
const iterator = (hook: NavigationGuard, next) => {
  // 确保期间还是当前路由
  if (this.pending !== route) return
  // 调用钩子
  hook(route, current, (to: any) => {
    // 如果说钩子函数在调用第三个参数（函数）` 时传入了 false
    // 则意味着要终止本次的路由切换
    if (to === false) {
      // next(false) -> abort navigation, ensure current URL
      // 重新保证当前 url 是正确的
      this.ensureURL(true)
    } else if (typeof to === 'string' || typeof to === 'object') {
      // next('/') or next({ path: '/' }) -> redirect
      // 如果传入的是字符串 或者对象的话 认为是一个重定向操作
      // 直接调用 push 走你
      this.push(to)
    } else {
      // confirm transition and pass on the value
      // 其他情况 意味着此次路由切换没有问题 继续队列下一个
      // 且把值传入了
      // 传入的这个值 在此时的 leave 的情况下是没用的
      // 注意：这是为了后边 enter 的时候在处理 beforeRouteEnter 钩子的时候
      // 可以传入一个函数 用于获得组件实例
      next(to)
    }
  })
}
// 执行队列 leave 和 beforeEnter 相关钩子
runQueue(queue, iterator, () => {
// ...
})
```

而 `queue` 是上边定义的一个切换周期的各种钩子函数以及处理异步组件的“异步”钩子函数所组成队列，在执行完后就会调用队列执行完成后毁掉函数，下面来看这个函数做的事情：

```js
runQueue(queue, iterator, () => {
  // enter 后的回调函数们 用于组件实例化后需要执行的一些回调
  const postEnterCbs = []
  // leave 完了后 就要进入 enter 阶段了
  const enterGuards = extractEnterGuards(activated, postEnterCbs, () => {
    return this.current === route
  })
  // enter 的回调钩子们依旧有可能是异步的 不仅仅是异步组件场景
  runQueue(enterGuards, iterator, () => {
// ...
  })
})
```

仔细看看这个 `extractEnterGuards`，从调用参数上来看还是和之前的 `extractLeaveGuards` 是不同的：

```js
function extractEnterGuards (
  matched: Array<RouteRecord>,
  cbs: Array<Function>,
  isValid: () => boolean
): Array<?Function> {
  // 依旧是调用 flatMapComponents
  return flatten(flatMapComponents(matched, (def, _, match, key) => {
    // 调用 extractGuard 得到组件上的 beforeRouteEnter 钩子
    const guard = extractGuard(def, 'beforeRouteEnter')
    if (guard) {
      // 特殊处理 依旧进行包装
      return Array.isArray(guard)
        ? guard.map(guard => wrapEnterGuard(guard, cbs, match, key, isValid))
        : wrapEnterGuard(guard, cbs, match, key, isValid)
    }
  }))
}
function wrapEnterGuard (
  guard: NavigationGuard,
  cbs: Array<Function>,
  match: RouteRecord,
  key: string,
  isValid: () => boolean
): NavigationGuard {
  // 代理 路由 enter 的钩子函数
  return function routeEnterGuard (to, from, next) {
// ...
  }
}
```

可以看出此时整体的思路还是和 `extractLeaveGuards` 的差不多的，只是多了 `cbs` 回调数组 和 `isValid` 校验函数，截止到现在还不知道他们的具体作用，继续往下看此时调用的 `runQueue`：

```js
// enter 的钩子们
runQueue(enterGuards, iterator, () => {
// ...
})
```

可以看到此时执行 `enterGuards` 队列的迭代函数依旧是上边定义的 `iterator`，在迭代过程中就会调用 `wrapEnterGuard` 返回的 `routeEnterGuard` 函数：

```js
function wrapEnterGuard (
  guard: NavigationGuard,
  cbs: Array<Function>,
  match: RouteRecord,
  key: string,
  isValid: () => boolean
): NavigationGuard {
  // 代理 路由 enter 的钩子函数
  return function routeEnterGuard (to, from, next) {
    // 调用用户设置的钩子函数
    return guard(to, from, cb => {
      // 此时如果说调用第三个参数的时候传入了回调函数
      // 认为是在组件 enter 后有了组件实例对象之后执行的回调函数
      // 依旧把参数传递过去 因为有可能传入的是
      // false 或者 字符串 或者 对象
      // 继续走原有逻辑
      next(cb)
      if (typeof cb === 'function') {
        // 加入到 cbs 数组中
        // 只是这里没有直接 push 进去 而是做了额外处理
        cbs.push(() => {
          // 主要是为了修复 #750 的bug
          // 如果说 router-view 被一个 out-in transition 过渡包含的话
          // 此时的实例不一定是注册了的（因为需要做完动画） 所以需要轮训判断
          // 直至 current route 的值不再有效
          poll(cb, match.instances, key, isValid)
        })
      }
    })
  }
}
```

这个 `poll` 又是做什么事情呢？

```js
function poll (
  cb: any, // somehow flow cannot infer this is a function
  instances: Object,
  key: string,
  isValid: () => boolean
) {
  // 如果实例上有 key
  // 也就意味着有 key 为名的命名视图实例了
  if (instances[key]) {
    // 执行回调
    cb(instances[key])
  } else if (isValid()) {
    // 轮训的前提是当前 cuurent route 是有效的
    setTimeout(() => {
      poll(cb, instances, key, isValid)
    }, 16)
  }
}
```

`isValid` 的定义就是很简单了，通过在调用 `extractEnterGuards` 的时候传入的：

```js
const enterGuards = extractEnterGuards(activated, postEnterCbs, () => {
  // 判断当前 route 是和 enter 的 route 是同一个
  return this.current === route
})
```

回到执行 `enter` 进入时的钩子函数队列的地方，在执行完所有队列中函数后会调用传入 `runQueue` 的回调：

```
runQueue(enterGuards, iterator, () => {
  // 确保当前的 pending 中的路由是和要激活的是同一个路由对象
  // 以防在执行钩子过程中又一次的切换路由
  if (this.pending === route) {
    this.pending = null
    // 执行传入 confirmTransition 的回调
    cb(route)
    // 在 nextTick 时执行 postEnterCbs 中保存的回调
    this.router.app.$nextTick(() => {
      postEnterCbs.forEach(cb => cb())
    })
  }
})
```

通过[上篇](http://blog.aijc.net/js/2016/11/23/vue-router%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E6%95%B4%E4%BD%93%E6%B5%81%E7%A8%8B)分析可以知道 `confirmTransition` 的回调做的事情：

```js
this.confirmTransition(route, () => {
  // 更新当前 route 对象
  this.updateRoute(route)
  // 执行回调 也就是 transitionTo 传入的回调
  cb && cb(route)
  // 子类实现的更新url地址
  // 对于 hash 模式的话 就是更新 hash 的值
  // 对于 history 模式的话 就是利用 pushstate / replacestate 来更新
  // 浏览器地址
  this.ensureURL()
})
```

针对于 `HashHistory` 来说，调用 `transitionTo` 的回调就是：

```js
// ...
  push (location: RawLocation) {
    // 调用 transitionTo
    this.transitionTo(location, route => {
      // 完成后 pushHash
      pushHash(route.fullPath)
    })
  }

  replace (location: RawLocation) {
    // 调用 transitionTo
    this.transitionTo(location, route => {
      // 完成后 replaceHash
      replaceHash(route.fullPath)
    })
  }
// ...
function pushHash (path) {
  window.location.hash = path
}

function replaceHash (path) {
  const i = window.location.href.indexOf('#')
  // 直接调用 replace 强制替换 以避免产生“多余”的历史记录
  // 主要是用户初次跳入 且hash值不是以 / 开头的时候直接替换
  // 其余时候和push没啥区别 浏览器总是记录hash记录
  window.location.replace(
    window.location.href.slice(0, i >= 0 ? i : 0) + '#' + path
  )
}
```

其实就是更新浏览器的 hash 值，`push` 和 `replace` 的场景下都是一个效果。

回到 `confirmTransition` 的回调，最后还做了一件事情 `ensureURL`：

```js
ensureURL (push?: boolean) {
  const current = this.current.fullPath
  if (getHash() !== current) {
    push ? pushHash(current) : replaceHash(current)
  }
}
```

此时 `push` 为 `undefined`，所以调用 `replaceHash` 更新浏览器 hash 值。

#### HTML5History

整个的流程和 `HashHistory` 是类似的，不同的只是一些具体的逻辑处理以及特性，所以这里呢就直接来看整个的 `HTML5History`：

```js
export class HTML5History extends History {
// ...
  go (n: number) {
    window.history.go(n)
  }
  
  push (location: RawLocation) {
    const current = this.current
    // 依旧调用基类 transitionTo
    this.transitionTo(location, route => {
      // 调用 pushState 但是 url 是 base 值加上当前 fullPath
      // 因为 fullPath 是不带 base 部分得
      pushState(cleanPath(this.base + route.fullPath))
      // 处理滚动
      this.handleScroll(route, current, false)
    })
  }

  replace (location: RawLocation) {
    const current = this.current
    // 依旧调用基类 transitionTo
    this.transitionTo(location, route => {
      // 调用 replaceState
      replaceState(cleanPath(this.base + route.fullPath))
      // 滚动
      this.handleScroll(route, current, false)
    })
  }
  // 保证 location 地址是同步的
  ensureURL (push?: boolean) {
    if (getLocation(this.base) !== this.current.fullPath) {
      const current = cleanPath(this.base + this.current.fullPath)
      push ? pushState(current) : replaceState(current)
    }
  }
  // 处理滚动
  handleScroll (to: Route, from: Route, isPop: boolean) {
    const router = this.router
    if (!router.app) {
      return
    }
    // 自定义滚动行为
    const behavior = router.options.scrollBehavior
    if (!behavior) {
      // 不存在直接返回了
      return
    }
    assert(typeof behavior === 'function', `scrollBehavior must be a function`)

    // 等待下重新渲染逻辑
    router.app.$nextTick(() => {
      // 得到key对应位置
      let position = getScrollPosition(_key)
      // 根据自定义滚动行为函数来判断是否应该滚动
      const shouldScroll = behavior(to, from, isPop ? position : null)
      if (!shouldScroll) {
        return
      }
      // 应该滚动
      const isObject = typeof shouldScroll === 'object'
      if (isObject && typeof shouldScroll.selector === 'string') {
        // 带有 selector 得到该元素
        const el = document.querySelector(shouldScroll.selector)
        if (el) {
          // 得到该元素位置
          position = getElementPosition(el)
        } else if (isValidPosition(shouldScroll)) {
          // 元素不存在 降级下
          position = normalizePosition(shouldScroll)
        }
      } else if (isObject && isValidPosition(shouldScroll)) {
        // 对象 且是合法位置 统一格式
        position = normalizePosition(shouldScroll)
      }

      if (position) {
        // 滚动到指定位置
        window.scrollTo(position.x, position.y)
      }
    })
  }
}

// 得到 不带 base 值的 location
export function getLocation (base: string): string {
  let path = window.location.pathname
  if (base && path.indexOf(base) === 0) {
    path = path.slice(base.length)
  }
  // 是包含 search 和 hash 的
  return (path || '/') + window.location.search + window.location.hash
}

function pushState (url: string, replace?: boolean) {
  // 加了 try...catch 是因为 Safari 有调用 pushState 100 次限制
  // 一旦达到就会抛出 DOM Exception 18 错误
  const history = window.history
  try {
    // 如果是 replace 则调用 history 的 replaceState 操作
    // 否则则调用 pushState
    if (replace) {
      // replace 的话 key 还是当前的 key 没必要生成新的
      // 因为被替换的页面是进入不了的
      history.replaceState({ key: _key }, '', url)
    } else {
      // 重新生成 key
      _key = genKey()
      // 带入新的 key 值
      history.pushState({ key: _key }, '', url)
    }
    // 保存 key 对应的位置
    saveScrollPosition(_key)
  } catch (e) {
    // 达到限制了 则重新指定新的地址
    window.location[replace ? 'assign' : 'replace'](url)
  }
}
// 直接调用 pushState 传入 replace 为 true
function replaceState (url: string) {
  pushState(url, true)
}
```

这样可以看出和 `HashHistory` 中不同的是这里增加了滚动位置特性以及当历史发生变化时改变浏览器地址的行为是不一样的，这里使用了新的 history api 来更新。

#### AbstractHistory

抽象模式是属于最简单的处理了，因为不涉及和浏览器地址相关记录关联在一起；整体流程依旧和 `HashHistory` 是一样的，只是这里通过数组来模拟浏览器历史记录堆栈信息。

```js
// ...
import { History } from './base'

export class AbstractHistory extends History {
  index: number;
  stack: Array<Route>;
// ...

  push (location: RawLocation) {
    this.transitionTo(location, route => {
      // 更新历史堆栈信息
      this.stack = this.stack.slice(0, this.index + 1).concat(route)
      // 更新当前所处位置
      this.index++
    })
  }

  replace (location: RawLocation) {
    this.transitionTo(location, route => {
      // 更新历史堆栈信息 位置则不用更新 因为是 replace 操作
      // 在堆栈中也是直接 replace 掉的
      this.stack = this.stack.slice(0, this.index).concat(route)
    })
  }
  // 对于 go 的模拟
  go (n: number) {
    // 新的历史记录位置
    const targetIndex = this.index + n
    // 超出返回了
    if (targetIndex < 0 || targetIndex >= this.stack.length) {
      return
    }
    // 取得新的 route 对象
    // 因为是和浏览器无关的 这里得到的一定是已经访问过的
    const route = this.stack[targetIndex]
    // 所以这里直接调用 confirmTransition 了
    // 而不是调用 transitionTo 还要走一遍 match 逻辑
    this.confirmTransition(route, () => {
      // 更新
      this.index = targetIndex
      this.updateRoute(route)
    })
  }

  ensureURL () {
    // noop
  }
}
```

### 小结

整个的和 history 相关的代码到这里已经分析完毕了，虽然有三种模式，但是整体执行过程还是一样的，唯一差异的就是在处理location更新时的具体逻辑不同。

欢迎拍砖。
