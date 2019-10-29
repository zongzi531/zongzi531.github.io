---
title: Vue实战经验分享
date: 2017-06-17 20:57:11
categories: "Vue"
comments: true
thumbnail: /gallery/vue实战经验分享/logo.png
tags:
- Vue
---

<!-- no node -->

<!-- more -->

>这周接到了一个SPA的活动页，借着这次机会直接在单位用起了Vue框架。
>因为之前自己练习过几个小Demo，所以对一些API还是能够比较熟悉的使用，不用直接操纵DOM的感觉真的是很棒的。
>当然在使用框架的时候，不能忘了“初心”。记住必要**探其究竟**。
>这两周没有经常更新Github，但是我一直活跃，我关注新技术，我仍然在这条路上。
>那么下面来说说我在这个SPA活动页项目中踩到的坑吧。

## Build生成的文件如何与后端配合

目前我单位的工作模式是前后端分离的。

之前我一直认为使用`npm run build`生成的文件，需要使用`node`起一个服务来进行使用。

但是并不是这样的，生成的文件丢到服务器后，可以直接配置`Apache`和`Tomcat`的配置文件，来引导`css`和`js`文件的相对路径。

那么是怎么发现这个问题的呢，这里很有意思，把文件直接丢上服务器，发现`css`和`js`文件的相对路径有问题，导致`index.html`没有办法正确的加载这些文件，所以导致显示内容空白。

那么虽然我不是一个后端，但是我简单的了解了下和这个问题的相关的内容。

下面直接上配置：

### Apache

配置`httpd.conf`文件`DocumentRoot`的引导路径即可。

### Tomcat

Build生成的文件目录在`dist`下，那么这里以`dist`为例。我们把`dist`目录丢到`webapps`下，然后配置`conf/server.xml`：

```xml

<Host name="localhost" appBase="webapps/dist" unpackWARs="true" autoDeploy="true">

	<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs" 
	prefix="localhost_access_log." suffix=".txt" 
	pattern="%h %l %u %t "%r" %s %b"/>  
	
    <Context path="/dist" reloadable="true" docBase="/home/tomcat/webapps/dist" />  

</Host>

```

## Build生成的文件去掉map文件

想要做到这一步其实很简单，直接将`config/index.js`中的`productionSourceMap`设置为`false`即可。

### 那么问题来了，map文件到底是做什么用的呢？

简单说，Source map就是一个信息文件，里面储存着位置信息。也就是说，转换后的代码的每一个位置，所对应的转换前的位置。

有了它，出错的时候，除错工具将直接显示原始代码，而不是转换后的代码。这无疑给开发者带来了很大方便。

### 那么为什么会有map文件的到来呢？

JavaScript脚本正变得越来越复杂。大部分源码（尤其是各种函数库和框架）都要经过转换，才能投入生产环境。常见的源码转换，主要是以下三种情况：

>（1）压缩，减小体积。比如jQuery 1.9的源码，压缩前是252KB，压缩后是32KB。
>（2）多个文件合并，减少HTTP请求数。
>（3）其他语言编译成JavaScript。最常见的例子就是CoffeeScript。

这三种情况，都使得实际运行的代码不同于开发代码，除错（debug）变得困难重重。

通常，JavaScript的解释器会告诉你，第几行第几列代码出错。但是，这对于转换后的代码毫无用处。举例来说，jQuery 1.9压缩后只有3行，每行3万个字符，所有内部变量都改了名字。你看着报错信息，感到毫无头绪，根本不知道它所对应的原始位置。

这就是Source map想要解决的问题。

## Vue插件使用

### [pagekit/vue-resource](https://github.com/pagekit/vue-resource)

一个Ajax插件，使用过程中遇到了这么一个坑，就是发起`POST`请求的时候，浏览器默认发起了`OPTIONS`请求。

那么按照理解来说，一个“非简单请求”在跨域时，浏览器会默认自动帮你发一个`OPTIONS`请求，到服务器端请求服务器确认该请求的合法性，服务器端必须得有相应的路由处理该请求，并认真返回`200`响应，然后浏览器才会再次发出正常的、你需要的请求。

那么后端并不会对`OPTIONS`响应，我们的解决方案就是直接在`main.js`中加入`Vue.http.options.emulateJSON = true`。

vue-resource/docs中有这么一段话：

---

#### Legacy web servers

If your web server can't handle requests encoded as `application/json`, you can enable the `emulateJSON` option. This will send the request as `application/x-www-form-urlencoded` MIME type, as if from an normal HTML form.

`Vue.http.options.emulateJSON = true;`

If your web server can't handle REST/HTTP requests like `PUT`, `PATCH` and `DELETE`, you can enable the `emulateHTTP` option. This will set the `X-HTTP-Method-Override` header with the actual HTTP method and use a normal `POST` request.

`Vue.http.options.emulateHTTP = true;`

---

### [vue-router](https://github.com/vuejs/vue-router)

使用vue-router的想法很简单，因为需求方需要将一个页面放在不同地方，链接是带参数的，那么我想到的是用vue-router利用动态路由匹配来控制跳转链接的地址。

## 踩坑事件

先简单的说下我遇到的问题：

```javascript
export default {
  data () {
    return {
      exampleData: ''
    }
  },
  methods: {
    exampleMethods: function () {

      // example 1

        let self = this
        setTimeout(function() {
          self.exampleData = 'example 1'
        }, 3000)

      // example 2

        setTimeout(function() {
          this.exampleData = 'example 2'
        }, 3000)
    }
  }
}
```

那么在调用`exampleMethods`方法后，`exampleData`的结果是什么呢？

>setTimeout方法是挂在window对象下的。《JavaScript高级程序设计》第二版中，写到：“超时调用的代码都是在全局作用域中执行的，因此函数中this的值在非严格模式下指向window对象，在严格模式下是undefined”。

所以，在setTimeout中所执行函数中的this，永远指向window。

先卖一个关子，或者你可以尝试下，下次将为大家分享学习过程。

## 参考

* [Tomcat下部署多个项目](http://blog.csdn.net/philosophyatmath/article/details/30246631)
* [JavaScript Source Map 详解](http://www.ruanyifeng.com/blog/2013/01/javascript_source_map.html)