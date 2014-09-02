---
layout: post
category : css
tagline: ""
tags : [css, CSS颜色, CSS背景]
---
{% include JB/setup %}

CSS属性允许作者指定一个元素的前景色color和背景background。背景可以是颜色也可以是图片。background属性允许作者定位背景图、重复他以及声明他是否应该相对于viewport规定或者随着文档滚动。


### 前景色：color属性

* __color__
	
	_值：_  <颜色color> | inherit

	_初始化：_ 取决于用户代理

	_应用在：_ 所有元素

	_可继承：_ 可以

	_百分比：_ 不可用

	_媒介：_ 可见媒体

	_计算值：_ 和指定值一样

<!--more-->

这个属性描述了一个元素文本内容的前景色。下边的是不同的方式来指定值为红色：

```css
em { color: red }              /* predefined color name */
em { color: rgb(255,0,0) }     /* RGB range 0-255   */
```

### 背景background

作者可以指定一个元素的背景，可以是颜色也可以是图片。依据[盒模型](/css/2014/08/05/CSS%E7%9B%92%E6%A8%A1%E5%9E%8B)，background和content，padding以及border区域相关。border的颜色和样式通过border属性设置。margin通常都是透明的。

background属性不可以继承，但是父盒的背景默认可以透过=底层发亮，因为background-color的初始值是transparent。

根元素的背景变成画布的背景，且会覆盖整个画布，固定在一个点就好像他仅仅职位根元素自身绘制的。根元素不会再次绘制背景。

然而，对于HTML文档，我们建议作者给BODY元素指定背景而不是给根元素HTML指定背景。对于根元素是HTML中的'HTML'元素或者XHTML中的'html'元素（他们的background-color的计算值是transparent且background-image的计算值是none）的文档来说，用户代理必须使用在HTML中的第一个BODY元素或者XHTML中的第一个body元素的背景属性作为根元素的背景来为画布绘制背景，且不能为那个子元素绘制背景。这样的背景也必须固定在一个点就好像他们仅仅只对于根元素绘制。

_通过这些规则，接下来的HTML文档中的画布会有一个marble的背景：_

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN">
    <TITLE>Setting the canvas background</TITLE>
    <STYLE type="text/css">
       BODY { background: url("http://example.com/marble.png") }
    </STYLE>
    <P>My background is marble.
```

_注意BODY元素的规则依旧起效即使在这个HTML源文档中不存在BODY标签，这是因为HTML解析器会对丢失的标签进行推断。_

在一个层叠上下文中元素的背景绘制在这个元素的层叠上下文的底部，在这个层叠上下文中的任何东西之下。

#### background属性：background-color，background-image，background-attachment，background-position和background

* __background-color__
	
	_值：_  <颜色color> | transparent | inherit

	_初始化：_ transparent

	_应用在：_ 所有元素

	_可继承：_ 不可以

	_百分比：_ 不可用

	_媒介：_ 可见媒体

	_计算值：_ 和指定值一样

这个属性指定来看一个元素的背景色，无论是<颜色color>还是transparent，使得通过底层的颜色发亮。

```css
h1 { background-color: #F00 }
```

* __background-image__
	
	_值：_  <统一资源标识符uri> | none | inherit

	_初始化：_ none

	_应用在：_ 所有元素

	_可继承：_ 不可以

	_百分比：_ 不可用

	_媒介：_ 可见媒体

	_计算值：_ 绝对的URI或者none

这个属性设置一个元素的背景图。当设置了背景图时，作者应该同时指定一个背景色，当图片不可用时背景色就会使用。当图片可用时，他是在background-color之上渲染的。

这个属性的值可以是uri来指定图片或者none不使用任何图片。

```css
body { background-image: url("marble.png") }
p { background-image: none }
```

background-position属性值如果是用百分比表示的固有尺寸，那么必须通过相对于建立的坐标系统的矩形尺寸来解决。

如果图片有一个固有的宽或者固有的宽，还有一个固有的宽高比，那么丢失的尺寸就通过给定的尺寸和比例计算。

如果图片有一个固有的宽或者固有的宽，没有一个固有的宽高比，那么丢失的尺寸就是假定的矩形（为了background-position属性所建立的坐标系统）的尺寸。

如果图片没有固有的尺寸但有一个固有的比例，那么假定的尺寸必须是在这个比例下的最大尺寸（不能超出为了background-position属性建立的坐标系统形成的矩形的尺寸）。

如果图片也没有固有的比例，那么假定的尺寸就是为了background-position属性建立的坐标系统形成的矩形的尺寸。

* __background-repeat__
	
	_值：_  repeat | repeat-x | repeat-y | no-repeat | inherit

	_初始化：_ repeat

	_应用在：_ 所有元素

	_可继承：_ 不可以

	_百分比：_ 不可用

	_媒介：_ 可见媒体

	_计算值：_ 和指定值一样

如果指定了背景图，这个属性指定了背景图是否重复repeat（平铺）以及怎样重复。所有的切片覆盖盒的content，padding以及border区域。

在本规范中一个行内元素background-image的位置和切片都是未定义的。未来可能会有相关的定义。

值有如下含义：

* __repeat__

	图片在水平和垂直方向都重复。

* __repeat-x__

	图片在水平方向重复。

* __repeat-y__

	图片在垂直方向重复。

* __no-repeat__

	图片不重复：只画了一份图片的拷贝。

```css
body { 
  background: white url("pendant.png");
  background-repeat: repeat-y;
  background-position: center;
}
```

![示意图](http://www.w3.org/TR/CSS21/images/bg-repeat.png)

* __background-attachment__
	
	_值：_  scroll | fixed | inherit

	_初始化：_ scroll

	_应用在：_ 所有元素

	_可继承：_ 不可以

	_百分比：_ 不可用

	_媒介：_ 可见媒体

	_计算值：_ 和指定值一样

如果指定了背景图，这个属性指定的是图是否相对于viewport固定或者随着文档滚动而滚动。

注意每一个view只有一个viewport。如果一个元素有一个滚动机制，一个固定fixed的背景不随着元素移动，一个scroll滚动的背景不和滚动机制一起移动。

即使图片是固定的，他也只是对于这个元素的content，padding和border区域可见。因此，除非图片是平铺repeat的，否则就是不可见的。

_这个例子创建了一个无线的垂直的带，当元素滚动的时候，看起来背景依然“粘”在了viewport上。_

```css
body { 
  background: red url("pendant.png");
  background-repeat: repeat-y;
  background-attachment: fixed;
}
```

不支持fixed背景的用户代理应该忽略带有fixed的声明。例如：

```css
body {
  background: white url(paper.png) scroll; /* for all UAs */
  background: white url(ledger.png) fixed; /* for UAs that do fixed backgrounds */
}
```

* __background-position__
	
	_值：_  [ [ <百分比percentage> | <长度length> | left | center | right ] [ <百分比percentage> | <长度length> | top | center | bottom ]? ] | [ [ left | center | right ] || [ top | center | bottom ] ] | inherit

	_初始化：_ 0% 0%

	_应用在：_ 所有元素

	_可继承：_ 不可以

	_百分比：_ 相对于盒自身的尺寸

	_媒介：_ 可见媒体

	_计算值：_ 如果是长度的话就是绝对的值，否则就是百分比

如果指定了背景图，这个属性指定了他初始位置。如果只有一个值指定了，第二个值就假定为center。如果至少一个值不是关键词，那么第一个值代表了水平的位置第二个值代表了垂直的位置。负的百分比和长度是允许的。

* __<百分比percentage>__

	一个是X的百分比使得横穿（对于水平方向）或者竖穿（对于垂直方向）图片的%X的点和横穿或者竖穿这个元素的padding盒的X%的点对齐。例如，0% 0%这对值，图片的左上角和padding盒的左上角对齐。100% 100%的这对值使得图片的右下角和padding盒的右下角对齐。14% 84%这对值来说，相对于图片来说横穿14%和竖穿84%的点和相对于padding盒的横穿14%和竖穿84%的点对齐。

* __<长度length>__

	一个长度L使得相对于图片左上角位置和padding盒的左上角的右边（水平）或者下边（垂直）L距离的点对齐。例如，2cm 1cm这对值，图片的左上角就在padding盒的左上角的右边2cm，下边1cm的位置。

* __top__

	和0%的垂直位置相等。

* __right__

	和100%的水平位置相等。

* __bottom__

	和100%的垂直位置相等。

* __left__

	和0%的水平位置相等。

* __center__

	和50%的垂直位置相等或者50%的水平位置相等。

然而，如果图片有一个固有比例但是没有固有尺寸的图片来说，在CSS2.1中，其位置就是未定义的。

```css
body { background: url("banner.jpeg") right top }    /* 100%   0% */
body { background: url("banner.jpeg") top center }   /*  50%   0% */
body { background: url("banner.jpeg") center }       /*  50%  50% */
body { background: url("banner.jpeg") bottom }       /*  50% 100% */
```

在本规范中一个行内元素background-image的位置和切片都是未定义的。未来可能会有相关的定义。

如果背景图相对于viewport固定的话，图片就相对于viewport定位而不是相对于元素的padding盒了。例如：

```css
body { 
  background-image: url("logo.png");
  background-attachment: fixed;
  background-position: 100% 100%;
  background-repeat: no-repeat;
} 
```

_上边的例子中，一个单独的图片放置在了viewport的右下角的位置。_

* __background__
	
	_值：_  [<'background-color'> || <'background-image'> || <'background-repeat'> || <'background-attachment'> || <'background-position'>] | inherit

	_初始化：_ 参见单独属性

	_应用在：_ 所有元素

	_可继承：_ 不可以

	_百分比：_ 只允许给background-position设置

	_媒介：_ 可见媒体

	_计算值：_ 参见单独属性

background是为了在样式表中的同一位置设置单个background属性的一个简写属性。

给定一个可用的声明，background属性首先设置所有单个background属性的值为他们的初始值，然后分配声明中详细的值。

_下边示例的第一条规则，只有一个background-color的值设置了，其他的单个属性设置的就是他们的初始值。在第二条规则中，所有的单个属性都被指定了。_

```css
BODY { background: red }
P { background: url("chess.png") gray 50% repeat fixed }
```