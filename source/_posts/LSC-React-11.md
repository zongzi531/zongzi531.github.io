---
title: React 源码学习（十一）：Scheduling
date: 2019-12-20 00:05:33
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

> 由于文中含有代码较多，建议收藏再做阅读

## 设计原则

> 摘录自[官网](https://zh-hans.reactjs.org/docs/design-principles.html#scheduling)

即便你的组件以 function 的方式声明，在 React 中你也并不会直接调用他们。每个组件返回一个该渲染什么的描述，该描述会包含开发者写的组件如 `<LikeButton>` 和 平台特定的组件如 `<div>`。由 React 决定在未来的某个时间点展开 `<LikeButton>`，并根据组件的渲染结果递归地把这些变更实际应用到 UI 树上。

虽然只是微小的区别，但这样做意义重大。因为你不需要调用组件方法而是让 React 调用他，这意味着如果必要 React 可以延迟调用。在 React 当前的实现中，React 在单个 tick 周期中递归地走完这棵树，然后调用整个更新后树的渲染方法。但是以后 React 可能会[延迟一些更新操作来防止掉帧](https://github.com/facebook/react/issues/6170)。

这在 React 的设计中很常见。有一些流行的库实现了 “push” 模式，即当新数据到达时再计算。然而 React 坚持 “pull” 模式，即计算可以延迟到必要时再执行。

React 不是一个常规的数据处理库，他是开发用户界面的库。我们认为 React 在一个应用中的位置很独特，他知道当前哪些计算当前是相关的，哪些不是。

如果不在当前屏幕，我们可以延迟执行相关逻辑。如果数据数据到达的速度快过帧速，我们可以合并、批量更新。我们优先执行用户交互（例如按钮点击形成的动画）的工作，延后执行相对不那么重要的后台工作（例如渲染刚从网络上下载的新内容），从而避免掉帧。

要清楚我们现在还没有利用调度。然而，我们之所以偏好自己控制调度以及异步setState()，是因为拥有了选择的自由度。

如果我们允许用户使用在一些变体的函数反应式编程范式中常见的“推”模式直接拼接视图，我们将会很难获得调度的控制权。

React 的一个关键目标是在把控制权转交给 React 之前执行的用户代码量最少。这确保 React 保持调度的能力，并根据他所知道的 UI 的情况把工作切分成小块处理。

在团队内部有个笑话，React 本该叫做“调度”因为 React 不想变得完全“反应式（reactive）”。

## 调度

让我们从代码的角度来看 Scheduling ，看到 [Scheduler.js](https://github.com/facebook/react/blob/16.8.6/packages/scheduler/src/Scheduler.js) 文件，接下来将分段进行解读：

打开文件一进入眼帘的就是 5 个优先级以及对应的超时时间。

```javascript
var ImmediatePriority = 1;
var UserBlockingPriority = 2;
var NormalPriority = 3;
var LowPriority = 4;
var IdlePriority = 5;

var IMMEDIATE_PRIORITY_TIMEOUT = -1;
var USER_BLOCKING_PRIORITY = 250;
var NORMAL_PRIORITY_TIMEOUT = 5000;
var LOW_PRIORITY_TIMEOUT = 10000;
var IDLE_PRIORITY = maxSigned31BitInt;
```

顾名思义，通过名字你就可以知道 `ImmediatePriority` 优先级最高， `IdlePriority` 优先级最低。再来让我们认识一下模块中的全局变量：

```javascript
// 用于存储回调函数的循环双向链表，链表头
var firstCallbackNode = null;

var currentDidTimeout = false;
// Pausing the scheduler is useful for debugging.
var isSchedulerPaused = false;

var currentPriorityLevel = NormalPriority;
var currentEventStartTime = -1;
var currentExpirationTime = -1;

// This is set when a callback is being executed, to prevent re-entrancy.
var isExecutingCallback = false;

var isHostCallbackScheduled = false;

var hasNativePerformanceNow =
  typeof performance === 'object' && typeof performance.now === 'function';
```

还是比较好理解的，同样通过名称和注释就能理解。接下来，让我们看到代码，先让我们看到 `flushFirstCallback` 函数：

```javascript
function flushFirstCallback() {
  // 获取链表头，视为需要被执行回调函数的节点
  var flushedNode = firstCallbackNode;
  // 获取链表头的下一个节点
  var next = firstCallbackNode.next;
  if (firstCallbackNode === next) {
    // 若链表头（当前节点）与下一个节点相等的话，说明当前节点也是链表的最后一个节点，操作链表清空
    firstCallbackNode = null;
    next = null;
  } else {
    // 否则连接链表头的前一个节点和后一个节点
    // 并将链表头的下一个节点更新为 firstCallbackNode 用做起始节点
    var lastCallbackNode = firstCallbackNode.previous;
    firstCallbackNode = lastCallbackNode.next = next;
    next.previous = lastCallbackNode;
  }
  // 移除引用
  flushedNode.next = flushedNode.previous = null;

  // Now it's safe to call the callback.
  var callback = flushedNode.callback;
  var expirationTime = flushedNode.expirationTime;
  var priorityLevel = flushedNode.priorityLevel;
  // 暂时缓存全局变量中的 currentPriorityLevel 和 currentExpirationTime 的值，在 finally 中进行还原
  var previousPriorityLevel = currentPriorityLevel;
  var previousExpirationTime = currentExpirationTime;
  currentPriorityLevel = priorityLevel;
  currentExpirationTime = expirationTime;
  var continuationCallback;
  try {
    // 尝试执行回调函数，并获取回调函数返回值，用于稍后操作
    continuationCallback = callback();
  } finally {
    // 还原
    currentPriorityLevel = previousPriorityLevel;
    currentExpirationTime = previousExpirationTime;
  }
  // 返回值依然是函数
  if (typeof continuationCallback === 'function') {
    // 创建循环双向链表节点，优先级和到期时间使用原回调函数值
    var continuationNode: CallbackNode = {
      callback: continuationCallback,
      priorityLevel,
      expirationTime,
      next: null,
      previous: null,
    };
    // 若循环双向链表已被清空，则这个新节点则是第一个
    if (firstCallbackNode === null) {
      firstCallbackNode = continuationNode.next = continuationNode.previous = continuationNode;
    } else {
      // 新节点被插入的位置（插入在之前）
      var nextAfterContinuation = null;
      // 遍历准备
      var node = firstCallbackNode;
      // 仅遍历一次，再次遇到头时结束
      do {
        // 直到被遍历到的节点到期时间大于等于重新创建回调节点的当前时间时，更新 nextAfterContinuation 值，然后退出循环
        if (node.expirationTime >= expirationTime) {
          // 当出现第一个遍历节点的到期时间大于或等于新节点的到期时间的时候，说明新节点需要被插入在之前
          nextAfterContinuation = node;
          break;
        }
        node = node.next;
      } while (node !== firstCallbackNode);
      // 遍历完都没有找到的话，那新节点就是最尾部的节点，所以新节点的后一个就是链表头
      if (nextAfterContinuation === null) {
        nextAfterContinuation = firstCallbackNode;
      } else if (nextAfterContinuation === firstCallbackNode) {
        // 与上一个逻辑位置一样，只不过优先级是最高，而不是最低
        firstCallbackNode = continuationNode;
        ensureHostCallbackIsScheduled();
      }
      // 插入新节点
      var previous = nextAfterContinuation.previous;
      previous.next = nextAfterContinuation.previous = continuationNode;
      continuationNode.next = nextAfterContinuation;
      continuationNode.previous = previous;
    }
  }
}
```

`flushFirstCallback` 函数的操作就是执行循环双向链表中到期时间最小（链表头）的回调函数，如果回调函数执行后依然有返回函数，则按照原先的优先级和到期时间重新按照到期时间的降序插入到循环双向链表中。接下来我们看到上面出现的 `ensureHostCallbackIsScheduled` 函数：

```javascript
function ensureHostCallbackIsScheduled() {
  // 当正在执行回调函数时，退出
  // 通过检索源码可以总结出：
  // - flushImmediateWork 中执行 flushFirstCallback(); 时，满足。
  // - flushWork 中执行 flushFirstCallback(); 时，满足。
  if (isExecutingCallback) {
    return;
  }

  var expirationTime = firstCallbackNode.expirationTime;
  if (!isHostCallbackScheduled) {
    isHostCallbackScheduled = true;
  } else {
    // 若当前正在主机回调，则取消，重置相关内容
    cancelHostCallback();
  }
  // 发起主机回调请求
  requestHostCallback(flushWork, expirationTime);
}
```

到目前为止，若已经正在执行主机回调，则先取消，最后再发起主机回调请求，有关主机回调相关，我们先来认识一下模块中的全局变量：

```javascript
var scheduledHostCallback = null;
var isMessageEventScheduled = false;
var timeoutTime = -1;

var isAnimationFrameScheduled = false;

var isFlushingHostCallback = false;

var frameDeadline = 0;
// We start out assuming that we run at 30fps but then the heuristic tracking
// will adjust this value to a faster fps if we get more frequent animation
// frames.
var previousFrameTime = 33;
var activeFrameTime = 33;
```

在认识完他们后，我们来看到 `requestHostCallback` 函数：

```javascript
requestHostCallback = function(callback, absoluteTimeout) {
  // 更新至模块中的全局变量
  scheduledHostCallback = callback;
  timeoutTime = absoluteTimeout;

  if (isFlushingHostCallback || absoluteTimeout < 0) {
    // 消息通道发送消息，有关 Channel Messaging API
    port.postMessage(undefined);
  } else if (!isAnimationFrameScheduled) {
    // 若没有安排动画帧则安排一个
    isAnimationFrameScheduled = true;
    requestAnimationFrameWithTimeout(animationTick);
  }
};
```

OK ，读到这里也明白了 `requestHostCallback` 函数执行内容，将传入回调函数和超时时间更新至模块中的全局变量后，满足条件的情况下进入消息通道监听函数或者是请求动画帧，请求动画帧会涉及到 `window.requestAnimationFrame` API ，先让我们来看到 Channel Messaging API 相关的代码：

```javascript
// We use the postMessage trick to defer idle work until after the repaint.
var channel = new MessageChannel();
// 对应到 port.postMessage(undefined);
var port = channel.port2;
channel.port1.onmessage = function(event) {
  // 进入监听函数
  isMessageEventScheduled = false;
  // 缓存
  var prevScheduledCallback = scheduledHostCallback;
  var prevTimeoutTime = timeoutTime;
  // 初始化
  scheduledHostCallback = null;
  timeoutTime = -1;

  var currentTime = getCurrentTime();

  var didTimeout = false;

  if (frameDeadline - currentTime <= 0) {
    // There's no time left in this idle period. Check if the callback has
    // a timeout and whether it's been exceeded.
    if (prevTimeoutTime !== -1 && prevTimeoutTime <= currentTime) {
      // Exceeded the timeout. Invoke the callback even though there's no
      // time left.
      didTimeout = true;
    } else {
      // 没有超时
      if (!isAnimationFrameScheduled) {
        // 若没有安排动画帧则安排一个
        isAnimationFrameScheduled = true;
        requestAnimationFrameWithTimeout(animationTick);
      }
      // 还原
      scheduledHostCallback = prevScheduledCallback;
      timeoutTime = prevTimeoutTime;
      return;
    }
  }
  // 当前时间没有超过截止时间
  if (prevScheduledCallback !== null) {
    isFlushingHostCallback = true;
    try {
      // => flushWork(didTimeout);
      prevScheduledCallback(didTimeout);
    } finally {
      isFlushingHostCallback = false;
    }
  }
};
```

`flushWork` 函数我们稍后再提，我们先来看到已经出现了 2 次的 `requestAnimationFrameWithTimeout` 函数：

```javascript
// 动画帧超时 100 ms
var ANIMATION_FRAME_TIMEOUT = 100;
// 编号用于取消动画帧或定时器
var rAFID;
var rAFTimeoutID;
// 关于 window.requestAnimationFrame 为了提高性能和电池寿命，因此在大多数浏览器里，当requestAnimationFrame() 运行在后台标签页或者隐藏的<iframe> 里时，requestAnimationFrame() 会被暂停调用以提升性能和电池寿命。
// 所以 setTimeout 作为 requestAnimationFrame 的 fallback 方案。
var requestAnimationFrameWithTimeout = function(callback) {
  rAFID = localRequestAnimationFrame(function(timestamp) {
    // cancel the setTimeout
    localClearTimeout(rAFTimeoutID);
    callback(timestamp);
  });
  rAFTimeoutID = localSetTimeout(function() {
    // cancel the requestAnimationFrame
    localCancelAnimationFrame(rAFID);
    callback(getCurrentTime());
  }, ANIMATION_FRAME_TIMEOUT);
};
// requestAnimationFrameWithTimeout 中执行的 callback 函数： animationTick
var animationTick = function(rafTime) {
  if (scheduledHostCallback !== null) {
    // Eagerly schedule the next animation callback at the beginning of the
    // frame. If the scheduler queue is not empty at the end of the frame, it
    // will continue flushing inside that callback. If the queue *is* empty,
    // then it will exit immediately. Posting the callback at the start of the
    // frame ensures it's fired within the earliest possible frame. If we
    // waited until the end of the frame to post the callback, we risk the
    // browser skipping a frame and not firing the callback until the frame
    // after that.
    // 发起下一次动画帧，若 scheduledHostCallback 一直存在则无限发起
    requestAnimationFrameWithTimeout(animationTick);
  } else {
    // 没有待处理的工作，退出
    isAnimationFrameScheduled = false;
    return;
  }
  // rafTime 为 requestAnimationFrame 返回的时间戳或者 setTimeout 传入的当前时间戳
  // 配合截止时间和每一帧时间（默认 33）计算下一帧时间
  var nextFrameTime = rafTime - frameDeadline + activeFrameTime;
  // 更新每一帧时间或前一帧时间，由初始的 33 可能会缩小，但是最小 8 帧
  if (
    nextFrameTime < activeFrameTime &&
    previousFrameTime < activeFrameTime
  ) {
    if (nextFrameTime < 8) {
      // Defensive coding. We don't support higher frame rates than 120hz.
      // If the calculated frame time gets lower than 8, it is probably a bug.
      nextFrameTime = 8;
    }
    // If one frame goes long, then the next one can be short to catch up.
    // If two frames are short in a row, then that's an indication that we
    // actually have a higher frame rate than what we're currently optimizing.
    // We adjust our heuristic dynamically accordingly. For example, if we're
    // running on 120hz display or 90hz VR display.
    // Take the max of the two in case one of them was an anomaly due to
    // missed frame deadlines.
    activeFrameTime =
      nextFrameTime < previousFrameTime ? previousFrameTime : nextFrameTime;
  } else {
    previousFrameTime = nextFrameTime;
  }
  // 更新截止时间
  frameDeadline = rafTime + activeFrameTime;

  if (!isMessageEventScheduled) {
    // 若没有进入消息通道则进入
    isMessageEventScheduled = true;
    port.postMessage(undefined);
  }
};
```

那么现在我们回到之前说先放一放的 `flushWork` 函数，就是在消息通道监听函数中尝试执行的 `prevScheduledCallback(didTimeout);` 代码：

```javascript
function flushWork(didTimeout) {
  // Exit right away if we're currently paused

  if (enableSchedulerDebugging && isSchedulerPaused) {
    return;
  }

  // 标记
  isExecutingCallback = true;
  // 缓存
  const previousDidTimeout = currentDidTimeout;
  currentDidTimeout = didTimeout;
  try {
    if (didTimeout) {
      // Flush all the expired callbacks without yielding.
      while (
        firstCallbackNode !== null &&
        !(enableSchedulerDebugging && isSchedulerPaused)
      ) {
        // TODO Wrap in feature flag
        // Read the current time. Flush all the callbacks that expire at or
        // earlier than that time. Then read the current time again and repeat.
        // This optimizes for as few performance.now calls as possible.
        var currentTime = getCurrentTime();
        if (firstCallbackNode.expirationTime <= currentTime) {
          do {
            flushFirstCallback();
          } while (
            firstCallbackNode !== null &&
            firstCallbackNode.expirationTime <= currentTime &&
            !(enableSchedulerDebugging && isSchedulerPaused)
          );
          continue;
        }
        break;
      }
    } else {
      // Keep flushing callbacks until we run out of time in the frame.
      if (firstCallbackNode !== null) {
        do {
          if (enableSchedulerDebugging && isSchedulerPaused) {
            break;
          }
          flushFirstCallback();
        } while (firstCallbackNode !== null && !shouldYieldToHost());
      }
    }
  } finally {
    // 还原
    isExecutingCallback = false;
    currentDidTimeout = previousDidTimeout;
    if (firstCallbackNode !== null) {
      // There's still work remaining. Request another callback.
      ensureHostCallbackIsScheduled();
    } else {
      isHostCallbackScheduled = false;
    }
    // Before exiting, flush all the immediate work that was scheduled.
    flushImmediateWork();
  }
}

shouldYieldToHost = function() {
  return frameDeadline <= getCurrentTime();
};
```

进入 `flushWork` 函数若发现已经超时，则将链表中所有已超时的回调函数任务都执行掉；若没有超时，则链表中任然存在回调函数任务，则发起主机请求，最后将所有优先级为 `ImmediatePriority` 的任务执行掉。让我们来看到 `flushImmediateWork` 函数：

```javascript
function flushImmediateWork() {
  if (
    // Confirm we've exited the outer most event handler
    currentEventStartTime === -1 &&
    firstCallbackNode !== null &&
    firstCallbackNode.priorityLevel === ImmediatePriority
  ) {
    isExecutingCallback = true;
    try {
      do {
        flushFirstCallback();
      } while (
        // Keep flushing until there are no more immediate callbacks
        firstCallbackNode !== null &&
        firstCallbackNode.priorityLevel === ImmediatePriority
      );
    } finally {
      isExecutingCallback = false;
      if (firstCallbackNode !== null) {
        // There's still work remaining. Request another callback.
        ensureHostCallbackIsScheduled();
      } else {
        isHostCallbackScheduled = false;
      }
    }
  }
}
```

这里看到这段代码是不是会很轻松的理解他在做什么，是不是感觉忘了什么内容，来看一下  `cancelHostCallback` 函数在做什么：

```javascript
cancelHostCallback = function() {
  scheduledHostCallback = null;
  isMessageEventScheduled = false;
  timeoutTime = -1;
};
```

很明显，是放弃正在执行的主机回调。到此为止，我们将调度过程已经解读完，你是不是会好奇，这个链表在哪里添加的呢？ OK ，那么下面我们开始解读模块暴露给外部的 API 方法：

```javascript
function unstable_runWithPriority(priorityLevel, eventHandler) {
  switch (priorityLevel) {
    case ImmediatePriority:
    case UserBlockingPriority:
    case NormalPriority:
    case LowPriority:
    case IdlePriority:
      break;
    default:
      priorityLevel = NormalPriority;
  }

  var previousPriorityLevel = currentPriorityLevel;
  var previousEventStartTime = currentEventStartTime;
  currentPriorityLevel = priorityLevel;
  currentEventStartTime = getCurrentTime();

  try {
    // 执行传入回调
    return eventHandler();
  } finally {
    currentPriorityLevel = previousPriorityLevel;
    currentEventStartTime = previousEventStartTime;

    // Before exiting, flush all the immediate work that was scheduled.
    flushImmediateWork();
  }
}
```

可以看到 `unstable_runWithPriority` 函数是按照传入优先级为准正常执行传入回调，最后同样执行 `flushImmediateWork` 函数。
- 同样的 `unstable_next` 也是大同小异的逻辑，唯一区别是他将高于 `NormalPriority` 的优先级进行了降级。
- `unstable_wrapCallback` 则是将运行时的 `currentPriorityLevel` 存入闭包，在调用回调函数时使用当时的 `currentPriorityLevel` 值。
- `unstable_pauseExecution` 则是用于暂停，执行 `isSchedulerPaused = true;` 。
- `unstable_continueExecution` 则是继续执行，复原值后如果有任务的话执行 `ensureHostCallbackIsScheduled();` 。
- `unstable_getFirstCallbackNode` 则是获取 `firstCallbackNode` 。
- `unstable_getCurrentPriorityLevel` 则是获取 `currentPriorityLevel` 。

简单的逻辑，我们一并概括了，接下来我们看到添加链表节点的函数， `unstable_scheduleCallback` 函数：

```javascript
function unstable_scheduleCallback(callback, deprecated_options) {
  var startTime =
    currentEventStartTime !== -1 ? currentEventStartTime : getCurrentTime();

  var expirationTime;
  if (
    typeof deprecated_options === 'object' &&
    deprecated_options !== null &&
    typeof deprecated_options.timeout === 'number'
  ) {
    // FIXME: Remove this branch once we lift expiration times out of React.
    expirationTime = startTime + deprecated_options.timeout;
  } else {
    // 根据当前优先级计算对应的到期时间
    switch (currentPriorityLevel) {
      case ImmediatePriority:
        expirationTime = startTime + IMMEDIATE_PRIORITY_TIMEOUT;
        break;
      case UserBlockingPriority:
        expirationTime = startTime + USER_BLOCKING_PRIORITY;
        break;
      case IdlePriority:
        expirationTime = startTime + IDLE_PRIORITY;
        break;
      case LowPriority:
        expirationTime = startTime + LOW_PRIORITY_TIMEOUT;
        break;
      case NormalPriority:
      default:
        expirationTime = startTime + NORMAL_PRIORITY_TIMEOUT;
    }
  }
  // 创建新的回调节点
  var newNode = {
    callback,
    priorityLevel: currentPriorityLevel,
    expirationTime,
    next: null,
    previous: null,
  };

  // Insert the new callback into the list, ordered first by expiration, then
  // by insertion. So the new callback is inserted any other callback with
  // equal expiration.
  if (firstCallbackNode === null) {
    // This is the first callback in the list.
    firstCallbackNode = newNode.next = newNode.previous = newNode;
    ensureHostCallbackIsScheduled();
  } else {
    // 根据到期时间进行插入
    var next = null;
    var node = firstCallbackNode;
    do {
      if (node.expirationTime > expirationTime) {
        // The new callback expires before this one.
        next = node;
        break;
      }
      node = node.next;
    } while (node !== firstCallbackNode);
    // 操作同 flushFirstCallback 中的部分代码
    if (next === null) {
      // No callback with a later expiration was found, which means the new
      // callback has the latest expiration in the list.
      next = firstCallbackNode;
    } else if (next === firstCallbackNode) {
      // The new callback has the earliest expiration in the entire list.
      firstCallbackNode = newNode;
      ensureHostCallbackIsScheduled();
    }
    // 插入节点
    var previous = next.previous;
    previous.next = next.previous = newNode;
    newNode.next = next;
    newNode.previous = previous;
  }

  return newNode;
}
```

既然有 `unstable_scheduleCallback` 函数进行插入，当然也有 `unstable_cancelCallback` 函数进行移除：

```javascript
function unstable_cancelCallback(callbackNode) {
  var next = callbackNode.next;
  if (next === null) {
    // Already cancelled.
    return;
  }

  if (next === callbackNode) {
    // This is the only scheduled callback. Clear the list.
    firstCallbackNode = null;
  } else {
    // Remove the callback from its position in the list.
    if (callbackNode === firstCallbackNode) {
      firstCallbackNode = next;
    }
    var previous = callbackNode.previous;
    previous.next = next;
    next.previous = previous;
  }

  callbackNode.next = callbackNode.previous = null;
}
```

最后让我们来看到 `unstable_shouldYield` 函数：

```javascript
function unstable_shouldYield() {
  return (
    // 未超时
    !currentDidTimeout &&
    ((firstCallbackNode !== null &&
    // 链表头存在且已经过了到期时间
      firstCallbackNode.expirationTime < currentExpirationTime) ||
      // 或者帧到期时间已到
      shouldYieldToHost())
  );
}
```

到此，我们已经解读完 React 调度机制，至于如何发挥调度机制的作用，我们就得看到协调了。
