---
title: 在 JavaScript 设计模式上盘旋
date: 2018-09-06 19:53:24
categories: "JavaScript"
comments: true
thumbnail: /gallery/在JavaScript设计模式上盘旋/pic1.jpg
tags:
- JavaScript
---

<!-- no node -->

<!-- more -->

>9月天气逐渐转凉
>
>8月抽时间阅读了《JavaScript设计模式》
>
>对设计模式有了初步的认识，对在 JavaScript 中运用到的设计模式也似乎加深了印象

## 前后端不分离的情况下基于 Webpack 实现调试开发

在写上一篇博客的时候，我还没有解决这个问题。当时，我一直纠结于`webpack-dev-server`，但是后来我发现，我可以放弃`webpack-dev-server`，而是直接对 `webpack.prod.conf.js` 进行轻度改写，增加`watch`配置项来实现调试开发，这时候问题也就迎刃而解了。

那么我们来简单了解一下 Webpack 的这一功能 watch 和 watchOptions

>webpack 可以监听文件变化，当它们修改后会重新编译。这个页面介绍了如何启用这个功能，以及当 watch 无法正常运行的时候你可以做的一些调整。

为什么要在前后端不分离的项目中使用 Webpack 呢，因为考虑到编程体验，在不影响编程体验的情况下，使用 Webpack 来解决一些工程化的问题，比如ES6语法转换，CSS浏览器兼容性问题等等。

最后我将这个解决方案命名为：fedevb

关于为什么命名为fedevb，可以详见 GitHub 仓库 [fedevb](https://github.com/zongzi531/fedevb)

## 放弃 amfe-flexible 使用 vw 的兼容方案

前往 [amfe-flexible](https://github.com/amfe/lib-flexible) 仓库 可以看到这样一段话：

>由于`viewport`单位得到众多浏览器的兼容，`lib-flexible`这个过渡方案已经可以放弃使用，不管是现在的版本还是以前的版本，都存有一定的问题。建议大家开始使用`viewport`来替代此方案。`vw`的兼容方案可以参阅[《如何在Vue项目中使用vw实现移动端适配》](https://www.w3cplus.com/mobile/vw-layout-in-vue.html)一文。

文中呢也是很有趣的提到了：

>说句题外话，由于Flexible的出现，也造成很多同学对`rem`的误解。正如当年大家对`div`的误解一样。也因此，大家都觉得`rem`是万能的，他能直接解决移动端的适配问题。事实并不是如此，至于为什么，我想大家应该去阅读[flexible.js](https://github.com/amfe/lib-flexible)源码，我相信你会明白其中的原委。

对着那50行不到的源码，我竟是没看出什么原委，哎，能力还不够啊！

最后，`.postcssrc.js` 配置如下：

```javascript
module.exports = {
  "plugins": {
    "postcss-import": {},
    "postcss-url": {},
    "postcss-aspect-ratio-mini": {},
    "postcss-write-svg": {
      utf8: false
    },
    "postcss-cssnext": {},
    "postcss-px-to-viewport": {
      viewportWidth: 375,
      unitPrecision: 3,
      viewportUnit: 'vw',
      minPixelValue: 1,
      mediaQuery: false
    },
    "postcss-viewport-units": {
      filterRule: rule => rule.selector.indexOf('::after') === -1 && rule.selector.indexOf('::before') === -1 && rule.selector.indexOf(':after') === -1 && rule.selector.indexOf(':before') === -1
    },
    "cssnano": {
      preset: 'advanced',
      autoprefixer: false,
      "postcss-zindex": false
    }
  }
}
```

来特别提一下`postcss-viewport-units`，找到 Viewport Units Buggyfill™ 你会发现`content`也会引起一定的副作用。比如`img`和伪元素`::before`(`:before`)或`::after`（`:after`）。在`img`中`content`会引起部分浏览器下，图片不会显示。这个时候需要全局添加：

```css
img {
  content: normal !important;
}
```

但是你会发现这还不够，类似 iconfont 也会同样受到这个副作用的影响，所以利用提供的接口设置过滤规则即可

```javascript
const filterRule = rule => rule.selector.indexOf('::after') === -1 &&
rule.selector.indexOf('::before') === -1 &&
rule.selector.indexOf(':after') === -1 &&
rule.selector.indexOf(':before') === -1
```

## 借助 WebUploader 实现秒传与断点续传

直接上源码了，重点部分我已经写下了相应的注释，关于用法的话可以去看官网以及 FAQ

```javascript
;(function ($) {
  WebUploader.Uploader.register({
    'before-send-file': 'beforeSendFile',
    'before-send': 'beforeSend'
  }, {
    beforeSendFile: function (file) {
      var self = this,
          owner = this.owner,
          deferred = WebUploader.Deferred()

      owner.md5File(file).then(function(md5) {
        $.ajax({
          type: 'POST',
          url: '/gallery/uploadCheck',
          data: {
            md5: md5
          },
          cache: false,
          dataType: 'json',
          success: function (response) {
            // If file upload completed you need use `owner.skipFile( file )` to skip file uploading
            // Otherwise, you need record uploading progress and something you need
            deferred.resolve()
          }
        })
      })
      return deferred.promise()
    },
    beforeSend: function (block) {
      // You need get uploading progress and do something
      var chunkCurr = 'uploading progress',
          deferred = WebUploader.Deferred()
      if (block.chunk < chunkCurr) {
        deferred.reject()
      }
      deferred.resolve()
      return deferred.promise()
    }
  })

  var uploader = WebUploader.create({
    auto: true,
    swf: '/gallery/Uploader.swf',
    server: '/gallery/upload',
    pick: '#filePicker',
    chunked: true,
    threads: 1,
    chunkRetry: false,
    formData: {
      // ...
    }
  })

  uploader.on('uploadAccept', function (file, response) {
    // If upload failed you can `return false` stop uploading
  })

  uploader.on('uploadSuccess', function (file, response) {
    // Success do something
  })
})(jQuery)

```

## 《JavaScript设计模式》读后感

此书首先介绍了一些关于设计模式的基础概念，其次开始从 JavaScript 介绍几种经典模式的实现

1. Constructor（构造器）模式
2. Module（模块）模式
3. Revealing Module（揭示模块）模式
4. Singleton（单例）模式
5. Observer（观察者）模式
6. Mediator（中介者）模式
7. Prototype（原型）模式
8. Command（命令）模式
9. Facade（外观）模式
10. Factory（工厂）模式
11. Mixin模式
12. Decorator（装饰者）模式
13. Flyweight（享元）模式

这些设计模式，在现在 JavaScript 中很多地方都有运用到，只是我暂时还没有办法很明确的认识到它们，我认为这还需要时间的沉淀。

关于 JavaScript MV* 模式的发展历程，从MVC演化出MVP和MVVM模式，可以参考阮一峰老师的[MVC，MVP 和 MVVM 的图示](http://www.ruanyifeng.com/blog/2015/02/mvcmvp_mvvm.html)，等等其他的内容也同样可以在网上搜索到，我之所以去网上搜，是因为书中提到的内容我看的云里雾里的。

接下来介绍了模块化 JavaScript 设计模式的发展历程，从AMD，CommonJS到UMD到ES6 Module，AMD比较适用于客户端，CommonJS比较适用于服务端，ES6 Module成为了现在模块化开发的主流

最后，介绍了jQuery中的设计模式以及jQuery插件使用到的设计模式，考虑读jQuery源码学习设计模式，理解AMD模块化，书中提到的插件基本基于jQuery-ui和jQuery-mobile。

种种迹象让我去读jQuery源码，所以我创建了一个 GitHub 仓库 [read-jquery](https://github.com/zongzi531/read-jquery)。

## 将 ToDo List 部署至服务器

比较浪费的是，去年双11在阿里云买的一年的云服务器，放着大半年没动过，最近才想着把这个写好的服务放上去，不过之前呢也没啥好放的，毕竟没写过Node服务。

服务器配置遇到的问题，就拿这个项目为例子，当我进入`/`路由的时候，是没有的问题，并且会自动跳转至`/signin`路由，内容也是同样可以显示的，但是如果我在`/signin`路由刷新页面时，浏览器去请求的是`/signin`路由下的html文件，就会出现服务器返回404的现象。那么解决办法就是使用：

```conf
server {
  ...
  location / {
    try_files $uri /index.html
  }
}
```

那么为什么会出现这样的情况呢，原因是 Browser history 是使用 React Router 的应用推荐的 history。它使用浏览器中的 History API 用于处理 URL，创建一个像 `example.com/some/path` 这样 **真实的** URL 。

起初我这样配置nginx的时候，发现问题解决了，页面刷新不会出现404的现象，但是这时我发现，接口不会请求了。很糟糕，解决方案呢有两种，一种是重新配置nginx，但是如果页面路由多的话，会导致配置文件难维护，我不可能是说项目修改了，我还要重新去配一遍nginx，因为我这里接口的路由是跟页面路由同级的，那么另一种就是修改我的node服务，为接口都添加一层，比如`/getTodoList`改为`/api/getTodoList`，都统一改成`/api/..`这样，然后配置一遍nginx即可。当然在这个项目中我选择了第一种简单的，我暂时懒得改node服务，改改确实是小事，主要改完以后页面项目也要跟着一起改。

再一个问题是，我将node服务部署在服务器上，发现需要借助工具，这里我选择使用PM2，好用的一批，不多说。

## 相关链接

* [Webpack Watch and WatchOptions（中文）](https://www.webpackjs.com/configuration/watch/)
* [Webpack Watch and WatchOptions](https://webpack.js.org/configuration/watch/)
* [vw适配中使用伪类选择器遇到的问题](https://blog.csdn.net/perryliu6/article/details/80965734)
* [webuploader - 如何实现秒传与断点续传](https://github.com/fex-team/webuploader/issues/142)
* [Install MongoDB Community Edition on Ubuntu](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/#install-mongodb-community-edition-on-ubuntu)
* [React Router Histories](https://react-guide.github.io/react-router-cn/docs/guides/basics/Histories.html)
* [PM2实用入门指南](https://www.cnblogs.com/chyingp/p/pm2-documentation.html)
* [为什么用Object.prototype.toString.call(obj)检测对象类型?](https://www.cnblogs.com/youhong/p/6209054.html)

---

>上一篇博客内容提到的jQuery html API 与 原生 innerHTML API，这里可以延展阅读一下这篇内容

* [JavaScript Note.我想插入一个动态的脚本](https://codesky.me/archives/javascript-note-i-want-to-insert-a-script.wind)
