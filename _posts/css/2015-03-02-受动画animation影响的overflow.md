---
layout: post
category : css
tagline: ""
tags : [css, css3, CSS animation]
---
{% include JB/setup %}

最近一直在搞移动端的spa，中间遇到了一个很诡异的问题，第一次页面load，局部滚动是有效果的，但是如果经过动画切换回来之后，发现局部滚动无效了，最起码在ios8.1.3上的safari和chrome中都是如此，这肯定藏了些什么猫腻。

基本页面结构是很简单的，就是在`body`中包含了几个`page-view`，可以把他看做一个个的画面，用来切换。关于移动端重构的一些知识点，可以参见[w3cplus mobile](http://www.w3cplus.com/mobile)上有关的介绍，很详细的系列。

<!--more-->

最基本切换效果，也就是`slide`切换效果，基本`css`如下：

```css
/*前进方向进入*/
.slide.in {-webkit-animation-name:slideinfromright;animation-name:slideinfromright;}
/*以后退方向进入*/
.slide.in.reverse {-webkit-animation-name:slideinfromleft;animation-name:slideinfromleft;}
/*前进方向离开*/
.slide.out {-webkit-animation-name:slideouttoleft;animation-name:slideouttoleft;}
/*后退方向离开*/
.slide.out.reverse {-webkit-animation-name:slideouttoright;animation-name:slideouttoright;}

@-webkit-keyframes slideinfromright {
	from { -webkit-transform: translate3d(100%, 0, 0); }
	to { -webkit-transform: translate3d(0, 0, 0); }
}
@keyframes slideinfromright {
	from { transform: translateX(100%); }
	to { transform: translateX(0); }
}

@-webkit-keyframes slideinfromleft {
	from { -webkit-transform: translate3d(-100%, 0, 0); }
	to { -webkit-transform: translate3d(0, 0, 0); }
}
@keyframes slideinfromleft {
	from { transform: translateX(-100%); }
	to { transform: translateX(0); }
}

@-webkit-keyframes slideouttoleft {
	from { -webkit-transform: translate3d(0, 0, 0); }
	to { -webkit-transform: translate3d(-100%, 0, 0); }
}
@keyframes slideouttoleft {
	from { transform: translateX(0); }
	to { transform: translateX(-100%); }
}

@-webkit-keyframes slideouttoright {
	from { -webkit-transform: translate3d(0, 0, 0); }
	to { -webkit-transform: translate3d(100%, 0, 0); }
}
@keyframes slideouttoright {
	from { transform: translateX(0); }
	to { transform: translateX(100%); }
}
```

最开始的时候，每次做完动画之后，用于做动画的`class`（也就是`slide`）都是在`page-view`元素上的，在pc上当然毫无压力；反在手机上做测试的时候发现，局部滚动部分在经过动画切换回来之后不能滚动了。这肯定是要解决的，因为safari和chrome都是不行的；最后经过一番实验发现__去掉用于动画的`class`就没这样的问题了__。

暂且把这个归结于`受动画animation影响的overflow`吧，对于移动端本身来说，最好还是不要有那么的动画效果为好，简要说几点：

* 移动端内存比较小，用的多了，尤其是启用了硬件加速的情况下（当然要启用，不启用还不如不要动画）可能会引起崩溃；这是很快的体验了。

* 省电，一定要省电！做了多了，电就少很多了。

* 存在坑，例如本文中所说的这个；可能还会引起其他的一些，待发掘吧。