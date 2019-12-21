---
title: React 源码学习（十二）：Reconciliation
date: 2019-12-21 09:02:56
categories: "React 源码学习"
comments: true
tags:
- React
mathjax: true
intro: '继 0.3-stable （以下简称 v0.3）后，这里开始将解读 16.8.6 （以下简称 v16.8...'
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

> 由于文中含有代码较多，建议收藏再做阅读

## 设计动力

> 摘录自[官网](https://zh-hans.reactjs.org/docs/reconciliation.html#motivation)

在某一时间节点调用 React 的 `render()` 方法，会创建一棵由 React 元素组成的树。在下一次 state 或 props 更新时，相同的 `render()` 方法会返回一棵不同的树。React 需要基于这两棵树之间的差别来判断如何有效率的更新 UI 以保证当前 UI 与最新的树保持同步。

这个算法问题有一些通用的解决方案，即生成将一棵树转换成另一棵树的最小操作数。 然而，即使在[最前沿的算法中](https://grfia.dlsi.ua.es/ml/algorithms/references/editsurvey_bille.pdf)，该算法的复杂程度为 \\(O(n^3)\\)，其中 n 是树中元素的数量。

如果在 React 中使用了该算法，那么展示 1000 个元素所需要执行的计算量将在十亿的量级范围。这个开销实在是太过高昂。于是 React 在以下两个假设的基础之上提出了一套 \\(O(n)\\) 的启发式算法：

1. 两个不同类型的元素会产生出不同的树；
2. 开发者可以通过 `key` prop 来暗示哪些子元素在不同的渲染下能保持稳定；

在实践中，我们发现以上假设在几乎所有实用的场景下都成立。

## 协调

在解读协调之前，我们得先来了解一下如何触发协调的，比如从 `ReactDOM.render` 开始（当然最常见的则是调用 `setState()` API ）：

```javascript
ReactDOM.render(
  React.createElement('button', null, 'Like'),
  document.getElementById('root')
)
```

你是不是会好奇 `React.createElement` 在做什么？追溯到源码你会发现这个方法依靠 `type, config, children` 三个参数根据 React 的规则生成了一个对象，没错，就是对象。下面就让我们看到 `ReactDOM.render` 中的代码吧：

```javascript
const ReactDOM: Object = {
  render(
    element: React$Element<any>,
    container: DOMContainer,
    callback: ?Function,
  ) {
    return legacyRenderSubtreeIntoContainer(
      null,
      element,
      container,
      false,
      callback,
    );
  },
};
```

我们可以看出， `ReactDOM.render` 实际上就是一个包装函数，对 `legacyRenderSubtreeIntoContainer` 函数的入参进行了特殊情况的包装， OK ，让我们继续学习：

```javascript
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: DOMContainer,
  forceHydrate: boolean,
  callback: ?Function,
) {
  // _reactRootContainer 是用于判断该 DOM 容器是否挂载过 React
  let root: Root = (container._reactRootContainer: any);
  if (!root) {
    // 没有挂载过，我们进行挂载
    // 挂载的同时根据条件移除 Child ，最后实例化 ReactRoot
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        // 获取根节点的第一个非 HostComponent 实例
        const instance = getPublicRootInstance(root._internalRoot);
        // 包装传入回调函数的上下文
        originalCallback.call(instance);
      };
    }
    // Initial mount should not be batched.
    // root 存在情况逻辑同此，仅区别 unbatchedUpdates 函数
    unbatchedUpdates(() => {
      if (parentComponent != null) {
        // ...
      } else {
        // ReactDOM.render 传入的 parentComponent 为 null
        // 现在调用 ReactRoot.prototype.render
        root.render(children, callback);
      }
    });
  } else {
    // ... 此处逻辑同理，差异仅存在 unbatchedUpdates 函数
  }
  return getPublicRootInstance(root._internalRoot);
}
```

上面有提到，当 `_reactRootContainer` 在 DOM 中不存在实例的情况下，则会实例化 `ReactRoot` ，并且这个实例化内容在上面后续代码中有直接调用到比如 `root.render(children, callback);` ，让我们来看到 `ReactRoot` ：

```javascript
function ReactRoot(
  container: DOMContainer,
  isConcurrent: boolean,
  hydrate: boolean,
) {
  // 创建一个 FiberRoot 数据结构， isConcurrent 则用于控制 Fiber 的模式（ mode ）
  // 注意：Fiber 则存放于 root.current 下
  const root = createContainer(container, isConcurrent, hydrate);
  this._internalRoot = root;
}
ReactRoot.prototype.render = function(
  children: ReactNodeList,
  callback: ?() => mixed,
): Work {
  const root = this._internalRoot;
  /**
   * ReactWork 介绍
   * ReactWork 是一个回调函数队列，仅提供 2 个方法
   * - then 将回调函数插入队列（若已经执行，则直接执行回调函数）
   * - _onCommit 执行回调函数队列（仅执行一次）
   */
  const work = new ReactWork();
  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    work.then(callback);
  }
  updateContainer(children, root, null, work._onCommit);
  return work;
};
```

FiberRoot 是包装在 Fiber 之上的数据结构，其属性则是更多的用于协调所需要。有关于 FiberRoot 数据结构这里就不做更多的解读了，当然你也可以查看 [ReactFiberRoot.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberRoot.js#L103) 了解更多。

那么让我们继续来看到 `updateContainer` ：

```javascript
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): ExpirationTime {
  const current = container.current;
  // 根据不同的情况获取“正确”的当前时间
  const currentTime = requestCurrentTime();
  // 根据“当前正处在的优先级”（使用到 unstable_getCurrentPriorityLevel 方法）计算返回到期时间
  const expirationTime = computeExpirationForFiber(currentTime, current);
  return updateContainerAtExpirationTime(
    element,
    container,
    parentComponent,
    expirationTime,
    callback,
  );
}
```

若对 `requestCurrentTime` 函数有兴趣，建议移步至 [ReactFiberScheduler.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberScheduler.js#L2040) 学习细节。

在根据情况计算出了到期时间后，追加此参数继续调用 `updateContainerAtExpirationTime` 函数： 

```javascript
export function updateContainerAtExpirationTime(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  expirationTime: ExpirationTime,
  callback: ?Function,
) {
  // TODO: If this is a nested container, this won't be the root.
  const current = container.current;
  // 获取上下文，可能包含子节点
  const context = getContextForSubtree(parentComponent);
  // 更新上下文，更新 FiberRoot 字段
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  return scheduleRootUpdate(current, element, expirationTime, callback);
}
```

我们可以看出，实际上 `updateContainer` 也是包装函数，他经过层层包装进入了 `scheduleRootUpdate` 函数，其中计算 Fiber 的到期时间，对 FiberRoot 的上下文状态更新，现在让我们看到 `scheduleRootUpdate` 函数：

```javascript
function scheduleRootUpdate(
  current: Fiber,
  element: ReactNodeList,
  expirationTime: ExpirationTime,
  callback: ?Function,
) {
  /**
   * 创建用于更新队列的节点结构：
   * export type Update<State> = {
   *   expirationTime: ExpirationTime,
   *  
   *   tag: 0 | 1 | 2 | 3,
   *   payload: any,
   *   callback: (() => mixed) | null,
   *  
   *   next: Update<State> | null,
   *   nextEffect: Update<State> | null,
   * };
   **/
  const update = createUpdate(expirationTime);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    // callback 实际上就是 work._onCommit
    update.callback = callback;
  }
  // 执行 unstable_cancelCallback 和 unstable_scheduleCallback 函数
  flushPassiveEffects();
  // 入队 update ，即根据条件更新 current.updateQueue 和 current.alternate.updateQueue
  // 即 WorkInProgress.updateQueue
  enqueueUpdate(current, update);
  scheduleWork(current, expirationTime);

  return expirationTime;
}
```

实际上，我们在从 `updateContainer` 直到 `scheduleWork` 做的是计算 Fiber 到期时间，更新 FiberRoot 上下文，更新 Fiber 更新队列链表结构，我们现在来一探究竟协调工作 `scheduleWork` 函数：

```javascript
function scheduleWork(fiber: Fiber, expirationTime: ExpirationTime) {
  // 更新 Fiber 到期时间以及父的 childExpirationTime 时间
  const root = scheduleWorkToRoot(fiber, expirationTime);
  if (root === null) {
    return;
  }

  if (
    !isWorking &&
    nextRenderExpirationTime !== NoWork &&
    expirationTime > nextRenderExpirationTime
  ) {
    // This is an interruption. (Used for performance tracking.)
    interruptedBy = fiber;
    // 重置栈，源码内部可以看到有关 ReactFiberStack.js 出入栈的内容
    // 所以具体目的是为了什么……🤣
    resetStack();
  }
  // 更新 FiberRoot 上的有关最早和最迟的时间
  markPendingPriorityLevel(root, expirationTime);
  if (
    // If we're in the render phase, we don't need to schedule this root
    // for an update, because we'll do it before we exit...
    !isWorking ||
    isCommitting ||
    // ...unless this is a different root than the one we're rendering.
    nextRoot !== root
  ) {
    const rootExpirationTime = root.expirationTime;
    // 请求工作
    requestWork(root, rootExpirationTime);
  }
}
```

上面源码中本含有 `enableSchedulerTracing` 分支内容，有关 React DevTools ，详情请移步至 [Tracing.js](https://github.com/facebook/react/blob/16.8.6/packages/scheduler/src/Tracing.js) 。

进入 `requestWork` 即标志着协调工作正式开始，先为大家介绍一些协调的大致流程： ① 比对 Fiber 变化 => ② 生成 updateQueue 更新队列 => ③ 更新 DOM 。从简单意义上讲，就是 diff Virtual DOM 然后更新 DOM 。当然了，结合新的协调，以 Fiber 为工作单位细度去完成工作，使用新的调度来调整工作的优先级别（如用户级优先级或立即执行优先级都会较普通优先级而提前），并更新更新队列，最后结合更新队列进行（或批量）更新 DOM 操作（DOM 操作依然是无法中断的，和以前一样）。让我们回到代码：

```javascript
// requestWork is called by the scheduler whenever a root receives an update.
// It's up to the renderer to call renderRoot at some point in the future.
// 即当 FiberRoot 收到更新时即调用 requestWork 函数
function requestWork(root: FiberRoot, expirationTime: ExpirationTime) {
  // 为 FiberRoot 更新到期时间，并可能将其插入至调度根链表尾部
  addRootToSchedule(root, expirationTime);
  if (isRendering) {
    // Prevent reentrancy. Remaining work will be scheduled at the end of
    // the currently rendering batch.
    // 当进入渲染，则不可中断
    return;
  }

  if (isBatchingUpdates) {
    // Flush work at the end of the batch.
    if (isUnbatchingUpdates) {
      // ...unless we're inside unbatchedUpdates, in which case we should
      // flush it now.
      nextFlushedRoot = root;
      nextFlushedExpirationTime = Sync;
      performWorkOnRoot(root, Sync, false);
    }
    return;
  }

  // TODO: Get rid of Sync and use current time?
  if (expirationTime === Sync) {
    performSyncWork();
  } else {
    scheduleCallbackWithExpirationTime(root, expirationTime);
  }
}
```

进入 `performWorkOnRoot` 函数即开启 `isRendering` ，目前都为同步工作，之后检查 FiberRoot 中 `finishedWork` 是否存在，若不存在则表示工作已完成，则执行 `completeRoot` 函数，反之执行 `renderRoot` 函数，执行完成后在此检查 `finishedWork` 并执行 `completeRoot` 函数，最后关闭 `isRendering` 。可以看到 [ReactFiberScheduler.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberScheduler.js#L2351) 。

再来到 `performSyncWork` 函数，他其实就是 `performWork` 的包装函数，同样也仅执行同步工作，在执行前先执行 `findHighestPriorityRoot` 函数，用于查找最高优先级的 FiberRoot ， FiberRoot 则被存放在一个链表中（可以查看 [ReactFiberScheduler.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberScheduler.js#L1919)），可以通过 [ReactFiberRoot.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberRoot.js#L85) 看到 `nextScheduledRoot` 用于连接下一个 FiberRoot 。在函数内部依然会回到 `performWorkOnRoot` 函数的执行，只不过入参是当前模块中的全局变量，在执行完成后依然再执行一次 `findHighestPriorityRoot` 函数，之后根据条件执行 `scheduleCallbackWithExpirationTime` 函数，功能雷同于 `flushPassiveEffects` 函数，有兴趣的同学可以去看一下 [scheduleCallbackWithExpirationTime](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberScheduler.js#L1953) 。最后执行 `finishRendering` 函数，执行已完成的批量更新，可以看到 [ReactFiberScheduler.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberScheduler.js#L2330) ，或者通过[《Batch Update 浅析》 - 饿了么大前端（官方）](https://zhuanlan.zhihu.com/p/28532725)了解更多有关 Batch 的资料。

那么下面让我们分别来看到 `renderRoot` 函数和 `completeRoot` 函数，即可以理解为 Diff 阶段和渲染阶段：

```javascript
function renderRoot(root: FiberRoot, isYieldy: boolean): void {
  // ... 省略代码，工作开始
  do {
    try {
      workLoop(isYieldy);
    } catch (thrownValue) {
      // 抛出异常，继续循环执行 workLoop
      // 若为致命错误则退出循环
    }
    break;
  } while (true);
  // ... 省略代码，工作完成
  // 后续工作：
  // ① 进入致命错误分支
  // ② 进入依然有工作需要完成分支
  // ③ 进入异步分支

  // Ready to commit.
  onComplete(root, rootWorkInProgress, expirationTime);
}
```

省略代码含有较多逻辑，若对此感兴趣可以查看 [`renderRoot`](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberScheduler.js#L1223) 函数。

其中最重要的则是 `workLoop` 函数，函数则是对 `performUnitOfWork` 函数基于异步的条件包装，借助此循环不停地处理 `nextUnitOfWork` 的值，接下来，让我们看到 `performUnitOfWork` 函数：

```javascript
function performUnitOfWork(workInProgress: Fiber): Fiber | null {
  // The current, flushed, state of this fiber is the alternate.
  // Ideally nothing should rely on this, but relying on it here
  // means that we don't need an additional field on the work in
  // progress.
  const current = workInProgress.alternate;

  let next;

  next = beginWork(current, workInProgress, nextRenderExpirationTime);
  // 更新 props
  workInProgress.memoizedProps = workInProgress.pendingProps;

  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    // 尝试完成当前的工作单元，然后移至 sibling Fiber 。如果没有更多的同级，请返回父 Fiber。
    next = completeUnitOfWork(workInProgress);
  }

  ReactCurrentOwner.current = null;
  // next 即 nextUnitOfWork
  return next;
}
```

`performUnitOfWork` 函数其实就是在执行 `beginWork` 函数，若返回值已经没有了，则代表没有新的工作，完成当前工作。

让我们继续：

```javascript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
  // ... 省略代码

  // Before entering the begin phase, clear the expiration time.
  workInProgress.expirationTime = NoWork;
  // 进入标签分支，选择相应的工作内容
  switch (workInProgress.tag) {
    case ClassComponent: {
      const Component = workInProgress.type;
      // 等待中的 props
      const unresolvedProps = workInProgress.pendingProps;
      // 将要更新的 props
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderExpirationTime,
      );
    }
  }
}
```

`beginWork` 函数中仅以 `ClassComponent` 分支为例，其余 Fiber Tag 代码内容，建议移步至 [ReactFiberBeginWork.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberBeginWork.js#L1893) 查看。

以 `ClassComponent` 分支为例的过程也非常的简单，继续：

```javascript
function updateClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps,
  renderExpirationTime: ExpirationTime,
) {
  // ... 省略代码
  const instance = workInProgress.stateNode;
  let shouldUpdate;
  if (instance === null) {
    // ClassComponent 未被初始化
    if (current !== null) {
      // An class component without an instance only mounts if it suspended
      // inside a non- concurrent tree, in an inconsistent state. We want to
      // tree it like a new mount, even though an empty version of it already
      // committed. Disconnect the alternate pointers.
      current.alternate = null;
      workInProgress.alternate = null;
      // Since this is conceptually a new fiber, schedule a Placement effect
      workInProgress.effectTag |= Placement;
    }
    // In the initial pass we might need to construct the instance.
    // 进入 construct ，即 new 操作
    constructClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime,
    );
    // 挂载 ClassComponent ，即执行生命周期函数 getDerivedStateFromProps 或 UNSAFE_componentWillMount
    // 标记生命周期函数 componentDidMount
    mountClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime,
    );
    shouldUpdate = true;
  } else if (current === null) {
    // In a resume, we'll already have an instance we can reuse.
    // 执行生命周期函数 UNSAFE_componentWillReceiveProps 或 getDerivedStateFromProps 或 UNSAFE_componentWillMount
    // 标记生命周期函数 componentDidMount
    shouldUpdate = resumeMountClassInstance(
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime,
    );
  } else {
    // 涉及较多生命周期函数，即更新
    shouldUpdate = updateClassInstance(
      current,
      workInProgress,
      Component,
      nextProps,
      renderExpirationTime,
    );
  }
  // 进入 Diff
  const nextUnitOfWork = finishClassComponent(
    current,
    workInProgress,
    Component,
    shouldUpdate,
    hasContext,
    renderExpirationTime,
  );
  return nextUnitOfWork;
}
```

通过代码大致的学习到整个 `ClassComponent` 分支的创建或更新，以及最终都会进入 Diff 。

有关生命周期即 `ClassComponent` 的初始化、挂载、更新请移步至 [ReactFiberClassComponent.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactFiberClassComponent.js) 查看。当然这只是其中之一的分支，如果你对更多的内容如函数组件感兴趣的话，请自行追溯源码学习。

关于 Diff 这次我们也不会重点学习，毕竟官方已经解释了整个 Diff 过程，如果对 Diff 有兴趣的话，请移步至 [ReactChildFiber.js](https://github.com/facebook/react/blob/16.8.6/packages/react-reconciler/src/ReactChildFiber.js) 学习，相信你会有不少收获。

那么整个协调流程大半已过，结合调度实现 Fiber 工作单元任务可中断，陆续完成 Diff 工作，正式进入渲染阶段。

接下来请看到 `completeRoot` 函数：

```javascript
function completeRoot(
  root: FiberRoot,
  finishedWork: Fiber,
  expirationTime: ExpirationTime,
): void {
  // Check if there's a batch that matches this expiration time.
  // 可以理解为将任务合并为一个批次
  const firstBatch = root.firstBatch;
  if (firstBatch !== null && firstBatch._expirationTime >= expirationTime) {
    if (completedBatches === null) {
      completedBatches = [firstBatch];
    } else {
      completedBatches.push(firstBatch);
    }
    if (firstBatch._defer) {
      // This root is blocked from committing by a batch. Unschedule it until
      // we receive another update.
      root.finishedWork = finishedWork;
      root.expirationTime = NoWork;
      return;
    }
  }

  // Commit the root.
  root.finishedWork = null;

  // Check if this is a nested update (a sync update scheduled during the
  // commit phase).
  if (root === lastCommittedRootDuringThisBatch) {
    // If the next root is the same as the previous root, this is a nested
    // update. To prevent an infinite loop, increment the nested update count.
    nestedUpdateCount++;
  } else {
    // Reset whenever we switch roots.
    lastCommittedRootDuringThisBatch = root;
    nestedUpdateCount = 0;
  }
  runWithPriority(ImmediatePriority, () => {
    commitRoot(root, finishedWork);
  });
}
```

`completeRoot` 函数的尾声即以最高优先级 `ImmediatePriority` 执行 `commitRoot` 函数，即开始渲染工作。

```javascript
function commitRoot(root: FiberRoot, finishedWork: Fiber): void {
  // 标记“正在工作”和“正在 commit ”开始
  const committedExpirationTime = root.pendingCommitExpirationTime;

  root.pendingCommitExpirationTime = NoWork;
  // ... 省略代码

  // Reset this to null before calling lifecycles
  ReactCurrentOwner.current = null;
  // 寻找第一个副作用，作为副作用头，用于后续还原链表遍历所用
  let firstEffect;
  if (finishedWork.effectTag > PerformedWork) {
    // A fiber's effect list consists only of its children, not itself. So if
    // the root has an effect, we need to add it to the end of the list. The
    // resulting list is the set that would belong to the root's parent, if
    // it had one; that is, all the effects in the tree including the root.
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork;
      firstEffect = finishedWork.firstEffect;
    } else {
      firstEffect = finishedWork;
    }
  } else {
    // There is no effect on the root.
    firstEffect = finishedWork.firstEffect;
  }
  // Transaction 事物机制开始
  prepareForCommit(root.containerInfo);

  // Invoke instances of getSnapshotBeforeUpdate before mutation.
  nextEffect = firstEffect;
  while (nextEffect !== null) {
    commitBeforeMutationLifecycles();
  }

  // Commit all the side-effects within a tree. We'll do this in two passes.
  // The first pass performs all the host insertions, updates, deletions and
  // ref unmounts.
  nextEffect = firstEffect;
  while (nextEffect !== null) {
    commitAllHostEffects();
  }
  // Transaction 事物机制完成
  resetAfterCommit(root.containerInfo);

  // The work-in-progress tree is now the current tree. This must come after
  // the first pass of the commit phase, so that the previous tree is still
  // current during componentWillUnmount, but before the second pass, so that
  // the finished work is current during componentDidMount/Update.
  root.current = finishedWork;

  // In the second pass we'll perform all life-cycles and ref callbacks.
  // Life-cycles happen as a separate pass so that all placements, updates,
  // and deletions in the entire tree have already been invoked.
  // This pass also triggers any renderer-specific initial effects.
  nextEffect = firstEffect;
  while (nextEffect !== null) {
    commitAllLifeCycles(root, committedExpirationTime);
  }

  if (firstEffect !== null && rootWithPendingPassiveEffects !== null) {
    // This commit included a passive effect. These do not need to fire until
    // after the next paint. Schedule an callback to fire them in an async
    // event. To ensure serial execution, the callback will be flushed early if
    // we enter rootWithPendingPassiveEffects commit phase before then.
    let callback = commitPassiveEffects.bind(null, root, firstEffect);
    passiveEffectCallbackHandle = runWithPriority(NormalPriority, () => {
      return schedulePassiveEffects(callback);
    });
    passiveEffectCallback = callback;
  }

  // 标记“正在工作”和“正在 commit ”结束
  onCommitRoot(finishedWork.stateNode);
  // ... 省略代码
  onCommit(root, earliestRemainingTimeAfterCommit);
}
```

渲染工作实际意义上是在 `commitAllHostEffects` 函数中完成的，那么我们一步一步来解释整个渲染阶段的流程。

首先还是老样子，进入 Transaction 事物阶段，执行 `commitBeforeMutationLifecycles` 函数，即目前调用 `getSnapshotBeforeUpdate` 函数。

再是进入真正的 DOM 操作阶段，即 `commitAllHostEffects` 函数，在完成整个 DOM 操作后即结束 Transaction 事物阶段。

最后进入 `commitAllLifeCycles` 函数完成剩余生命周期及 `ref` 回调。

有关操作 DOM 的源码其实也大同小异，包括事件机制 SyntheticEvent ，这些内容请自行追溯源码学习。

非常感谢您阅读至此！本次源码阅读系列接近尾声，花了近半年的时间陆陆续续阅读 React 源码，对自己来说一种收获，一种成长。

同样的，希望对正在阅读的你也有所帮助，相信你可以从中学习到数据结构、设计模式、开发模式等内容。

关于 React 其他更新内容，未来有机会的话，再做解读吧！
