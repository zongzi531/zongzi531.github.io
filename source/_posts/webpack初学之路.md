---
title: Webpack初学之路
date: 2018-03-13 21:43:37
categories: "Tools"
comments: true
tags:
- Webpack
- Tools
---

<!-- no node -->

<!-- more -->

>之前可以说有简单学习过[ruanyf/webpack-demos](https://github.com/ruanyf/webpack-demos)，但并没有很好的理解。
>然后学习了webpack 中文文档，但似乎任然是云里雾里。
>那么为了更好的去理解webpack，所以我决定去改造老项目（机会来了 :)）。
>故事从这里开始

改造前结构：

```bash
│  .gitignore
│  app.js
│  package.json
│  index.html
│
├─css
│    index.css
│
├─fonts
│    index.ttf
│
├─img
│    index.png
│
├─jade
│    index.jade
│
└─js
     index.js
```

老项目存在的问题：
1. 无浏览器缓存优化
2. 代码体积未压缩
3. 非模块化开发模式
4. 手动生成html文件
5. 文件目录复杂不清晰

那么改造开始

## 入口

类似于Vue或者React项目，入口文件一般只有一个index.js或者main.js文件，因为基本此类项目属于单页面应用。

但是老项目却多数为多页面应用，那么入口文件会有多个，这么建议将入口文件设置成对象使用。

使用对象的好处是，输出的时候可以使`[name]`动态获取`key`的值。

我将js、jade、css文件统一归类至src文件目录下，将静态资源归类至static目录下。

同时利用glob编写了一个符合此场景并且可以扩展的小工具`getArrayToSomething`，它可以帮我检索我需要的文件目录下所需的文件，并以数组的形式返回给我，并且我可以进行自定义的扩展或者改造。

例如生成我想要的入口文件对象：

```javascript
{
  'name': 'src/**/**.js',
  // ...
}
```

## 输出

输出对象就相对容易配置许多，但是这里需要特别清楚`[chunkhash]`和`[hash]`的区别，因为这影响到浏览器缓存。

* 用`[hash]`的话，由于每次使用 webpack 构建代码的时候，此 hash 字符串都会更新，因此相当于**强制刷新浏览器缓存**。

* 用`[chunkhash]`的话，则会根据具体 chunk 的内容来形成一个 hash 字符串来插入到文件名上；换句说， chunk 的内容不变，该 chunk 所对应生成出来的文件的文件名也不会变，由此，**浏览器缓存便能得以继续利用**。

## Loaders

可以查看[Loaders](https://doc.webpack-china.org/loaders/)或者检索Github。

这里值得一提的是，在引入一些非模块化的工具时，遇到的一些问题，查看源码你会发现它是一个立即执行函数（IIFE），并且最后会把方法最终暴露在window上。如：

```javascript
;(function (factory) {
  // ...
}(function ($, window, undefined) {
  // ...
  window.API = API;
}));
```

此类工具并没有使用`module.exports`或者`export.default`暴露出接口。

这使得此类工具引入后，方法并没有像预期的那样添加到全局，而是它自己的模块上下文中。

解决方法可以参考《如何在 webpack 中引入未模块化的库，如 Zepto》或者一个我未实践过的[老式jQuery插件还不能丢，怎么兼容？](https://segmentfault.com/a/1190000006887523)。

## 插件

* [ExtractTextWebpackPlugin](https://github.com/webpack-contrib/extract-text-webpack-plugin)

Extract text from a bundle, or bundles, into a separate file.

>默认使用`css-loader`和`style-loader`将css样式插入至html中，但是重复的css对于多页面来说会产生不必要的重新加载和渲染，所以使用此插件解决问题。

* [CommonsChunkPlugin](https://doc.webpack-china.org/plugins/commons-chunk-plugin/)

CommonsChunkPlugin 插件，是一个可选的用于建立一个独立文件(又称作 chunk)的功能，这个文件包括多个入口 chunk 的公共模块。通过将公共模块拆出来，最终合成的文件能够在最开始的时候加载一次，便存到缓存中供后续使用。这个带来速度上的提升，因为浏览器会迅速将公共的代码从缓存中取出来，而不是每次访问一个新页面时，再去加载一个更大的文件。

>我一般会选择抽离出`import`次数大于2次的js模块文件

* [HtmlWebpackPlugin](https://github.com/jantimon/html-webpack-plugin)

Plugin that simplifies creation of HTML files to serve your bundles

>因为是多页面应用，所以同样使用`getArrayToSomething`自定义返回`new HtmlWebpackPlugin()`数组列表

* [CopyWebpackPlugin](https://github.com/webpack-contrib/copy-webpack-plugin)

Copies individual files or entire directories to the build directory

>拷贝static目录文件

* [HotModuleReplacementPlugin](https://doc.webpack-china.org/plugins/hot-module-replacement-plugin/)

启用热替换模块(Hot Module Replacement)，也被称为 HMR。

* [NamedModulesPlugin](https://doc.webpack-china.org/plugins/named-modules-plugin/)

当开启 HMR 的时候使用该插件会显示模块的相对路径，建议用于开发环境。

* [NoEmitOnErrorsPlugin](https://doc.webpack-china.org/plugins/no-emit-on-errors-plugin/)

在编译出现错误时，使用`NoEmitOnErrorsPlugin`来跳过输出阶段。这样可以确保输出资源不会包含错误。对于所有资源，统计资料(stat)的`emitted`标识都是`false`。

* [CleanWebpackPlugin](https://github.com/johnagan/clean-webpack-plugin)

A webpack plugin to remove/clean your build folder(s) before building

>用于生产配置

* [UglifyjsWebpackPlugin](https://github.com/webpack-contrib/uglifyjs-webpack-plugin)

This plugin uses UglifyJS v3 (`uglify-es`) to minify your JavaScript

>用于生产配置

* [OptimizeCssAssetsPlugin](https://github.com/NMFR/optimize-css-assets-webpack-plugin)

A Webpack plugin to optimize \ minimize CSS assets.

>用于生产配置

* [BundleAnalyzerPlugin](https://github.com/webpack-contrib/webpack-bundle-analyzer)

Visualize size of webpack output files with an interactive zoomable treemap.
![](https://cloud.githubusercontent.com/assets/302213/20628702/93f72404-b338-11e6-92d4-9a365550a701.gif)

## 开发中 Server(DevServer)

开发环境配置遇到的一个奇怪的坑。

启用 webpack 的模块热替换特性：
```javascript
hot: true
```

>Note that `webpack.HotModuleReplacementPlugin` is required to fully enable HMR. If `webpack` or `webpack-dev-server` are launched with the --hot option, this plugin will be added automatically, so you may not need to add this to your `webpack.config.js`. See the HMR concepts page for more information.

在我没有手动引入`webpack.HotModuleReplacementPlugin`浏览器则会提示报错，引入后会导致我修改jade文件浏览器不会进行模块热替换。当我将`hot`属性移除，则一切恢复正常。（很奇怪）

## webpack-merge

开发环境(*development*)和生产环境(*production*)的构建目标差异很大。在开发环境中，我们需要具有强大的、具有实时重新加载(live reloading)或热模块替换(hot module replacement)能力的 source map 和 localhost server。而在生产环境中，我们的目标则转向于关注更小的 bundle，更轻量的 source map，以及更优化的资源，以改善加载时间。由于要遵循逻辑分离，我们通常建议为每个环境编写**彼此独立的 webpack 配置**。

虽然，以上我们将生产环境和开发环境做了略微区分，但是，请注意，我们还是会遵循不重复原则(Don't repeat yourself - DRY)，保留一个“通用”配置。为了将这些配置合并在一起，我们将使用一个名为`webpack-merge`的工具。通过“通用”配置，我们不必在环境特定(environment-specific)的配置中重复代码。

---

改造后结构：

```bash
│  .gitignore
│  package.json
│
├─build
│    untils.js
│    webpack.base.conf.js
│    webpack.dev.conf.js
│    webpack.prod.conf.js
│
├─config
│    index.js
│
├─src
│  └─index
│      index.jade
│      index.js
│
└─static
     .gitkeep
```

Github仓库：[zongzi531/jade-webpack3](https://github.com/zongzi531/jade-webpack3)

## 学习资源
* [webpack 中文文档](https://doc.webpack-china.org/)
* [webpack多页应用架构系列](https://segmentfault.com/a/1190000006843916)
* [glob](https://github.com/isaacs/node-glob)
* [如何在 webpack 中引入未模块化的库，如 Zepto](https://sebastianblade.com/how-to-import-unmodular-library-like-zepto/)
