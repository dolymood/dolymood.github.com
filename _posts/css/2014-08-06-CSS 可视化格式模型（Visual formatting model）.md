---
layout: post
category : css
tagline: ""
tags : [css, CSS可视化格式模型]
---
{% include JB/setup %}

### 可视化格式模型简介

这一章以及[下一章](/css/2014/08/08/CSS%20%E5%8F%AF%E8%A7%86%E5%8C%96%E6%A0%BC%E5%BC%8F%E6%A8%A1%E5%9E%8B%E7%BB%86%E8%8A%82%EF%BC%88Visual%20formatting%20model%20details%EF%BC%89)来描述可视化格式模型：用户代理针对于可视化媒介是怎样处理文档树的。

在可视化格式模型中，在文档树中的每一个元素会通过盒模型生成0个或者多个盒子。这些盒子的布局是这样管理的：

* 框的尺寸和类型。

* 定位体系（普通流、浮动float、和绝对定位）。

* 文档树中的元素之间的关系。

* 额外的信息（例如，viewport的大小、图片的固有尺寸等等）。

<!--more-->

本章以及下一章中定义的属性是应用在‘连续媒体’和‘分页媒体’上的。然而，当应用到分页媒体上的时候，margin属性的意义会有所不同。

可视化格式模型不会涉及格式化的所有细节（方面）。本规范中不包括那些符合规范的用户代理们的不同行为的那些格式化问题。

#### 视口 viewport

连续媒体的用户代理通常会提供给用户一个viewport，用户可以用来查阅文档。当viewport的大小发生改变的时候，用户代理可能会改变文档的布局。

当viewport比文档渲染您所需的画布区域小的时候，用户代理应该提供一个滚动条。每一个画布上最多有一个viewport，但是用户代理可以渲染多个画布。

#### 包含块

在CSS2.1中，很多的盒子的定位和尺寸的计算都取决于一个矩形的边界，这个矩形就是包含块。一般来说，生成的盒子扮演者他的子孙盒子的包含块的角色；我们称：一个盒子为他的子孙节点创建了包含块。

每一个盒子都相对于其包含块摆放，但是不会被包含块限制。他可能会溢出（overflow）。

在下一章中将会讨论包含块的细节。

### 控制盒子的产生

接下来的段落将会描述在CSS2.1中有的盒子类型。从部分角度来说，一个盒子的类型影响了他在可视化格式模型中的行为。下边所描述的display属性，决定了一个盒子的类型。

#### 块级元素（Block-level elements）和块盒（block boxes）

块级元素是源文档中的那些被可视化格式化为盒状的元素。这些display属性可以使得一个元素是块级的： 'block', 'list-item' 和 'table'。

块级盒是在一个BFC（块级格式化上下文）中的盒子。每一个块级元素会生成一个包含着子孙盒子的主块级盒（主盒），并且生成内容且这个盒子会参与任何定位方案。一些块级元素除了主盒外可能还会生成额外的盒，例如：'list-item'元素。这些额外的框相对于主盒摆放。

除了table盒子（将会在后边的章节介绍）和替换元素，一个块级盒子一般也是一个块容器盒。一个块容器盒要么只包含块级盒，要么只包含创建了IFC（inline formatting context 行内级格式化上下文）的行内级盒。并不是所有的块容器盒就是块级盒：非替换的inline blocks和非替换的table cells是块容器，但是却不是块级盒。块级盒同样是块容器的就成为块级盒（块盒）。

这三个术语“块级框”，“块容器盒”和“块盒”有时会清晰的简称为“块”。

#### 匿名块盒

在一个像这样的文档中：

```html
<DIV>
  Some text
  <P>More text
</DIV>
```

(DIV和P的display都是block)，DIV中同时出现了行内内容和块内容。为了更容易的格式化，我们假定围绕着“Some text”有一个匿名块盒。

![匿名块盒](http://www.w3.org/TR/CSS21/images/anon-block.png)

换句话说：如果一个块容器盒里边有一个块级盒，那么我们强制其内部只有块级盒。

当一个行内盒包含着一个在普通流中的块级盒的时候，这个行内盒（和他的在相同线盒中的行内祖先们）会在块级盒周围断掉（和任何的连续的或者仅仅只是被空白white space和/ 或者不在普通流中的元素分开的同级块级兄弟们），被拆分成两个盒子（及时一边是空的），分别在块级盒的两边。这个线盒在断掉之前和之后的线框被匿名的块盒包着，并且这个块级盒成了那些匿名盒的兄弟。当这样的行内盒被相对定位影响到了，因此产生的任何换算也会影响包含在这个行内盒的块级盒。

_这种模式会被应用到下边的示例中：_

```css
p    { display: inline }
span { display: block }
```

且在HTML文档中是这样的：

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN">
<HEAD>
<TITLE>Anonymous text interrupted by a block</TITLE>
</HEAD>
<BODY>
<P>
This is anonymous text before the SPAN.
<SPAN>This is the content of SPAN.</SPAN>
This is anonymous text after the SPAN.
</P>
</BODY>
```

_这个P元素包含着一个在一个块级元素之前的匿名文本区块C1和在其之后的匿名文本区块C2。结果上来看，BODY的块级盒，包含了一个围绕着C1的匿名块盒，SPAN块盒和另外一个围绕着C2的匿名块盒。_

匿名盒的属性是继承自最近的非匿名盒（例如之前的例子中的匿名块盒，他最近的非匿名盒就是DIV这个盒子）。非继承性的属性会有他们的初始值。例如，匿名盒的font就会继承自DIV，但是margin的值将会是0。

一个元素生设置了属性导致产生匿名块盒同样适用于其内容。例如，如果在上边的例子中，如果P元素设置了边框，那么border就会围绕着C1和C2绘制。

![示例结果](https://github.com/dolymood/blog/raw/master/pics/001.png)

匿名块盒在计算相对于他的百分比的值的时候会被忽略掉：利用最近的非匿名的父级（祖先）框替换。例如，上边的例子中，如果在DIV内的匿名块盒的孩子需要知道他的包含块的高height去计算一个百分比的高的话，那么他就会使用由DIV格式化出的包含块的高而不是匿名块盒的。

#### 行内级元素和行盒

行内级元素就是在源文档中的那些不会形成新的内容块的元素；内容成行分布。这些display的属性会使得一个元素行内级：'inline', 'inline-table', 和 'inline-block'。行内级元素生成行内级盒（inline-level boxes），这些盒子在一个行内格式化上下文中（IFC）。

一个行盒是一个既是行内级并且它的内容在一个IFC中的盒子。一个非替换元素，假设说display属性是inline的话，那么就会生成一个行盒。是行内级盒但是不是行盒（例如替换的行内级元素，inline-block元素和inline-table元素）被称作‘原子行内级盒（atomic inline-level boxes）’，因为他们在他们的IFC中作为单独的不透明的盒子存在。

##### 匿名行盒

在一个块容器元素（不是在一个行内元素中）中直接包含的任何文本必须当成一个匿名的行内元素对待。

在一个文档中的HTML是类似于这样的：

```html
	<p>Some <em>emphasized</em> text</p>
```

P生成一个块盒，在其内部有几个行内盒。EM生成一个行盒，但是其他的盒都是P生成的行盒。他们被称作匿名行盒，因为他们并没有被行内级元素包裹。

这样的行盒从他们的父块盒上继承可继承的属性。不可继承的属性将会有他们的初始值。在这个例子中，匿名盒的颜色是继承自P的，但是背景就会是透明的。

根据white-space的属性，折叠了的空白内容是不会产生任何匿名行盒的。

如果上下文中匿名盒的意义清楚的话，在本规范中，不管是匿名的行盒还是匿名的块盒都能简单的称为匿名盒。

当格式化table的时候会遇到更多的匿名盒类型。

#### display属性

* __display__
	
	_值：_  inline | block | list-item | inline-block | table | inline-table | table-row-group | table-header-group | table-footer-group | table-row | table-column-group | table-column | table-cell | table-caption | none | inherit

	_初始化：_ inline

	_应用在：_ 一切元素

	_可继承：_ 不可以

	_百分比：_ 不可用

	_媒介：_ 所有媒介

	_计算值：_ 依赖于文本

这个属性的值有一下的意思：

* __block__
	
	一个元素可以生成一个块盒。

* __inline-block__
	
	一个元素可以生成一个行内级块容器。在其内部被格式化为一个块盒且这个元素本身是被格式化为一个原子行级块。

* __inline__
	
	一个元素可以生成一个或者多个行盒。

* __list-item__
	
	一个元素（在HTML中的LI）可以生成一个主块盒和一个标记盒。

* __none__
	
	这个值导致元素不会出现在格式化结构中。也不会生成任何的盒，这个元素以及他的内容完全从格式化的结构中的移除掉。这种行为不能被他的子孙的display属性覆盖掉。

	请注意display是none的元素不会创建一个可视的盒；他根本就不创建盒。注意和visibility的区别。

* __table, inline-table, table-row-group, table-column, table-column-group, table-header-group, table-footer-group, table-row, table-cell, 和 table-caption__
	
	一个元素可以表现的像一个table元素（在后边的章节中会详细介绍table）。

计算值和指定值是一样的，除了定位了的和浮动了的元素以及根元素。针对于根元素来说，计算值的变化会在稍后的段落（display、position和float之间的关系）中介绍。

需要注意的值display的初始值是inline，在用户代理中的默认样式表可能会覆盖这个值。

_下边有关于display属性的几个例子：_

```css
p   { display: block }
em  { display: inline }
li  { display: list-item } 
img { display: none }      /* Do not display images */
```

### 定位体系

在CSS2.1中，一个盒可以根据三种定位体系布局：

1. 普通流。在CSS2.1中，普通流包括块级盒的块格式化，行内级盒的行格式化以及块级和行级的相对定位。

1. 浮动。在浮动模式中，一个盒首先按照普通流去布局，然后脱离出普通流并且尽可能的向左或向右偏移。内容可以沿着浮动边排列。

1. 绝对定位。在绝对定位模式中，一个盒完全从普通流中移除并且根据其包含块定位。

一个元素如果浮动了、绝对定位了或者是根元素，那么他叫做脱离（普通）流。如果不是脱离流的话，就叫做在流中（普通）。A元素的流由A以及在其流中的元素（这些元素的最近的脱离流的祖先是A）组成的。

#### 选择定位体系：'position'属性

在CSS2.1的定位算法中规定的position和float属性是用来计算一个盒的位置（position）的。

* __position__
	
	_值：_  static | relative | absolute | fixed | inherit

	_初始化：_ static

	_应用在：_ 一切元素

	_可继承：_ 不可以

	_百分比：_ 不可用

	_媒介：_ 可见媒介

	_计算值：_ 和指定值一样

这个属性的值有以下意思：

* __static__
	
	这个盒是一个普通的盒，根据普通流布局。top、right、bottom和left属性是无效的。

* __relative__
	
	这个盒根据普通流布局（叫做在普通流中的位置）。然后这个盒会偏移出他的原本的位置。当盒B是相对定位的，紧邻其后的盒的计算就像B没有便宜一样。在table-row-group, table-header-group, table-footer-group, table-row, table-column-group, table-column, table-cell, 和table-caption元素的position:relative是无效的。

* __absolute__
	
	这个盒的位置（可能大小）是由top、right、bottom和left属性确定的。这些属性的偏移是根据其包含块的。绝对定位的盒是从普通流中脱离的。这意味着他对紧邻其后的同级元素没有影响。同样的，尽管绝对定位的盒有margin，但是不会和其他的margin折叠。

* __fixed__
	
	这个盒的位置是通过绝对定位的模式，但是，这个盒需要根据一些参考来固定。和absolute模式一样，他的margin不会和其他的margin发生折叠。在手持设备、投影、屏幕、打字机以及电视的媒介类型的情况下，这个盒子是根据viewport固定的（不会随着滚定移动）。在打印的媒介类型的情况下，这个盒子在每一张上都会渲染，并且是根据page盒来固定，即使page是穿过一个viewport的。而其他的媒介类型，没啥效果。作者可以用依赖媒介的方式去指定fixed。就像：

	```css
	@media screen { 
	  h1#first { position: fixed } 
	}
	@media print { 
	  h1#first { position: static }
	}
	```

	用户代理必须不能对绝对定位的盒分页。

在根元素上，用户代理可以将position看成是static的。

#### 盒偏移：top、right、bottom、left

一个元素的position属性如果不是static的话就可以说他是一个定位（positioned）元素。定位元素会生成定位盒，通过下边的四个属性进行布局：

* __top__
	
	_值：_  <长度length> | <百分比percentage> | auto | inherit

	_初始化：_ auto

	_应用在：_ 定位元素

	_可继承：_ 不可以

	_百分比：_ 根据其包含块的高

	_媒介：_ 可见媒介

	_计算值：_ 如果指定了长度，那么就转换成绝对长度；如果指定了百分比，就是指定的值；其他情况就是auto。

这是属性指定了绝对定位了的盒的top margin边界到他的包含块的上边界有多远。对于相对定位了的盒，就是根据其自身的上边界进行偏移。

* __right__
	
	_值：_  <长度length> | <百分比percentage> | auto | inherit

	_初始化：_ auto

	_应用在：_ 定位元素

	_可继承：_ 不可以

	_百分比：_ 根据其包含块的宽

	_媒介：_ 可见媒介

	_计算值：_ 如果指定了长度，那么就转换成绝对长度；如果指定了百分比，就是指定的值；其他情况就是auto。

和top类似，但是是指定了绝对定位了的盒的right margin边界到他的包含块的右边界有多远。对于相对定位了的盒，就是根据其自身的右边界进行偏移。

* __bottom__
	
	_值：_  <长度length> | <百分比percentage> | auto | inherit

	_初始化：_ auto

	_应用在：_ 定位元素

	_可继承：_ 不可以

	_百分比：_ 根据其包含块的高

	_媒介：_ 可见媒介

	_计算值：_ 如果指定了长度，那么就转换成绝对长度；如果指定了百分比，就是指定的值；其他情况就是auto。

和top类似，但是是指定了绝对定位了的盒的bottom margin边界到他的包含块的下边界有多远。对于相对定位了的盒，就是根据其自身的下边界进行偏移。

* __left__
	
	_值：_  <长度length> | <百分比percentage> | auto | inherit

	_初始化：_ auto

	_应用在：_ 定位元素

	_可继承：_ 不可以

	_百分比：_ 根据其包含块的宽

	_媒介：_ 可见媒介

	_计算值：_ 如果指定了长度，那么就转换成绝对长度；如果指定了百分比，就是指定的值；其他情况就是auto。

和top类似，但是是指定了绝对定位了的盒的left margin边界到他的包含块的左边界有多远。对于相对定位了的盒，就是根据其自身的左边界进行偏移。

这四个属性的值有以下的这几种意思：

* __<长度length>__
	
	偏移相对的边界的一个固定的值。负值也是可以的。

* __<百分比percentage>__
	
	偏移的值是相对于包含块的宽或者高计算的。负值也是可以的。

* __auto__
	
	对于非替换元素，这个值的效果取决于和其相关的属性的值是auto一样。详情参见‘绝对定位了的非替换元素的宽和高’段落。针对于替换元素，这个值的效果只取决于被替换的内容的原始尺寸。详情参见‘绝对定位了的替换元素的宽和高’段落。

### 普通流

在普通流中的盒属于同一个格式化上下文（可能是块级也可能是行级，但是不会都是）。块盒参与块级格式化上下文。行内盒参与行内格式化上下文。

#### BFC 块级格式化上下文

浮动元素、绝对定位元素、不是块盒的块容器（例如inline-block table-cell table-caption）元素，以及overflow不是visible的块盒会给他们的内容创建新的BFC。

在一个BFC中，盒是从其包含块的顶部一个挨着一个在垂直方向上布局的。两个在垂直方向上相邻的的盒子之间的间距是由margin属性确定的。在同一个BFC中的两个毗邻的块级盒在垂直方向的margin会发生折叠。

在一个BFC中，每一个盒的左外边挨着其包含块的左边界（如果是从右到左的格式化的话，挨着右边）。即使存在浮动也是如此（尽管一个盒的线盒会由于浮动而收缩），除非这个盒创建了新的BFC（在这种情况下盒本身可能由于浮动而变得更加狭窄）。

#### IFC 行内格式化上下文

在一个IFC中，盒是从其包含块的顶部一个挨着另一个在水平方向上布局的。在这些盒的水平方向上的margin、border以及padding都是有效的。盒可以通过几种不同的方式在垂直方向上对齐：他们的top或者bottom对齐或者他们里边的文本的baseline（基线）对齐。包含着盒的矩形区域，会形成一条线，称作行盒。

一个行盒的宽是由包含块和浮动的出现决定的。一个行盒的高是由在‘行高计算’段落中的规则决定的。

一个行盒的高往往对于其包含的全部盒都是足够的。然而，他可能要比他包含的内容中的最高的盒的高还要高（例如以基线对齐的盒）。当一个盒B的高比包含着他的行盒的高要低的时候，行盒中的B的垂直对齐就由vertical-align这个属性决定。当多个行级盒在水平方向上无法放在一个单独的行盒中时，他们就会被分配在两个或者多个垂直堆叠的行盒中。因此，一个段落就是行盒在垂直方向上堆叠形成的。行框是不会存在垂直方向上的分割（除非其他地方规定了）和重叠的。

一般来说，一个行盒的左边界会和其包含块的左边挨着并且右边界会和其包含块的右边界挨着。然而，浮动盒可能处在包含块边缘和行盒边缘之间。因此，尽管一般情况下在相同的IFC中的的行盒都是一样宽的（包含块的宽），但是他们可能会因浮动存在缩短了可用宽度，也就是在宽度上发生了变化。在同一个IFC中的行盒的高往往都是不一样的（例如，一行包含着图片，而其他的包含的只有文本内容）。

当一行中的行内级盒的总宽小于包含着他们的行盒的宽的时候，他们在行盒中水平方向的对齐主要取决于text-align这个属性。如果属性的值是justify的话，用户代理也可以拉伸行内盒（inline-table和inline-block盒除外）中的空间和文字。

当一个行内盒超出了他的行盒的宽的时候，他就会被分成几个盒，这几个盒会分布到几个行盒内。如果一个行内盒不能被分割（例如，行内盒只包含单个字符，或者语言特殊的断字规则不允许在行内盒里换行，或者行内盒受到带有nowrap或pre值的white-space属性的影响），那么行内盒会溢出行盒。

当一个行内盒被分割了，margin、padding和border在所有的分割处没有视觉效果。

行内盒还可能由于双向文本处理（bidirectional text processing），在同一个行盒中被分割成几个盒。

行盒是为了控制在IFC中的行内级内容的需要而创建的。不包含文本、保留空白符，和margin、padding以及border是非0的行内元素，以及在其他普通流中的内容和不是以换行结束的行盒必须当做零高度行盒对待以达到确定在他们中的元素的位置的目的且以不存在其他目的来对待。

_下边的就是一个行内盒结构的例子。P包含了匿名文本以及用EM和STRONG包含的文本内容：_

```html
<P>Several <EM>emphasized words</EM> appear
<STRONG>in this</STRONG> sentence, dear.</P>
```

_P元素生成了一个块盒，这个块盒包含了5个行内盒，其中的3个是匿名的：_

* _匿名的："Several"_

* _EM："emphasized words"_

* _匿名的："appear"_

* _STRONG："in this"_

* _匿名的："sentence, dear."_

_对于段落的格式化，用户代理将这5个盒放在行盒中。在这个例子中，P元素生成的盒为了这些行盒建立了包含块。如果这个包含块是足够宽的话，所有的行内盒都会放在一个单独的行盒中：_

Several _emphasized words_ appear __in this__ sentence, dear.

_如果不够宽的话，这些行内盒就会被分割成跨越几个行盒分布。上边的段落可能是下边的样子：_

Several _emphasized words_ appear

__in this__ sentence, dear.

_或者是这样的：_

Several _emphasized_

_words_ appear __in this__

sentence, dear.

在上边的例子中，EM盒被拆分成了两个EM盒（"split1"和"split2"）。在split1之后和split2之前margin、border、padding或者文本装饰都是没有视觉效果的。

_考虑下边的例子：_

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN">
<HTML>
  <HEAD>
    <TITLE>Example of inline flow on several lines</TITLE>
    <STYLE type="text/css">
      EM {
        padding: 2px; 
        margin: 1em;
        border-width: medium;
        border-style: dashed;
        line-height: 2.4em;
      }
    </STYLE>
  </HEAD>
  <BODY>
    <P>Several <EM>emphasized words</EM> appear here.</P>
  </BODY>
</HTML>
```

_取决于P的宽，盒可能表现如下：_

![行内布局示例](http://www.w3.org/TR/CSS21/images/inline-layout.png)

_* margin插在了"emphasized"之前，"words"之后。_

_* padding插在了"emphasized"的前面、上面以及下面，和"words"的后面、上面以及下面。每一种情况dashed的border都是渲染在了3个边上。_

#### 相对定位

一旦一个盒按照普通流或者浮动定位了，那么他还可以相对于该位置进行偏移。这就叫相对定位。用这种方式偏移一个盒B1，对其之后的盒B2是没有任何影响的：如果B1没有偏移，B2就已经给定了一个位置，那么当B1偏移了之后，B2还在原来的位置不发生变化。这也就意味着相对定位可能会产生盒的重叠。然而，如果相对定位引起"overflow:auto"或者"overflow:scroll"框的溢出，用户代理必须允许用户访问里边的内容，也就是需要创建滚动条，这可能会影响布局。

一个相对定位了的盒保持着他在普通流中的尺寸，包括换行以及其原来的位置。‘包含块’段落解释了当一个相对定位了的盒会建立一个新的包含块。

针对于相对定位的元素，left和right使得盒在水平方向上移动，不改变其大小。left能够让盒子向右边移动，right能让盒子向左边移动。由于盒不能被分割或者拉伸，所以使用值通常是：left = -right。

* 如果left和right都是auto的话，使用值就是0。

* 如果left是auto，他的使用值就是right的负值。

* 如果right被指定为auto，那么他的使用值就是left的负值。

* 如果left和right都不是auto，那么定位就很难了，他们中的一个必须被忽略。如果包含块的direction是ltr的话，使用left的值，right变成left的负值。如果direction是rtl的话，使用right的值，left的值是right的负值。

_例子，下边的三条规则是相同的：_

```css
div.a8 { position: relative; direction: ltr; left: -1em; right: auto }
div.a8 { position: relative; direction: ltr; left: auto; right: 1em }
div.a8 { position: relative; direction: ltr; left: -1em; right: 5em }
```

top和bottom属性使得相对定位元素在上下放下上移动，但是不改变尺寸。top使得盒子向下移动，bottom使得盒子向上移动。同样的盒子不能被分割或者拉伸，所以有一个必须被忽略，使用值通常是：top = -bottom。

* 如果top和bottom都是auto，那么使用值就是0

* 如果他们其中之一死auto，那么他的值就是另一个值的负值

* 如果他们都不是auto，bottom会被忽略

在‘普通流、浮动和绝对定位’段落中提供了相对定位的例子。

### 浮动

浮动就是一个盒在当前行向左或者向右偏移。浮动最有意思的地方就是内容可以沿着他的边排列（除非有clear属性）。内容会沿着一个左浮动的盒的右边向下排列且会沿着一个右浮动的盒的左边向下排列。接下来就是关于浮动定位和内容浮动的介绍；float属性准确描述控制浮动行为的规则。

一个浮动的盒子会向左或者向右偏移，直到遇到了他的包含块的边界或者另一个浮动元素的外边界。如果有一个行盒存在，那么浮动盒的外顶部会和当前的线盒的顶部对齐。

如果在水平方向上没有足够的空间浮动的话，他就会向下移动，直到有足够的空间或者没有更多浮动存在。

由于浮动不在普通流中，在该浮动盒之前或者之后的非定位块盒垂直排列就如同浮动不存在一样。然而，浮动盒之后创建的行盒为了给浮动盒提供空间而缩短。

如果被缩短的行盒太小了以至于不能包含任何内容的话，那么这个行盒就会下移，直到有足够的空间或者没有更多的浮动存在。在当前行浮动盒之前的内容都将被重排到浮动另一边相同的行里。换句话说，如果在一行中，行内级盒在一个左浮动之前（同时左浮动也被放置在当前行），剩余的空间能够放下这个行内级盒的话，他就会和当前线盒的顶端对齐，并且在这一行的的行内级盒将会移动到这个浮动的右边；反之亦然，对于rtl和右浮动框也是一样的。

table、块级替换元素或者在普通流中的创建了新的BFC的元素的border box必须不能覆盖在同一个BFC中的任何的浮动元素。如果有必要的话，实现者应该通过把元素放到前面的浮动的下边来清理之前说的元素，但是如果有足够的空间，也可以把他紧邻浮动放置。他们甚至可以让border box比在下一章中定义的还要窄。在CSS2中没有定义用户代理把先前说的紧邻浮动的元素如何放置或者说这个元素可以变得多窄。

_示例，下边的文档片段中，包含块太窄了包不下和浮动紧邻的内容，因此内容就会移动下浮动的下边，他的对齐方式取决于text-align属性：_

```
p { width: 10em; border: solid aqua; }
span { float: left; width: 5em; height: 5em; border: solid blue; }


...


<p>
  <span> </span>
  Supercalifragilisticexpialidocious
</p>
```

_看起来可能是下边这样的：_

![空间够示例](http://www.w3.org/TR/CSS21/images/supercal.png)

几个浮动可能是相邻的，并且这种模式同样可以应用到在相同行中的相邻的浮动。

_下边的示例中的几条规则使得所有的带有class="icon"的IMG的盒浮动到左边：_

```css
img.icon { 
  float: left;
  margin-left: 0;
}
```

_考虑下边的HTML源码以及样式：_

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN">
<HTML>
  <HEAD>
    <TITLE>Float example</TITLE>
    <STYLE type="text/css">
      IMG { float: left }
      BODY, P, IMG { margin: 2em }
    </STYLE>
  </HEAD>
  <BODY>
    <P><IMG src=img.png alt="This image will illustrate floats">
       Some sample text that has no other...
  </BODY>
</HTML>
```

_IMG盒浮动到了左边。接下来的内容格式化到了浮动的右边，和浮动在同一行的位置开始。在浮动右边的行盒由于浮动而缩短了，但是在浮动之后的他们的宽就是normal宽了（也就是由P创建的包含块的宽度）。这个文档可能被格式化为下边这样：_

![示例](http://www.w3.org/TR/CSS21/images/floateg.png)

_如果文档是这样的也会有同样的效果：_

```html
<BODY>
  <P>Some sample text 
  <IMG src=img.png alt="This image will illustrate floats">
           that has no other...
</BODY>
```

_因为浮动左边的内容被浮动替换掉了，然后重排到了他的右边。_

就像之前margin折叠的状态一样，浮动的盒的margin永远不会和他相邻的盒发生折叠。因此，在上边的例子中，P盒和浮动的IMG盒在垂直方向的margin是不会折叠的。

如果浮动创建了新的层叠上下文（除了定位了的元素和的确创建了新的堆叠上下文但是是其父堆叠上下文的一部分的元素）的话，那么他的内容会堆叠。浮动可以覆盖在普通流中的盒（例如，一个普通的盒紧邻着有着负margin的浮动的情况下）。当这种情况发生的时候，浮动会渲染到在流中的非定位块盒之前，但是在流中的行内盒之后。

_下边是另一个图，展示了当一个浮动覆盖在普通流中元素的border之上的时候发生的：_

![示例](http://www.w3.org/TR/CSS21/images/float2p.png)

接下来额这个例子展示了利用clear属性来阻止紧邻浮动的内容发生排列的样子。

_指定这样的规则：_

```css
p { clear: left }
```

_格式化之后的样子：_

![示例](http://www.w3.org/TR/CSS21/images/floatclear.png)

#### 浮动定位：float属性

* __'float'__

	_值：_ left | right | none | inherit

	_初始化：_ none

	_应用在：_ 一切元素上，但是详情请看‘display、position和float之间的关系’

	_可继承：_ 不可以

	_百分比：_ 相对于包含块的宽度

	_媒介：_ 可见媒介

	_计算值：_ 和指定值一样

这个属性指定了一个盒是应该往左、右浮动或者根本不浮动。他可以设置在任何元素上，但是只能应用在那些没有绝对定位的元素上面。属性的值有如下的意思：

* __left__
	
	这个元素会生成一个向左浮动的块盒。内容排到这个盒子的右边，从顶部开始。

* __right__
	
	和left很像，除了这个盒是向右浮动的，且内容会排到这个盒子的左侧，从顶部开始。

* __none__
	
	这个盒不浮动。

用户代理应该把根元素上的float当none处理，也就是说在根元素上的浮动是无效的。

下边给出了控制浮动行为的一些规则说明：

1. 一个左浮动的盒的左外边不能出现在他的包含块的左边界的左边。对于右浮动也有类似的规则。也就是说，左浮动元素的左外边界不能超出他的包含块的左边界；右浮动元素的右外边界不能超出他的包含块的右边界。

1. 如果当前盒是左浮动的，且在源文档中存在更早生成的左浮动的盒的话，那么对于每一个之前的盒来说，要么当前盒的左外边出现在之前盒的右边界的右侧，要么他的顶部必须在之前盒的底部之下。也就是说，当前浮动盒的定位会受到先前生成的同向浮动盒的影响，它们不能相互覆盖。当前浮动盒需要紧挨着先前同向浮动盒的外边界进行定位，如果当前行空间不足，则折行，放置到它之前浮动盒的下面。

1. 一个左浮动的盒的右外边不能出现在他右侧的任何右浮动盒的左外边的右侧。对于右浮动也有类似规则。也就是说，同一行中不同向的浮动盒不能够有重叠的现象。

1. 一个浮动盒的顶外边（outer edge）不能比他的包含块的顶部还要高。当一个浮动盒发生在两个叠加的外边距之间的时候，浮动的定位好像是他有另外一个空的匿名块级父盒在普通流中。这个匿名的父盒的位置是在margin折叠的段落中规则定义的。也就是说，当浮动盒处于两个折叠外边距中间的时候，会被当做包含在一个空的块盒中，他上边和下边的外边距会穿过它发生外边距叠加，如同这个浮动盒不存在一样。

1. 一个浮动盒的顶外边不能高于源文档中之前元素产生的块或者浮动盒的顶。

1. 一个元素的浮动盒的顶外边不能高于源文档中之前元素产生的任何一个包含着盒的行盒的顶。

1. 一个在其左边有另一个左浮动的盒的左浮动的盒，他的右外边不能出现在他的包含块的右边界的右侧（宽松点的话：一个左浮动不能超出右边界，除非他已经尽可能的靠左了）。对于右浮动元素也有类似的规则。

1. 一个浮动盒必须放置的尽可能的高。

1. 一个左浮动盒必须尽量靠左，右浮动的盒必须尽量的靠右。在更高的位置或者更左/右的选择中选择前者，也就意味着高优先。

但是在CSS2.1中，如果在BFC中有一个在普通流中的垂直方向上有负margin的盒，那么浮动的位置是在把那个负margin被设置为0的情况下的位置之上的，浮动的位置是未确定的。

在这些规则中涉及到的其他元素和浮动元素在相同的BFC中的。

_下面的HTML片段的结果是b浮动到了右边。_

```html
<P>a<SPAN style="float: right">b</SPAN></P>
```

_如果P元素够宽的话，a和b将会是在在两边。像这样：_

![示例](http://www.w3.org/TR/CSS21/images/float-right.png)

#### 控制紧邻浮动的排列：clear属性

* __'clear'__

	_值：_ none | left | right | both | inherit

	_初始化：_ none

	_应用在：_ 块级元素

	_可继承：_ 不可以

	_百分比：_ 相对于包含块的宽度

	_媒介：_ 可见媒介

	_计算值：_ 和指定值一样

这个属性指定了一个元素的盒在哪一边不可以和之前的浮动盒相邻。clear属性不考虑这个元素自身内部的或者在另外的BFC中的浮动。

当这个属性应用在非浮动的块级盒上的时候，值有如下的意思：

* __left__
	
	这个盒的顶外边要比出现在源文档中之前元素生成的任何左浮动盒的下外边低。

* __right__
	
	这个盒的顶外边要比出现在源文档中之前元素生成的任何右浮动盒的下外边低。

* __right__
	
	这个盒的顶外边要比出现在源文档中之前元素生成的任何左浮动盒、右浮动盒的下外边低。

* __none__
	
	相对于浮动没有任何位置上的限制。

当clear的属性值不是none的时候可以产生间隙（clearance）。间隙可以阻止margin折叠，同时还作为元素的margin-top之上的空白（spacing）。他用来把元素从垂直方向上推过浮动。

一个设置了clear的元素间隙的值可先通过确定该元素的上外边假定的位置来计算。这个位置就是假设这个元素设置了clear属性是none的时候的真实的顶外边。

如果这个元素假定的顶外边的位置没有通过相关的浮动的话，那么间隙就会被引入，且根据之前章节说的规则影响margin折叠。

然后间隙的量（值amount）就是下边的较大的：

1. 需要的量就是使得这个块的border边界在 需要清除的浮动中的最靠下的那个下外边。

1. 需要的量就是使得这个块的顶外边在假定的位置。

或者，精确设置间隙的量就是使得这个块的border边界在需要清除的浮动中的最靠下的那个下外边界。

_注意：间隙可以是0也可以是负值。_

_例子1，假定我们只有三个盒，他们是这样排的：块B1的bottom margin叫M1（B1没有孩子也没有padding和border），浮动盒F的高是H，且块B2有一个top margin叫M2（没有padding和border，也没孩子）。B2有clear属性设置为both。我们同样可以假定B2不是空的。_

_不考虑B2的clear属性，我们会有下边的方案。B1和B2的margin折叠了。我们可以说B1的下border边界是在y=0的位置，然后F的上border的位置是在y=M1的位置，B2的上border边界是在y=max(M1,M2)的位置，且F的下border是在y=M1+H的位置。_

![示例](http://www.w3.org/TR/CSS21/images/clearance.png)

_我们也假定B2不在F下边等等，我们是描述需要空隙的情况。这意味着：_

max(M1, M2) < M1 + H

_我们需要计算间隙C两次，C1和C2，然后取较大值：C=max(C1,C2)。第一种方式是将B2的顶和F的低对齐，在y=M1+H的位置。这意味着由于margin不再折叠了，因为他们之间有空隙了：_

bottom of F	= top border edge of B2

M1 + H	= M1 + C1 + M2

C1	= M1 + H - M1 - M2

C1  = H - M2

_第二种方式就是保留B2顶部的位置y=max(M1,M2)。这意味着：_

max(M1,M2)	= M1 + C2 + M2

C2	= max(M1,M2) - M1 - M2

_我们假设了max(M1, M2) < M1 + H，所以_

C2 = max(M1,M2) - M1 - M2	< M1 + H - M1 - M2 = H - M2
C2	< H - M2
C2  < C1

_所以说：_

C = max(C1, C2) = C1

_例子2，这个例子是考虑间隙是负值的情景下的，间隙的值是-1em：_

```html
<p style="margin-bottom: 4em">第一段
<p style="float: left; height: 2em; margin: 0">浮动段
<p style="clear: left; margin-top: 3em">最后一段
```

_解释：当没有clear的时候，第一段和最后一段的margin会折叠的，最后一段的上border边会和浮动段的顶部齐平。但是clear需要顶border边在float之下，低2em。这意味着空隙必须被引入。相应的margin不再折叠了，所以说clearance+margin-top=2em，所以clearance=2em-3em=-1em._

当这个属性设置在了浮动元素上时，他会导致浮动盒定位规则的修正。额外的第10条需要添加：

* 浮动的顶外边必须低于前面的所有左浮动盒（clear是left）或者右浮动盒（clear是right）或者左右浮动盒（clear是both）的下外边界。

_注意：在CSS1中，这个属性可以应用在所有元素上。在CSS2和CSS2.1中，clear属性仅仅只能用在块级元素上。_

### 绝对定位

在绝对定位模式中，一个盒子根据其包含块明确的偏移。他完全从普通流中脱离。一个绝对定位盒会为他的普通流中的孩子们和绝对定位子孙们（不是fixed）创建新的包含块。然而，绝对定位元素的内容不会绕着其他盒排列。他们可能会遮盖其他盒（或者遮挡自身）的内容，这取决于互相重合的盒的‘层叠级别’。

在本规范中规定的绝对定位元素（或者他的盒）意味着这个元素的position属性是absolute或者fixed。

#### 固定定位 fixed positioning

固定定位是绝对定位的一个子集。对于一个固定定位盒来说唯一的不同就是，他的包含块是由viewport创建的。针对于‘可持续媒体’来说，固定定位盒不会随着滚动而滚动。这种情况有点类似于‘固定背景图fixed background images’。针对于‘分页媒体’来说，固定定位盒在每一页都会重复。这对于需要在每一页底部放置一个签名时是很有用的。固定定位盒如果比页的区域要大的话，将会被裁剪。部分的固定定位盒在初始化包含块的时候不可见的话就不会被打印。

_作者可以使用固定定位来创建框架（frame）状演示文稿。考虑下边的片段布局：_

![frame](http://www.w3.org/TR/CSS21/images/frame.png)

_可以通过如下的HTML文档和样式规则实现：_

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN">
<HTML>
  <HEAD>
    <TITLE>A frame document with CSS 2.1</TITLE>
    <STYLE type="text/css" media="screen">
      BODY { height: 8.5in } /* Required for percentage heights below */
      #header {
        position: fixed;
        width: 100%;
        height: 15%;
        top: 0;
        right: 0;
        bottom: auto;
        left: 0;
      }
      #sidebar {
        position: fixed;
        width: 10em;
        height: auto;
        top: 15%;
        right: auto;
        bottom: 100px;
        left: 0;
      }
      #main {
        position: fixed;
        width: auto;
        height: auto;
        top: 15%;
        right: 0;
        bottom: 100px;
        left: 10em;
      }
      #footer {
        position: fixed;
        width: 100%;
        height: 100px;
        top: auto;
        right: 0;
        bottom: 0;
        left: 0;
      }
    </STYLE>
  </HEAD>
  <BODY>
    <DIV id="header"> ...  </DIV>
    <DIV id="sidebar"> ...  </DIV>
    <DIV id="main"> ...  </DIV>
    <DIV id="footer"> ...  </DIV>
  </BODY>
</HTML>
```

### display、position以及float之间的关系

影响盒生成和布局的三个属性——display、position和float——之间会像下边列的互相影响：

1. 如果display的的值是none的话，那么position和float不起作用。在这种情况下，元素不会生成盒。

1. 其他情况，如果position的值是absolute或者fixed，那么这个盒就是绝对定位的，float的计算值是none，切display的值会根据下边的表来设置。这个盒的位置将会是由top、right、bottom和left属性相对于其包含块确定的。

1. 其他情况，如果float的值不是none，这个盒就会浮动并且display会依照下表设置。

1. 其他情况，如果这个元素是根元素，display的值将会按照下表来设置，除了在CSS2.1中未确定的，一个规定的值list-item的计算值是否应该是block或者list-item。

1. 其他应用指定的display属性值。

_PS: 这里加上一个来自[w3help](http://www.w3help.org/zh-cn/kb/009/)的流程图，更清晰明了：_

![流程图](http://www.w3help.org/zh-cn/kb/009/009/display_float_position.png)

| 设定值 | 计算值 |
| ------ | ------ |
| inline-table | table |
| inline,run-in,table-row-group,table-column,table-column-group,table-header-group,table-footer-group,table-row,table-cell,table-caption,inline-block | block |
| 其他 | 同设定值 |

### 普通流、浮动和绝对定位的对比（比较）

为了阐述普通流、相对定位、浮动以及绝对定位之间的不同，我们提供了一系列的例子，基本的HTML：

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN">
<HTML>
  <HEAD>
    <TITLE>Comparison of positioning schemes</TITLE>
  </HEAD>
  <BODY>
    <P>Beginning of body contents.
      <SPAN id="outer"> Start of outer contents.
      <SPAN id="inner"> Inner contents.</SPAN>
      End of outer contents.</SPAN>
      End of body contents.
    </P>
  </BODY>
</HTML>
```

在这个文档中，我们假定有一下的规则：

```css
body { display: block; font-size:12px; line-height: 200%; 
       width: 400px; height: 400px }
p    { display: block }
span { display: inline }
```

在每一个例子中，outer和inner元素生成的最终的盒的位置都是不一样的。在每一个图的左侧用双倍行距线（为了清楚起见）来表示普通流的位置。

#### 普通流

考虑下边的关于inner和outer的CSS声明，不会改变盒的普通流：

```css
#outer { color: red }
#inner { color: blue }
```

P元素包含着所有行内内容：匿名的行内文本和两个SPAN元素。因此，所有的内容将会在同一个IFC中布局，在同一个由P元素生成的包含块中，如下样子：

![示意图](http://www.w3.org/TR/CSS21/images/flow-generic.png)

#### 相对定位

为了看看相对定位的效果，我们设定：

```css
#outer { position: relative; top: -12px; color: red }
#inner { position: relative; top: 12px; color: blue }
```

outer元素的文本通常会向上排列。outer中文本在第1行的末端排列到他普通流中的位置和尺寸。然后行内盒包含着文本（跨越3行）向上移动了12px（向上）。

inner的内容，作为outer的子元素，会立即在of outer contents字之后正常布局（在行1.5位置）。然而，inner的内容他们自身是相对于outer内容便宜的（乡下），重新回到了他们的原始位置（第2行）。

注意紧接着outer的内容不受outer相对定位的影响。

![示意图](http://www.w3.org/TR/CSS21/images/flow-relative.png)

同时注意如果outer有-24px的偏移的话，outer的文本和body的文本就会叠加了。

#### 浮动盒

现在考虑让inner元素的文本浮动到右边的效果，需要如下规则：

```css
#outer { color: red }
#inner { float: right; width: 130px; color: blue }
```

inner盒的文本通常会向上排列，inner盒会脱离普通流，且会浮动到右margin位置（他的宽width被明确设置了）。行盒中浮动左侧缩短了，且文档剩余的文本排列在他们旁边。

![示意图](http://www.w3.org/TR/CSS21/images/flow-float.png)

为了看看clear的效果，我们给这个例子增加一个相邻的元素：

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN">
<HTML>
  <HEAD>
    <TITLE>Comparison of positioning schemes II</TITLE>
  </HEAD>
  <BODY>
    <P>Beginning of body contents.
      <SPAN id=outer> Start of outer contents.
      <SPAN id=inner> Inner contents.</SPAN>
      <SPAN id=sibling> Sibling contents.</SPAN>
      End of outer contents.</SPAN>
      End of body contents.
    </P>
  </BODY>
</HTML>
```

下面的规则：

```css
#inner { float: right; width: 130px; color: blue }
#sibling { color: red }
```

inner像之前一样浮动到右边，剩下的文字利用剩余空间排列：

![示意图](http://www.w3.org/TR/CSS21/images/flow-clear.png)

然而，如果在sibling元素上设置clear属性的值是right，那么这个sibling元素就会在浮动的下边开始排列：

```css
#inner { float: right; width: 130px; color: blue }
#sibling { clear: right; color: red }
```

![示意图](http://www.w3.org/TR/CSS21/images/flow-clear2.png)

#### 绝对定位

最后，我们看下绝对定位的效果。给outer和inner赋予下面的CSS声明：

```css
#outer { 
    position: absolute; 
    top: 200px; left: 200px; 
    width: 200px; 
    color: red;
}
#inner { color: blue }
```

这样会使得outer盒相对于其包含块顶部定位。一个定位盒的包含块是最近的定位了的祖先（如果没有就是初始包含块，我们例子中就是）。outer盒到包含块的顶部距离是200px，到左边界距离是200px。outer的子盒会根据他的父盒正常的排列。

![示意图](http://www.w3.org/TR/CSS21/images/flow-absolute.png)

接下来的例子展示了绝对定位盒是一个相对定位盒的孩子的场景。尽管outer盒没有实际上的偏移，设置他的position属性值为relative意味着他的盒可以作为包含块用来定位他的子孙们。由于outer盒是一个行内盒，且被分割成了好几行，第一行行内盒的上和左边界作为top和left的偏移的根据。

```css
#outer { 
  position: relative; 
  color: red 
}
#inner { 
  position: absolute; 
  top: 200px; left: -100px; 
  height: 130px; width: 130px; 
  color: blue;
}
```

结果是这样的：

![示意图](http://www.w3.org/TR/CSS21/images/flow-abs-rel.png)

如果我们不定位outer盒子：

```css
 #outer { color: red }
 #inner {
  position: absolute; 
  top: 200px; left: -100px; 
  height: 130px; width: 130px; 
  color: blue;
 }
```

inner的包含块就成了初始包含块。然后就变成了下边的样子：

![示意图](http://www.w3.org/TR/CSS21/images/flow-static.png)

_相对定位和绝对定位可以用来实现可变bars，像下边的例子中展示的那样：_

```html
<P style="position: relative; margin-right: 10px; left: 10px;">
I used two red hyphens to serve as a change bar. They
will "float" to the left of the line containing THIS
<SPAN style="position: absolute; top: auto; left: -1em; color: red;">--</SPAN>
word.</P>
```

_结果：_

![示意图](http://www.w3.org/TR/CSS21/images/changebar.png)

_最开始，段落正常的排列。然后他相对于其包含块的左边界便宜10px。两个连字符作为从普通流中拿出作为可变bars，并定位到当前行（因为top:auto），相对于它的包含块的左边界偏移-1em（通过P最终位置确定的）。结果上来看就像是可变bars“浮动”到了当前行的左边。_

### 分层显示

#### 指定层叠级别：z-index属性

* _'z-index'_

	_值：_ auto | <整数integer> | inherit

	_初始化：_ auto

	_应用在：_ 定位元素

	_可继承：_ 不可以

	_百分比：_ 不可用

	_媒介：_ 可见媒介

	_计算值：_ 和指定值一样

对于一个定位盒，z-index属性指定了：

1. 这个盒在当前的层叠上下文中的层叠级别。

1. 盒是否生成局部层叠上下文。

值有如下意思：

* __<整数integer>__
	
	该整数是生成框在当前层叠上下文中的层叠级别。该框也会生成一个局部层叠上下文。

* __<auto>__
	
	生成盒在当前的层叠上下文中的层叠级别是0。该盒不会创建新的局部层叠上下文除非他是根元素。

_在这段落中，"在xx之前"意味着看起来离用户更近。_

在CSS2.1中，每一个盒都有一个三个坐标的位置。除了他们在水平方向和垂直放上的位置，盒会沿着Z轴一个在另一个之上这样格式化。Z轴的位置在盒视觉重叠的时候是很重要的。这个段落主要就是讨论盒在Z轴上怎样定位的。

画到画布上的渲染树的顺序是用层叠上下文描述的。层叠上下文能够包含深层的层叠上下文。一个层叠上下文是他父层叠上下文中一个不可分割的最小单位（从他的父层叠上下文的角度来看）；在其他层叠上下文中的盒，不可能出现在他的任何盒之间。

每一个盒属于一个层叠上下文。在给定的层叠上下文中的每一个定位盒都有一个整数的层叠级别，他就是在同一层叠上下文中的在Z轴上相对于其他层叠级别的位置。具有更大层叠级别的盒往往被格式化到低层叠级别的盒之前。盒可以有负的层叠级别。在同一个层叠上下文中相同层叠级别的盒会按照在文档树中的顺序由后到前层叠。

根元素形成根层叠上下文。其他的层叠上下文都是z-index的计算值不是auto的定位元素生成的。包含块和层叠上下文没有必要联系。在未来的CSS中，其他属性也可能会有层叠上下文，例如opacity。

在每一个层叠上下文中，都有如下的由后到前的顺序的显示顺序层：

1. 形成层叠上下文的元素的背景和边框。

1. 层叠级别为负值的子层叠上下文。

1. 普通流中非行内非定位的子元素。

1. 非定位的浮动。

1. 普通流中行内非定位子元素（包括inline tables和inline blocks）组成的层。

1. 层叠级别是0的子层叠上下文，以及层叠级别是0的定位子元素。

1. 层叠级别为正值的子层叠上下文。

_PS: 来自w3help的示意图：_

![示意图](http://www.w3help.org/zh-cn/kb/013/013/stacklevel.png)

在每一个层叠上下文中，层叠级别是0的定位元素，非定位的浮动，inline blocks和inline tables，是被显示成如同他们自身创建了新的层叠上下文一样，除了他们的定位子元素和任何可能的当前层叠上下文的子层叠上下文。

显示的顺序是有每一个层叠上下文决定的。层叠上下文有关的比较详细的部分请参见[层叠上下文的详细描述](http://www.w3.org/TR/CSS21/zindex.html)。

_在接下来的示例中，盒的层叠级别是：text2=0, image=1, text3=2, text1=3.text2的层叠级别是继承自根元素形成的盒。其他的都是通过z-index属性指定的。_

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN">
<HTML>
  <HEAD>
    <TITLE>Z-order positioning</TITLE>
    <STYLE type="text/css">
      .pile { 
        position: absolute; 
        left: 2in; 
        top: 2in; 
        width: 3in; 
        height: 3in; 
      }
    </STYLE>
  </HEAD>
  <BODY>
    <P>
      <IMG id="image" class="pile" 
           src="butterfly.png" alt="A butterfly image"
           style="z-index: 1">

    <DIV id="text1" class="pile" 
         style="z-index: 3">
      This text will overlay the butterfly image.
    </DIV>

    <DIV id="text2">
      This text will be beneath everything.
    </DIV>

    <DIV id="text3" class="pile" 
         style="z-index: 2">
      This text will underlay text1, but overlay the butterfly image
    </DIV>
  </BODY>
</HTML>
```

这个例子演示了透明（transparency）的概念。background默认的行为是允许其之后的盒可见。在这个例子中，每一个盒子透明覆盖在其之下的盒。这种行为可以使用已存在的‘background属性’来重写。

### 最后关于文本方向的：direction和unicode-bidi属性，请参见w3c的[文本方向](http://www.w3.org/TR/CSS21/visuren.html#direction)