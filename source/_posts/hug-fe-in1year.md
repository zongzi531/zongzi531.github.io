---
title: 迎接学习FE一周年
date: 2017-08-07 21:43:25
categories: "Vue"
comments: true
tags:
- Vue
- Vux
- axios
---

<!-- no node -->

<!-- more -->

好久不见，本篇博客内容将延续上一篇博客内容[《初试VUX》](http://zongzi531.github.io/2017/07/01/%E5%88%9D%E8%AF%95vux/)，我将总结在完成此项目中学习到的一些有趣的事情。

>阅读记录：
>6月份，我阅读了《图解HTTP》
>7月份，我阅读了《JavaScript语言精粹》
>8月份，我正在阅读《CSS SECRETS》——很高兴的是，至今年8月底，我学习FE一年了
>9月份，我已经准备好迎接《ECMAScript 6 入门》以及《深入理解ES6》啦，已经阅读电子版的《ECMAScript 6 入门》，对ES6有些简单的认识，并慢慢实践中

7月初，分析公司业务需求后，我选择使用VUX完成项目，项目也很顺利的在7月底上线了。

那么以下内容都是通过我查阅我的Commits日志产出。

首先介绍下，本项目中使用到的技术栈：

* [vue](https://cn.vuejs.org/)
* [vue-router](https://github.com/vuejs/vue-router/blob/dev/docs/zh-cn/SUMMARY.md)
* [vuex](https://github.com/vuejs/vuex/blob/dev/docs/zh-cn/SUMMARY.md)
* [vue-infinite-scroll](https://github.com/ElemeFE/vue-infinite-scroll)
* [vux](https://vux.li/)
* [axios](https://github.com/mzabriskie/axios)
* [weui.js](https://github.com/Tencent/weui.js)

## axios

axios不需要过多介绍了吧，Vue更新至2.0版本后，作者就宣告不再对vue-resource更新，而是推荐的axios。

### URLSearchParams API 解决CORS预检请求

问题出在，非简单请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求（preflight）。

那么URLSearchParams API就是axios官方提供的解决方案。[使用方法传送门>>](https://github.com/mzabriskie/axios#using-applicationx-www-form-urlencoded-format)

以及浏览器发起XHR的两种请求：简单请求和非简单请求。可以参阅[《阮老师 - 跨域资源共享 CORS 详解》](http://www.ruanyifeng.com/blog/2016/04/cors.html)

### 实现文件上传

```html
<input class="file" name="file" type="file" accept="image/*" @change="upload"/>
```

```javascript
upload (event) {
  let file = event.target.files[0]
  let param = new FormData()  //  创建form对象
  param.append('upLoad', file, file.name)  //  通过append向form对象添加数据
  let config = {headers: {'Content-Type': 'multipart/form-data'}}  //  添加请求头
  axios.post(url, param, config)
  .then(function (response) {
    // ...
  })
  .catch(function (error) {
    // ...
  })
}
```

## vue-router

### [导航钩子](https://github.com/vuejs/vue-router/blob/dev/docs/zh-cn/advanced/navigation-guards.md)

因为业务逻辑需要，在载入某个路由后执行某些事。

```javascript
const router = new VueRouter({ ... })

router.beforeEach((to, from, next) => {
  // ...
})
```

## Vue

### [props](https://cn.vuejs.org/v2/api/#props)

props 可以是数组或对象，用于接收来自父组件的数据。props 可以是简单的数组，或者使用对象作为替代，对象允许配置高级选项，如类型检测、自定义校验和设置默认值。

项目开发过程中，我将数据存在父组件中，由于某些业务场景，需要在子组件A中修改父组件数据，从而更新至子组件B中，最后统一移除props，使用vuex管理数据。

## Vuex

每一个 Vuex 应用的核心就是 store（仓库）。"store" 基本上就是一个容器，它包含着你的应用中大部分的状态(state)。Vuex 和单纯的全局对象有以下两点不同：

1. Vuex 的状态存储是响应式的。当 Vue 组件从 store 中读取状态的时候，若 store 中的状态发生变化，那么相应的组件也会相应地得到高效更新。

2. 你不能直接改变 store 中的状态。改变 store 中的状态的唯一途径就是显式地提交(commit) mutations。这样使得我们可以方便地跟踪每一个状态的变化，从而让我们能够实现一些工具帮助我们更好地了解我们的应用。

Vuex 应用的核心完全契合项目开发。

## 组件化开发

**[什么是组件？](https://cn.vuejs.org/v2/guide/components.html)**

组件 (Component) 是 Vue.js 最强大的功能之一。组件可以扩展 HTML 元素，封装可重用的代码。在较高层面上，组件是自定义元素，Vue.js 的编译器为它添加特殊功能。在有些情况下，组件也可以是原生 HTML 元素的形式，以 `is` 特性扩展。

我认为使用组件是有必要的，其必要性在于：
1. 代码复用
2. 代码解耦

![](https://cn.vuejs.org/images/props-events.png)

## [HTML5范围样式 scoped](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/style)

## [CSS属性 fill](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Attribute/fill)

## [for ...of](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/for...of)

**for ...of 循环有以下几个特征：**

* 这是最简洁、最直接的遍历数组元素的语法。
* 这个方法避开了 for ...in 循环的所有缺陷。
* 与 forEach 不同的是，它可以正确响应 break、continue 和 return 语句。
* 其不仅可以遍历数组，还可以遍历类数组对象和其他可迭代对象。

**那 for ...of 到底可以干什么呢？**

* 跟 forEach 相比，可以正确响应 break, continue, return。
* for ...of 循环不仅支持数组，还支持大多数类数组对象，例如 DOM nodelist 对象。
* for ...of 循环也支持字符串遍历，它将字符串视为一系列 Unicode 字符来进行遍历。
* for ...of 也支持 Map 和 Set （两者均为 ES6 中新增的类型）对象遍历。

参考： [《知乎 - 深入了解 JavaScript 中的 for 循环》](https://zhuanlan.zhihu.com/p/23812134)

## 过场动画

参考vux源码：

* [main.js](https://github.com/airyland/vux/blob/v2/src/main.js)
* [App.vue](https://github.com/airyland/vux/blob/v2/src/App.vue)

以下为提取内容：

main.js

```javascript
import Vuex from 'vuex'

Vue.use(Vuex)

let store = new Vuex.Store({})

store.registerModule('vux', {
  state: {
    demoScrollTop: 0,
    isLoading: false,
    direction: 'forward',
  },
  mutations: {
    updateDemoPosition (state, payload) {
      state.demoScrollTop = payload.top
    },
    updateLoadingStatus (state, payload) {
      state.isLoading = payload.isLoading
    },
    updateDirection (state, payload) {
      state.direction = payload.direction
    }
  },
  actions: {
    updateDemoPosition ({commit}, top) {
      commit({type: 'updateDemoPosition', top: top})
    }
  }
})

// simple history management
const history = window.sessionStorage
history.clear()
let historyCount = history.getItem('count') * 1 || 0
history.setItem('/', 0)

router.beforeEach(function (to, from, next) {
  store.commit('updateLoadingStatus', {isLoading: true})

  const toIndex = history.getItem(to.path)
  const fromIndex = history.getItem(from.path)

  if (toIndex) {
    if (!fromIndex || parseInt(toIndex, 10) > parseInt(fromIndex, 10) || (toIndex === '0' && fromIndex === '0')) {
      store.commit('updateDirection', {direction: 'forward'})
    } else {
      store.commit('updateDirection', {direction: 'reverse'})
    }
  } else {
    ++historyCount
    history.setItem('count', historyCount)
    to.path !== '/' && history.setItem(to.path, historyCount)
    store.commit('updateDirection', {direction: 'forward'})
  }

  if (/\/http/.test(to.path)) {
    let url = to.path.split('http')[1]
    window.location.href = `http${url}`
  } else {
    next()
  }
})

new Vue({
  store,
  router,
  render: h => h(App)
}).$mount('#app')

```

App.vue

```html
<template>
  <div id="app">
    <transition :name="'vux-pop-' + (direction === 'forward' ? 'in' : 'out')">
    <!-- @direction 判断动画样式为in或者out -->
      <router-view class="app"></router-view>
    </transition>
  </div>
</template>

<script>
import { mapState } from 'vuex'   //  用引入vuex的mapState

export default {
  name: 'app',
  computed: { //  vuex管理页面
    ...mapState({
      direction: state => state.vux.direction
    })
  }
}
</script>

<style lang='less'>
.vux-pop-out-enter-active,
.vux-pop-out-leave-active,
.vux-pop-in-enter-active,
.vux-pop-in-leave-active {
  will-change: transform;
  transition: all 250ms;
  height: 100%;
  top: 0;
  position: absolute;
  backface-visibility: hidden;
  perspective: 1000;
}
.vux-pop-out-enter {
  opacity: 0;
  transform: translate3d(-100%, 0, 0);
}
.vux-pop-out-leave-active {
  opacity: 0;
  transform: translate3d(100%, 0, 0);
}
.vux-pop-in-enter {
  opacity: 0;
  transform: translate3d(100%, 0, 0);
}
.vux-pop-in-leave-active {
  opacity: 0;
  transform: translate3d(-100%, 0, 0);
}
.weui-btn:after {
  border: none !important;
}
</style>
```

## Webpack 轻度调整

vue-cli已经集成Webpack，执行`npm run build`后，想实现直接运行index.html即可运行页面。修改以下参数即可实现：

build/utils.js

```javascript
ExtractTextPlugin.extract({
  publicPath: '../../'  //  解决CSS样式导出路径问题
})
```

config/index.js

```javascript
build: {
  assetsPublicPath: './'  //  解决JS、CSS路径问题
}
```

修改后`src = ""`，或者`url (...)`使用相对路径即可。

## [vue-infinite-scroll - 下拉加载](https://github.com/ElemeFE/vue-infinite-scroll)

## [ESLint](http://eslint.org/)

这里要提到ESLint是因为，他可以帮助开发者减少开发中一些不必要的错误（除了逻辑），并且在合作开发时，起到大大的作用，所以请拥抱ESLint。

---

好了，先这样吧，我要抓紧学习去了~