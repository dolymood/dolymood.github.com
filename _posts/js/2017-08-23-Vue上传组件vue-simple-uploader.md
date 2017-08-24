---
layout: post
category : js
tagline: ""
tags : [vue-simple-uploader, simple-uploader.js, vue]
---
{% include JB/setup %}

在日常开发中经常会遇到文件上传的需求，[vue-simple-uploader](https://github.com/simple-uploader/vue-uploader) 就是一个基于 [simple-uploader.js](https://github.com/simple-uploader/Uploader) 和 Vue 结合做的一个上传组件，自带 UI，可覆盖；先来张动图看看效果：

![example](http://demo.aijc.net/js/vue-uploader/example/simple-uploader.gif)

其主要特点就是：

* 支持文件、多文件、文件夹上传

* 支持拖拽文件、文件夹上传

* 统一对待文件和文件夹，方便操作管理

* 可暂停、继续上传

* 错误处理

* 支持“快传”，通过文件判断服务端是否已存在从而实现“快传”

* 上传队列管理，支持最大并发上传

* 分块上传

* 支持进度、预估剩余时间、出错自动重试、重传等操作

<!--more-->

### 安装

通过`npm`安装：`npm install vue-simple-uploader --save`即可。

### 使用

#### 初始化

``` js
import Vue from 'vue'
import uploader from 'vue-simple-uploader'
import App from './App.vue'

Vue.use(uploader)

/* eslint-disable no-new */
new Vue({
  render(createElement) {
    return createElement(App)
  }
}).$mount('#app')
```

### App.vue

``` vue
<template>
  <uploader :options="options" class="uploader-example">
    <uploader-unsupport></uploader-unsupport>
    <uploader-drop>
      <p>Drop files here to upload or</p>
      <uploader-btn>select files</uploader-btn>
      <uploader-btn :attrs="attrs">select images</uploader-btn>
      <uploader-btn :directory="true">select folder</uploader-btn>
    </uploader-drop>
    <uploader-list></uploader-list>
  </uploader>
</template>

<script>
  export default {
    data () {
      return {
        options: {
          // 可通过 https://github.com/simple-uploader/Uploader/tree/develop/samples/Node.js 示例启动服务
          target: '//localhost:3000/upload',
          testChunks: false
        },
        attrs: {
          accept: 'image/*'
        }
      }
    }
  }
</script>

<style>
  .uploader-example {
    width: 880px;
    padding: 15px;
    margin: 40px auto 0;
    font-size: 12px;
    box-shadow: 0 0 10px rgba(0, 0, 0, .4);
  }
  .uploader-example .uploader-btn {
    margin-right: 4px;
  }
  .uploader-example .uploader-list {
    max-height: 440px;
    overflow: auto;
    overflow-x: hidden;
    overflow-y: auto;
  }
</style>
```

### 组件

#### Uploader

上传根组件，可理解为一个上传器。

##### Props

* `options {Object}`

  参考 [simple-uploader.js 配置](https://github.com/simple-uploader/Uploader#configuration)。

* `autoStart {Boolean}`

  默认 `true`, 是否选择文件后自动开始上传。

##### 事件

* `upload-start`

  开始上传。

* `file-added(file)`

  添加了一个文件，一般用作文件校验，如果给 `file` 增加 `ignored` 属性为 `true` 的话就会被过滤掉。

* `file-removed(file)`

  移除一个文件（文件夹）。

* `files-submitted(files, fileList)`

  所选择的文件们添加到上传队列后触发。

##### 作用域插槽

* `files {Array}`

  纯文件列表，没有文件夹概念。

* `fileList {Array}`

  统一对待文件、文件夹列表。

* `started`

  是否开始上传了。

#### UploaderBtn

点选上传文件按钮。

##### Props

* `directory {Boolean}`

  默认 `false`, 是否是文件夹上传。

* `single {Boolean}`

  默认 `false`, 如果设为 `true`，则代表一次只能选择一个文件。

* `attrs {Object}`

  默认 `{}`, 添加到 input 元素上的额外属性。

#### UploaderDrop

拖拽上传区域。

#### UploaderList

文件、文件夹列表，同等对待。

##### 作用域插槽

* `fileList {Array}`

  文件、文件夹组成数组。

#### UploaderFiles

文件列表，没有文件夹概念，纯文件列表。

##### 作用域插槽

* `files {Array}`

  文件列表。

#### UploaderUnsupport

不支持 HTML5 File API 的时候会显示。

#### UploaderFile

文件、文件夹单个组件。

##### Props

* `file {Uploader.File}`

  封装的文件实例。

* `list {Boolean}`

  如果是在 `UploaderList` 组件中使用的话，请设置为 `true`。

##### 作用域插槽

* `file {Uploader.File}`

  文件实例。

* `list {Boolean}`

  是否在 `UploaderList` 组件中使用。

* `status {String}`

  当前状态，可能是：`success`, `error`, `uploading`, `paused`, `waiting`

* `name {String}`

  文件名字。

* `paused {Boolean}`

  是否暂停了。

* `error {Boolean}`

  是否出错了。

* `averageSpeed {Number}`

  平均上传速度，单位字节每秒。

* `formatedAverageSpeed {String}`

  格式化后的平均上传速度，类似：`3 KB / S`。

* `currentSpeed {Number}`

  当前上传速度，单位字节每秒。

* `isComplete {Boolean}`

  是否已经上传完成。

* `isUploading {Boolean}`

  是否在上传中。

* `size {Number}`

  文件或者文件夹大小。

* `formatedSize {Number}`

  格式化后文件或者文件夹大小，类似：`10 KB`.

* `uploadedSize {Number}`

  已经上传大小，单位字节。

* `progress {Number}`

  介于 0 到 1 之间的小数，上传进度。

* `progressStyle {String}`

  进度样式，transform 属性，类似：`{transform: '-50%'}`.

* `progressingClass {String}`

  正在上传中的时候值为：`uploader-file-progressing`。

* `timeRemaining {Number}`

  预估剩余时间，单位秒。

* `formatedTimeRemaining {String}`

  格式化后剩余时间，类似：`3 miniutes`.

* `type {String}`

  文件类型。

* `extension {String}`

  文件名后缀，小写。

* `fileCategory {String}`

  文件分类，其中之一：`folder`, `document`, `video`, `audio`, `image`, `unknown`。

### 项目

地址：[https://github.com/simple-uploader/vue-uploader](https://github.com/simple-uploader/vue-uploader)。

欢迎使用、拍砖，顺便求 Star O(∩_∩)O。
