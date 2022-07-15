---
title: 微前端下 Webpack Runtime 遇坑记录
date: 2022-07-15 09:36:12
categories: "Webpack"
comments: true
tags:
- webpack
- micro
---

<!-- no node -->

<!-- more -->

故事发生在微前端的场景下，基于 single-spa 定制，在路由切换进行子应用交替时发生的情况。

举例说明， 有子应用 A 和子应用 B ， A 使用 Vue3 相关技术栈， B 使用 Vue2 相关技术栈。

默认情况下，第一次进入 A 或者第一次进入 B ，相应的内容都可以正常的载入，并且不会报错。

但是如果说是先进入 A ，再进入 B ，或者反之操作，会引起后者无法正常载入，并且可以从控制台查看到报错信息。

定位到的问题也非常诡异，检查 A 和 B 本身并没有什么问题会造成这样的情况。

尝试从 Webpack 的编译产物入手排查发现， Webpack Runtime 的代码中，有一个 `modules` 的对象，用于在加载模块时进行查询使用。

仔细来回切换发现， A 和 B 的 `modules` 在第一次进入时都很正常，但是进行切换后会发现 `modules` 出现了污染。

简单点来说就是 A 和 B 的 `modules` 意外的合并了……

仔细检查 Runtime 代码和翻阅 Webpack 文档发现，是因为 async chunk 引起此问题。

并且发现 A 和 B 的代码中的 [`jsonpFunction`](https://www.webpackjs.com/configuration/output/#output-jsonpfunction) 同名了，所以导致了此问题。

> 只在 target 是 web 时使用，用于按需加载(load on-demand) chunk 的 JSONP 函数。
>
> JSONP 函数用于异步加载(async load) chunk，或者拼接多个初始 chunk(CommonsChunkPlugin, AggressiveSplittingPlugin)。
>
> 如果在同一网页中使用了多个（来自不同编译过程(compilation)的）webpack runtime，则需要修改此选项。
>
> 如果使用了 output.library 选项，library 名称时自动追加的。

比较奇怪的是，明明 A 和 B 的构建配置已经设置了独立的 `library` 选项，但是还是造成了这么多困惑。

总之，问题得以解决，记录一下。
