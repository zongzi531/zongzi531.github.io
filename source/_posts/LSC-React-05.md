---
title: React 源码学习（五）：事件机制
date: 2019-04-05 08:25:52
categories: "React"
comments: true
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

## React 事件

我们先来看到整体逻辑图：

![事件机制](05-01.png)

React 采用将事件挂载至 `document` 或者 `window` 上来实现顶级事件。接下来我们会一一来介绍事件的实现过程。

### 事件插件注入

我们来回忆下， `ReactNativeComponent.js` 中 `_createOpenTagMarkup` 方法中的 `putListener` 函数和 `ReactMount.js` 中 `renderComponent` 方法中的 `prepareTopLevelEvents` 函数都是事件相关的方法，前者是生成 HTML 标记时，对符合要求的“键”进行注册事件，后者则是在执行 `React.renderComponent` 方法时进行准备顶级事件的过程。当然之前有一个内容没有提到，就是在 React 注入时，默认注入了事件插件排序顺序、实例操作函数、事件插件。那让我们直接先看到代码：

```javascript
// core/ReactDefaultInjection.js
function inject() {
  // 注入模块，用于解析DOM层次结构和插件排序。
  // 这里是注入事件插件排序顺序
  EventPluginHub.injection.injectEventPluginOrder(DefaultEventPluginOrder);
  // 注入实例操作函数，用于事件传播
  EventPluginHub.injection.injectInstanceHandle(ReactInstanceHandles);

  // 默认包含两个重要事件插件（而不必要求他们）。
  // 注入事件插件
  EventPluginHub.injection.injectEventPluginsByName({
    'SimpleEventPlugin': SimpleEventPlugin,
    'EnterLeaveEventPlugin': EnterLeaveEventPlugin
  });

  // 用于兼容 IE8 创建的复合组件，后续讲到组件的时候再做解读，但不对 `ReactDOMForm.js` 进行解读
  ReactDOM.injection.injectComponentClasses({
    form: ReactDOMForm
  });
}
```

当然你需要知道，这个 `inject` 函数时需要在 `React.js` 中引入并执行的。

我们一步步往下看。先来看看 `DefaultEventPluginOrder.js` ：

```javascript
// eventPlugins/DefaultEventPluginOrder.js
var DefaultEventPluginOrder = [
  keyOf({ResponderEventPlugin: null}),
  keyOf({SimpleEventPlugin: null}),
  keyOf({TapEventPlugin: null}),
  keyOf({EnterLeaveEventPlugin: null}),
  keyOf({AnalyticsEventPlugin: null})
];
```

就是一个事件插件的默认顺序列表，使用 `EventPluginHub.injection.injectEventPluginOrder` 方法进行注入。

```javascript
// event/EventPluginHub.js
var injection = {
  EventPluginOrder: null,
  injectEventPluginOrder: function(InjectedEventPluginOrder) {
    injection.EventPluginOrder = InjectedEventPluginOrder;
    // 这个方法稍后再做解读，因为 injectEventPluginsByName 方法也使用到了他
    injection._recomputePluginsList();
  }
};
```

`EventPluginHub.injection.injectInstanceHandle(ReactInstanceHandles);` 也是差不多的操作，他操作的是 `EventPropagators.js` 下的 `InstanceHandle` 。 `ReactInstanceHandles` 在后续事件相关中都会使用到。我们先来看看 `injectEventPluginsByName` 方法：

```javascript
// event/EventPluginHub.js
var injection = {
  plugins: [],
  injectEventPluginsByName: function(pluginsByName) {
    // 这里讲传入插件与已存插件进行合并
    injection.pluginsByName = merge(injection.pluginsByName, pluginsByName);
    injection._recomputePluginsList();
  },
  pluginsByName: {},
  // 重新计算事件插件顺序列表
  _recomputePluginsList: function() {
    var injectPluginByName = function(name, PluginModule) {
      // 查找插件顺序列表，不存在则抛出错误
      var pluginIndex = injection.EventPluginOrder.indexOf(name);
      throwIf(pluginIndex === -1, ERRORS.DEPENDENCY_ERROR + name);
      // 对应的插件列表不存在的情况，则在对应索引赋值其对应的插件模块
      if (!injection.plugins[pluginIndex]) {
        injection.plugins[pluginIndex] = PluginModule;
        // 遍历其插件模块对应的抽象事件类型
        for (var eventName in PluginModule.abstractEventTypes) {
          var eventType = PluginModule.abstractEventTypes[eventName];
          // 记录所有的注册名称
          recordAllRegistrationNames(eventType, PluginModule);
        }
      }
    };
    // 事件插件的默认顺序列表存在情况
    if (injection.EventPluginOrder) {  // Else, do when plugin order injected
      var injectedPluginsByName = injection.pluginsByName;
      // 遍历注入的插件
      for (var name in injectedPluginsByName) {
        injectPluginByName(name, injectedPluginsByName[name]);
      }
    }
  }
};

var registrationNames = {};
var registrationNamesArr = [];

function recordAllRegistrationNames(eventType, PluginModule) {
  var phaseName;
  // 从 eventType 获得 phasedRegistrationNames
  var phasedRegistrationNames = eventType.phasedRegistrationNames;
  // 查看相关源码可以发现，这里适用于 `SimpleEventPlugin`
  if (phasedRegistrationNames) {
    // 遍历 phasedRegistrationNames
    for (phaseName in phasedRegistrationNames) {
      if (!phasedRegistrationNames.hasOwnProperty(phaseName)) {
        continue;
      }
      // 对应位置进行添加
      registrationNames[phasedRegistrationNames[phaseName]] = PluginModule;
      registrationNamesArr.push(phasedRegistrationNames[phaseName]);
    }
  } else if (eventType.registrationName) {
    // 若注册名称在的情况，查看相关源码可以发现，这里适用于 `EnterLeaveEventPlugin`
    registrationNames[eventType.registrationName] = PluginModule;
    registrationNamesArr.push(eventType.registrationName);
  }
  // 看到源码项目结构或者事件插件的默认顺序列表你都会发现，有着 `ResponderEventPlugin`
  // `TapEventPlugin`, `AnalyticsEventPlugin` 的存在。
}
```

比较抽象对吧，因为这里操作的都是事件插件的默认顺序列表中的对象，我们接下来看到 `SimpleEventPlugin` 和 `EnterLeaveEventPlugin` 来做相应的解释：

```javascript
// eventPlugins/SimpleEventPlugin.js
var SimpleEventPlugin = {
  abstractEventTypes: {
    // Note: We do not allow listening to mouseOver events. Instead, use the
    // onMouseEnter/onMouseLeave created by `EnterLeaveEventPlugin`.
    // 这里提到 onMouseEnter/onMouseLeave 请查看到 `EnterLeaveEventPlugin.js`
    mouseDown: {
      // 这是上面注册所有事件名称使用到的位置
      phasedRegistrationNames: {
        bubbled: keyOf({onMouseDown: true}),
        captured: keyOf({onMouseDownCapture: true})
      }
    },
    // ...
  }
};
```

```javascript
// eventPlugins/EnterLeaveEventPlugin.js
var abstractEventTypes = {
  mouseEnter: {registrationName: keyOf({onMouseEnter: null})},
  mouseLeave: {registrationName: keyOf({onMouseLeave: null})}
};
```

那么从中获取相应的对象如 `phasedRegistrationNames` 或是对应的标识 `registrationName`。

## 注册顶级事件

我们现在来看到 `ReactMount.js` 中 `renderComponent` 方法中的 `prepareTopLevelEvents` 函数：

```javascript
// core/ReactMount.js
var ReactMount = {
  useTouchEvents: false,
  prepareTopLevelEvents: function(TopLevelCallbackCreator) {
    ReactEvent.ensureListening(
      // 用于控制是否使用 TouchEvents ， 默认为 false
      ReactMount.useTouchEvents,
      TopLevelCallbackCreator
    );
  }
};
```

但是你可以调用 `React.initializeTouchEvents(true);` 来提前开启他，当然这是你需要使用到他的时候。

```javascript
// core/ReactEvent.js
var _isListening = false;

function ensureListening(touchNotMouse, TopLevelCallbackCreator) {
  if (!_isListening) {
    // 赋值顶级回调创建者
    ReactEvent.TopLevelCallbackCreator = TopLevelCallbackCreator;
    listenAtTopLevel(touchNotMouse);
    // 一个私有变量，用于控制 ensureListening 方法只执行一次
    _isListening = true;
  }
}

function listenAtTopLevel(touchNotMouse) {
  var mountAt = document;

  registerDocumentScrollListener();
  registerDocumentResizeListener();
  trapBubbledEvent(topLevelTypes.topMouseOver, 'mouseover', mountAt);
  // ...
  // TouchEvents 相关
  if (touchNotMouse) {
    trapBubbledEvent(topLevelTypes.topTouchStart, 'touchstart', mountAt);
    // ...
  }
  // ... 以及一些兼容相关，涉及使用到 `isEventSupported` 方法
}
```

```javascript
// event/EventConstants.js
var topLevelTypes = keyMirror({
  topBlur: null,
  // ...
});
```

### 捕获/冒泡事件监听

上面代码直接完成顶级事件的监听，我们顺势来看看 `trapBubbledEvent` ，与之相关的还有 `trapCapturedEvent` ，分别是冒泡和捕获阶段：

```javascript
// core/ReactEvent.js
function trapBubbledEvent(topLevelType, handlerBaseName, onWhat) {
  listen(
    onWhat,
    handlerBaseName,
    ReactEvent.TopLevelCallbackCreator.createTopLevelCallback(topLevelType)
  );
}

function trapCapturedEvent(topLevelType, handlerBaseName, onWhat) {
  capture(
    onWhat,
    handlerBaseName,
    ReactEvent.TopLevelCallbackCreator.createTopLevelCallback(topLevelType)
  );
}
```

其实从这里你就可以发现，不管是冒泡或者是捕获，我在 `document` 或者 `window` 这样的顶级对象上，都会先被捕获到，至于最后想到哪个方法或者函数，那么是由事件传播器控制，查找对应的回调注册表来完成功能。

```javascript
// event/NormalizedEventListener.js
function createNormalizedCallback(cb) {
  return function(unfixedNativeEvent) {
    cb(normalizeEvent(unfixedNativeEvent));
  };
}

var NormalizedEventListener = {
  listen: function(el, handlerBaseName, cb) {
    EventListener.listen(el, handlerBaseName, createNormalizedCallback(cb));
  },
  capture: function(el, handlerBaseName, cb) {
    EventListener.capture(el, handlerBaseName, createNormalizedCallback(cb));
  }
};
```

`normalizeEvent` 函数是一个补丁，这里不做解读。你可以直接忽略理解为 `cb(unfixedNativeEvent);` 。这里调用的又是另一个模块 `EventListener` ， `EventListener` 则的对原生浏览器 API 进行了封装。

```javascript
// vendor/stubs/EventListener.js
var EventListener = {
  listen: function(el, handlerBaseName, cb) {
    if (el.addEventListener) {
      el.addEventListener(handlerBaseName, cb, false);
    } else if (el.attachEvent) {
      el.attachEvent('on' + handlerBaseName, cb);
    }
  },
  capture: function(el, handlerBaseName, cb) {
    if (!el.addEventListener) {
      // ...
      return;
    } else {
      el.addEventListener(handlerBaseName, cb, true);
    }
  }
};
```

## React 顶级事件回调方法

那么由此可以看出， `trapBubbledEvent` 在操作的其实就是 `addEventListener` API，唯独还没有解读的是 `ReactEvent.TopLevelCallbackCreator.createTopLevelCallback(topLevelType)` ，其实你会发现，从刚刚开始我都没有解读 `TopLevelCallbackCreator` ，那么我们现在就来聊聊：

```javascript
// core/ReactEventTopLevelCallback.js
var ReactEventTopLevelCallback = {
  // 这里的 `topLevelType` 的值已经在 `listenAtTopLevel` 函数执行阶段就完成了赋值
  // 当 `addEventListener` 执行后，返回的是 `unfixedNativeEvent` 参数
  // 经过由 `normalizeEvent` 补丁处理后，返回 `fixedNativeEvent`
  createTopLevelCallback: function(topLevelType) {
    return function(fixedNativeEvent) {
      // 这个在之前也有提到过，当执行 React 调度事务时，将控制事件
      if (!_topLevelListenersEnabled) {
        return;
      }
      // 获得第一个 React DOM
      var renderedTarget = ReactInstanceHandles.getFirstReactDOM(
        fixedNativeEvent.target
      ) || ExecutionEnvironment.global;
      // 获得对应的 ID
      var renderedTargetID = getDOMNodeID(renderedTarget);
      var event = fixedNativeEvent;
      var target = renderedTarget;
      // 例如： ReactEvent.handleTopLevel('topClick', event, '.reactRoot[0]', target)
      ReactEvent.handleTopLevel(topLevelType, event, renderedTargetID, target);
    };
  }
};
```

## 创建抽象事件及执行至销毁

```javascript
// core/ReactEvent.js
function handleTopLevel(
    topLevelType,
    nativeEvent,
    renderedTargetID,
    renderedTarget) {
  // 提取抽象事件，返回 AbstractEvents 实例
  // 这个抽象事件可能是 AbstractEvents 或者可能是 AbstractEvents[]
  var abstractEvents = EventPluginHub.extractAbstractEvents(
    topLevelType,
    nativeEvent,
    renderedTargetID,
    renderedTarget
  );

  // The event queue being processed in the same cycle allows preventDefault.
  // 加入抽象事件队列
  EventPluginHub.enqueueAbstractEvents(abstractEvents);
  EventPluginHub.processAbstractEventQueue();
}
```

```javascript
// event/EventPluginHub.js
var extractAbstractEvents = function(topLevelType, nativeEvent, renderedTargetID, renderedTarget) {
  var abstractEvents;
  // 重新计算的 plugins 列表，就是执行 _recomputePluginsList 函数后的结果
  var plugins = injection.plugins;
  var len = plugins.length;
  // 遍历插件列表
  for (var i = 0; i < len; i++) {
    // Not every plugin in the ordering may be loaded at runtime.
    var possiblePlugin = plugins[i];
    // possiblePlugin 存在的情况下执行
    // 这里的 extractAbstractEvents 方法是 possiblePlugin 的
    // 比如 SimpleEventPlugin.extractAbstractEvents 方法
    // 对 SimpleEventPlugin.extractAbstractEvents 或者是 EnterLeaveEventPlugin.extractAbstractEvents
    // 这里不做具体解读，但是需要提到的是，进过函数返回的内容是 AbstractEvent 的实例，他使用 PooledClass 注入过
    // 可以直接调用 getPooled 及 release 方法
    var extractedAbstractEvents =
      possiblePlugin &&
      possiblePlugin.extractAbstractEvents(
        topLevelType,
        nativeEvent,
        renderedTargetID,
        renderedTarget
      );
    // 若最终这个返回值存在
    if (extractedAbstractEvents) {
      // 合并 abstractEvents 和 extractedAbstractEvents
      abstractEvents = accumulate(abstractEvents, extractedAbstractEvents);
    }
  }
  return abstractEvents;
};

var abstractEventQueue = [];

var enqueueAbstractEvents = function(abstractEvents) {
  if (abstractEvents) {
    // 对这个抽象事件队列进行合并
    abstractEventQueue = accumulate(abstractEventQueue, abstractEvents);
  }
};

var processAbstractEventQueue = function() {
  var processingAbstractEventQueue = abstractEventQueue;
  // 释放内存
  abstractEventQueue = null;
  // 若 processingAbstractEventQueue 是数组则遍历执行 executeDispatchesAndRelease 函数
  // 反之 executeDispatchesAndRelease(processingAbstractEventQueue)
  forEachAccumulated(processingAbstractEventQueue, executeDispatchesAndRelease);
};

var executeDispatchesAndRelease = function(abstractEvent) {
  if (abstractEvent) {
    // 获得对应的注册事件名称
    var PluginModule = getPluginModuleForAbstractEvent(abstractEvent);
    // 比如获得了 SimpleEventPlugin.executeDispatch ，用于实现与默认实现相同，但在返回值为 false 时取消事件除外的功能
    var pluginExecuteDispatch = PluginModule && PluginModule.executeDispatch;
    EventPluginUtils.executeDispatchesInOrder(
      abstractEvent,
      pluginExecuteDispatch || EventPluginUtils.executeDispatch
    );
    // 销毁抽象事件
    AbstractEvent.release(abstractEvent);
  }
};

function getPluginModuleForAbstractEvent(abstractEvent) {
  if (abstractEvent.type.registrationName) {
    return registrationNames[abstractEvent.type.registrationName];
  } else {
    for (var phase in abstractEvent.type.phasedRegistrationNames) {
      if (!abstractEvent.type.phasedRegistrationNames.hasOwnProperty(phase)) {
        continue;
      }
      var PluginModule = registrationNames[
        abstractEvent.type.phasedRegistrationNames[phase]
      ];
      if (PluginModule) {
        return PluginModule;
      }
    }
  }
  return null;
}
```

### 实例化抽象事件

这里将对 `SimpleEventPlugin.extractAbstractEvents` 或者是 `EnterLeaveEventPlugin.extractAbstractEvents` 进行解读。

```javascript
// eventPlugins/SimpleEventPlugin.js
var SimpleEventPlugin = {
  // 'topClick', event, '.reactRoot[0]', target
  extractAbstractEvents: function(topLevelType, nativeEvent, renderedTargetID, renderedTarget) {
    var data;
    // SimpleEventPlugin.abstractEventTypes 的映射
    var abstractEventType =
      SimpleEventPlugin.topLevelTypesToAbstract[topLevelType];
    if (!abstractEventType) {
      return null;
    }
    switch(topLevelType) {
      // ...
    }
    // 实例化 AbstractEvent
    var abstractEvent = AbstractEvent.getPooled(
      abstractEventType,
      renderedTargetID,
      topLevelType,
      nativeEvent,
      data
    );
    // 这里是个关键
    // 对捕获和冒泡分别进行派发这个 AbstractEvent 实例
    EventPropagators.accumulateTwoPhaseDispatches(abstractEvent);
    return abstractEvent;
  }
};

SimpleEventPlugin.topLevelTypesToAbstract = {
  topMouseDown:   SimpleEventPlugin.abstractEventTypes.mouseDown,
  // ...
};
```

在 `EnterLeaveEventPlugin.js` 中， `extractAbstractEvents` 方法返回了 `[leave, enter]` 的 AbstractEvent 实例化，并且调用 `EventPropagators.accumulateEnterLeaveDispatches(leave, enter, fromID, toID);` 进行派发。

`EventPropagators.js` 相关的方法稍后做解读。

那么在之前调用 `executeDispatchesAndRelease` 函数时，执行相应函数涉及到的函数如下：

```javascript
// event/EventPluginUtils.js
function executeDispatch(abstractEvent, listener, domID) {
  // forEachEventDispatch 函数中对应的 cb(abstractEvent, dispatchListeners, dispatchIDs)
  listener(abstractEvent, domID);
}

function executeDispatchesInOrder(abstractEvent, executeDispatch) {
  forEachEventDispatch(abstractEvent, executeDispatch);
  abstractEvent._dispatchListeners = null;
  abstractEvent._dispatchIDs = null;
}

function forEachEventDispatch(abstractEvent, cb) {
  // 在初始化的时候，这两个内容都是被置为 null 的
  // 他们都在 EventPropagators 被赋值
  // 若存在的情况下，这就是你在元素上定义的事件回调及对应的 ID
  var dispatchListeners = abstractEvent._dispatchListeners;
  var dispatchIDs = abstractEvent._dispatchIDs;
  if (Array.isArray(dispatchListeners)) {
    var i;
    for (
      i = 0;
      i < dispatchListeners.length && !abstractEvent.isPropagationStopped;
      i++) {
      // Listeners and IDs are two parallel arrays that are always in sync.
      cb(abstractEvent, dispatchListeners[i], dispatchIDs[i]);
    }
  } else if (dispatchListeners) {
    cb(abstractEvent, dispatchListeners, dispatchIDs);
  }
}
```

### 抽象事件 AbstractEvent

大家对 `AbstractEvent` 的属性有些疑问，因为刚刚在提到的时候并没有提到 `AbstractEvent` 的实现。那么接下来解释下，这样能把全部都串联起来：

```javascript
// event/AbstractEvent.js
var MAX_POOL_SIZE = 20;

function AbstractEvent(
    abstractEventType,
    abstractTargetID,  // Allows the abstract target to differ from native.
    originatingTopLevelEventType,
    nativeEvent,
    data) {
  // SimpleEventPlugin.abstractEventTypes.click, .reactRoot[0]', 'topClick', event, data
  this.type = abstractEventType;
  this.abstractTargetID = abstractTargetID || '';
  this.originatingTopLevelEventType = originatingTopLevelEventType;
  this.nativeEvent = nativeEvent;
  this.data = data;
  this.target = nativeEvent && nativeEvent.target;
  this._dispatchListeners = null;
  this._dispatchIDs = null;
  this.isPropagationStopped = false;
}

AbstractEvent.poolSize = MAX_POOL_SIZE;

AbstractEvent.prototype.destructor = function() {
  this.target = null;
  this._dispatchListeners = null;
  this._dispatchIDs = null;
};

PooledClass.addPoolingTo(AbstractEvent, PooledClass.fiveArgumentPooler);
```

### 模拟捕获/冒泡（1）

现在我们来看到刚刚忽略掉的 `EventPropagators.accumulateTwoPhaseDispatches` 和 `EventPropagators.accumulateEnterLeaveDispatches` 方法：

```javascript
// event/EventPropagators.js
function accumulateTwoPhaseDispatches(abstractEvents) {
  forEachAccumulated(abstractEvents, accumulateTwoPhaseDispatchesSingle);
}

function accumulateTwoPhaseDispatchesSingle(abstractEvent) {
  if (abstractEvent && abstractEvent.type.phasedRegistrationNames) {
    // 获得 phasedRegistrationNames 对象
    injection.InstanceHandle.traverseTwoPhase(
      abstractEvent.abstractTargetID,
      accumulateDirectionalDispatches,
      abstractEvent
    );
  }
}

function accumulateEnterLeaveDispatches(leave, enter, fromID, toID) {
  injection.InstanceHandle.traverseEnterLeave(
    fromID,
    toID,
    accumulateDispatches,
    leave,
    enter
  );
}

function listenerAtPhase(id, abstractEvent, propagationPhase) {
  // 获取对应的注册名称
  var registrationName =
    abstractEvent.type.phasedRegistrationNames[propagationPhase];
  // 这个 getListener 函数需要看到 CallbackRegistry.js 相关的方法
  return getListener(id, registrationName);
}

function accumulateDirectionalDispatches(domID, upwards, abstractEvent) {
  // domID = 当前 traverseParentPath 的 ID
  // upwards = true / false
  var phase = upwards ? PropagationPhases.bubbled : PropagationPhases.captured;
  // 获得对应的 cb
  var listener = listenerAtPhase(domID, abstractEvent, phase);
  if (listener) {
    // 如果回调存在的情况下对 _dispatchListeners 和 _dispatchIDs 进行合并
    // 本身不存在的情况下，返回新的 listener 和 domID
    // 存在的情况下返回 listener[] 和 domID[]
    abstractEvent._dispatchListeners =
      accumulate(abstractEvent._dispatchListeners, listener);
    abstractEvent._dispatchIDs = accumulate(abstractEvent._dispatchIDs, domID);
  }
}

// 也是同样
function accumulateDispatches(id, ignoredDirection, abstractEvent) {
  if (abstractEvent && abstractEvent.type.registrationName) {
    var listener = getListener(id, abstractEvent.type.registrationName);
    if (listener) {
      abstractEvent._dispatchListeners =
        accumulate(abstractEvent._dispatchListeners, listener);
      abstractEvent._dispatchIDs = accumulate(abstractEvent._dispatchIDs, id);
    }
  }
}
```

### 存储事件回调

```javascript
// event/CallbackRegistry.js
// 回调监听银行，用于存储各种 cb
var listenerBank = {};

var CallbackRegistry = {
  // 加入 cb
  putListener: function(id, registrationName, listener) {
    var bankForRegistrationName =
      listenerBank[registrationName] || (listenerBank[registrationName] = {});
    bankForRegistrationName[id] = listener;
  },
  // 获得 cb
  getListener: function(id, registrationName) {
    var bankForRegistrationName = listenerBank[registrationName];
    return bankForRegistrationName && bankForRegistrationName[id];
  },
  // 移除 cb
  deleteListener: function(id, registrationName) {
    var bankForRegistrationName = listenerBank[registrationName];
    if (bankForRegistrationName) {
      delete bankForRegistrationName[id];
    }
  },
  // 清空所有
  __purge: function() {
    listenerBank = {};
  }
};
```

### 模拟捕获/冒泡（2）

这里所调用的 `injection.InstanceHandle` 下的方法其实就是 `ReactInstanceHandles` 的方法，回忆到在默认注入的时候，那么这里来看下：

```javascript
// core/ReactInstanceHandles.js
var ReactInstanceHandles = {
  traverseTwoPhase: function(targetID, cb, arg) {
    if (targetID) {
      // 直接看到 traverseParentPath 函数
      // 比如:
      // .reactRoot[0].[3].0
      // 捕获阶段： .reactRoot[0] => .reactRoot[0].[3] => .reactRoot[0].[3].0
      // 冒泡阶段： .reactRoot[0].[3].0 => .reactRoot[0].[3] => .reactRoot[0]
      // 他里面是通过前后 '.' 来控制判断的
      // 具体就是那行 for 循环代码以及相关的操作函数
      traverseParentPath('', targetID, cb, arg, true, false);
      traverseParentPath(targetID, '', cb, arg, false, true);
    }
  },
  traverseEnterLeave: function(leaveID, enterID, cb, upArg, downArg) {
    // 获取两个ID最近的共同祖先ID
    var longestCommonID = ReactInstanceHandles.getFirstCommonAncestorID(
      leaveID,
      enterID
    );
    if (longestCommonID !== leaveID) {
      traverseParentPath(leaveID, longestCommonID, cb, upArg, false, true);
    }
    if (longestCommonID !== enterID) {
      traverseParentPath(longestCommonID, enterID, cb, downArg, true, false);
    }
  },
};

// 在两个ID之间遍历父路径（向上或向下）。ID不能相同，并且它们之间必须存在父路径。
// 方法的作用因为比较抽象，我会举个例子在上面。
function traverseParentPath(start, stop, cb, arg, skipFirst, skipLast) {
  start = start || '';
  stop = stop || '';
  invariant(
    start !== stop,
    'traverseParentPath(...): Cannot traverse from and to the same ID, `%s`.',
    start
  );
  var ancestorID = ReactInstanceHandles.getFirstCommonAncestorID(start, stop);
  var traverseUp = ancestorID === stop;
  invariant(
    traverseUp || ancestorID === start,
    'traverseParentPath(%s, %s, ...): Cannot traverse from two IDs that do ' +
    'not have a parent path.',
    start,
    stop
  );
  // Traverse from `start` to `stop` one depth at a time.
  var depth = 0;
  // 通过 traverseUp 来控制是使用 parentID 函数还是 nextDescendantID 方法。
  // 就是从前往后取或者是从后往前取
  var traverse = traverseUp ? parentID : ReactInstanceHandles.nextDescendantID;
  for (var id = start; /* until break */; id = traverse(id, stop)) {
    if ((!skipFirst || id !== start) && (!skipLast || id !== stop)) {
      cb(id, traverseUp, arg);
    }
    if (id === stop) {
      // Only break //after// visiting `stop`.
      break;
    }
    invariant(
      depth++ < MAX_TREE_DEPTH,
      'traverseParentPath(%s, %s, ...): Detected an infinite loop while ' +
      'traversing the React DOM ID tree. This may be due to malformed IDs: %s',
      start, stop
    );
  }
}
```

最后呢就是在 `ReactNativeComponent.js` 中 `_createOpenTagMarkup` 方法中的 `putListener` 函数，看过 `putListener` 函数就是用于存入回调函数的，在 `getListener` 获得则会去执行。

那么到此为止，实现事件机制。
