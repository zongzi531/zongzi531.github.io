---
title: 加深 Webcontainer 和 Agent 的融合
date: 2025-09-01 09:56:11
categories: "JavaScript"
comments: true
featured_image: pic1.jpg
tags:
- WebContainer
- LLM
- Agent
---

<!-- no node  -->

<!-- more -->

简单点说，目前在做的是在浏览器内基于 Webcontainer 实现的 Agent 。（侧重在展示，并支持代码修改）

## 重构 Webcontainer 封装

发现同事之前写的代码对 Webcontainer 的 `spawn` 方法封装并没有很好的管控，不是单例级别的调用和销毁，多次调用会产生多个 `WebContainerProcess` 的泄露和污染。

于是对 Webcontainer 的封装进行了重构，以更清晰的形式进行管理和使用。

另外，在启动时，开启 `forwardPreviewErrors` 配置项，可以获得到更多的报错信息，以保证流程完整性。

### Webcontainer 内运行的工程优化

之前在 Webcontainer 中使用的 Node.js 工程是基于 vite 脚手架的工程代码，并且专门写了一个 vite 插件来监控 vite 的 dev 编译流程。

在此工程中充斥着一些与工程无关的业务逻辑，导致模型在修改代码的时候，可能会无意改动到关键位置，从而影响到正常的业务流程。

在此背景下，也进行了优化，将大部分逻辑迁移到 Webcontainer 外层，通过 Webcontainer 提供的 `setPreviewScript` 方法进行注入，此过程中还接触到了一些 vite 编译态的代码风格。

进行相应的兼容处理，从而让工程相对更为纯净，目前还无法将工程做的百分之百纯净（努力中）。

也在模型和用户修改代码时，附加了校验代码，利用 Babel 对剩余的部分文件进行了必要的监管，以防止被意外修改，保证业务的完整性。

## Agent 的流程完善

在 Agent 的流程上，基于原流程补充前端自校验代码流程。

原流程模块有：模型路由、用户意图判断、图片解析、生成需求文档、生成UI设计文档、生成当前任务TODO清单、生成代码、校验代码等流程完善模块。

前端自校验代码则是，通过通信的方式使得前端能够进行自主的校验检测，任务开始直至运行完成，会便利所有的前端路由。

若期间产生报错信息，则进入到错误修复模式，待修复完成后重新进行前端自校验代码，直至没有错误产生。

则代表完成此次任务。
