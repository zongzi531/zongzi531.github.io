---
title: React 源码学习（四）：事务机制
date: 2019-04-04 00:01:36
categories: "React"
comments: true
thumbnail: /gallery/LSC-React/logo.png
tags:
- React
---

<!-- no node -->

<!-- more -->

> 阅读源码成了今年的学习目标之一，在选择 Vue 和 React 之间，我想先阅读 React 。
> 在考虑到读哪个版本的时候，我想先接触到源码早期的思想可能会更轻松一些，最终我选择阅读 `0.3-stable` 。
> 那么接下来，我将从几个方面来解读这个版本的源码。

1. [React 源码学习（一）：HTML 元素渲染](https://zongzi531.com/2019/04/01/LSC-React-01/)
2. [React 源码学习（二）：HTML 子元素渲染](https://zongzi531.com/2019/04/02/LSC-React-02/)
3. [React 源码学习（三）：CSS 样式及 DOM 属性](https://zongzi531.com/2019/04/03/LSC-React-03/)
4. [React 源码学习（四）：事务机制](https://zongzi531.com/2019/04/04/LSC-React-04/)
5. [React 源码学习（五）：事件机制](https://zongzi531.com/2019/04/05/LSC-React-05/)
6. [React 源码学习（六）：组件渲染](https://zongzi531.com/2019/04/06/LSC-React-06/)
7. [React 源码学习（七）：生命周期](https://zongzi531.com/2019/04/07/LSC-React-07/)
8. [React 源码学习（八）：组件更新](https://zongzi531.com/2019/04/08/LSC-React-08/)

## 什么是事务

我们需要直接看到事务的整个概念：

> **英文版请看源码，以下为 Google 翻译内容**

`Transaction` 创建一个黑盒子，它能够包装任何方法，以便在调用方法之前和之后维护某些不变量（即使在调用包装方法时抛出异常）。 实例化事务的人可以在创建时提供不变量的执行者。 `Transaction` 类本身将为您提供一个额外的自动不变量 - 任何事务实例在运行时不应该运行的不变量。您通常会创建一个 `Transaction` 的单个实例，以便多次重用，这可能用于包装多个不同的方法。包装器非常简单 - 它们只需要实现两种方法。

```javascript
/**
 * <pre>
 *                                 wrappers (创建时注入)
 *                                      +        +
 *                                      |        |
 *                    +-----------------|--------|--------------+
 *                    |                 v        |              |
 *                    |      +---------------+   |              |
 *                    |   +--|    wrapper1   |---|----+         |
 *                    |   |  +---------------+   v    |         |
 *                    |   |          +-------------+  |         |
 *                    |   |     +----|   wrapper2  |--------+   |
 *                    |   |     |    +-------------+  |     |   |
 *                    |   |     |                     |     |   |
 *                    |   v     v                     v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(任何方法)   | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->| 任何方法 |---|---|-|---|-|-------->
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | +---+ +---+   +---------+   +---+ +---+ |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
 * </pre>
 */
```

奖金：

- 按方法名称和包装器索引报告时间度量标准。

用例：

- 在对帐之前/之后保留输入选择范围。即使出现意外错误也可以恢复选择。
- 在重新排列DOM时停用事件，防止模糊/焦点，同时保证事后系统重新激活。
- 在工作线程中进行协调后，将收集的DOM突变队列刷新到主UI线程。
- 在呈现新内容后调用任何收集的 `componentDidRender` 回调。
- （未来用例）：包装 `ReactWorker` 队列的特定刷新以保留 `scrollTop`  （自动滚动感知DOM）。
- （未来用例）：DOM更新之前和之后的布局计算。

事务性插件 API:

- 具有返回任何预计算的 `initialize` 方法的模块。
- 和一个接受预计算的 `close` 方法。当包装过程完成或失败时，调用 `close` 。

## 事务 Transaction

```javascript
// utils/Transaction.js
var Mixin = {
  /**
   * 设置此实例，以便为收集指标做好准备。 这样做是为了使这个设置方法可以在已经初始化的实例上使用，
   * 其方式是在重用时不消耗额外的内存。 如果您决定将此 mixin 的子类化为 "PooledClass" ，那么这将非常有用。
   */
  reinitializeTransaction: function() {
    // this.getTransactionWrappers() 用于获取上面概念中的 wrappers
    // this.transactionWrappers 就是 wrappers
    this.transactionWrappers = this.getTransactionWrappers();
    // 初始化用于存储 initialize 时返回的内容的 this.wrapperInitData
    if (!this.wrapperInitData) {
      this.wrapperInitData = [];
    } else {
      this.wrapperInitData.length = 0;
    }
    // 时间方面，这里不做解读
    if (!this.timingMetrics) {
      this.timingMetrics = {};
    }
    this.timingMetrics.methodInvocationTime = 0;
    if (!this.timingMetrics.wrapperInitTimes) {
      this.timingMetrics.wrapperInitTimes = [];
    } else {
      this.timingMetrics.wrapperInitTimes.length = 0;
    }
    if (!this.timingMetrics.wrapperCloseTimes) {
      this.timingMetrics.wrapperCloseTimes = [];
    } else {
      this.timingMetrics.wrapperCloseTimes.length = 0;
    }
    // 初始化事务标记
    this._isInTransaction = false;
  },
  _isInTransaction: false,
  getTransactionWrappers: null,
  isInTransaction: function() {
    return !!this._isInTransaction;
  },
  // 在安全窗口内执行该功能。 将此用于顶级方法，这些方法会导致需要进行安全检查的大量计算/突变。
  perform: function(method, scope, a, b, c, d, e, f) {
    throwIf(this.isInTransaction(), DUAL_TRANSACTION);
    var memberStart = Date.now();
    // 报错存储
    var err = null;
    // method 返回内容
    var ret;
    try {
      // 开始 initialize
      this.initializeAll();
      // initialize 完成后调用 method
      ret = method.call(scope, a, b, c, d, e, f);
    } catch (ie_requires_catch) {
      // 抓报错，这里报错会被覆盖
      err = ie_requires_catch;
    } finally {
      var memberEnd = Date.now();
      this.methodInvocationTime += (memberEnd - memberStart);
      try {
        // 开始 close
        this.closeAll();
      } catch (closeAllErr) {
        // 抓报错，这里若前面存在报错，则取前面的报错
        err = err || closeAllErr;
      }
    }
    // 抛出报错
    if (err) {
      throw err;
    }
    // 返回 method 执行结果
    return ret;
  },
  initializeAll: function() {
    // 事务开始
    this._isInTransaction = true;
    var transactionWrappers = this.transactionWrappers;
    var wrapperInitTimes = this.timingMetrics.wrapperInitTimes;
    var err = null;
    // 遍历 wrappers
    for (var i = 0; i < transactionWrappers.length; i++) {
      var initStart = Date.now();
      var wrapper = transactionWrappers[i];
      try {
        // 执行 initialize ，并把返回值存入对应的 this.wrapperInitData[i]
        this.wrapperInitData[i] =
          wrapper.initialize ? wrapper.initialize.call(this) : null;
      } catch (initErr) {
        err = err || initErr;  // Remember the first error.
        // 有报错的话对应位置存入 Transaction.OBSERVED_ERROR
        this.wrapperInitData[i] = Transaction.OBSERVED_ERROR;
      } finally {
        var curInitTime = wrapperInitTimes[i];
        var initEnd = Date.now();
        wrapperInitTimes[i] = (curInitTime || 0) + (initEnd - initStart);
      }
    }
    // 抛出错误
    if (err) {
      throw err;
    }
  },
  /**
   * 调用 `this.transactionWrappers.close[i]`  函数中的每一个，向它们传递
   * `this.transactionWrappers.init[i]` 的相应返回值（对应于失败的初始值设定项的 `close`rs 将不被调用）。
   */
  closeAll: function() {
    throwIf(!this.isInTransaction(), MISSING_TRANSACTION);
    var transactionWrappers = this.transactionWrappers;
    var wrapperCloseTimes = this.timingMetrics.wrapperCloseTimes;
    var err = null;
    for (var i = 0; i < transactionWrappers.length; i++) {
      var wrapper = transactionWrappers[i];
      var closeStart = Date.now();
      // 获取 initialize 时对应的返回值
      var initData = this.wrapperInitData[i];
      try {
        // 返回值不等于 Transaction.OBSERVED_ERROR 时才执行 close
        if (initData !== Transaction.OBSERVED_ERROR) {
          wrapper.close && wrapper.close.call(this, initData);
        }
      } catch (closeErr) {
        err = err || closeErr;  // Remember the first error.
      } finally {
        var closeEnd = Date.now();
        var curCloseTime = wrapperCloseTimes[i];
        wrapperCloseTimes[i] = (curCloseTime || 0) + (closeEnd - closeStart);
      }
    }
    // 释放内存
    this.wrapperInitData.length = 0;
    // 事务关闭
    this._isInTransaction = false;
    // 抛出错误
    if (err) {
      throw err;
    }
  }
};

var Transaction = {
  Mixin: Mixin,
  OBSERVED_ERROR: {}
};
```

## React 事务

上面这段代码看完，我相信你对事务的整个概念已经有所了解，那么事务到底在处理什么的时候用到了呢？接下来就要揭晓本次同样重点的内容 React 调度事务， `ReactReconcileTransaction` ：

```javascript
// core/ReactReconcileTransaction.js
// 这个就是 React 调度事务的 wrappers
var TRANSACTION_WRAPPERS = [
  // 确保在可能的情况下，执行事务不会干扰选择范围（当前选定的文本输入）。
  // 通俗点讲就是，在 React 调度事务开始之时将选择的文本信息获取，再结束时还原选择信息。
  // 关于 Selection 的具体实现这里不做解读。
  SELECTION_RESTORATION,
  // 抑制由于高级别的DOM操作（如临时从DOM中删除文本输入）而可能无意中调度的事件（模糊/焦点）。
  // 通俗点讲就是，在 React 调度事务结束之前，抑制事件的传播。
  // 于 core/ReactEventTopLevelCallback.js 中 createTopLevelCallback 函数，关于 _topLevelListenersEnabled 的判断来抑制。
  // 事件控制，稍后做简单解读，但内容不会提及事件具体实现相关。
  EVENT_SUPPRESSION,
  // 提供 `ReactOnDOMReady` 队列，用于在执行事务期间收集 `onDOMReady` 回调。
  ON_DOM_READY_QUEUEING
];

function ReactReconcileTransaction() {
  // 初始化事务
  this.reinitializeTransaction();
  // 这里涉及到 PooledClass
  // 看到 ReactOnDOMReady 时末端也有这样一句话：
  // PooledClass.addPoolingTo(ReactOnDOMReady)
  // 调用 getPooled 方法其实就是 new
  // 具体在下面解读 PooledClass
  this.reactOnDOMReady = ReactOnDOMReady.getPooled(null);
}

var Mixin = {
  // reinitializeTransaction 方法中使用，用于获取 React 调度事务的 wrappers
  getTransactionWrappers: function() {
    // 判断执行环境是否可以使用 DOM
    if (ExecutionEnvironment.canUseDOM) {
      return TRANSACTION_WRAPPERS;
    } else {
      return [];
    }
  },
  getReactOnDOMReady: function() {
    return this.reactOnDOMReady;
  },
  // 这里也牵扯到 PooledClass
  destructor: function() {
    // PooledClass 中 standardReleaser 方法
    ReactOnDOMReady.release(this.reactOnDOMReady);
    // 释放内存，同样的，在 ReactOnDOMReady 中也进行了操作： this._queue = null
    this.reactOnDOMReady = null;
  }
};

mixInto(ReactReconcileTransaction, Transaction.Mixin);
mixInto(ReactReconcileTransaction, Mixin);

// 这是一个加入 PooledClass 的方法
PooledClass.addPoolingTo(ReactReconcileTransaction);
```

在看完 `ReactReconcileTransaction` 其实似懂非懂，没关系，接下来要依次解读 `PooledClass` ， `ReactOnDOMReady` 。

### 工具方法 PooledClass

以下关于 `PooledClass` 仅解读一个参数的方法，其余多参数情况都是一样的。

```javascript
// utils/PooledClass.js
var oneArgumentPooler = function(copyFieldsFrom) {
  // 相当于执行 CopyConstructor.getPooled()
  // CopyConstructor 就是 addPoolingTo 传入的第一参数
  var Klass = this;
  // 一但执行过 release ， CopyConstructor.instancePool.length 才会大于 0
  if (Klass.instancePool.length) {
    // 这个 instance 是做什么用的，相信看到 call ，你变会理解。
    // 用来绑定执行上下文。
    var instance = Klass.instancePool.pop();
    // 这里不再进行实例化，而是以 instance 为执行上下文直接调用，并将其返回。
    Klass.call(instance, copyFieldsFrom);
    return instance;
  } else {
    // 相当于直接 new CopyConstructor(copyFieldsFrom)
    return new Klass(copyFieldsFrom);
  }
};

var standardReleaser = function(instance) {
  var Klass = this;
  // 在 CopyConstructor.getPooled() 后，保存返回结果
  // 在 release 时作为参数传入，若存在 destructor 则执行。
  if (instance.destructor) {
    instance.destructor();
  }
  // 长度未超过默认大小，则将 instance 推入数组
  if (Klass.instancePool.length < Klass.poolSize) {
    Klass.instancePool.push(instance);
  }
};

// 默认的大小
var DEFAULT_POOL_SIZE = 10;
// 默认的 getPooled 方法
var DEFAULT_POOLER = oneArgumentPooler;

// 直接在 CopyConstructor 上添加属性的方法，并返回 CopyConstructor
var addPoolingTo = function(CopyConstructor, pooler) {
  var NewKlass = CopyConstructor;
  NewKlass.instancePool = [];
  NewKlass.getPooled = pooler || DEFAULT_POOLER;
  if (!NewKlass.poolSize) {
    NewKlass.poolSize = DEFAULT_POOL_SIZE;
  }
  NewKlass.release = standardReleaser;
  return NewKlass;
};

var PooledClass = {
  addPoolingTo: addPoolingTo,
  oneArgumentPooler: oneArgumentPooler,
  twoArgumentPooler: twoArgumentPooler,
  fiveArgumentPooler: fiveArgumentPooler
};
```

要说 `PooledClass` 是做什么用的，具体我也还是没有 get 到，但是经过我的实践，在调用 `var c = CopyConstructor.getPooled()` 后，若在 `c` 上添加属性，如： `c.v = '0.3'` 。并且在调用 `CopyConstructor.release(c)` 这样的情况，在重新进行 `CopyConstructor.getPooled()` 时，这个 `v` 属性及值任然存在，当然前提是，你自己定义的 `destructor` 方法里不会销毁 `v` 的值。

### React DOM 准备完成后的执行队列

那么现在我们来看下 `ReactOnDOMReady` ：

```javascript
// core/ReactOnDOMReady.js
// 初始化队列
function ReactOnDOMReady(initialCollection) {
  this._queue = initialCollection || null;
}

mixInto(ReactOnDOMReady, {
  // 入队
  enqueue: function(component, callback) {
    this._queue = this._queue || [];
    // 组件及回调
    this._queue.push({component: component, callback: callback});
  },
  // 执行队列
  notifyAll: function() {
    var queue = this._queue;
    if (queue) {
      // 清空队列
      this._queue = null;
      for (var i = 0, l = queue.length; i < l; i++) {
        var component = queue[i].component;
        var callback = queue[i].callback;
        // 回调传入组件，上下文绑定 component
        // 关于 getDOMNode 方法稍后解读（获得当前 component 的 node）
        callback.call(component, component.getDOMNode());
      }
      queue.length = 0;
    }
  },
  // 清空队列
  reset: function() {
    this._queue = null;
  },
  // PooledClass.release 时候使用
  destructor: function() {
    this.reset();
  }
});

// 添加 PooledClass 方法
PooledClass.addPoolingTo(ReactOnDOMReady);
```

### 和事务结合

这段代码很简单了， `ReactOnDOMReady` 就是一个执行队列。回顾到上面代码发现， React 调度事务在被初始化的时候，同样 `ReactOnDOMReady` 也被初始化，在调度事务执行 wrappers 的过程时， `ReactOnDOMReady` 被相继执行。这个具体的过程，你看到 `ON_DOM_READY_QUEUEING` 变会明白。

```javascript
// core/ReactReconcileTransaction.js
var ON_DOM_READY_QUEUEING = {
  // 在 React 调度事务 initialize 时， ReactOnDOMReady 队列被重置。
  initialize: function() {
    this.reactOnDOMReady.reset();
  },
  // 在 React 调度事务 close 时， ReactOnDOMReady 队列被依次执行。
  close: function() {
    this.reactOnDOMReady.notifyAll();
  }
};
```

同样的，在 React 调度事务执行 `release` 时， `ReactOnDOMReady` 也会执行 `release` 。

### getDOMNode 方法

差点忘了提下， `getDOMNode` 方法是做什么用的，请直接看到源码：

```javascript
// core/ReactComponent.js
var ReactComponent = {
  Mixin: {
    getDOMNode: function() {
      // 尝试获得 _rootNode
      var rootNode = this._rootNode;
      if (!rootNode) {
        // 尝试获得 _rootNodeID
        rootNode = document.getElementById(this._rootNodeID);
        if (!rootNode) {
          // TODO: Log the frequency that we reach this path.
          // 这里代码就不做详细解读了，反正就是为了获得对应的 Node ，一级级往下查找。
          rootNode = ReactMount.findReactRenderedDOMNodeSlow(this._rootNodeID);
        }
        // 对其进行赋值，用于下次查询。
        this._rootNode = rootNode;
      }
      return rootNode;
    },
  }
};
```

### React 事务中的事件控制

那么现在，我们来看到 `SELECTION_RESTORATION` ：

```javascript
// core/ReactReconcileTransaction.js
var EVENT_SUPPRESSION = {
  initialize: function() {
    // 获取 ReactEvent.isEnabled() 用于 close 时接收。
    // 其实就是 true
    var currentlyEnabled = ReactEvent.isEnabled();
    // 设置为 false
    ReactEvent.setEnabled(false);
    return currentlyEnabled;
  },
  // close 时设置为 true
  close: function(previouslyEnabled) {
    ReactEvent.setEnabled(previouslyEnabled);
  }
};
```

那么在这个真个 wrappers 执行过程，这个有什么用呢？ 这需要看到 `ReactEventTopLevelCallback` 的一段分支逻辑：

```javascript
// core/ReactEventTopLevelCallback.js
var ReactEventTopLevelCallback = {
  createTopLevelCallback: function(topLevelType) {
    return function(fixedNativeEvent) {
      // setEnabled 方法修改的就是 _topLevelListenersEnabled 的值。
      if (!_topLevelListenersEnabled) {
        return;
      }
      // 直接掉过后续逻辑
    };
  }
};
```

### 回顾渲染 HTML 元素事务调度过程

那么到此，我们来回顾一下，之前说到的事务机制的运用是如何进行的，重新看到这段代码是不是会清晰了很多：

```javascript
// core/ReactComponent.js
var ReactComponent = {
  Mixin: {
    mountComponentIntoNode: function(rootID, container) {
      // 初始化 React 调度事务
      var transaction = ReactComponent.ReactReconcileTransaction.getPooled();
      // 进入 React 调度事务 wrappers 环节
      transaction.perform(
        this._mountComponentIntoNode,
        this,
        rootID,
        container,
        transaction
      );
      // 销毁这个 React 调度事务
      ReactComponent.ReactReconcileTransaction.release(transaction);
      // 整个挂载方法结束
    }
  }
}
```

这里附一张此方法的图：

```javascript
/**
 * <pre>
 *                    TRANSACTION_WRAPPERS (ExecutionEnvironment.canUseDOM 为 true 的情况)
 *                                       +                +     +
 *                                       |                |     |
 *                     +-----------------|----------------|-----|-------------------+
 *                     |                 v                |     |                   |
 *                     |      +-----------------------+   |     |                   |
 *                     |   +--| SELECTION_RESTORATION |---|-----|---+               |
 *                     |   |  +-----------------------+   v     |   |               |
 *                     |   |             +-------------------+  |   |               |
 *                     |   |     +-------| EVENT_SUPPRESSION |--|---|-----+         |
 *                     |   |     |       +-------------------+  v   |     |         |
 *                     |   |     |        +-----------------------+ |     |         |
 *                     |   |     |     +--| ON_DOM_READY_QUEUEING |-|-----|-----+   |
 *                     |   |     |     |  +-----------------------+ |     |     |   |
 *                     |   |     |     |                            |     |     |   |
 * perform(_mount      |   v     v     v                            v     v     v   | wrapper
 *         Component   | +---+ +---+ +---+   +----------------+   +---+ +---+ +---+ | invariants
 *         IntoNode)   | |   | |   | |   |   |                |   |   | |   | |   | | maintained
 * +------------------>|-|---|-|---|-|---|-->|     _mount     |---|---|-|---|-|---|-|-------->
 *                     | |   | |   | |   |   |    Component   |   |   | |   | |   | |
 *                     | |   | |   | |   |   |    IntoNode    |   |   | |   | |   | |
 *                     | |   | |   | |   |   |                |   |   | |   | |   | |
 *                     | +---+ +---+ +---+   +----------------+   +---+ +---+ +---+ |
 *                     |     initialize                                 close       |
 *                     +------------------------------------------------------------+
 * </pre>
 */
```

那么至此，实现事务机制及 React 调度事务。

---

## 关于 Reconciler

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

> 参阅[《React Fiber初探》](http://blog.codingplayboy.com/2017/12/02/react_fiber)
