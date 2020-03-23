---
title: React Redux 源码学习
date: 2020-03-23 17:06:07
categories: "React"
comments: true
featured_image: pic.png
tags:
- Redux
- React
---

<!-- no node -->

<!-- more -->

> 继 Redux 源码学习之后，我们来看一看 React Redux 是如何将 Redux 和 React 组合起来的。
> 我正在阅读的则是 `7.2.0` 版本代码。

我们依然围绕着 `src` 进行学习。可以看到源码中已经在使用 Hook 代码。如果你暂时还对 Hook 不是很了解的话，建议先前往 React 官网学习 Hook 内容。

从目录来看 `utils` 依然是简单的工具， `hooks` 下则是一些对外提供的 Hook ， `connect` 下则是我们每次都要使用的**高阶组件**函数， `components` 则也是我们每次都要使用的 `Provider` 组件。其中， `connect/connect.js` 其实是对 `components/connectAdvanced.js` 的封装。

示例代码中我会移除并省略掉一些逻辑代码以及一些我认为不那么重要的代码，即只展现我想要说明的内容。

有必要提到的是 `utils/batch.js` 和 `index.js` 中的部分源码，其中涉及到 React 协调中的批量更新，并且在 `utils/Subscription.js` 有使用到。

```javascript
// utils/batch.js
let batch = callback => callback()

export const setBatch = newBatch => (batch = newBatch)
export const getBatch = () => batch
// index.js
import { setBatch } from './utils/batch'
import { unstable_batchedUpdates as batch } from 'react-dom'

setBatch(batch)
```

## Subscription

从上面我们可以看出 React Redux 这里使用的更新模式并不是“普通”的执行函数，而是依赖于 React DOM 的批量更新，作为了解学习即可，可以增进一些自己的思路。现在，让我们来看到 `utils/Subscription.js` 的源码：

```javascript
function createListenerCollection() {
  // 依赖于 React DOM 的批量更新
  const batch = getBatch()
  // 监听器链表头和尾
  let first = null
  let last = null
  // 典型的闭包结构
  return {
    // 清空链表
    clear() {
      first = null
      last = null
    },
    // 通知函数
    notify() {
      // 调用依赖于 React DOM 的批量更新
      batch(() => {
        let listener = first
        while (listener) {
          listener.callback()
          listener = listener.next
        }
      })
    },
    // 获取监听器列表
    get() {
      let listeners = []
      let listener = first
      while (listener) {
        listeners.push(listener)
        listener = listener.next
      }
      return listeners
    },
    // 订阅回调函数，添加至监听器链表
    subscribe(callback) {
      let isSubscribed = true

      let listener = (last = {
        callback,
        next: null,
        prev: last
      })

      if (listener.prev) {
        listener.prev.next = listener
      } else {
        first = listener
      }
      // 返回取消订阅函数（典型的闭包结构）
      return function unsubscribe() {
        if (!isSubscribed || first === null) return
        isSubscribed = false
        // 将加入的监听器移除
        if (listener.next) {
          listener.next.prev = listener.prev
        } else {
          last = listener.prev
        }
        if (listener.prev) {
          listener.prev.next = listener.next
        } else {
          first = listener.next
        }
      }
    }
  }
}

export default class Subscription {
  constructor(store, parentSub) {
    // 初始化
    this.store = store
    this.parentSub = parentSub
    this.unsubscribe = null
    this.listeners = nullListeners
    // 绑定执行上下文
    this.handleChangeWrapper = this.handleChangeWrapper.bind(this)
  }
  // 添加监听器回调
  addNestedSub(listener) {
    this.trySubscribe()
    return this.listeners.subscribe(listener)
  }
  // 监听器通知
  notifyNestedSubs() {
    this.listeners.notify()
  }
  // 处理指定 API onStateChange
  handleChangeWrapper() {
    if (this.onStateChange) {
      this.onStateChange()
    }
  }
  // 是否尝试过订阅
  isSubscribed() {
    return Boolean(this.unsubscribe)
  }
  // 订阅
  trySubscribe() {
    if (!this.unsubscribe) {
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.handleChangeWrapper)
        : this.store.subscribe(this.handleChangeWrapper)
      // 创建监听器集合
      this.listeners = createListenerCollection()
    }
  }
  // 退订清除缓存
  tryUnsubscribe() {
    if (this.unsubscribe) {
      this.unsubscribe()
      this.unsubscribe = null
      this.listeners.clear()
      this.listeners = nullListeners
    }
  }
}
```

## Provider

上面是一个订阅器的简单实现，也是 `Provider` 组件触发更新的原理，那么话不多说，我们来看到 `components` 目录下 `Provider.js` 的相关源码：

```javascript
const ReactReduxContext = /*#__PURE__*/ React.createContext(null)

function Provider({ store, context, children }) {
  const contextValue = useMemo(() => {
    // 实例化 Subscription
    const subscription = new Subscription(store)
    // API onStateChange 设置
    subscription.onStateChange = subscription.notifyNestedSubs
    return {
      store,
      subscription
    }
  }, [store])
  // Redux state 获取 API 调用 
  const previousState = useMemo(() => store.getState(), [store])

  useEffect(() => {
    const { subscription } = contextValue
    // 尝试订阅并创建监听器集合
    subscription.trySubscribe()

    if (previousState !== store.getState()) {
      // 若新旧 state 不一样则发起通知，执行监听器列表
      subscription.notifyNestedSubs()
    }
    return () => {
      // 清除副作用
      subscription.tryUnsubscribe()
      subscription.onStateChange = null
    }
  }, [contextValue, previousState])
  // 用户自定义 Context 优先
  const Context = context || ReactReduxContext
  // contextValue 包含整个 Redux 实例和订阅器
  return <Context.Provider value={contextValue}>{children}</Context.Provider>
}
```

## connect

看到这里其实已经明白 `Provider` 组件的整个逻辑，即依据 `store` 或 `previousState` 的变化触发订阅退订的生命周期以及通知更新操作。那么再和 `connect` 高阶组件函数配合实现 React Redux 的功能。显然，到了这里我想大家都应该能猜到， `Provider` 组件用于注入 Redux 并控制逻辑，`connect` 高阶组件函数来处理用户想要的 `store` 以 `props` 的形式传入。我们先来看到 `connectAdvanced.js` 的源码吧（下面内容有些多，我陪你一点点看下去）：

```javascript
export default function connectAdvanced(
  /*
    选择器工厂函数（默认官方提供）作用是生成类似这样的代码内容：
    export default connectAdvanced((dispatch, options) => (state, props) => ({
      thing: state.things[props.thingId],
      saveThing: fields => dispatch(actionCreators.saveThing(props.thingId, fields)),
    }))(YourComponent)
  */
  selectorFactory,
  // 配置参数，官方已在 connect 进行封装（当然用户可以自己魔改），所以我们在使用的时候用的是 connect
  {
    // the func used to compute this HOC's displayName from the wrapped component's displayName.
    // probably overridden by wrapper functions such as connect()
    getDisplayName = name => `ConnectAdvanced(${name})`,
    // shown in error messages
    // probably overridden by wrapper functions such as connect()
    methodName = 'connectAdvanced',
    // REMOVED: if defined, the name of the property passed to the wrapped element indicating the number of
    // calls to render. useful for watching in react devtools for unnecessary re-renders.
    renderCountProp = undefined,
    // determines whether this HOC subscribes to store changes
    shouldHandleStateChanges = true,
    // REMOVED: the key of props/context to get the store
    storeKey = 'store',
    // REMOVED: expose the wrapped component via refs
    withRef = false,
    // use React's forwardRef to expose a ref of the wrapped component
    forwardRef = false,
    // the context consumer to use
    context = ReactReduxContext,
    // additional options are passed through to the selectorFactory
    ...connectOptions
  } = {}
) {
  const Context = context
  // 典型的 HOC 写法
  return function wrapWithConnect(WrappedComponent) {
    const wrappedComponentName =
      WrappedComponent.displayName || WrappedComponent.name || 'Component'

    const displayName = getDisplayName(wrappedComponentName)
    // 选择器工厂函数所需参数
    const selectorFactoryOptions = {
      ...connectOptions,
      getDisplayName,
      methodName,
      renderCountProp,
      shouldHandleStateChanges,
      storeKey,
      displayName,
      wrappedComponentName,
      WrappedComponent
    }
    // “ prue ”即一种模式，默认为开启。影响 Memo 相关内容，可以视做是一种性能优化
    const { pure } = connectOptions
    const usePureOnlyMemo = pure ? useMemo : callback => callback()
    // 函数组件 ConnectFunction
    function ConnectFunction(props) {
      // props 分类
      const [propsContext, forwardedRef, wrapperProps] = useMemo(() => {
        const { forwardedRef, ...wrapperProps } = props
        return [props.context, forwardedRef, wrapperProps]
      }, [props])
      // 选择默认 Context 或用户自己传入的 Context
      const ContextToUse = useMemo(() => {
        return propsContext &&
          propsContext.Consumer &&
          isContextConsumer(<propsContext.Consumer />)
          ? propsContext
          : Context
      }, [propsContext, Context])

      const contextValue = useContext(ContextToUse)
      // 典型的鸭子模型辨认法
      const didStoreComeFromProps =
        Boolean(props.store) &&
        Boolean(props.store.getState) &&
        Boolean(props.store.dispatch)
      const didStoreComeFromContext =
        Boolean(contextValue) && Boolean(contextValue.store)
      const store = didStoreComeFromProps ? props.store : contextValue.store
      // props 选择器，对应源码为 finalPropsSelectorFactory 函数，可以理解为一个包装层
      const childPropsSelector = useMemo(() => {
        return selectorFactory(store.dispatch, selectorFactoryOptions)/* createChildSelector(store) */
      }, [store])
      // 订阅器及通知函数创建
      const [subscription, notifyNestedSubs] = useMemo(() => {
        if (!shouldHandleStateChanges) return [null, null]/* NO_SUBSCRIPTION_ARRAY */
        // 实例化 Subscription
        const subscription = new Subscription(
          store,
          didStoreComeFromProps ? null : contextValue.subscription
        )
        // 绑定 notifyNestedSubs 的执行上下文环境
        const notifyNestedSubs = subscription.notifyNestedSubs.bind(
          subscription
        )
        return [subscription, notifyNestedSubs]
      }, [store, didStoreComeFromProps, contextValue])
      // Determine what {store, subscription} value should be put into nested context, if necessary,
      // and memoize that value to avoid unnecessary context updates.
      const overriddenContextValue = useMemo(() => {
        if (didStoreComeFromProps) {
          return contextValue
        }

        return {
          ...contextValue,
          subscription
        }
      }, [didStoreComeFromProps, contextValue, subscription])

      const lastChildProps = useRef()
      const lastWrapperProps = useRef(wrapperProps)
      const childPropsFromStoreUpdate = useRef()
      const renderIsScheduled = useRef(false)
      // 实际的 props
      const actualChildProps = usePureOnlyMemo(() => {
        if (
          childPropsFromStoreUpdate.current &&
          wrapperProps === lastWrapperProps.current
        ) {
          return childPropsFromStoreUpdate.current
        }
        // props 选择器，同包装层
        return childPropsSelector(store.getState(), wrapperProps)
      }, [store, wrapperProps])
      // Hook useEffect 或 useLayoutEffect
      useIsomorphicLayoutEffect(/* captureWrapperProps */() => {
        // 用于更新最新的 props 相关
        lastWrapperProps.current = wrapperProps
        lastChildProps.current = actualChildProps
        renderIsScheduled.current = false
        // 根据条件通知
        if (childPropsFromStoreUpdate.current) {
          childPropsFromStoreUpdate.current = null
          notifyNestedSubs()
        }
      })
      // Our re-subscribe logic only runs when the store/subscription setup changes
      useIsomorphicLayoutEffect(/* subscribeUpdates */ () => {
        if (!shouldHandleStateChanges) return

        let didUnsubscribe = false
        // 检查更新函数
        const checkForUpdates = () => {
          if (didUnsubscribe) { return }
          const latestStoreState = store.getState()
          // props 选择器（非包装层，可以理解为 impureFinalPropsSelector 函数，因为省略源码的关系，重名）
          let newChildProps = childPropsSelector(
            latestStoreState,
            lastWrapperProps.current
          )
          // 如果 props 未变化，则此处无事可做 - 级联订阅更新
          if (newChildProps === lastChildProps.current) {
            if (!renderIsScheduled.current) {
              // 通知
              notifyNestedSubs()
            }
          } else {
            // props 变化，触发通知
            lastChildProps.current = newChildProps
            childPropsFromStoreUpdate.current = newChildProps
            renderIsScheduled.current = true
          }
        }
        // 订阅 checkForUpdates
        subscription.onStateChange = checkForUpdates
        subscription.trySubscribe()
        // 首次检查
        checkForUpdates()
        // 取消订阅函数用于返回
        const unsubscribeWrapper = () => {
          didUnsubscribe = true
          subscription.tryUnsubscribe()
          subscription.onStateChange = null
        }
        return unsubscribeWrapper
      }, [store, subscription, childPropsSelector/* 包装层 */])

      const renderedWrappedComponent = useMemo(
        () => <WrappedComponent {...actualChildProps} ref={forwardedRef} />,
        [forwardedRef, WrappedComponent, actualChildProps]
      )

      const renderedChild = useMemo(() => {
        // 性能优化标示，若为 false 则不重新渲染
        if (shouldHandleStateChanges) {
          return (
            <ContextToUse.Provider value={overriddenContextValue}>
              {renderedWrappedComponent}
            </ContextToUse.Provider>
          )
        }

        return renderedWrappedComponent
      }, [ContextToUse, renderedWrappedComponent, overriddenContextValue])

      return renderedChild
    }
    const Connect = pure ? React.memo(ConnectFunction) : ConnectFunction
    Connect.WrappedComponent = WrappedComponent
    Connect.displayName = displayName
    // 此处省略含有 React.forwardRef 相关的特殊场景下的代码内容，若感兴趣请移步至 Github 仓库查看
    // 关于 hoistStatics 请查看： Copies non-react specific statics from a child component to a parent component.
    // Similar to Object.assign, but with React static keywords blacklisted from being overridden.
    return hoistStatics(Connect, WrappedComponent)
  }
}
```

到这里， `connectAdvanced` 高阶组件函数的内容已阅读完，因为篇幅关系，我移除了部分代码，可能会导致理解不足，就像我在前面说的，这个高阶组件的功能就是如此，若大家想深入建议前往 Github 仓库查看，大部分内容是一些细节的处理。

是不是觉得内容有些爆炸，有些多。确实有些，不如，先去喝杯水，上个洗手间，放松一下，我们再继续。

我们在看到后面的源码（ `connect.js` ）之前需要先看一些内容，还记得 `connect` 函数执行的时候传入的参数类似于 `mapStateToProps` 或者 `mapDispatchToProps` 的过滤参数，源码中我会移除这部份内容，所以我在开头先来举例说明。

源码中以 `match` 函数来对参数进行识别，以条件选项最多的 `mapDispatchToProps` 为例子。

即对入参进行如下顺序校验，优先满足即返回：

1. `whenMapDispatchToPropsIsFunction` 传入识别为函数
2. `whenMapDispatchToPropsIsMissing` 未传入
3. `whenMapDispatchToPropsIsObject` 传入识别为对象

在了解过这个后，我们来回到源码：

```javascript
export function createConnect({
  connectHOC = connectAdvanced/* HOC */,
  mapStateToPropsFactories = defaultMapStateToPropsFactories,
  mapDispatchToPropsFactories = defaultMapDispatchToPropsFactories,
  mergePropsFactories = defaultMergePropsFactories,
  selectorFactory = defaultSelectorFactory
} = {}) {
  // 看到这里是不是少许熟悉了一些
  return function connect(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    {
      pure = true,
      areStatesEqual = strictEqual,
      areOwnPropsEqual = shallowEqual,
      areStatePropsEqual = shallowEqual,
      areMergedPropsEqual = shallowEqual,
      ...extraOptions
    } = {}
  ) {
    // 分别进行入参识别
    const initMapStateToProps = match(mapStateToProps, mapStateToPropsFactories, 'mapStateToProps')
    const initMapDispatchToProps = match(mapDispatchToProps, mapDispatchToPropsFactories, 'mapDispatchToProps')
    const initMergeProps = match(mergeProps, mergePropsFactories, 'mergeProps')
    // 对 connectAdvanced 的封装
    return connectHOC(selectorFactory, {
      // used in error messages
      methodName: 'connect',
      // used to compute Connect's displayName from the wrapped component's displayName.
      getDisplayName: name => `Connect(${name})`,
      // if mapStateToProps is falsy, the Connect component doesn't subscribe to store state changes
      shouldHandleStateChanges: Boolean(mapStateToProps),
      // passed through to selectorFactory
      initMapStateToProps,
      initMapDispatchToProps,
      initMergeProps,
      pure,
      areStatesEqual,
      areOwnPropsEqual,
      areStatePropsEqual,
      areMergedPropsEqual,
      // any extra options args can override defaults of connect or connectAdvanced
      ...extraOptions
    })
  }
}

export default /*#__PURE__*/ createConnect()
```

那么到此，我们已经看完 `connect` 函数的内容。

回到前面的内容，整个高阶组件函数是为了实现：

```javascript
export default connectAdvanced((dispatch, options) => (state, props) => ({
  thing: state.things[props.thingId],
  saveThing: fields => dispatch(actionCreators.saveThing(props.thingId, fields)),
}))(YourComponent)
```

类似于这样的代码，所以到目前为止是不是漏了点什么？是的，被你发现了。是 `selectorFactory` ：

```javascript
export function impureFinalPropsSelectorFactory(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  dispatch
) {
  return function impureFinalPropsSelector(state, ownProps) {
    return mergeProps(
      mapStateToProps(state, ownProps),
      mapDispatchToProps(dispatch, ownProps),
      ownProps
    )
  }
}
```

我省略了大部分优化代码，很直观的理解就是过滤、合并最终你想要的 `props` 。剩下关于源码中 Hook 即是一些简单的封装，这里就不做解读了，很简单，如果感兴趣的话，请移步至 Github 仓库查看，那么这次的 React Redux 源码学习到此，感谢阅读至此！
