---
title: React 源码学习（十）：Fiber
date: 2019-12-19 15:00:25
categories: "React 源码学习"
comments: true
tags:
- React
---

<!-- no node -->

<!-- more -->

> 继 `0.3-stable` （以下简称 v0.3）后，这里开始将解读 `16.8.6` （以下简称 v16.8.6）版本，此版本上标签于 2019 年 3 月 28 日。
> 那么接下来，我将从几个方面来解读这个版本的源码。（目录含有 v0.3 ）

1. [React 源码学习（一）：HTML 元素渲染](https://zongzi531.com/2019/04/01/LSC-React-01/)
2. [React 源码学习（二）：HTML 子元素渲染](https://zongzi531.com/2019/04/02/LSC-React-02/)
3. [React 源码学习（三）：CSS 样式及 DOM 属性](https://zongzi531.com/2019/04/03/LSC-React-03/)
4. [React 源码学习（四）：事务机制](https://zongzi531.com/2019/04/04/LSC-React-04/)
5. [React 源码学习（五）：事件机制](https://zongzi531.com/2019/04/05/LSC-React-05/)
6. [React 源码学习（六）：组件渲染](https://zongzi531.com/2019/04/06/LSC-React-06/)
7. [React 源码学习（七）：生命周期](https://zongzi531.com/2019/04/07/LSC-React-07/)
8. [React 源码学习（八）：组件更新](https://zongzi531.com/2019/04/08/LSC-React-08/)
9. [React 源码学习（九）：“脱胎换骨”](https://zongzi531.com/2019/12/18/LSC-React-09/)
10. [React 源码学习（十）：Fiber](https://zongzi531.com/2019/12/19/LSC-React-10/)
11. [React 源码学习（十一）：Scheduling](https://zongzi531.com/2019/12/20/LSC-React-11/)
12. [React 源码学习（十二）：Reconciliation](https://zongzi531.com/2019/12/21/LSC-React-12/)

## 什么是 Fiber ？

Fiber 是 React 16 中新的协调引擎。他的主要目的是使 [Virtual DOM](https://zh-hans.reactjs.org/docs/faq-internals.html#what-is-the-virtual-dom) 可以进行增量式渲染。**[了解更多](https://github.com/acdlite/react-fiber-architecture)**

> 建议阅读**了解更多**来理解 Fiber ，或者阅读译文[《【翻译】React Fiber 架构》](https://juejin.im/post/5ddb722cf265da7e264736a0)了解。

从数据结构来理解 Fiber ，他其实是一个链表数据结构，分别通过 `return` ， `child` ， `sibling` 连接着另一个 Fiber 。同样的也可以通过 `nextEffect` 通往下一个副作用 Fiber ，或是通过 `firstEffect` 或 `lastEffect` 跳转到第一个或最后一个副作用 Fiber 。接下来，让我们看到源码：

```javascript
export type Fiber = {|
  // Fiber 标签，用于判断 Fiber 类型
  tag: WorkTag,
  // 唯一标识
  key: null | string,
  // 元素类型，协调时使用
  // 检索源码可以发现 elementType 有存在与 type 、 String 类型、 ReactSymbols 进行比较
  elementType: any,
  // 类型
  type: any,
  // 本地状态
  stateNode: any,
  // 父级 Fiber
  return: Fiber | null,
  // 单链列表树结构。
  // 子 Fiber
  child: Fiber | null,
  // 兄弟 Fiber
  sibling: Fiber | null,
  // 当前索引
  index: number,
  // ref 引用支持 callback 函数或者 React.createRef
  ref: null | (((handle: mixed) => void) & {_stringRef: ?string}) | RefObject,
  // 更新时等待的 props
  pendingProps: any,
  // 创建时使用的 props
  memoizedProps: any,
  // 更新队列，即协调时使用
  updateQueue: UpdateQueue<any> | null,
  // 创建时使用的 state
  memoizedState: any,
  // 上下文以来链表结构
  contextDependencies: ContextDependencyList | null,
  // 位域，描述有关 Fiber 及其子树的属性。例如， ConcurrentMode 标志指示子树是否应默认为异步。
  // 创建 Fiber 时，他将继承其父级的模式。可以在创建时设置其他标志，但是此后，该值应在 Fiber 的整个生命周期中保持不变，尤其是在创建其子 Fiber 之前。
  mode: TypeOfMode,
  // 副作用标签
  effectTag: SideEffectTag,
  // 单链表列出了具有副作用的通往下一个 Fiber 的快速路径。
  nextEffect: Fiber | null,
  // 该子树中具有副作用的第一个和最后一个 Fiber 。 当我们重用此 Fiber 中完成的工作时，这使我们可以重用链表的一部分。
  firstEffect: Fiber | null,
  lastEffect: Fiber | null,
  // 当前 Fiber 的到期时间，用于调度时排序使用，将到期时间越早的排在越前面进行执行。
  expirationTime: ExpirationTime,
  // 子的到期时间，用于快速确定子树是否没有挂起的更改。
  childExpirationTime: ExpirationTime,
  // WorkInProgress 存放处（双缓存技术）
  alternate: Fiber | null,
|};
```

源码中依然会有一些难以理解的字段内容，但是没关系，相信在看完调度和调和，你会对 Fiber 有更深刻的理解，因为他贯穿在整个调和中。

当然啦，也可以在 [ReactFiber.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiber.js) 中看到有关 Fiber 创建函数的内容，有兴趣可以点击查看。

同样的，在调和中会认识到一个新的数据结构 `FiberRoot` ，他和调和密切相关，也可以先通过 [ReactFiberRoot.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberRoot.js) 进行了解。

那么后面我们将介绍以 Fiber 为最小单位的时间切片（ Time Slicing ）概念的一种实现：**调度**。
