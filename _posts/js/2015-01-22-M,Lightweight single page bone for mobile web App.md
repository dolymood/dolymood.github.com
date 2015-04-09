---
layout: post
category : js
tagline: ""
tags : [js, web端开发, web端单页面, web端单页面框架]
---
{% include JB/setup %}

### M(mobile-router.js)，一个轻量级web移动端单页面骨架

请看[DEMO](http://demo.aijc.net/js/M/examples/)页的示例。

#### 优势：

* 使用简单、方便、轻量，基于 [history](https://developer.mozilla.org/en-US/docs/Web/Guide/API/DOM/Manipulating_the_browser_history)、[window.onpopstate](https://developer.mozilla.org/en-US/docs/WindowEventHandlers.onpopstate)。

* 无依赖，可与其他框架（库）搭配自由使用，例如：`jquery`, `zepto`, `iscroll`等。

* 任意选择字符串式模板引擎，当然最简单的就是自己拼接字符串了；同时支持异步（远程获取模板，或者去请求数据在前端构建模板）；可配置是否缓存结果模板。

* 考虑后端渲染首屏的情况，只需要按结构输出响应的片段即可，利于`SEO`，且可以实现前后端模板公用。

* 自动缓存部分画面，可配置缓存数量，默认3个。

* 每个路由都有对应的`callback`和`onDestroy`配置方法，分别用于显示了对应画面后的回调以及当该画面销毁时回调。

* 利用`CSS animation`控制动画变换效果，也可设置关闭动画效果。

* “保留”浏览器原生`hash`功能，根据`hash`，可自由跳转到对应`id`元素位置。

* 可配置`enablePushState`决定是否使用`pushstate`功能，默认启用；不启用的话，仅仅影响的是不产生历史，但是路由依旧好使的，也就是还是基于`url`的。

<!--more-->

#### 一些注意点：

* 不管画面是否已缓存在页面中，只要切换回显示了，那么就会调用`callback`，而`callback`中大多数情况需要处理监听事件、操作`DOM`，这时候可根据`this.cached`来区分；当没有缓存在页面上时为`false`，或者缓存在页面上了，但是模板更新了，这时候也为`false`。

* `getTemplate`配置方法，如果带有参数，那么该参数就是得到模板字符串后的回调函数，一定要回调的；如果没有参数，直接返回模板字符串即可。这样做，主要是为了考虑异步获取（render）模板的场景。

* `M.history`的默认的 base path 是页面中`base`元素的`href`的值，如果没有，则默认是`/`；也可以在`M.history.start()`时传入。

* 对于[history](https://developer.mozilla.org/en-US/docs/Web/Guide/API/DOM/Manipulating_the_browser_history)、[window.onpopstate](https://developer.mozilla.org/en-US/docs/WindowEventHandlers.onpopstate)不支持或者支持不够好的浏览器来说，能够正常匹配对应`route`，也就是说能够正常调用`route`配置项中的`getTemplate`以及`callback`（`onDestroy`除外），其他功能都没有，点击链接直接刷新页面。这样就可以在不改变代码的情况下，适配了不支持的浏览器。当然这种情况也可以通过取得`M.history.support`来判断，如果不支持的话，可以在调用`M.history.start`时设置参数`enablePushState`为`false`也可以，但不建议，因为没有历史记录了。

#### 使用方法：

```js
M.router.init([
	{
		path: '/',
		cacheTemplate: false, // 针对于当前的route，是否缓存模板
		getTemplate: function() {
			return '/index';
		},
		callback: function() {
			if (this.cached) return;
			// 处理操作...
		},
		onDestroy: function() {
			// 例如，处理一些解绑操作，销毁和DOM关联
		}
	},
	{
		path: '/c/:paramName',
		cacheTemplate: false, // 针对于当前的route，是否缓存模板
		getTemplate: function(cb) {
			// 这里模拟异步得到模板内容
			var that = this;
			// that.params 参数信息
			// that.query query信息
			setTimeout(function() {
				cb('/c/' + that.params.paramName);
			}, 200);
		},
		callback: function(paramName) {
			if (this.cached) return;
			// 处理操作...
		},
		onDestroy: function() {
			// 例如，处理一些解绑操作，销毁和DOM关联
		}
	}
], {
	/*是否缓存模板*/
	cacheTemplate: true,

	/*views容器选择器 如果为空，或者没有符合元素，那么views的容器元素就为body了*/
	viewsSelector: '',

	/*view的class*/
	viewClass: 'page-view',

	/*是否有动画*/
	animation: true,
	/*有动画的话，动画的类型*/
	aniClass: 'slide',

	/*蒙层class 主要是显示loading时的蒙层*/
	maskClass: 'mask',
	/*显示loading*/
	showLoading: true,

	/*缓存view数*/
	cacheViewNum: 3
});

// 也可以通过这种形式添加
M.router.get('/ddd/{dddID:int}', function(dddID) {
	// 这是 callback 回调
}, {
	cacheTemplate: true,
	getTemplate: function() {
		return '/ddd/' + this.params.dddID;
	},
	onDestroy: function() {
		// destroy
	}
});

/* 监听route change */
/* routeChangeStart 是刚开始的时候被触发，此时还没有调用getTemplate得到模板内容 */
M.router.on('routeChangeStart', function(currentRouteState) {
	
});
/*已经完成动画切换（如有动画效果的话）显示出来之后触发*/
M.router.on('routeChangeEnd', function(currentRouteState) {
	
});

// 开始 监听history
M.history.start({
	// base: '/', // base path
	// enablePushState: true // 启用pushstate
});

```

如果首屏需要后端渲染好，那么只需要在页面上加入响应的页面结构即可：

```html
<div class="page-view">后端渲染内容</div>
```

因为默认第一次初始化时，会查找页面上带有`viewClass`的元素，如果找到了，且`innerHTML`不为空，那么就不会再去调用`getTemplate`来得到模板内容了。

#### 协议

[MIT](https://github.com/dolymood/M/blob/master/LICENSE)