---
title: React 源码学习（六）：组件渲染
date: 2019-04-06 09:52:00
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

## React 组件渲染 React.createClass

终于要说到组件渲染了，其实组件渲染就是基于 HTML 元素渲染，我们在重新回到熟悉的例子，这是一个完整的例子：

```javascript
// #examples
var ExampleApplication = React.createClass({
  render: function() {
    var elapsed = Math.round(this.props.elapsed  / 100);
    var seconds = elapsed / 10 + (elapsed % 10 ? '' : '.0' );
    var message =
      'React has been successfully running for ' + seconds + ' seconds.';

    return React.DOM.p(null, message);
  }
});
var start = new Date().getTime();

React.renderComponent(
  ExampleApplication({elapsed: new Date().getTime() - start}),
  document.getElementById('container')
);
```

使用 `type="text/jsx"` 的形式编写：

```jsx
/** @jsx React.DOM */
// #examples
var ExampleApplication = React.createClass({
  render: function() {
    var elapsed = Math.round(this.props.elapsed  / 100);
    var seconds = elapsed / 10 + (elapsed % 10 ? '' : '.0' );
    var message =
      'React has been successfully running for ' + seconds + ' seconds.';

    return <p>{message}</p>;
  }
});
var start = new Date().getTime();

React.renderComponent(
  <ExampleApplication elapsed={new Date().getTime() - start} />,
  document.getElementById('container')
);
```

### 复合组件

在这里，我们看到创建组件的方法是 `React.createClass` ，组件就是一堆 HTML 元素的集合，但是组件具有状态 (`state`) 和属性 (`props`) ，还具有生命周期，并且组件可以更新。所以我们会一一将其到来，那么本次我们仅讨论组件渲染。我们看到复合组件

```javascript
// core/ReactCompositeComponent.js
var ReactCompositeComponent = {
  createClass: function(spec) {
    var Constructor = function(initialProps, children) {
      this.construct(initialProps, children);
    };
    Constructor.prototype = new ReactCompositeComponentBase();
    Constructor.prototype.constructor = Constructor;
    mixSpecIntoComponent(Constructor, spec);
    invariant(
      Constructor.prototype.render,
      'createClass(...): Class specification must implement a `render` method.'
    );

    var ConvenienceConstructor = function(props, children) {
      return new Constructor(props, children);
    };
    ConvenienceConstructor.componentConstructor = Constructor;
    ConvenienceConstructor.originalSpec = spec;
    return ConvenienceConstructor;
  },
};
```

看到 `React.createClass` 是不是发现和 `ReactDOM.js` 下的 `createDOMComponentClass` 函数很相似。确实如此，在你看懂了 HTML 元素渲染的实现后来看这个确实会容易很多，那么接受参数 `spec` 是什么呢？是 `specification` 规范，按照官方注释翻译为：在给定类规范的情况下创建复合组件类。我们从上面代码也可以看出 `render` 方法是必不可少的。他的原型映射到 `ReactCompositeComponentBase` ，那么让我们来看看他。

```javascript
// core/ReactCompositeComponent.js
var ReactCompositeComponentBase = function() {};
mixInto(ReactCompositeComponentBase, ReactComponent.Mixin);
mixInto(ReactCompositeComponentBase, ReactOwner.Mixin);
mixInto(ReactCompositeComponentBase, ReactPropTransferer.Mixin);
mixInto(ReactCompositeComponentBase, ReactCompositeComponentMixin);
```

### 渲染至 DOM 节点方法 mountComponent

熟悉吧， React 这种写法，原型链上的属性疯狂合并，并且原型链上的属性互相被连续调用。当然在方法中也有这样一行代码 `mixSpecIntoComponent(Constructor, spec);` ，他在 `mixInto` 的基础上做了更多的处理，但是他与 `mixInto` 的功能效果是一样的。那么接下来，先让我们看到当执行 `React.renderComponent` 方法后，复合组件会做出什么样的反应。我们看到代码：

```javascript
// core/ReactCompositeComponent.js
var ReactCompositeComponentMixin = {
  construct: function(initialProps, children) {
    ReactComponent.Mixin.construct.call(this, initialProps, children);
    this.state = null;
    this._pendingState = null;
    this._compositeLifeCycleState = null;
  },
  mountComponent: function(rootID, transaction) {
    // 这里与 ReactNativeComponent.Mixin.mountComponent 一致
    // 都执行了此方法，剩下的区别只是针对 HTML 组件和复合组件的区别
    // HTML 组件只需产出 markup 标记，添加 CSS 样式， DOM 属性，事件监听
    // 复合组件却不一样，上面已经提到过。
    ReactComponent.Mixin.mountComponent.call(this, rootID, transaction);

    // 组件的生命周期和复合组件的生命周期
    // Unset `this._lifeCycleState` until after this method is finished.
    this._lifeCycleState = ReactComponent.LifeCycle.UNMOUNTED;
    this._compositeLifeCycleState = CompositeLifeCycle.MOUNTING;

    // 参数校验
    if (this.constructor.propDeclarations) {
      this._assertValidProps(this.props);
    }

    // 绑定 this 执行上下文有关，与 mixSpecIntoComponent 有关联
    if (this.__reactAutoBindMap) {
      this._bindAutoBindMethods();
    }

    // 组件状态相关
    this.state = this.getInitialState ? this.getInitialState() : null;
    this._pendingState = null;

    // 生命周期相关
    if (this.componentWillMount) {
      this.componentWillMount();
      // When mounting, calls to `setState` by `componentWillMount` will set
      // `this._pendingState` without triggering a re-render.
      if (this._pendingState) {
        this.state = this._pendingState;
        this._pendingState = null;
      }
    }

    // 生命周期相关
    if (this.componentDidMount) {
      // 这里需要特别提一下，你可以看到，这是 React 调度事务，并且 ReactOnDOMReady 在调度事务 close 时
      // 会执行队列中的事务，所以你可以理解组件被挂载后这个生命周期是如何实现的。
      transaction.getReactOnDOMReady().enqueue(this, this.componentDidMount);
    }

    // 调用 _renderValidatedComponent 方法
    // 返回 ReactNativeComponent
    this._renderedComponent = this._renderValidatedComponent();

    // 生命周期相关
    // Done with mounting, `setState` will now trigger UI changes.
    this._compositeLifeCycleState = null;
    this._lifeCycleState = ReactComponent.LifeCycle.MOUNTED;

    // 调用 ReactNativeComponent.mountComponent 生成 markup 标记
    return this._renderedComponent.mountComponent(rootID, transaction);
  },
  _renderValidatedComponent: function() {
    // 这一步使用来控制正确的 Owner
    ReactCurrentOwner.current = this;
    // 调用 render 方法的返回
    var renderedComponent = this.render();
    // 这一步使用来控制正确的 Owner
    // 在 ReactNativeComponent 实例化时 ReactCurrentOwner.current 会被记录
    ReactCurrentOwner.current = null;
    invariant(
      ReactComponent.isValidComponent(renderedComponent),
      '%s.render(): A valid ReactComponent must be returned.',
      this.constructor.displayName || 'ReactCompositeComponent'
    );
    return renderedComponent;
  },
};
```

那么组件渲染讲的也差不多了，我们对其他一些，也先在这边提起来。回到这段代码：

```javascript
// core/ReactCompositeComponent.js
var ReactCompositeComponentBase = function() {};
// 一些 DOM 操作， React 调度事务相关，并包括生命周期
// ReactNativeComponent 也同样含有
mixInto(ReactCompositeComponentBase, ReactComponent.Mixin);
// 用于操作 ref ，相关代码随行在 ReactComponent 中，这里也不做详细解读
mixInto(ReactCompositeComponentBase, ReactOwner.Mixin);
// props 传输功能
mixInto(ReactCompositeComponentBase, ReactPropTransferer.Mixin);
mixInto(ReactCompositeComponentBase, ReactCompositeComponentMixin);
```

### React.autoBind 处理

简单介绍了这些后我们来看看， `mixSpecIntoComponent` 方法，既然看到这个方法，我们也要同样提到 `React.autoBind` 方法。本次我们仅讨论绑定上下文，不讨论其他相关：

```javascript
// core/ReactCompositeComponent.js
var ReactCompositeComponent = {
  // 使用 autoBind 的时候返回 unbound 函数
  // 将需要被绑定上下文的方法赋值至 __reactAutoBind
  autoBind: function(method) {
    function unbound() {
      invariant(
        false,
        'React.autoBind(...): Attempted to invoke an auto-bound method that ' +
        'was not correctly defined on the class specification.'
      );
    }
    unbound.__reactAutoBind = method;
    return unbound;
  }
};

function mixSpecIntoComponent(Constructor, spec) {
  var proto = Constructor.prototype;
  for (var name in spec) {
    if (!spec.hasOwnProperty(name)) {
      continue;
    }
    var property = spec[name];

    // 目前仅保留了与 autoBind 相关代码
    // 合并 specification 时，检查是否存在 __reactAutoBind 属性
    // 并加入 __reactAutoBindMap
    if (property && property.__reactAutoBind) {
      if (!proto.__reactAutoBindMap) {
        proto.__reactAutoBindMap = {};
      }
      proto.__reactAutoBindMap[name] = property.__reactAutoBind;
    } else {
      // 其他属性正常合并
      proto[name] = property;
    }
  }
}



var ReactCompositeComponentMixin = {
  mountComponent: function(rootID, transaction) {
    //...
    // 挂载组件时检查 __reactAutoBindMap 存在则执行 _bindAutoBindMethods 方法
    if (this.__reactAutoBindMap) {
      this._bindAutoBindMethods();
    }
    //...
  },
  _bindAutoBindMethods: function() {
    // 遍历 __reactAutoBindMap
    for (var autoBindKey in this.__reactAutoBindMap) {
      if (!this.__reactAutoBindMap.hasOwnProperty(autoBindKey)) {
        continue;
      }
      // 获取对应的方法
      var method = this.__reactAutoBindMap[autoBindKey];
      // 重新返回被绑定过后的方法
      this[autoBindKey] = this._bindAutoBindMethod(method);
    }
  },
  _bindAutoBindMethod: function(method) {
    // 获取到当前的组件上下文作用域
    var component = this;
    var hasWarned = false;
    function autoBound(a, b, c, d, e, tooMany) {
      invariant(
        typeof tooMany === 'undefined',
        'React.autoBind(...): Methods can only take a maximum of 5 arguments.'
      );
      // 判断组件声明周期为已挂载进行绑定
      if (component._lifeCycleState === ReactComponent.LifeCycle.MOUNTED) {
        return method.call(component, a, b, c, d, e);
      } else if (!hasWarned) {
        hasWarned = true;
        if (__DEV__) {
          console.warn(
            'React.autoBind(...): Attempted to invoke an auto-bound method ' +
            'on an unmounted instance of `%s`. You either have a memory leak ' +
            'or an event handler that is being run after unmounting.',
            component.constructor.displayName || 'ReactCompositeComponent'
          );
        }
      }
    }
    return autoBound;
  }
};
```

## 卸载组件

那么到这里关于绑定组件上下文的内容也解读完了，既然有组件渲染（挂载），那么就有组件（移除）卸载，这里我们主要看到 `React.unmountAndReleaseReactRootNode` 方法：

```javascript
// core/ReactMount.js
var ReactMount = {
  unmountAndReleaseReactRootNode: function(container) {
    // 意思很简单，获取 container 对应的 reactRootID
    // 从映射中找到对应的 component
    var reactRootID = getReactRootID(container);
    var component = instanceByReactRootID[reactRootID];
    // TODO: Consider throwing if no `component` was found.
    // 执行卸载方法
    component.unmountComponentFromNode(container);
    // 清除对应映射缓存
    delete instanceByReactRootID[reactRootID];
    delete containersByReactRootID[reactRootID];
  },
};
```

`unmountComponentFromNode` 方法我们看到 `ReactComponent.js` ：

```javascript
// core/ReactComponent.js
var ReactComponent = {
  Mixin: {
    unmountComponentFromNode: function(container) {
      // 调用对应的 ReactNativeComponent 或 ReactCompositeComponent 组件的 unmountComponent 方法
      this.unmountComponent();
      // http://jsperf.com/emptying-a-node
      // 遍历移除节点
      while (container.lastChild) {
        container.removeChild(container.lastChild);
      }
    },
    unmountComponent: function() {
      // 释放内存相关，会被 ReactNativeComponent 或 ReactCompositeComponent 调起
      invariant(
        this._lifeCycleState === ComponentLifeCycle.MOUNTED,
        'unmountComponent(): Can only unmount a mounted component.'
      );
      var props = this.props;
      if (props.ref != null) {
        ReactOwner.removeComponentAsRefFrom(this, props.ref, props[OWNER]);
      }
      this._rootNode = null;
      this._rootNodeID = null;
      this._lifeCycleState = ComponentLifeCycle.UNMOUNTED;
    },
  }
};
```

我们来分别看下 `ReactNativeComponent` 或 `ReactCompositeComponent` 组件的 `unmountComponent` 方法：

```javascript
// core/ReactNativeComponent.js
ReactNativeComponent.Mixin = {
  unmountComponent: function() {
    // 释放内存
    ReactComponent.Mixin.unmountComponent.call(this);
    // 移除子节点，方法来自于 ReactMultiChild.unmountMultiChild
    this.unmountMultiChild();
    // 移除事件监听
    ReactEvent.deleteAllListeners(this._rootNodeID);
  }
};
```

### 遍历卸载子组件

```javascript
// core/ReactMultiChild.js
var ReactMultiChildMixin = {
  unmountMultiChild: function() {
    var renderedChildren = this._renderedChildren;
    // 遍历卸载
    for (var name in renderedChildren) {
      if (renderedChildren.hasOwnProperty(name) && renderedChildren[name]) {
        var renderedChild = renderedChildren[name];
        renderedChild.unmountComponent && renderedChild.unmountComponent();
      }
    }
    // 释放内存
    this._renderedChildren = null;
  },
};
```

```javascript
// core/ReactCompositeComponent.js
var ReactCompositeComponentMixin = {
  unmountComponent: function() {
    this._compositeLifeCycleState = CompositeLifeCycle.UNMOUNTING;
    // 执行生命周期
    if (this.componentWillUnmount) {
      this.componentWillUnmount();
    }
    this._compositeLifeCycleState = null;

    ReactComponent.Mixin.unmountComponent.call(this);
    // 执行 ReactNativeComponent.unmountComponent
    this._renderedComponent.unmountComponent();
    // 释放内存
    this._renderedComponent = null;

    if (this.refs) {
      this.refs = null;
    }

    // Some existing components rely on this.props even after they've been
    // destroyed (in event handlers).
    // TODO: this.props = null;
    // TODO: this.state = null;
  },
};
```

那么到此，实现组件渲染功能。

---

## 验证组件方法

这里还有一点需要提的到是 `React.isValidComponent` 方法，用来校验是否为 React 组件：

```javascript
// core/ReactComponent.js
var ReactComponent = {
  isValidComponent: function(object) {
    // 鸭子类型校验法
    // object 存在切具有 mountComponentIntoNode 方法 和 receiveProps 方法
    return !!(
      object &&
      typeof object.mountComponentIntoNode === 'function' &&
      typeof object.receiveProps === 'function'
    );
  },
};
```
