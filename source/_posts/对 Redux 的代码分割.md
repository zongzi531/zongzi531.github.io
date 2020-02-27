---
title: 对 Redux 的代码分割
date: 2020-02-27 14:22:21
categories: "React"
comments: true
featured_image: pic.png
tags:
- Redux
- Code Splitting
- 代码分割
---

<!-- no node -->

<!-- more -->

相信大家都知道利用 React 脚手架 Create React App 构建出来的 React 项目是基于 Webpack 构建的。那么官方也提供了关于拆分 React 代码，实现代码分割的功能。

那么需要使用到的就是 `import()` 也就是 dynamic import 动态导入功能，配合 `React.lazy` 进行使用，即可实现代码分割。

关于这一块部分如果你还没有很好的了解，建议你可以看一下[《代码分割 - React》](https://zh-hans.reactjs.org/docs/code-splitting.html#code-splitting)。

在我司项目的快迭代环境下，我也选择这套方案来进行优化，但是我在后续的开发过程中发现，这套解决方案并不完善，因为对于 Redux ，我并没有做到代码分割，当迭代过程中仅涉及到修改 Redux reducer 时会出现大批量代码更新。

因为项目中仅使用了一个 `store` 来使用 Redux ，当里面出现一个细小的变化会导致整个 `store` 的打包 `hash` 值变化，从而影响到大部分被分割的 React 代码，包括很多并未做修改的文件。

所以我开始着手对 Redux 进行代码分割，官方[《Code Splitting - Redux》](https://redux.js.org/recipes/code-splitting/)也提供了很好的支持。

基于此，我开始修改项目中的代码， 依照官方的例子开始修改：

```javascript
/**
 * rootReducer 即认为静态 reducer
 * asyncReducers 即认为需要合并的异步 reducer
 */
const createReducer = asyncReducers => combineReducers({
  ...rootReducer,
  ...asyncReducers
});

// 暴露给外界的注入函数
export let injectReducer;

const configureStore = preloadedState => {
  const store = createStore(
    combineReducers(rootReducer),
    preloadedState,
    middleware
  );

  store.asyncReducers = {};

  // 注入函数
  injectReducer = store.injectReducer = (key, asyncReducer) => {
    if (Object.hasOwnProperty.call(store.asyncReducers, key)) { return; };
    
    // 赋值
    store.asyncReducers[key] = asyncReducer;
    // 合并 reducer 并调用 replaceReducer 方法更新
    store.replaceReducer(createReducer(store.asyncReducers));
  };

  return store;
};

export default configureStore;
```

对原先的创建 `store` 过程进行了更新，暴露了 `injectReducer` 函数用于异步注入，结合 `import()` 就可以实现对 React 的代码分割，真是非常喜悦。

那我们来看看怎么用：

```javascript
import { lazy } from 'react';
import { injectReducer } from '@/store/configureStore';

export default {
  DictManager: lazy(async () => {
    await import(/* webpackChunkName: 'Something' */'@/reducers/something').then(res => {
      injectReducer('something', res.default);
    });
    return import(/* webpackChunkName: 'Something' */'@/containers/Something');
  }),
  ...
};
```

使用方法也特别简单，即动态导入后注入 reducer 即可，以实现代码分割。

那么到此，相对实现了一个比较完善的 React + Redux 的代码分割解决方案，可以以页面维度，或者你想要的细度去控制代码分割的程度以应对项目的快迭代。
