---
title: 初试VUX
date: 2017-07-01 12:22:47
categories: "Vue"
comments: true
thumbnail: /gallery/初试vux/logo.jpeg
tags:
- Vux
- Vue
---

<!-- no node -->

<!-- more -->

>借机，重写单位项目，那么挑了一些框架，最终决定用VUX。
>讲真，文档看起来有点吃力，要不断尝试。
>有趣，可以自己扯UI。
>费时，真的费时。
>那么，来分享下吧。

## vux-loader

在`build/webpack.base.conf.js`进行单独配置：

```javascript
const vuxLoader = require('vux-loader')
const webpackConfig = originalConfig // 原来的 module.exports 代码赋值给变量 webpackConfig

module.exports = vuxLoader.merge(webpackConfig, {
  plugins: ['vux-ui','duplicate-style']
})
```

## vue-router

### 编程式的导航

#### `router.push(location)`

| 声明式 | 编程式 |
|-------------|--------------|
| `<router-link :to="...">` | `router.push(...)` |

### 重定向

重定向也是通过 `routes` 配置来完成，下面例子是从 `/a` 重定向到 `/b`：

```javascript
const router = new VueRouter({
  routes: [
    { path: '/a', redirect: '/b' }
  ]
})
```

## XIcon

**深坑！**

根本不需要写这句：

`import { XIcon } from 'vux'`

直接用就好了

```html
<x-icon type="ios-arrow-up" class="icon-red"></x-icon>
<x-icon type="ios-arrow-up" size="30"></x-icon>
```

## VUEG

来添加转场动画，配置`src/main.js`：

```javascript
import Vue from 'vue'
import router from '/gallery/router'
import vueg from 'vueg'
import 'vueg/css/transition-min.css'

const options = {
  duration: '0.3',
  firstEntryDisable: false,
  firstEntryDuration: '.6',
  forwardAnim: 'fadeInRight',
  backAnim: 'fadeInLeft',
  sameDepthDisable: false,
  tabs: [{name: 'Home'}, {name: 'OwnerInfo'}],
  tabsDisable: false,
  disable: true
}

Vue.use(vueg, router, options)
```

[配置项详解](https://github.com/jaweii/vueg#配置项--config)

也可以用`animate.css`自己写一个……

## vue-dropzone

一个上传文件的插件，由于文档内容比较多，没有深入去读，已经写在源码里，但是被我注释掉了。后续再去了解吧。

## DEMO

![](/gallery/初试vux/demo1.gif)
[源码](https://github.com/zongzi531/vux-demo-test)