---
title: May
date: 2019-06-02 21:54:01
categories: "React"
comments: true
thumbnail: /gallery/may/may.jpeg
tags:
- React
---

<!-- no node -->

<!-- more -->

5月，天气忽冷忽热的，挺难受的，这个天气也挺容易感冒。近段时间，晚上会下楼运动🏃一下，主要是跳绳、拉伸、跑步、俯卧撑、哑铃这些运动。这个月月初的我还在肝游戏，但是发现这样有些浪费时间，于是决定利用通勤时间做点事情，那么最容易做到的就是阅读文章啦，刚好在博客中开设了一个[位置](https://zongzi531.com/commuting/)来记录我阅读过得文章，以便后续查阅学习。

阅读 React 源码计划也在继续，当然遇到了一些困难，我准备慢慢的一一的克服他们，在阅读 `packages/react` 部分其实还是很好理解的，这与 `0.3` 中有很多的相似点，但是 `packages/react-dom` 时，会以为与之前也有很多相似之处，但是却恰恰相反，因为引入的 Fiber 数据结构。那么在此之前，我需要对这部分知识进行整理，讲实话牵扯的内容有些多。

为了给自己一个缓冲区，所以我觉得先学习一些其他部分，来缓解缓解心情。这时候我选择看一下之前面试的时候了解到的 [Vue Slot](https://cn.vuejs.org/v2/guide/components-slots.html) ，讲实话，这个月有小迭代之前写的 Vue 2.x 项目，但是由于长时间没有写 Vue 有点陌生，不过也很快适应过来了（毕竟可读性还算过得去）。让我们再回到 Vue Slot ，很巧的是，在通勤阅读的时候，有阅读到关于原生自己支持 Slot 的文章：[《JavaScript 的工作原理：Shadow DOM 的内部结构以及如何构建可封装的组件》](https://mp.weixin.qq.com/s/-jkmCrNe35qD5vUizuv16g)，这一系列的文章都是围绕着 Web Components 来讲述的，还是很值得一看的，同样的 Vue Slot 的设计灵感也是源于 [Web Components 规范草案](https://github.com/w3c/webcomponents/blob/gh-pages/proposals/Slots-Proposal.md)，刚好想到想找点轮子，借着 React 来实现 Slot ，其实 React 本身就含有这样的功能，就是对 `this.props.children` 稍作改造，就是插槽功能了。可以查看到我的项目 [zongzi531/react-slot-element](https://github.com/zongzi531/react-slot-element) 或 `yarn add react-slot-element` 使用。

再来说说通勤那点事吧，看了那么些文章，总得学到一些东西拿来总结总结吧。我如果把里面的东西全部照搬过来也不好，那么我就脱离着讲讲感受吧。先来说说关于 React 及其周边的学习，对于 Redux 有了更多的了解，以及 immutable 这个概念，比较幸运的是，我司的项目都使用了 Redux 的黄金搭档 Immutable.js ，使用起来还是非常的方便快捷的；对于 HOC 也有了跟多的学习和实践，比如上面的 react-slot-element 就是有包含 HOC 的实践；对于 Hook 有了浅显的认识；对 CSS 变量也有了新的学习，我司 PC 端项目是无需兼容旧版本浏览器的，也在跟换主题，封装组件时统一调整视窗高度等地方直接进行了实践，效果拔群，那种和 JavaScript 解耦的感觉很是美妙；对 Web Components 也有了新的认识；对一些前端周边的职位，职业发展规划等等。

说到这里，其实可以引申出一个问题，也就是通勤时读到的[《【第1613期】项目优化却体现不出自己的价值？》](https://mp.weixin.qq.com/s/rM2Jw4UimIZJLZBb5XVJYQ)，怎样来体现自己的价值呢？用数据！那么数据从哪里来呢？平时的点点滴滴！所以要试图让自己养成这样的习惯，让自己多思考这类问题，来表现出自己的价值。

当然啦，不要忘了 React 源码阅读计划，在遇到困难时，从其他点切入或许是个更好的方法，比如看看资料，或文档什么的，当然我知道 16.8.6 这个版本没那么容易很快读下来，但是必然会被我读下来！

在我司后端项目开发过程中，那么我这次采用了 `React.lazy()` + `import()`，来实现分包。具体实践了以后，我对这块部分也有了想要的了解，希望可以在阅读源码时帮助到我，参考[《react v16.6 动态 import，React.lazy()、Suspense、Error boundaries》](http://www.ptbird.cn/react-lazy-suspense-error-boundaries.html)

对了，这个月我买了《极客时间 - winter 重学前端》，其实之前就有在朋友圈推广，但是之前并没有很感兴趣，但是恍然之间，我看到了他的目录，我认为里面可能有我需要的知识，所以我买了，慢慢读起来。

这个月在我司内部部署了 npm 服务，但是因为我司内部服务器太垃圾了，速度慢的让人没办法接受，但是至少部署了，采用 [verdaccio](https://github.com/verdaccio/verdaccio/) ，有兴趣的可以去尝试一下。

再来说说又写了个中国地图穿梭组件，因为公司内部项目有分析，那么采用地图的形式看全国的业务数据，之前写的类是基于 Vue 写的，并不适合 React 。于是决定按照之前的逻辑编写个 React 组件，以便于后续迭代，目前功能以及实现，可以查看我的项目 [zongzi531/react-echarts-map-china](https://github.com/zongzi531/react-echarts-map-china) 或 `yarn add react-echarts-map-china` 使用。但是在单元测试这一方面的实践还是遇到了一些问题，后续解决后再统一进行总结。

最后补充一些关于职业规划和思考的内容，如何从长远来看待自己的职业，从平时就考虑问题就可以试着从小问题映射到大问题，以考虑大问题来解决这个问题，这样有利于自己的思考，并且可以相应的积累下一些经验来，同时当公司遇到类似的问题时，也能及时的应对及解决，职业规划中不可缺少的当然是上面提到的价值，这里说道的也是价值的体现。包括在写代码时，如何把一些重复的内容（复制黏贴工程师）转化成工程化的内容，为开发提供高效，这也是需要去思考的，不要苟且与现状，要勇于尝试，其实我总结的也很模糊笼统，在看完大佬写的文章后，自己模模糊糊有些意识出来，相信在未来自己会有时间去实践！
