---
title: React 源码学习（二）：HTML 子元素渲染
date: 2019-04-02 23:03:50
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

## 生成 Markup 标记时如何生成多重子元素

接下来，我们来解读如何实现 HTML 子元素渲染：

```javascript
// core/ReactNativeComponent.js
// For quickly matching children type, to test if can be treated as content.
var CONTENT_TYPES = {'string': true, 'number': true};

ReactNativeComponent.Mixin = {
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

### 扁平化工具方法 flattenChildren

首先，我们来看看 `flattenChildren` 函数做了什么：

```javascript
// utils/flattenChildren.js
var ONLY_CHILD_NAME = '0';

var flattenChildrenImpl = function(res, children, nameSoFar) {
  // 如果 children 是个数组的话，遍历进行压扁
  // 数组的情况下，nameSoFar 会加上 [number]
  if (Array.isArray(children)) {
    for (var i = 0; i < children.length; i++) {
      flattenChildrenImpl(res, children[i], nameSoFar + '[' + i + ']');
    }
  } else {
    // 这里是非数组的情况分支
    var type = typeof children;
    var isOnlyChild = nameSoFar === '';
    var storageName = isOnlyChild ? ONLY_CHILD_NAME : nameSoFar;
    if (children === null || children === undefined || type === 'boolean') {
      res[storageName] = null;
      // 若 children 还有 mountComponentIntoNode 方法，则直接认定为组件
    } else if (children.mountComponentIntoNode) {
      /* We found a component instance */
      res[storageName] = children;
    } else {
      if (type === 'object') {
        // 若是对象的情况下，也是进行遍历压扁
        // 并且 nameSoFar 会加上 {key}
        throwIf(children && children.nodeType === 1, INVALID_CHILD);
        for (var key in children) {
          if (children.hasOwnProperty(key)) {
            flattenChildrenImpl(
              res,
              children[key],
              nameSoFar + '{' + escapeTextForBrowser(key) + '}'
            );
          }
        }
      // 若是 string 或 number 则返回 ReactTextComponent 的实例化对象
      } else if (type === 'string') {
        res[storageName] = new ReactTextComponent(children);
      } else if (type === 'number') {
        res[storageName] = new ReactTextComponent('' + children);
      }
    }
  }
};

function flattenChildren(children) {
  if (children === null || children === undefined) {
    return children;
  }
  // children 存在的情况下，对其进行压扁处理
  var result = {};
  flattenChildrenImpl(result, children, '');
  return result;
}
```

### 针对 String 或 Number 类型生成的文本组件

看到这里，我们先需要了解下，什么是 `ReactTextComponent` ：

这里的内容，我相信大家看的都很眼熟吧，这和 `ReactDOM.js` 中的 `Constructor` 一样，只不过这里不接受 `children` 参数，而是只接受一个 `text` 参数。

```javascript
// core/ReactTextComponent.js
var ReactTextComponent = function(initialText) {
  this.construct({text: initialText});
};

mixInto(ReactTextComponent, ReactComponent.Mixin);
mixInto(ReactTextComponent, {
  // 这个我相信大家也不陌生，在 React.renderComponent 时候，会被调用到。
  // 将 string 或 number 以 <span> 标签的形式进行返回
  mountComponent: function(rootID) {
    ReactComponent.Mixin.mountComponent.call(this, rootID);
    return (
      '<span id="' + rootID + '">' +
        escapeTextForBrowser(this.props.text) +
      '</span>'
    );
  }
});
```

### 多重子元素生成 Markup 标记

那么在压扁之后，我们调用的是 `ReactMultiChild.Mixin.mountMultiChild` 方法，至于为什么会调用到这里去，回忆 `ReactNativeComponent.js` 末端的几行代码，你就会想起为什么。那么接下来我们话不多说，来看到 `ReactMultiChild.js` 的相关源码：

```javascript
var ReactMultiChildMixin = {
  // 渲染子元素
  mountMultiChild: function(children, transaction) {
    var accum = '';
    var index = 0;
    for (var name in children) {
      var child = children[name];
      if (children.hasOwnProperty(name) && child) {
        // 依次渲染元素 markup 标记，并且拼接
        accum += child.mountComponent(
          this._rootNodeID + '.' + name,
          transaction
        );
        // 标记 DOM 索引，用于更新组件时使用。
        child._domIndex = index;
        index++;
      }
    }
    this._renderedChildren = children; // children are in just the right form!
    // 初始化 DOM 操作的数组，这个用于组件更新时。
    this.domOperations = null;
    return accum;
  }
};

var ReactMultiChild = {
  Mixin: ReactMultiChildMixin
};
```

那么到此，实现 HTML 子元素渲染。
