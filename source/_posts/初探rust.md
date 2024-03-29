---
title: 初探 Rust
date: 2022-04-01 10:05:50
categories: "Blog"
comments: true
featured_image: pic1.jpeg
tags:
- Blog
intro: '羡慕老婆先去三亚旅游～而我还在搬砖...'
---

<!-- no node -->

<!-- more -->

羡慕老婆先去三亚旅游～而我还在搬砖...

Rust 依靠着每天一些碎片时间来学习着，等着把基础知识学完成后，开始实践起来，争取写一些工具出来使用。

## 低代码基础建设

**我们团队目前有 HC, 需要招从事低代码基础建设的开发人员...**

这个月主要完成的工作是基于团队内部的低代码协议，打通团队内两款低代码平台和工具的资产，比如组件、配置项等。

这也是当初编写协议的初衷，为了在早期建立起低代码相关的规范。实现自己的低代码引擎！

提供了一个运行时适配器工具，能够实现类似于微前端形式的不同 UI 框架的渲染 SDK ，同时为内部两款低代码平台和工具提供了 `preset` 适配。

再一个就是既然有运行时，那么就必然存在编译时的工具，当然了，编译适配器这么叫肯定不合理，那么我也找了开源社区中并没有像我们这样的可怕需求，就是使用 Vue3 开发，还得兼容 IE 11 。

对于这个需求来说，我基于 `unplugin` 实现了一款编译插件，可以将 Vue3 的 SFC 或者 TSX 文件编译成 Vue2 的编译结果，可以使用在 Vue2 环境下的降级兼容。参考了 `vue-loader` 、 `@vue/compiler-sfc` 、 `vue-template-compiler` 、 `vue-template-es2015-compiler` 、 `@vue/babel-preset-app` 等工具来实现。截止当前插件已实现基础的降级兼容能力，还有不少边界问题没有处理。

以 SFC 为例，对其内容进行解析，解析结果主要产出 `template` 、 `script` 、 `styles` 。

`template` 会带有 `with(this) { ...` 将其包装后编译成 `es2015` 后使用， `script` 则解析语言类型，若是 Typescript 则进入 Babel 处理后更新， `styles` 则会遍历处理，当然若语言是 `postcss` 则进行再处理，等等其他补丁操作。

最后生成 Vue2 编译结果以达成降级兼容。

那么今年也将主要发力低代码基础建设了，加油！

## 绩效

绩效的调整，对未来的工作来说，即是机遇也是挑战。可以接触到更多的管理模式，对自己来说也是增加压力，达成目标，同样加油！

## 开源贡献

这方面这个月主要集中在 Rust 书籍上的一些修正，既然认真阅读了，那也顺手帮忙修正了一些问题。

## 内部建设

其实非常喜欢开源这套协同工作流程，异步的方式去完成各自的目标。也是参考开源的这套协同流程，现在在内部的 Issues 和 PR 上面也有了不错的的进展。在内部 Wiki 上规范了开设 Issues 和 PR 的模板和要求，大家也能按照要求在上面反映和修复问题。内部组件库也进入了一个更好的迭代更新模式～

## 任务进度

* **「日本語学習計画」**：语法及词汇 **第1页**
* **「计算机图形学」**：材质与外观
* **「锻炼身体」**：步行、上楼梯、Just Dance、健身环 **待加强**
* **「ECMA-262」**：6.2.4 The Reference Specification Type
* **「Rust语言圣经(Rust教程 Rust Course)」**：6.3.5 工作空间 Workspace (跳过 4 Rust 异步编程)
