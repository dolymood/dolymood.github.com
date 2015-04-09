---
layout: post
category : js
tagline: ""
tags : [js, angular, jquery]
---
{% include JB/setup %}

之前看到的一个关于`angular`的问题，如题，做的是tab选项，中间大概问道：
	
	如果我用jquery的话，做起来的大概步骤就是：绑定事件，移除所有选项的class，给点击项加上class。然后如果用angular怎么做？

看起来很简单的一个问题，但是涉及的确实一个“改变”。这里就那做一个tab来看这个改变。

这里做了简化，只保留tab导航部分，内容部分就忽略了；不过不影响。

大概效果（源码）请看：[jquery版](http://demo.aijc.net/js/angular-tab/jquery.html), [angular版1](http://demo.aijc.net/js/angular-tab/angular.html), [angular版2](http://demo.aijc.net/js/angular-tab/angular2.html)

<!--more-->

#### jquery版本

目标就是完成一个依赖于`jquery`的简易组件。这里设定下：

1. 最基本切换
1. 可配置active的class
1. 可以配置选中之后的回调

基本的HTML：

```html
<div class="tab">
	<ul>
		<li class="active"><a href='#'>选项卡 1</a>
		<li><a href='#'>选项卡 2</a>
		<li><a href='#'>选项卡 3</a>
		<li><a href='#'>选项卡 4</a>
		<li><a href='#'>选项卡 5</a>
	</ul>
</div>
```

对应的CSS：

```css
a {text-decoration:none;color:#333;}
ul, li {list-style:none;margin:0;padding:0;}

.tab {border-bottom:1px solid #999;}
.tab > ul {margin-bottom:-1px;}
.tab > ul > li {display:inline-block;vertical-align:top;margin:0 5px;}
.tab > ul > li > a {display:inline-block;position:relative;padding:3px 10px;border:1px solid transparent;border-bottom-color:#999;background:#f3f3f3;transition:all .2s;}
.tab > ul > li > a:hover {border-color:#999;border-bottom-color:#f3f3f3;}
.tab > ul > .active > a {color:#fff;cursor:default;border-color:#999;border-bottom-color:#25ab5e!important;background:#25ab5e;}
```

下面就是设计这个组件了，先不去纠结于做成插件形式，或者要不要new操作。下面给出我自己写的基本实现：

```js
function Tab(ele, options) {
	this.tabEle = $(ele);
	this.options = $.extend({
		activeClass: 'active',
		onSelected: $.noop // 选中后回调
	}, options || {});

	this.currentEle = this.tabEle.find('.' + this.options.activeClass);
	this._bindEvents();
}

$.extend(Tab.prototype, {

	_bindEvents: function() {
		this.tabEle.on('click', 'li > a', $.proxy(this.handlerClick, this));
	},

	handlerClick: function(e) {
		e.preventDefault();

		var newEle = $(e.currentTarget).parent();
		
		if (newEle.is(this.currentEle)) return;

		var activeClass = this.options.activeClass;

		// 移除掉所有的li的 activeClass
		this.tabEle.find('li').removeClass(activeClass);

		// 给目标元素增加 activeClass
		newEle.addClass(activeClass);

		this.currentEle = newEle;

		// 回调
		this.options.onSelected.call(this, this.currentEle);

	},

	/**
	 * 简单销毁
	 */
	destroy: function() {
		this.tabEle.off('click');
		this.currentEle = null;
		this.tabEle = null;
		this.options = null;
	}

});
```

然后，只需要创建一个新的`Tab`“类”的实例就好：

```js
var tab = new Tab('.tab');
```

然后就完成了，可以见[jquery版](http://demo.aijc.net/js/angular-tab/jquery.html)。

#### angular版本

`jquery`版本就是按照那个基本思路完成的“先绑定事件，然后移除class，然后增加class”这样就搞定了。那么利用`angular`__该__怎么实现呢？

首先，要创建app，其实就是利用`angular`创建一个module，暂定是app的话，也就是这样：

```js
var app = angular.module('app', []);
```

之后，希望有一个控制器，来控制整个应用，取名叫main：

```js
app.controller('main', function() {
	
	// ....
})
```

对应的基本的HTML：

```html
<body ng-app="app">
	<div ng-controller="main as tabCtrl">
		<!-- tab 部分 -->
	</div>
</body>
```

然后就需要加上tab部分了，基本结构和jquery版本一样，差异的就是每一个选项怎么写呢，当然，也可以直接和jquery版本一样，直接列出来，然后给每一个选项绑定上click事件；那有没有更方便的做法，当然有，谨记：__angular是以数据驱动的框架__，咱们可以把各个选项放到js中，当然考虑数组，存放有哪些选项等信息的数组。然后通过`angular`的`ng-repeat`指令得到需要的选项们。然后上边的HTML就变成了这样：

```html
<div ng-controller="main as tabCtrl">
	<div class="tab">
		<ul>
			<li ng-class="{active:tabCtrl.currentTab === tab}" ng-repeat="tab in tabCtrl.tabs"><a href='#' ng-bind="tab.name" ng-click="tabCtrl.handC(tab)"></a>
		</ul>
	</div>
</div>
```

具体每个指令怎么使用，这里不说了，请看官方文档就好了。简单来看就是利用`ng-repeat`生成每一个选项，然后利用`ng-click`指令给每一个选项绑定click，用于处理逻辑。到处突出了声名式语法。

js中的`controller`怎么写呢，只需要加上“数据”就好。

```js
app.controller('main', function() {
	
	this.tabs = [
		{name: '选项卡 1'},
		{name: '选项卡 2'},
		{name: '选项卡 3'},
		{name: '选项卡 4'},
		{name: '选项卡 5'}
	];
	
	// 当前tab 默认选中第一个
	this.currentTab = this.tabs[0];

	var that = this;
	this.handC = function(tab) {
		if (tab === that.currentTab) return;
		that.currentTab = tab; // 当然的tab 赋值为点击tab
		handlerClick(that.currentTab);
	};

	var handlerClick = function() {
		// ...
	};

});
```

这样就完成了。可以看到js代码中没有任何的DOM操作代码，仅做的就是处理“数据”，这样关注的层面就彻底的关注于处理纯粹的业务逻辑了。代码量显而易见，效率大大提高。

见[angular版](http://demo.aijc.net/js/angular-tab/angular.html)。

#### angular版本2

能不能将上边的例子做成类似于组件的形式呢，以便实现复用。答案是当然可以的，因为`angular`提供了给开发者自定义“组件”的接口：指令`directive`；通过自定义指令，可以做组件了。

依旧是在jquery版本中的目的，配置active class以及onSelected选中回调。怎么来配置呢，很简单，继承。angular中的scope是一级一级嵌套且是继承的。但是这里做的是组件，应该保持其独立，不和他之外的scope发生啥耦合；只需要给自定义指令指定单独的scope就好了，然后通过给定的语法“继承”外层scope的数据即可。

直接看代码吧：

```js
app.directive('ngdTab', function() {

	var tpl = '<div class="tab">' +
			'<ul>' +
				'<li ng-class="[$parent.currentTab === tab && activeClass || \'\']" ng-repeat="tab in tabs"><a href=\'#\' ng-bind="tab.name" ng-click="handlerClick(tab)"></a>' +
			'</ul>' +
		'</div>';

	return {
		
		replace: true,

		template: tpl,
		
		scope: {
			tabs: '=ngdTab',
			onSelected: '&'
		},

		link: function(scope, ele, attrs) {

			var activeClass = attrs.activeClass || 'active';

			scope.activeClass = activeClass;

			scope.currentTab = scope.tabs[0];

			scope.handlerClick = function(tab) {
				if (tab === scope.currentTab) return;
				scope.currentTab = tab;
				scope.onSelected({
					tab: tab
				});
			};

		}

	};

});
```

可以看到link中处理的一些逻辑和上边的angular版本基本没啥大的变化。那该怎么使用这个指令呢？

```html
<div ng-controller="main">
	<div ngd-tab="tabs" on-selected="seled(tab)">
	</div>

	<div ngd-tab="tabs" on-selected="seled(tab)">
	</div>
</div>
```

这样就OK了，可以看到这里有两个div加了ngd-tab属性，对应的结果就是有两个tab，他们互不影响；当然，需要定义数据：

```js
app.controller('main', ['$scope', function($scope) {
	
	$scope.tabs = [
		{name: '选项卡 1'},
		{name: '选项卡 2'},
		{name: '选项卡 3'},
		{name: '选项卡 4'},
		{name: '选项卡 5'}
	];

	$scope.seled = function(tab) {
		console.log(tab);
	};

}]);
```

见[angular版2](http://demo.aijc.net/js/angular-tab/angular.html)。

切记：

1. 以数据为核心
1. 拒绝操作DOM（自定义指令是唯一的一个可能去操作的地方）
1. 考虑换个角度来看“世界”

个人浅见，希望对有点“迷茫”的童鞋有所帮助吧，O(∩_∩)O~
