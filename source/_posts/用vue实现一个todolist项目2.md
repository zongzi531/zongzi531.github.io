---
title: 用Vue实现一个To Do List项目（二）
date: 2017-06-04 15:34:05
categories: "Vue"
comments: true
featured_image: demo3.gif
tags:
- Vue
---

<!-- no node -->

<!-- more -->

这里较上一篇[《用Vue实现一个To Do List项目》](http://zongzi531.github.io/2017/06/02/%E7%94%A8vue%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AAtodolist%E9%A1%B9%E7%9B%AE/)内容，做了一下更新：

* 增加颜色选择
* 增加修改功能
* 增加拖拽功能
* 界面优化

>那么下面还是来分享一些技术点。

## 增加颜色选择

使用到的API任然是`v-for`、`v-bind`、`v-on`来控制数据。并且将监听来的数据存入data中。

使用一个`flag`来控制当前选中的颜色。

## 增加修改功能

这里用到了`v-if`而不是`v-show`，因为我觉得不是每个`<input>`标签都需要被一次性编译的，虽然这个项目小，但是思路不能错。

这里在调整CSS样式的时候使用到了`calc()`一个CSS样式值得计算功能。并且也隐藏了文本超出的部分。

## 增加拖拽功能

首先依赖`awe-dnd`。此插件很方便，直接按照官网DOC进行配置即可使用。

>我自己也思考过如何去实现一个拖拽功能，思路一个就是直接操纵数据来改变页面内容，但是没想过drag和drop的使用。这个待我更熟悉Vue的时候，再尝试吧，但是我觉得这个思路是没有错的。

## 界面优化

### [Bootstrap栅格系统](http://v3.bootcss.com/css/#grid)

移除了之前固定的布局模式，调整为Bootstrap 提供了一套响应式、移动设备优先的流式栅格系统。

用法就是很无脑的这么一套

```html
<div class="row">
  <div class="col-md-4">.col-md-4</div>
  <div class="col-md-4">.col-md-4</div>
  <div class="col-md-4">.col-md-4</div>
</div>
```

### [Bootstrap导航标签页](http://v3.bootcss.com/components/#nav-tabs)

同样也是简单的一套，但是值得一提的是`border-bottom`的样式，我这里是使用Vue的数据控制`border-bottom`的样式。

使用要的Vue技术是[computed](https://cn.vuejs.org/v2/api/#computed)。

因为直译就是计算，没错就是计算，这是Vue提供的一个极赞的解决方案，在这个项目里，判断`border-bottom`样式的显示事件存在着两种情况，很好的解决了问题。

举个栗子：

```javascript
//判断当todo顶部菜单被点击或者todo列表内是否存在内容来控制border-bottom的显示
'tabs-bottom': !this.todoflag || !this.todos.length
```

感谢阅读至此。

**[GitHub仓库地址](https://github.com/zongzi531/vue-to-do-list)**