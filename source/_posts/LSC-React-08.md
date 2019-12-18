---
title: React 源码学习（八）：组件更新
date: 2019-04-08 19:15:34
categories: "React 源码学习"
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
9. [React 源码学习（九）：“脱胎换骨”](https://zongzi531.com/2019/12/18/LSC-React-09/)
10. [React 源码学习（十）：Fiber](https://zongzi531.com/2019/12/19/LSC-React-10/)
11. [React 源码学习（十一）：Scheduling](https://zongzi531.com/2019/12/20/LSC-React-11/)
12. [React 源码学习（十二）：Reconciliation](https://zongzi531.com/2019/12/21/LSC-React-12/)

## 是什么引起组件更新

引发组件更新的方法就是 `this.setState` ，按照注释代码看来 `this.setState` 是不可变的，则 `this._pendingState` 是用来存放挂起的 `state` ，他不会直接更新到 `this.state` ，让我们来看到代码：

```javascript
// core/ReactCompositeComponent.js
var ReactCompositeComponentMixin = {
  setState: function(partialState) {
    // 如果“挂起状态”存在，则与之合并，否则与现有状态合并。
    this.replaceState(merge(this._pendingState || this.state, partialState));
  },
  replaceState: function(completeState) {
    var compositeLifeCycleState = this._compositeLifeCycleState;
    // 生命周期校验
    invariant(
      this._lifeCycleState === ReactComponent.LifeCycle.MOUNTED ||
      compositeLifeCycleState === CompositeLifeCycle.MOUNTING,
      'replaceState(...): Can only update a mounted (or mounting) component.'
    );
    invariant(
      compositeLifeCycleState !== CompositeLifeCycle.RECEIVING_STATE &&
      compositeLifeCycleState !== CompositeLifeCycle.UNMOUNTING,
      'replaceState(...): Cannot update while unmounting component or during ' +
      'an existing state transition (such as within `render`).'
    );

    // 将合并完的状态给挂起状态，若不满足下面更新条件，则只存入挂起状态结束此函数
    this._pendingState = completeState;

    // 如果我们处于安装或接收道具的中间，请不要触发状态转换，因为这两者都已经这样做了。
    // 若复合组件生命周期不在挂载中和更新 props 时，我们会操作更新方法
    if (compositeLifeCycleState !== CompositeLifeCycle.MOUNTING &&
        compositeLifeCycleState !== CompositeLifeCycle.RECEIVING_PROPS) {
      // 变更复合组件生命周期为更新 state
      this._compositeLifeCycleState = CompositeLifeCycle.RECEIVING_STATE;

      // 准备更新 state ，并释放挂起状态
      var nextState = this._pendingState;
      this._pendingState = null;

      // 进入 React 调度事务，加入 _receivePropsAndState 方法
      var transaction = ReactComponent.ReactReconcileTransaction.getPooled();
      transaction.perform(
        this._receivePropsAndState,
        this,
        this.props,
        nextState,
        transaction
      );
      ReactComponent.ReactReconcileTransaction.release(transaction);

      // 调度事务完成后置空复合组件生命周期
      this._compositeLifeCycleState = null;
    }
  },
};
```

### setState 触发了什么

那么到此为止，你可以知道 `this.setState` 并非事实更新 `this.state` 的，比如我们看到在 `componentWillMount` 中去使用 `this.setState` 并不会马上更新到 `this.state` ，那么我们继续阅读后面代码：

```javascript
// core/ReactCompositeComponent.js
var ReactCompositeComponentMixin = {
  _receivePropsAndState: function(nextProps, nextState, transaction) {
    // shouldComponentUpdate 方法不存在或返回 true
    if (!this.shouldComponentUpdate ||
        this.shouldComponentUpdate(nextProps, nextState)) {
      // Will set `this.props` and `this.state`.
      this._performComponentUpdate(nextProps, nextState, transaction);
    } else {
      // 如果确定某个组件不应该更新，我们仍然需要设置props和state。
      // shouldComponentUpdate 返回 false 的情况
      this.props = nextProps;
      this.state = nextState;
    }
  },
  _performComponentUpdate: function(nextProps, nextState, transaction) {
    // 存入旧的 props 和 state
    // 用于传入 componentDidUpdate
    var prevProps = this.props;
    var prevState = this.state;

    if (this.componentWillUpdate) {
      this.componentWillUpdate(nextProps, nextState, transaction);
    }

    // 更新 props 和 state
    this.props = nextProps;
    this.state = nextState;

    // 更新组件
    this.updateComponent(transaction);

    if (this.componentDidUpdate) {
      transaction.getReactOnDOMReady().enqueue(
        this,
        this.componentDidUpdate.bind(this, prevProps, prevState)
      );
    }
  },
  updateComponent: function(transaction) {
    // 这里的更新比较硬核
    // 先把已渲染的旧的组件赋值至 currentComponent
    var currentComponent = this._renderedComponent;
    // 直接渲染新的组件（是不是很硬核）
    var nextComponent = this._renderValidatedComponent();
    // 如果是同样的组件则进入此判断
    // 通过 constructor 来判断是否为同一个
    // 比如：
    // React.DOM.a().constructor !== React.DOM.p().constructor
    // React.DOM.a().constructor === React.DOM.a().constructor
    // 或
    // React.createClass({ render: () => null }).constructor ===
    // React.createClass({ render: () => null }).constructor
    if (currentComponent.constructor === nextComponent.constructor) {
      // 若新的组件 props 下 isStatic 为真则不更新
      // 知道这一个可以对部分组件进行手动优化，以免不必要的计算
      if (!nextComponent.props.isStatic) {
        // 这里会调用对应的方法
        // ReactCompositeComponent.receiveProps
        // ReactNativeComponent.receiveProps
        // ReactTextComponent.receiveProps
        // 除了 ReactTextComponent 都会调用 ReactComponent.Mixin.receiveProps 来更新 ref 相关
        // 这个我们稍后来解读
        currentComponent.receiveProps(nextComponent.props, transaction);
      }
    } else {
      // These two IDs are actually the same! But nothing should rely on that.
      // 旧的 _rootNodeID 和新的 _rootNodeID
      var thisID = this._rootNodeID;
      var currentComponentID = currentComponent._rootNodeID;
      // 卸载旧组件
      currentComponent.unmountComponent();
      // 挂载新组件（也挺硬核的）
      var nextMarkup = nextComponent.mountComponent(thisID, transaction);
      // 在新 _rootNodeID 下更新 markup 标记
      ReactComponent.DOMIDOperations.dangerouslyReplaceNodeWithMarkupByID(
        currentComponentID,
        nextMarkup
      );
      // 赋值新的组件
      this._renderedComponent = nextComponent;
    }
  },
};
```

### 各个组件的 receiveProps 方法

上面代码看来，一个是不替换组件的情况下更新组件，另一个则是直接更新 `markup` 标记。我们按照顺序一个个看过来吧，先看到 `ReactCompositeComponent.receiveProps` ：

```javascript
// core/ReactCompositeComponent.js
var ReactCompositeComponentMixin = {
  receiveProps: function(nextProps, transaction) {
    // 校验参数
    if (this.constructor.propDeclarations) {
      this._assertValidProps(nextProps);
    }
    // 更新 ref
    ReactComponent.Mixin.receiveProps.call(this, nextProps, transaction);

    // 更新复合组件生命周期为更新 props
    this._compositeLifeCycleState = CompositeLifeCycle.RECEIVING_PROPS;
    // 执行钩子函数，在这个函数内执行 this.setState 是不会立即更新 this.state 的
    if (this.componentWillReceiveProps) {
      this.componentWillReceiveProps(nextProps, transaction);
    }
    // 进入复合组件生命周期更新 state
    this._compositeLifeCycleState = CompositeLifeCycle.RECEIVING_STATE;
    // When receiving props, calls to `setState` by `componentWillReceiveProps`
    // will set `this._pendingState` without triggering a re-render.
    // 如果上面执行过 componentWillReceiveProps ，并且里面操作了 this.setState
    // 那么 this._pendingState 会有值，并且是与 this.state 合并过的
    var nextState = this._pendingState || this.state;
    // 释放 this._pendingState
    this._pendingState = null;
    // 执行的是 currentComponent._receivePropsAndState 方法
    // 但是这个 currentComponent 一定是 ReactCompositeComponent
    this._receivePropsAndState(nextProps, nextState, transaction);
    // 置空复合组件生命周期
    this._compositeLifeCycleState = null;
  },
};
```

再是我们来看看 `ReactNativeComponent.receiveProps` ：

```javascript
// core/ReactNativeComponent.js
ReactNativeComponent.Mixin = {
  receiveProps: function(nextProps, transaction) {
    // 日常校验
    invariant(
      this._rootNodeID,
      'Trying to control a native dom element without a backing id'
    );
    assertValidProps(nextProps);
    // 日常更新 ref
    ReactComponent.Mixin.receiveProps.call(this, nextProps, transaction);
    // 重点来了，更新 DOM 属性
    this._updateDOMProperties(nextProps);
    // 更新 DOM 子节点
    this._updateDOMChildren(nextProps, transaction);
    // 都执行完后更新 props
    this.props = nextProps;
  },
  _updateDOMProperties: function(nextProps) {
    // 这里开始解读更新 DOM 属性
    // 保存旧 props
    var lastProps = this.props;
    // 遍历新 props
    for (var propKey in nextProps) {
      var nextProp = nextProps[propKey];
      var lastProp = lastProps[propKey];
      // 以新 props 键为准取对应的值
      // 若 2 个值相等则跳过
      if (!nextProps.hasOwnProperty(propKey) || nextProp === lastProp) {
        continue;
      }
      // CSS 样式
      if (propKey === STYLE) {
        if (nextProp) {
          nextProp = nextProps.style = merge(nextProp);
        }
        var styleUpdates;
        // 遍历 nextProp
        for (var styleName in nextProp) {
          if (!nextProp.hasOwnProperty(styleName)) {
            continue;
          }
          // 旧的 styleName 与新的 styleName 值不同时
          // 将新的值加入 styleUpdates
          if (!lastProp || lastProp[styleName] !== nextProp[styleName]) {
            if (!styleUpdates) {
              styleUpdates = {};
            }
            styleUpdates[styleName] = nextProp[styleName];
          }
        }
        // 操作更新 CSS 样式
        if (styleUpdates) {
          // ReactComponent.DOMIDOperations => ReactDOMIDOperations
          // 他会通过 ID 对真实 node 进行相应的更新
          ReactComponent.DOMIDOperations.updateStylesByID(
            this._rootNodeID,
            styleUpdates
          );
        }
        // 判断若是 dangerouslySetInnerHTML 则在不同的情况下进行相应的更新
      } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
        var lastHtml = lastProp && lastProp.__html;
        var nextHtml = nextProp && nextProp.__html;
        if (lastHtml !== nextHtml) {
          ReactComponent.DOMIDOperations.updateInnerHTMLByID(
            this._rootNodeID,
            nextProp
          );
        }
        // 判断 content 的情况更新
      } else if (propKey === CONTENT) {
        ReactComponent.DOMIDOperations.updateTextContentByID(
          this._rootNodeID,
          '' + nextProp
        );
        // 对事件进行监听
        // 比较好奇的是旧的 propKey 若存在着事件监听，这里似乎没有做什么处理
        // 这样不就内存溢出了吗？难道说不会有这种情况？？？
        // 想多了啦，更新 props 的情况，同样的事件会被覆盖
        // 在对应 this._rootNodeID 的情况下。（希望如此，没有证实过，但是理解如此）
      } else if (registrationNames[propKey]) {
        putListener(this._rootNodeID, propKey, nextProp);
      } else {
        // 剩余的就是更新 DOM 属性啦
        ReactComponent.DOMIDOperations.updatePropertyByID(
          this._rootNodeID,
          propKey,
          nextProp
        );
      }
    }
  },
  _updateDOMChildren: function(nextProps, transaction) {
    // 来更新 DOM 子节点了
    // 当前 this.props.content 类型
    var thisPropsContentType = typeof this.props.content;
    // 是否 thisPropsContentEmpty 为空
    var thisPropsContentEmpty =
      this.props.content == null || thisPropsContentType === 'boolean';
    // 新的 nextProps.content 类型
    var nextPropsContentType = typeof nextProps.content;
    // 是否 nextPropsContentEmpty 为空
    var nextPropsContentEmpty =
      nextProps.content == null || nextPropsContentType === 'boolean';

    // 最后使用的 content ：
    // 若 thisPropsContentEmpty 不为空则取 this.props.content 否则
    // this.props.children 类型为 string 或 number 的情况下取 this.props.children 否则
    // null
    var lastUsedContent = !thisPropsContentEmpty ? this.props.content :
      CONTENT_TYPES[typeof this.props.children] ? this.props.children : null;

    // 使用内容 content ：
    // 若 nextPropsContentEmpty 不为空则取 nextProps.content 否则
    // nextProps.children 类型为 string 或 number 的情况下取 nextProps.children 否则
    // null
    var contentToUse = !nextPropsContentEmpty ? nextProps.content :
      CONTENT_TYPES[typeof nextProps.children] ? nextProps.children : null;

    // Note the use of `!=` which checks for null or undefined.

    // 最后使用的 children ：
    // 若 lastUsedContent 不为 null or undefined 则取 null 否则
    // 取 this.props.children ，以 content 优先
    var lastUsedChildren =
      lastUsedContent != null ? null : this.props.children;
    // 使用 children ：
    // 若 contentToUse 不为 null or undefined 则取 null 否则
    // 取 nextProps.children ，以 content 优先
    var childrenToUse = contentToUse != null ? null : nextProps.children;

    // 需要使用 content 情况
    if (contentToUse != null) {
      // 是否需要移除 children 判断结果：
      // 最后使用的 children 存在并且 children 不再需要使用
      var childrenRemoved = lastUsedChildren != null && childrenToUse == null;
      if (childrenRemoved) {
        // 更新子节点
        this.updateMultiChild(null, transaction);
      }
      // 若没满足上面条件则说明不需要更新掉 children
      // 并且新旧 content 不相等的情况下进行 DOM 操作
      if (lastUsedContent !== contentToUse) {
        ReactComponent.DOMIDOperations.updateTextContentByID(
          this._rootNodeID,
          '' + contentToUse
        );
      }
    } else {
      // 反之看是否需要移除 content
      // 若最后使用的 content 存在且 content 不再需要使用
      var contentRemoved = lastUsedContent != null && contentToUse == null;
      if (contentRemoved) {
        // 进行 DOM 操作
        ReactComponent.DOMIDOperations.updateTextContentByID(
          this._rootNodeID,
          ''
        );
      }
      // 更新子节点
      // 压扁更新，与挂载时一样
      this.updateMultiChild(flattenChildren(nextProps.children), transaction);
    }
  },
};
```

## Diff

关于 DOM 操作一系列的方法这里不准备做解读，可以直接查看源码 `core/ReactDOMIDOperations.js` ，道理都是一样的。但是，这里需要看下 `updateMultiChild` 方法，因为这里已经涉及到 Diff 实现，但是在讲 Diff 之前，我们先把 `ReactTextComponent.receiveProps` 给解读掉，其实方法里面很简单，就是操作了 `ReactDOMIDOperations` 相关的方法，具体实现直接看源码就行，那么接下来，我们来看到 `updateMultiChild` ：

```javascript
// core/ReactMultiChild.js
// 直接看到 updateMultiChild
var ReactMultiChildMixin = {
  enqueueMarkupAt: function(markup, insertAt) {
    this.domOperations = this.domOperations || [];
    this.domOperations.push({insertMarkup: markup, finalIndex: insertAt});
  },
  enqueueMove: function(originalIndex, finalIndex) {
    this.domOperations = this.domOperations || [];
    this.domOperations.push({moveFrom: originalIndex, finalIndex: finalIndex});
  },
  enqueueUnmountChildByName: function(name, removeChild) {
    if (ReactComponent.isValidComponent(removeChild)) {
      this.domOperations = this.domOperations || [];
      this.domOperations.push({removeAt: removeChild._domIndex});
      removeChild.unmountComponent && removeChild.unmountComponent();
      delete this._renderedChildren[name];
    }
  },

  /**
   * Reconciles new children with old children in three phases.
   *
   * - Adds new content while updating existing children that should remain.
   * - Remove children that are no longer present in the next children.
   * - As a very last step, moves existing dom structures around.
   * - (Comment 1) `curChildrenDOMIndex` is the largest index of the current
   *   rendered children that appears in the next children and did not need to
   *   be "moved".
   * - (Comment 2) This is the key insight. If any non-removed child's previous
   *   index is less than `curChildrenDOMIndex` it must be moved.
   *
   * @param {?Object} children Flattened children object.
   */
  updateMultiChild: function(nextChildren, transaction) {
    // 一些补全判断操作
    if (!nextChildren && !this._renderedChildren) {
      return;
    } else if (nextChildren && !this._renderedChildren) {
      this._renderedChildren = {}; // lazily allocate backing store with nothing
    } else if (!nextChildren && this._renderedChildren) {
      nextChildren = {};
    }
    // 用于更新子节点时，记录的父节点 ID 前缀加 dot
    var rootDomIdDot = this._rootNodeID + '.';
    // DOM markup 标记缓冲
    var markupBuffer = null;  // Accumulate adjacent new children markup.
    // DOM markup 标记缓冲等待插入的数量
    var numPendingInsert = 0; // How many root nodes are waiting in markupBuffer
    // 新子节点的循环用索引 index
    var loopDomIndex = 0;     // Index of loop through new children.
    var curChildrenDOMIndex = 0;  // See (Comment 1)
    // 遍历新的 children
    for (var name in nextChildren) {
      if (!nextChildren.hasOwnProperty(name)) {continue;}
      var curChild = this._renderedChildren[name];
      var nextChild = nextChildren[name];
      // 通过 constructor 来判断 curChild 和 nextChild 是否为同一个
      if (shouldManageExisting(curChild, nextChild)) {
        if (markupBuffer) {
          // 若 DOM markup 标记缓冲存在，将其加入队列
          // 标记位置为 loopDomIndex - numPendingInsert
          // 这里和下面是一样的道理，请看到循环结束后
          this.enqueueMarkupAt(markupBuffer, loopDomIndex - numPendingInsert);
          // 清空 DOM markup 标记缓冲
          markupBuffer = null;
        }
        // 初始化 DOM markup 标记缓冲等待插入的数量为 0
        numPendingInsert = 0;
        // _domIndex 在挂载中依次按照顺序进行排序，若他小于目前的子节点顺序
        // 则进行移动操作，移动操作则是记录原 index 和现 index （也就是新子节点的循环用索引 index ）
        if (curChild._domIndex < curChildrenDOMIndex) { // (Comment 2)
          // 我没有办法联想到此情况
          this.enqueueMove(curChild._domIndex, loopDomIndex);
        }
        // curChildrenDOMIndex 则取大值
        curChildrenDOMIndex = Math.max(curChild._domIndex, curChildrenDOMIndex);
        // 硬核式递归更新！！同样会进入到 Diff
        !nextChild.props.isStatic &&
          curChild.receiveProps(nextChild.props, transaction);
        // 更新 _domIndex 属性
        curChild._domIndex = loopDomIndex;
      } else {
        // 若 curChild 和 nextChild 不为同一个的时候
        if (curChild) {               // !shouldUpdate && curChild => delete
          // 卸载旧子节点加入队列，并操作卸载组件
          this.enqueueUnmountChildByName(name, curChild);
          // curChildrenDOMIndex 则取大值
          curChildrenDOMIndex =
            Math.max(curChild._domIndex, curChildrenDOMIndex);
        }
        if (nextChild) {              // !shouldUpdate && nextChild => insert
          // 对应位置传入新子节点
          this._renderedChildren[name] = nextChild;
          // 生成新的 markup 标记
          // ID 为父 ID 加 dot 加现在的 name
          var nextMarkup =
            nextChild.mountComponent(rootDomIdDot + name, transaction);
          // 累加 DOM markup 标记缓冲
          markupBuffer = markupBuffer ? markupBuffer + nextMarkup : nextMarkup;
          // DOM markup 标记缓冲等待插入的数量
          numPendingInsert++;
          // 新的子节点 _domIndex 更新
          nextChild._domIndex = loopDomIndex;
        }
      }
      // 若新子节点存在，则新子节点的循环用索引 index 累加 1
      loopDomIndex = nextChild ? loopDomIndex + 1 : loopDomIndex;
    }
    if (markupBuffer) {
      // 将 DOM markup 标记缓冲加入队列
      // 这里的 loopDomIndex - numPendingInsert 可以解释下
      // 会使得 markupBuffer 存在的情况就是进入第二个分支，那么同样的，
      // 会使得 numPendingInsert 增加的情况也是第二个分支，那么在这里插入的 DOM markup 标记
      // 是最后插入的，他需要从整个循环 DOM 索引减去等待数量来确定插入位置
      // 举个例子，你在进入第二个分支时，旧节点存在的情况下一定会被移除
      // 新节点存在的情况下一定会被生成 DOM markup 标记 并且累加相应的数量
      // loopDomIndex 也会随之增加，loopDomIndex 也一定大于等于 numPendingInsert
      // 如：旧节点 <div></div><p></p>
      // 新节点 <div></div><span></span><p></p>
      // 这种情况下 loopDomIndex 为 3 ， numPendingInsert 为 2 ，插入位置为 1
      this.enqueueMarkupAt(markupBuffer, loopDomIndex - numPendingInsert);
    }
    // 遍历旧 children
    for (var childName in this._renderedChildren) { // from other direction
      if (!this._renderedChildren.hasOwnProperty(childName)) { continue; }
      var child = this._renderedChildren[childName];
      if (child && !nextChildren[childName]) {
        // 旧的存在，新的不存在加入队列
        this.enqueueUnmountChildByName(childName, child);
      }
    }
    // 执行 DOM 操作队列
    this.processChildDOMOperationsQueue();
  },
  processChildDOMOperationsQueue: function() {
    if (this.domOperations) {
      // 执行队列
      ReactComponent.DOMIDOperations
        .manageChildrenByParentID(this._rootNodeID, this.domOperations);
      this.domOperations = null;
    }
  },
};
```

在上面这个执行队列，我们需要看到相关的 DOM 操作：

```javascript
// domUtils/DOMChildrenOperations.js
var MOVE_NODE_AT_ORIG_INDEX = keyOf({moveFrom: null});
var INSERT_MARKUP = keyOf({insertMarkup: null});
var REMOVE_AT = keyOf({removeAt: null});

var manageChildren = function(parent, childOperations) {
  // 用于获得 DOM 中原生的 Node
  // 符合 MOVE_NODE_AT_ORIG_INDEX 和 REMOVE_AT
  var nodesByOriginalIndex = _getNodesByOriginalIndex(parent, childOperations);
  if (nodesByOriginalIndex) {
    // 移除对应的 Node
    _removeChildrenByOriginalIndex(parent, nodesByOriginalIndex);
  }
  // 对应的插入
  _placeNodesAtDestination(parent, childOperations, nodesByOriginalIndex);
};
```

## refs 引用

那么到此， Diff 实现算是解读完成，最后关于 ref 我们在这里也直接解读掉， ref 为引用，看到官方注释：“ ReactOwners are capable of storing references to owned components. ”，那么首先我们得知道 `[OWNER]` 是什么，他是：“引用组件所有者的属性键。”，那么他的值就是该组件的所有者（也就是父组件实例），这句话的依据在哪里呢？

```javascript
// core/ReactCompositeComponent.js
var ReactCompositeComponentMixin = {
  _renderValidatedComponent: function() {
    // render 方法执行前，我们将 this 也就是当前复合组件传入 ReactCurrentOwner.current
    // render 方法执行结束后，我们将置空 ReactCurrentOwner.current
    ReactCurrentOwner.current = this;
    var renderedComponent = this.render();
    ReactCurrentOwner.current = null;
    return renderedComponent;
  },
};
```

那么执行 `render` 方法时，发生了什么？回忆一下。返回的是 `ReactCompositeComponent` 或者 `ReactNativeComponent` 或者 `ReactTextComponent` ，那么他们在被实例化的过程中获得了 `ReactCurrentOwner.current` ：

```javascript
// core/ReactComponent.js
var ReactComponent = {
  Mixin: {
    construct: function(initialProps, children) {
      // Record the component responsible for creating this component.
      // 记录负责创建此组件的组件。
      // 将其记录下来。
      this.props[OWNER] = ReactCurrentOwner.current;
    },
  }
};
```

那么讲了这么多，他和 ref 有什么关系呢，那还确实有关系。在挂载、更新、卸载组件时都会发生 ref 的更新，若你对子组件添加了 ref 属性，那么他对应的键会出现在他拥有者的 `this.refs` 上，那么你就可以通过拥有者调用引用上的方法。

那么到此，实现组件更新。
