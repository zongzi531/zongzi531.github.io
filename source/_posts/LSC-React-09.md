---
title: React 源码学习（九）：“脱胎换骨”
date: 2019-12-18 09:38:33
categories: "React 源码学习"
comments: true
tags:
- React
---

<!-- no node -->

<!-- more -->

> 继 `0.3-stable` （以下简称 v0.3）后，这里开始将解读 `16.8.6` （以下简称 v16.8.6）版本，此版本上标签于 2019 年 3 月 28 日。
> 那么接下来，我将从几个方面来解读这个版本的源码。（目录含有 v0.3 ）

1. [React 源码学习（一）：HTML 元素渲染](https://zongzi531.github.io/2019/04/01/LSC-React-01/)
2. [React 源码学习（二）：HTML 子元素渲染](https://zongzi531.github.io/2019/04/02/LSC-React-02/)
3. [React 源码学习（三）：CSS 样式及 DOM 属性](https://zongzi531.github.io/2019/04/03/LSC-React-03/)
4. [React 源码学习（四）：事务机制](https://zongzi531.github.io/2019/04/04/LSC-React-04/)
5. [React 源码学习（五）：事件机制](https://zongzi531.github.io/2019/04/05/LSC-React-05/)
6. [React 源码学习（六）：组件渲染](https://zongzi531.github.io/2019/04/06/LSC-React-06/)
7. [React 源码学习（七）：生命周期](https://zongzi531.github.io/2019/04/07/LSC-React-07/)
8. [React 源码学习（八）：组件更新](https://zongzi531.github.io/2019/04/08/LSC-React-08/)
9. [React 源码学习（九）：“脱胎换骨”](https://zongzi531.github.io/2019/12/18/LSC-React-09/)
10. [React 源码学习（十）：Fiber](https://zongzi531.github.io/2019/12/19/LSC-React-10/)
11. [React 源码学习（十一）：Scheduling](https://zongzi531.github.io/2019/12/20/LSC-React-11/)
12. [React 源码学习（十二）：Reconciliation](https://zongzi531.github.io/2019/12/21/LSC-React-12/)

## 概况

历时近 6 年之久，从 0.3.0 (*May 29, 2013*) 至 16.8.6 (*March 27, 2019*) 整个 React 框架已经经历了可以说是“脱胎换骨”，具体更新内容细节可以移步至 [CHANGELOG.md](https://github.com/facebook/react/blob/16.8.6/CHANGELOG.md) 查看。

回看 v0.3 ，源码存放在 `src` 目录下，而 v16.8.6 则将源码存放在 `packages` 目录下，并且每个文件夹都含有自己单独的 `package.json` 文件，典型的 Monorepo 仓库管理方式。

再来比较目录结构，回看 v0.3 仅限使用在浏览器上，而 v16.8.6 已经将 React 核心部分与浏览器部分进行解耦，从这个角度讲其核心部分可以与任意“端”进行组合，这里我提到的“端”指的有些宽泛，比如说我们最为常见的浏览器。目录结构中也有一个文件夹可以表明这一现象，就是 `react-native-renderer` 。但是该文件夹中未包含 `package.json` 文件，我们移步至 [npm](https://www.npmjs.com/package/react-native-renderer) 可以看到简单的一句话：

> This package is the renderer that is used by the react-native package. It is intended to be used inside the react-native environment. It is not intended to be used stand alone.

妙就妙在， React 团队将其核心部分进行复用了，可用在浏览器和 Native ，在此情况下，我们是不是可以想到，使用其核心部分我们同样可以运用到比如小程序这样的“端”，真妙啊。

当然了， v16.8.6 中还含有一些不常用的小工具，建议可以去看一看，作为了解。比如说： [create-subscription](https://www.npmjs.com/package/create-subscription/v/16.8.6) 、 [react-art](https://www.npmjs.com/package/react-art/v/16.8.6) 、 [react-cache （LRU 算法实现）](https://www.npmjs.com/package/react-cache) 、 react-is 、包括单侧工具、 ESLint 插件等等。

## 变化

简单从目录结构的角度谈了一下 React 的变化，那么接下来，我们从细节入手，看看这些年到底都变化了什么。

### JSX

- v0.3 使用 JSXTransformer 来编译 JSX 语法
- v16.8.6 使用 Babel 来编译 JSX 语法

### 组件

- v0.3 中分为：
  - ReactTextComponent: 文本节点
  - ReactNativeComponent: 原生组件，即浏览器提供的 HTML 标签
  - ReactCompositeComponent: 复合组件，包含生命周期


- v16.8.6 中分为（ Fiber 标签）：
  - FunctionComponent: 函数组件
  - ClassComponent: 类组件
  - ... 具体请移步至 [ReactWorkTags.js](https://github.com/facebook/react/blob/16.8.6/packages/shared/ReactWorkTags.js#L31)


如果你熟悉 React 并且又看过我之前写的文章，你就会发现 `React.DOM.*` 不见了。是的，代替他的则是 `React.createElement` 。曾经的 `React.DOM.p()` 等同于现在的 `React.createElement('p')` 。

v0.3 中使用原生组件都是使用 `React.DOM.*` ，使用 React （复合）组件则是直接函数调用 。在 v16.8.6 中官方将其进行了 API 合并，统一使用 `React.createElement` 。函数调用则衍生出了函数组件。

同样的 `React.renderComponent` 也是，代替他的则是 `ReactDOM.render` 。  `React.isValidComponent` 被 `React.isValidElement` 代替。

以及 `React.createClass` 代替他的则是 `React.Component` 或 `React.PureComponent` 。

我们从一个例子来看一看，一些其他的区别：

```javascript
// v0.3
const ExampleApplication = React.createClass({
  // 初始化 state 方法被移除
  getInitialState: function () {
    return { message: 'zongzi' }
  },
  // 绑定上下文 React.autoBind 工具函数被移除
  handleClick: React.autoBind(function() { /* do something use this */ }),
  // 生命周期
  componentWillMount: function () {},
  componentDidMount: function () {},
  componentWillReceiveProps: function () {},
  shouldComponentUpdate: function () {},
  componentWillUpdate: function () {},
  componentDidUpdate: function () {},
  componentWillUnmount: function () {},
  updateComponent: function () {},

  render: function () {
    return React.DOM.p({
      onClick: this.handleClick,
      // ref 仅支持 String
      ref: 'example',
    }, this.state.message)
  }
})

React.renderComponent(
  ExampleApplication(),
  document.getElementById('container')
)
```

```javascript
// v16.8.6
class ExampleApplication extends React.Component {
  constructor(props) {
    super(props)
    // 取而代之的是，直接在类组件构造函数中定义
    // 初始化则自动更新 state
    this.state = { message: 'zongzi' }
    // 取而代之的是，使用 bind 方法
    this.handleClick = this.handleClick.bind(this)
  }
  // ref 禁用 String 需使用 React.createRef() 或 callback
  example = React.createRef()

  handleClick () { /* do something use this */ },
  // 或者使用箭头函数
  // handleClick = () => { /* do something use this */ },

  // 生命周期
  componentDidMount() {},
  componentDidUpdate(prevProps, prevState, snapshot) {},
  componentWillUnmount() {},
  shouldComponentUpdate(nextProps, nextState) {},
  static getDerivedStateFromProps(props, state) {},
  getSnapshotBeforeUpdate(prevProps, prevState) {},
  static getDerivedStateFromError(error) {},
  componentDidCatch(error, info) {},
  UNSAFE_componentWillMount() {}, // 建议： componentDidMount
  UNSAFE_componentWillReceiveProps(nextProps) {}, // 建议： static getDerivedStateFromProps
  UNSAFE_componentWillUpdate(nextProps, nextState) {}, // 建议： componentDidUpdate

  render() {
    return React.createElement('p', {
      onClick: this.handleClick,
      ref: this.example,
    }, this.state.message)
  }
}

ReactDOM.render(
  React.createElement(ExampleApplication),
  document.getElementById('root')
)
```

### 事物机制

是否还记得 v0.3 中的 Transaction 事物机制，但是在 v16.8.6 并没有像以前那样的那么明显，但是能从 [ReactDOMHostConfig.js](https://github.com/facebook/react/blob/16.8.6/packages/react-dom/src/client/ReactDOMHostConfig.js#L166) 中仍然可以找到他的影子：

```javascript
export function prepareForCommit(containerInfo: Container): void {
  eventsEnabled = ReactBrowserEventEmitterIsEnabled();
  selectionInformation = getSelectionInformation();
  ReactBrowserEventEmitterSetEnabled(false);
}

export function resetAfterCommit(containerInfo: Container): void {
  restoreSelection(selectionInformation);
  selectionInformation = null;
  ReactBrowserEventEmitterSetEnabled(eventsEnabled);
  eventsEnabled = null;
}
```

检索代码就可以发现 `prepareForCommit` 和 `resetAfterCommit` 两个函数贯穿在 Reconciliation 之间。这也是最大的变化，让我们回顾一下 Reconciliation 的转变：

**[Stack Reconciler](https://reactjs.org/docs/codebase-overview.html#stack-reconciler)**

我们知道浏览器渲染引擎是单线程的，在 React 15.x 版本及之前版本，计算组件树变更时将会阻塞整个线程，整个渲染过程是连续不中断完成的，而这时的其他任务都会被阻塞，如动画等，这可能会使用户感觉到明显卡顿，比如当你在访问某一网站时，输入某个搜索关键字，更优先的应该是交互反馈或动画效果，如果交互反馈延迟 200ms ，用户则会感觉较明显的卡顿，而数据响应晚200毫秒并没太大问题。这个版本的调和器可以称为栈调和器（ Stack Reconciler ），其调和算法大致过程见 [React Diff 算法](http://blog.codingplayboy.com/2016/10/27/react_diff/) 和 [React Stack Reconciler 实现](https://reactjs.org/docs/implementation-notes.html)。

Stack Reconcilier 的主要缺陷就是不能暂停渲染任务，也不能切分任务，无法有效平衡组件更新渲染与动画相关任务间的执行顺序，即不能划分任务优先级，有可能导致重要任务卡顿，动画掉帧等问题。

**[Fiber Reconciler](https://reactjs.org/docs/codebase-overview.html#fiber-reconciler)**

React 16 版本提出了一个更先进的调和器，它允许渲染进程分段完成，而不必须一次性完成，中间可以返回至主进程控制执行其他任务。而这是通过计算部分组件树的变更，并暂停渲染更新，询问主进程是否有更高需求的绘制或者更新任务需要执行，这些高需求的任务完成后才开始渲染。这一切的实现是在代码层引入了一个新的数据结构 - Fiber 对象，每一个组件实例对应有一个 fiber 实例，此 fiber 实例负责管理组件实例的更新，渲染任务及与其他 fiber 实例的联系。

这个新推出的调和器就叫做纤维调和器（ Fiber Reconciler ），它提供的新功能主要有：

1. 可切分，可中断任务；
2. 可重用各分阶段任务，且可以设置优先级；
3. 可以在父子组件任务间前进后退切换任务；
4. render 方法可以返回多元素（即可以返回数组）；
5. 支持异常边界处理异常；

### 事件机制

曾经的 AbstractEvent 更名为 SyntheticEvent 。当然啦，也不是简单的更名而已，相关的属性有着细微的变化，总体来看，本身重要的属性现在依旧存在，比如 `type` 和 `target` 。其他内容大同小异，具体我没有细读，本质上就是换汤不换药。

包括你还记得事件注入机制吗？是的，他还在那边，一样的存在着。

但是虽然这么说，这里还是得说明一点： `packages/events` 目录下是已经被抽象出来的内容，包括 SyntheticEvent 。而 `packages/react-dom/src/events` 下则是基于浏览器的实现，以及继承至 SyntheticEvent 的 Event 事件数据结构和浏览器抹平操作。

## 其他

1. 本次将不会解读有关 React Hooks 内容
2. `__DEV__` 等内容会被跳过
3. 本次将不会解读 Diff 相关内容
4. 本次将不会解读 DOM 操作相关内容
