---
layout: post
category : js
tagline: ""
tags : [js, 浏览器兼容]
---
{% include JB/setup %}


微软为了向标准靠拢，同时为了向后兼容，在IE9中，如果监听元素的事件的话，两种方法`addEventListener`和`attachEvent`都是可以用的；理论上来说应该是没有什么区别的，但是在最近项目上的一个小功能上需要用到iframe个form表单配合上传文件，这时候就遇到了奇葩的IE9产生的一个bug：

	IE9中iframe的load事件不会被触发，虽然从请求上来看，已经成功，也有响应

下面详细说下这个奇幻漂流之旅。

<!--more-->

#### 对比

由于这个功能需要android和ios客户端上建立简易server，而bug是在android手机上测试发现的，在ios上并没有这个bug，所以就是对比，查看请求响应有什么不同。

结果发现：

1. 在ios上响应头`Connection`是`close`，而在android上没有。

1. 在ios上响应头`Content-Type`是`text/plain; charset=utf-8`，而在android上响应的类型`Content-Type`是`text/json; charset=utf-8`。

对于第1点，只是对于此次连接是否断开（close）或者保持（keepalive），没有啥问题。详见[HTTP协议头部与Keep-Alive模式详解](http://blog.csdn.net/zfrong/article/details/6070608)博文。

所以就猜测是响应类型导致的问题。但是这个text/json是什么东东？

response的content-type的值，一般是如下几种情况：

1. 服务端需要返回一段普通文本给客户端

1. 务端需要返回一段HTML代码给客户端

1. 服务端需要返回一段XML代码给客户端

1. 服务端需要返回一段javascript代码给客户端

1. 服务端需要返回一段json串给客户端

对于前3中对于的content-type分别为`text/plain`、`text/html`、和`text/xml`；而对于一段javascript代码来说，按照最新标准写法是大家都知道的`application/javascript`，而常用的`text/javascript`已经被废弃了（[rfc4329](http://www.ietf.org/rfc/rfc4329.txt)）；对于响应是json的，常见写法有`text/json`、`text/javascript`，但是最新的标准是`application/json`（[rfc4627](http://www.ietf.org/rfc/rfc4627.txt)），这个`text/json`本应不存在的，百度谷歌了一阵，没什么有用的价值，大致意思就是

	text/json在android手机应用客户端和服务器端传送数据时用到

或者
	
	某些情况下，每次请求都要指定Content-Type:text/json。否则服务器会收不到请求参数或出错。

所以解决方案就是请android组同事帮忙改下响应头。到这里就结束了吗？android组同事在解决是需要时间的；我总觉得还是差点什么，为什么响应头影响到了iframe的load事件的触发？？

#### 尝试`addEventListener`和`attachEvent`

大胆猜测下，这两张监听事件的方法有什么不同吗？或者说在某些场景下会有什么不同吗？

##### iframe监听大尝试

这次只是针对于iframe的load事件的监听，首先直接使用`iframe.onload = function() {}`，实验发现能正常的触发load事件；接着使用`iframe.attachEvent('onload', function() {})`，发现也是可以触发的，这下就奇怪了，难道就`addEventListener`不可以，所以最后实验`iframe.addEventListener('load', function() {})`，果不其然，load事件不会被触发。

这真是惊天真相，这还会有区别？那对于其他元素会有bug吗？

##### button监听大尝试

这次创建`button`元素，然后监听`click`事件，事实证明三种方式都是可以的，没有问题。那如果是关于`load`事件的呢？

##### img监听大尝试

创建`img`元素，监听`load`事件。也都无问题。

目前来看只有iframe有这个不可思议的问题。

#### 结局

结局很简单，那就是android组同事把响应头`content-type`改为`text/plain`了，而js部分没有变，问题得到解决了。

虽然说是一个本不应存在的`content-type:text/json`引起的，但是对于IE9，这种不会触发`load`事件bug实在是万万不能想到的。

仍然把js代码判断应该用`addEventListener`或者`attachEvent`的部分稍稍修改了下，由

```js
function addEvent(element, type, handler) {
  if (element.addEventListener) {
    element.addEventListener(type, handler, false);
  } else if (element.attachEvent) {
    element.attachEvent("on" + type, handler);
  } else {
    element["on" + type] = handler;
  }
}
```

变为了

```js
function addEvent(element, type, handler) {
  if (element.attachEvent) {
  	element.attachEvent("on" + type, handler);
  } else if (element.addEventListener) {
    element.addEventListener(type, handler, false);
  } else {
    element["on" + type] = handler;
  }
}
```

好，到这里奇幻漂流之旅结束了，但是也有感慨：。。。省略。。。

参考：

1. [HTTP协议头部与Keep-Alive模式详解](http://blog.csdn.net/zfrong/article/details/6070608)

1. [纠结于ajax开发中 response的contentType 问题](http://fins.iteye.com/blog/289852)

1. [What is the correct JSON content type?](http://stackoverflow.com/questions/477816/what-is-the-correct-json-content-type)

1. [text/html 和 text/xml 和 text/json三者分别在什么地方用到?](http://zhidao.baidu.com/link?url=lrhQmvTUUvJbENHCOQfBf9JAE2l6yoAxQfIO04Tcn7sC3IAo-Tgj78NecZ_Plevp-mVHfjbwHqT2p5Xux9b09K)

1. [http://www.360doc.com/content/14/0902/09/281812_406432915.shtml](http://www.360doc.com/content/14/0902/09/281812_406432915.shtml)