---
title: React 源码学习（一）：HTML 元素渲染
date: 2019-04-01 21:10:31
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

## React.DOM.*

直接步入正题，在官方的例子中可以看到 `render` 函数会返回类似这样一段代码：

```javascript
// #examples
return React.DOM.h1(null, 'Zong is learning the source code of React.')
```

使用 `type="text/jsx"` 的形式编写：

```jsx
/** @jsx React.DOM */
// #examples
return <h1>Zong is learning the source code of React.</h1>
```

`JSXTransformer.js` 会将 `type="text/jsx"` 的形式转换成 `React.DOM.h1` 的函数形式。

结合 `React.renderComponent` 将 `<h1>` 标签最终渲染在指定的元素下，这段代码最终渲染至 DOM 下可能如下：

```html
<h1 id=".reactRoot[0]">Zong is learning the source code of React.</h1>
```

## 如何实现 HTML 元素的渲染

那么， React 是如何实现 HTML 元素的渲染呢？

### 工厂函数 objMapKeyVal.js

objMapKeyVal 是个工厂函数，他最终会返回一个“键”与 `obj` 对应的对象“值”则是 `func` 的执行结果。

```javascript
// utils/objMapKeyVal.js
function objMapKeyVal(obj, func, context) {
  if (!obj) {
    return null;
  }
  var i = 0;
  var ret = {};
  for (var key in obj) {
    if (obj.hasOwnProperty(key)) {
      ret[key] = func.call(context, key, obj[key], i++);
    }
  }
  return ret;
}
```

### 创建 React.DOM.* 方法

```javascript
// core/ReactDOM.js
/**
 * 用于创建 DOM 组件的类，其原型连接 ReactNativeComponent
 */
function createDOMComponentClass(tag, omitClose) {
  var Constructor = function(initialProps, children) {
    this.construct(initialProps, children);
  };

  Constructor.prototype = new ReactNativeComponent(tag, omitClose);
  Constructor.prototype.constructor = Constructor;

  return function(props, children) {
    return new Constructor(props, children);
  };
}

var ReactDOM = objMapKeyVal({
  // ...
  // Danger: this gets monkeypatched! See ReactDOMForm for more info.
  form: false,
  img: true,
  // ...
}, createDOMComponentClass);
```

`ReactDOM` 对象中的“键”并非包含目前所有的 HTML 元素，若你需要加入新的 HTML 元素，在此对象中添加即可。但，若你需要同时支持 `type="text/jsx"` 的编写形式，则同时需要在 `JSXTransformer.js` 进行添加，以实现转换。

在经过 `objMapKeyVal` 工厂函数的执行后， `ReactDOM` 得到的值已经并非是之前的布尔值，键值以 `tag, omitClose` 的形式作为参数供 `ReactNativeComponent` 进行实例化了。返回的则是接受 `props, children` 参数的函数，用于实例化这个 `Constructor` 。

所以例子中是以 `props = null, children = 'Zong is learning the source code of React.'` 的形式来接受的参数，并实例化 `Constructor` 。

### 原型 React 原生组件

当然，这里同样需要提到，`ReactNativeComponent` 实例化时操作了什么：

```javascript
// core/ReactNativeComponent.js
function ReactNativeComponent(tag, omitClose) {
  this._tagOpen = '<' + tag + ' ';
  this._tagClose = omitClose ? '' : '</' + tag + '>';
  this.tagName = tag.toUpperCase();
}
```

比如，键值对 `form: false` 返回的对象为：

```json
ReactNativeComponent {
  "_tagOpen": "<form ",
  "_tagClose": "</form>",
  "tagName": "FORM",
}
```

### 工具函数 - 混合

就这么点东西吗？当然不是，你需要注意到 `ReactNativeComponent.js` 末端的几行代码：

```javascript
// utils/mixInto.js
/**
 * Simply copies properties to the prototype.
 */
var mixInto = function(constructor, methodBag) {
  var methodName;
  for (methodName in methodBag) {
    if (!methodBag.hasOwnProperty(methodName)) {
      continue;
    }
    constructor.prototype[methodName] = methodBag[methodName];
  }
};
```

这里依次将 3 个对象混合到 `ReactNativeComponent.prototype` 。

```javascript
// core/ReactNativeComponent.js
mixInto(ReactNativeComponent, ReactComponent.Mixin);
mixInto(ReactNativeComponent, ReactNativeComponent.Mixin);
mixInto(ReactNativeComponent, ReactMultiChild.Mixin);
```

`mixInto` 方法的功能其实就是 `Object.assign` 的功能，上方的代码也可以这样写： `Object.assign(ReactNativeComponent.prototype, ReactComponent.Mixin)` 。

那么，接下来我们来看看 `this.construct(initialProps, children)` 这里到底做了什么，逆向寻找发现这个方法在 `ReactComponent.Mixin` 中。

```javascript
// core/ReactComponent.js
var ReactComponent = {
  Mixin: {
    construct: function(initialProps, children) {
      this.props = initialProps || {};
      if (typeof children !== 'undefined') {
        this.props.children = children;
      }
      // Record the component responsible for creating this component.
      this.props[OWNER] = ReactCurrentOwner.current;
      // All components start unmounted.
      this._lifeCycleState = ComponentLifeCycle.UNMOUNTED;
    },
  }
}
```

## 将 React 组件实例挂载至 DOM

到此为止，我们改如何将这个实例渲染到 DOM 上呢？我们来看到这段代码：

```javascript
// #examples
React.renderComponent(
  React.DOM.h1(null, 'Zong is learning the source code of React.'),
  document.getElementById('container')
);
```

### 注册 React 实例

是 `React.renderComponent` 方法将实例渲染到 DOM 上的，我们来看看这个方法究竟做了些什么，此次先忽略其他逻辑（组件更新/事件注册）：

```javascript
// core/ReactMount.js
// 用于统计公共挂载的数量
var globalMountPointCounter = 0;

/** Mapping from reactRoot DOM ID to React component instance. */
// React 组件实例基于 ReactRootID 的映射
var instanceByReactRootID = {};

/** Mapping from reactRoot DOM ID to `container` nodes. */
// container 基于 ReactRootID 的映射
var containersByReactRootID = {};

/**
 * @param {DOMElement} container DOM element that may contain a React component.
 * @return {?string} A "reactRoot" ID, if a React component is rendered.
 */
function getReactRootID(container) {
  return container.firstChild && container.firstChild.id;
}

var ReactMount = {
  renderComponent: function(nextComponent, container) {
    // 上面逻辑包含组件更新及事件注册
    // 获得/生成 reactRootID
    var reactRootID = ReactMount.registerContainer(container);
    // 映射 React 组件
    instanceByReactRootID[reactRootID] = nextComponent;
    // 调用组件自身方法
    nextComponent.mountComponentIntoNode(reactRootID, container);
    return nextComponent;
  },
  registerContainer: function(container) {
    // 获得 reactRootID
    var reactRootID = getReactRootID(container);
    if (reactRootID) {
      // 若存在的情况下确认 ID 是否为 "reactRoot" ID，否则返回 null
      // If one exists, make sure it is a valid "reactRoot" ID.
      reactRootID = ReactInstanceHandles.getReactRootIDFromNodeID(reactRootID);
    }
    if (!reactRootID) {
      // No valid "reactRoot" ID found, create one.
      // 若 ID 不存在，则返回新的 ID
      reactRootID = ReactInstanceHandles.getReactRootID(
        globalMountPointCounter++
      );
    }
    // 映射 container
    containersByReactRootID[reactRootID] = container;
    return reactRootID;
  },
}
```

```javascript
// core/ReactInstanceHandles.js
var ReactInstanceHandles = {
  getReactRootID: function(mountPointCount) {
    return '.reactRoot[' + mountPointCount + ']';
  },
  getReactRootIDFromNodeID: function(id) {
    var regexResult = /\.reactRoot\[[^\]]+\]/.exec(id);
    return regexResult && regexResult[0];
  },
}
```

### 将组件挂载至 DOM 节点方法

```javascript
// core/ReactComponent.js
var ReactComponent = {
  Mixin: {
    mountComponent: function(rootID, transaction) {
      // 组件生命周期和 ref 相关暂不做解读

      this._rootNodeID = rootID;
    },
    mountComponentIntoNode: function(rootID, container) {
      // 这里牵扯到 React 事务，后续再做解读
      var transaction = ReactComponent.ReactReconcileTransaction.getPooled();
      // 本次讨论，你只需简单理解为：
      // this._mountComponentIntoNode.call(this, rootID, container, transaction)
      transaction.perform(
        this._mountComponentIntoNode,
        this,
        rootID,
        container,
        transaction
      );
      ReactComponent.ReactReconcileTransaction.release(transaction);
    },
    _mountComponentIntoNode: function(rootID, container, transaction) {
      // 这里包含这一些时间计算，不做解读
      var renderStart = Date.now();
      // markup 即为返回的 HTML 标记
      // this.mountComponent 则为 ReactNativeComponent.Mixin.mountComponent
      var markup = this.mountComponent(rootID, transaction);
      ReactMount.totalInstantiationTime += (Date.now() - renderStart);

      var injectionStart = Date.now();
      // Asynchronously inject markup by ensuring that the container is not in
      // the document when settings its `innerHTML`.
      // 以下代码是用来判断 markup 需要被如何插入值 DOM 节点。
      var parent = container.parentNode;
      if (parent) {
        var next = container.nextSibling;
        parent.removeChild(container);
        container.innerHTML = markup;
        if (next) {
          parent.insertBefore(container, next);
        } else {
          parent.appendChild(container);
        }
      } else {
        container.innerHTML = markup;
      }
      ReactMount.totalInjectionTime += (Date.now() - injectionStart);
    },
  }
}
```

### 生成 Markup 标记

```javascript
// core/ReactNativeComponent.js
// For quickly matching children type, to test if can be treated as content.
var CONTENT_TYPES = {'string': true, 'number': true};

ReactNativeComponent.Mixin = {
  mountComponent: function(rootID, transaction) {
    ReactComponent.Mixin.mountComponent.call(this, rootID, transaction);
    // 参数校验不做解读
    assertValidProps(this.props);
    // 返回的就是 HTML 标记
    return (
      this._createOpenTagMarkup() +
      this._createContentMarkup(transaction) +
      this._tagClose
    );
  },
  _createOpenTagMarkup: function() {
    var ret = this._tagOpen;
    // 暂时不解读（事件注册/ CSS 样式/ DOM 属性）

    return ret + ' id="' + this._rootNodeID + '">';
  },
  _createContentMarkup: function(transaction) {
    // 这里忽略 dangerouslySetInnerHTML 的情况
    var contentToUse = this.props.content != null ? this.props.content :
      CONTENT_TYPES[typeof this.props.children] ? this.props.children : null;
    var childrenToUse = contentToUse != null ? null : this.props.children;
    if (contentToUse != null) {
      // content == null 并且 children 为 string / number 的情况
      // 直接返回 Zong is learning the source code of React.
      return escapeTextForBrowser(contentToUse);
    } else if (childrenToUse != null) {
      // 多个 children 的情况
      return this.mountMultiChild(
        flattenChildren(childrenToUse),
        transaction
      );
    }
    return '';
  },
}
```

```javascript
// utils/escapeTextForBrowser.js
var ESCAPE_LOOKUP = {
  "&": "&amp;",
  ">": "&gt;",
  "<": "&lt;",
  "\"": "&quot;",
  "'": "&#x27;",
  "/": "&#x2f;"
};

function escaper(match) {
  return ESCAPE_LOOKUP[match];
}

var escapeTextForBrowser = function (text) {
  var type = typeof text;
  var invalid = type === 'object';
  if (text === '' || invalid) {
    return '';
  } else {
    if (type === 'string') {
      return text.replace(/[&><"'\/]/g, escaper);
    } else {
      return (''+text).replace(/[&><"'\/]/g, escaper);
    }
  }
};
```

那么到此为止，实现 HTML 元素渲染功能。
