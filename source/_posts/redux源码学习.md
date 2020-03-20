---
title: Redux 源码学习
date: 2020-03-20 20:25:06
categories: "React"
comments: true
featured_image: pic.png
tags:
- Redux
- React
---

<!-- no node -->

<!-- more -->

> 想着对 React 生态更深入的学习，况且 React 源码内容也不多的情况，来读一下他的源码内容。
> 目前 Redux 已经是 TypeScript 版本的源码，我正在阅读的则是 `4.0.4` 版本代码。

很显然，可以看到这和 React 仓库一样，同样使用了 rollup 打包工具，也验证了 rollup 在开发库类仓库的优势，毕竟大厂都在用嘛。

我们先来看到 `package.json` ：

```json
{
  "name": "redux",
  "version": "4.0.4",
  "main": "lib/redux.js",
  "unpkg": "dist/redux.js",
  "module": "es/redux.js",
  "types": "types/index.d.ts",
}
```

我省略了大部分内容，可以看到开发团队针对不同使用场景，对外提供包对应的路径，比如使用 TypeScript 时，对应的则是 `types/index.d.ts` ，或者使用 ES6 模块化引入时则是 `es/redux.js` 。值得我们学习。

接下来，我们看到源码部分，整个源码学习内容，将围绕着 `src` 进行， `utils` 下的内容即一些简单的工具， `types` 下则是一些类型的定义。

示例代码中我会移除并省略掉一些逻辑代码和 TypeScript 相关的函数重载代码，以及一些我认为不那么重要的代码，即只展现我想要说明的内容。

那么我们先来看到 `createStore` 函数吧。

## createStore

```typescript
export default function createStore<
  S,
  A extends Action,
  Ext = {},
  StateExt = never
>(
  reducer: Reducer<S, A>,
  preloadedState?: PreloadedState<S> | StoreEnhancer<Ext, StateExt>,
  enhancer?: StoreEnhancer<Ext, StateExt>
): Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext {
  // 接受的 reducer
  let currentReducer = reducer
  // 预加载 state
  let currentState = preloadedState as S
  // 订阅队列
  let currentListeners: (() => void)[] | null = []
  // 下一个订阅队列，看起来更像是订阅队列的副本，下面我会来介绍他和订阅队列的不解之谜
  let nextListeners = currentListeners
  // 检查并创建这个副本的函数
  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      // 拷贝到一个新的数组引用，可以理解为 [...currentListeners] 使得 nextListeners !== currentListeners
      nextListeners = currentListeners.slice()
    }
  }
  // 获得 state
  function getState(): S {
    return currentState as S
  }
  // 订阅函数
  function subscribe(listener: () => void) {
    ensureCanMutateNextListeners()
    // 操作订阅队列副本，与 currentListeners 之间不受影响
    nextListeners.push(listener)
    // 返回取消订阅函数
    return function unsubscribe() {
      ensureCanMutateNextListeners()
      // 同样的效果
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
      currentListeners = null
    }
    // 看到这里你是不是会想，创建这个副本的目的是什么？
    // 官方给到的解释：This prevents any bugs around consumers calling subscribe/unsubscribe in the middle of a dispatch.
  }
  // dispatch 函数
  function dispatch(action: A) {
    currentState = currentReducer(currentState, action)
    // 遍历通知订阅队列，将 currentListeners 赋值为“最新”的队列
    const listeners = (currentListeners = nextListeners)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    return action
  }
  // 替换 reducer 函数
  function replaceReducer<NewState, NewActions extends A>(
    nextReducer: Reducer<NewState, NewActions>
  ): Store<ExtendState<NewState, StateExt>, NewActions, StateExt, Ext> & Ext {
    // 更新 reducer
    ;((currentReducer as unknown) as Reducer<
      NewState,
      NewActions
    >) = nextReducer
    // 同初始化功能一致，即生成 currentState ，下面代码中会提及
    dispatch({ type: ActionTypes.REPLACE } as A)
    return (store as unknown) as Store<
      ExtendState<NewState, StateExt>,
      NewActions,
      StateExt,
      Ext
    > &
      Ext
  }
  // 初始化，生成 currentState ，即调用 getState 函数返回的内容， ActionTypes.INIT 即可以理解为私有类型，不会影响到你自己写的 reducer 毕竟里面包含了随机数...
  dispatch({ type: ActionTypes.INIT } as A)
  // 暴露的 store 对象
  const store = ({
    dispatch: dispatch as Dispatch<A>,
    subscribe,
    getState,
    replaceReducer,
    // 观察者方法这里不做解读，可以理解为对 subscribe 的封装
    [$$observable]: observable
  } as unknown) as Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext
  return store
}
```

那么到此，我们看到了整个 `createStore` 函数的实现过程以及返回的内容。

这里我来简单的代过 `compose` 函数，也是业界的常用实现，不是很明白其功能的同学可以简单看到这段代码即可：

```typescript
(funcs as : Function[]).reduce((a, b) => (...args: any) => a(b(...args)))
```

接下来，我们来看看 `combineReducers` 的实现，你可以把这个函数理解为对 `reducer` 的合并，当然函数名就是这个意思。

## combineReducers

```typescript
export default function combineReducers(reducers: ReducersMapObject) {
  const reducerKeys = Object.keys(reducers)
  // 用于 combination 函数中使用所需，收集的有效 reducers
  const finalReducers: ReducersMapObject = {}
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]

    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  // 收集的 keys
  const finalReducerKeys = Object.keys(finalReducers)

  return function combination(
    state: StateFromReducersMapObject<typeof reducers> = {},
    action: AnyAction
  ) {
    let hasChanged = false
    // 新的 state
    const nextState: StateFromReducersMapObject<typeof reducers> = {}
    // 遍历 keys
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      const previousStateForKey = state[key]
      // 从遍历中更新 state
      const nextStateForKey = reducer(previousStateForKey, action)
      nextState[key] = nextStateForKey
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    hasChanged =
      hasChanged || finalReducerKeys.length !== Object.keys(state).length
    // 返回 state 或新的 state
    return hasChanged ? nextState : state
  }
}
```

在我们看完 `reducer` 的组合工具函数后，我们来看到绑定执行上下文的语法糖函数 `bindActionCreators` 。

## bindActionCreators

实现代码很少，即绑定执行上下文环境 `this` 。具体使用场景，可以看到官网 API 文档 [bindActionCreators(actionCreators, dispatch)](https://redux.js.org/api/bindactioncreators#bindactioncreatorsactioncreators-dispatch) ，那么直接看到源码：

```typescript
function bindActionCreator<A extends AnyAction = AnyAction>(
  actionCreator: ActionCreator<A>,
  dispatch: Dispatch
) {
  return function(this: any, ...args: any[]) {
    return dispatch(actionCreator.apply(this, args))
  }
}

export default function bindActionCreators(
  actionCreators: ActionCreator<any> | ActionCreatorsMapObject,
  dispatch: Dispatch
) {
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }

  const boundActionCreators: ActionCreatorsMapObject = {}
  for (const key in actionCreators) {
    const actionCreator = actionCreators[key]
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  }
  return boundActionCreators
}
```

最后，我们来看到中间件函数 `applyMiddleware` 。

## applyMiddleware

```typescript
export default function applyMiddleware(
  ...middlewares: Middleware[]
): StoreEnhancer<any> {
  return (createStore: StoreCreator) => <S, A extends AnyAction>(
    reducer: Reducer<S, A>,
    ...args: any[]
  ) => {
    const store = createStore(reducer, ...args)
    let dispatch: Dispatch = () => {
      throw new Error('...')
    }

    const middlewareAPI: MiddlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 对 store.dispatch 进行改写，将其包装在中间件的最后一层，以实现中间件的功能
    dispatch = compose<typeof dispatch>(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```

OK， Redux 源码学习完了，是不是没看够，确实！

这才是精妙的库设计，简洁的代码实现强大的功能，那么这次没过瘾没关系，下次我们来品一品 Redux 是如何和 React 结合在一起的吧。
