---
title: 进阶ES6（二）
date: 2017-11-25 18:22:55
categories: "JavaScript"
comments: true
featured_image: pic.jpeg
tags:
- JavaScript
- ECMAScript 6
---

<!-- no node -->

<!-- more -->

>**概要**
>计划本月读完《ECMAScript 6 标准入门》（第3版） => 完成
>购入新书《学习JavaScript数据结构与算法》和《深入React技术栈》
>计划12月读完《学习JavaScript数据结构与算法》
>计划12月入门React
>[前文](http://zongzi531.github.io/2017/10/12/%E8%BF%9B%E9%98%B6es6/)中提及的PC端管理系统（vue + element-ui）[以下简称：项目B]
>11月中旬团队开发工行大厂APP第三方页面积分商城项目 [以下简称：项目C]

## Vue-AMap

接触此组件并紧接着参加饿了么大前端秋季交流会后，对为主流框架提供现有其他服务组件的项目有了新的认识。详细可以查看[《浅谈第三方服务与 Vue 的结合》](http://zongzi531.github.io/2017/11/05/%E5%8F%82%E5%8A%A02017%E9%A5%BF%E4%BA%86%E4%B9%88%E5%A4%A7%E5%89%8D%E7%AB%AF%E7%A7%8B%E5%AD%A3%E4%BA%A4%E6%B5%81%E4%BC%9A/)

这里浅谈下使用Vue-AMap的感受。地图业务基本是异步业务，我使用`Promise对象`和 `async函数`来完成我的业务需求。

此外Vue-AMap提供[lazyAMapApiLoaderInstance.load](https://elemefe.github.io/vue-amap/#/zh-cn/introduction/init?id=promise)方法返回`Promise对象`为开发者带来便利。

```javascript
/**
 * **代码实践**
 * locate 和 cloudSearchNearBy 函数传入AMap实例调用高德地图API并返回Promise对象
 */
import { lazyAMapApiLoaderInstance } from 'vue-amap'

lazyAMapApiLoaderInstance.load().then(() => {
let init = async function () {
  await locate(AMap).catch((e) => {
    // ...定位失败处理
  })
  await cloudSearchNearBy(AMap, ...)
}
init()
})
```

## 团队开发

项目C开发过程中更深刻体会到以下几点重要性：
1. ESLint 代码风格约定
2. git commit风格约定
3. 代码版本管理
4. 团队分工安排
5. 项目前期需求分析

同时祝贺下，项目C已正式上线！

## CommonJS VS ES6 Module

简单总结下两者差异：

| 模块加载方案        | CommonJS         |  ES6 Module     |
| --------          | :----:           | :----:          |
| 加载模式差异        | 运行时加载        |   编译时加载     |
| 输出内容差异        | 输出值的复制      |   输出值的引用    |
| 加载内容差异        | 模块整体加载      |   模块需要加载    |

编译时加载带来的好处：
* 不再需要UMD模块格式，将来服务器和浏览器都会支持ES6模块格式。
* 将来浏览器的新API可以用模块格式提供，不在需要做成全局变量或者navigator对象属性。
* 不再需要对象作为命名空间（比如Math对象），未来这些功能可以通过模块来提供。

### 项目B

**实践ES6 Module**


1. 表单验证方法集合对象: [element-ui 表单验证方法集合对象](https://github.com/zongzi531/ZGadget/blob/master/element/element-ui-form-validator.js)

2. 国内城市选择器: [element-ui 级联选择器 城市数据](https://github.com/zongzi531/ZGadget/blob/master/element/element-ui-Cascader-china-address-v3.js)

```javascript
/**
 * HOW TO USE
 *
 * 1.import ca from './element-ui-Cascader-china-address-v3.js'
 * 2.<el-cascader :options="ca" ... > ... </el-cascader>
 */
```

### 微信小程序

微信小程序目前似乎不支持ES6 Module，而是采用CommonJS模块加载方案。

```javascript
// 官方文档示例代码
var common = require('common.js')
Page({
  helloMINA: function() {
    common.sayHello('MINA')
  },
  goodbyeMINA: function() {
    common.sayGoodbye('MINA')
  }
})
```

详细文档内容请参见[微信小程序-模块化](https://mp.weixin.qq.com/debug/wxadoc/dev/framework/app-service/module.html)

## Key值

key必须在其兄弟节点中是唯一的，而非全局唯一。万不得已，你可以传递他们在数组中的索引作为key。若元素没有重排，该方法效果不错，但重排会使得其变慢。

**权衡**

牢记协调算法的实现细节非常重要。React可能会在每次操作时渲染整个应用；而结果仍是相同的。为保证大多数场景效率能更快，我们通常提炼启发式的算法。

在目前实现中，可以表明一个事实，即子树在其兄弟节点中移动，但你无法告知其移动到哪。该算法会重渲整个子树。

由于React依赖于该启发式算法，若其背后的假设没得到满足，则其性能将会受到影响：

1. 算法无法尝试匹配不同组件类型的子元素。若你发现两个输出非常相似的组件类型交替出现，你可能希望使其成为相同类型。实践中，我们并非发现这是一个问题。
2. Keys应该是稳定的，可预测的，且唯一的。不稳定的key（类似由Math.random()生成的）将使得大量组件实例和DOM节点进行不必要的重建，使得性能下降并丢失子组件的状态。

相关资料：

* [Vue特殊特性Key](https://cn.vuejs.org/v2/api/#key)
* [React Key](https://doc.react-china.org/docs/lists-and-keys.html#keys)
* [深度解析Key的重要性](https://doc.react-china.org/docs/reconciliation.html#%E9%80%92%E5%BD%92%E5%AD%90%E8%8A%82%E7%82%B9)


## Vuex

论优雅的使用Vuex重要性，请认真阅读Vuex官方文档。

每一个 Vuex 应用的核心就是 store（仓库）。“store”基本上就是一个容器，它包含着你的应用中大部分的状态 (state)。Vuex 和单纯的全局对象有以下两点不同：

1. Vuex 的状态存储是响应式的。当 Vue 组件从 store 中读取状态的时候，若 store 中的状态发生变化，那么相应的组件也会相应地得到高效更新。

2. 你**不能直接改变** store 中的状态。改变 store 中的状态的唯一途径就是**显式地提交 (commit) mutation**。这样使得我们可以方便地跟踪每一个状态的变化，从而让我们能够实现一些工具帮助我们更好地了解我们的应用。

```javascript
// 如果在模块化构建系统中，请确保在开头调用了 Vue.use(Vuex)

const store = new Vuex.Store({
  state: {
    count: 0
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
})

// 不优雅
store.state.count++

// 优雅
store.commit('increment')
```

那么优雅的使用Vuex就是：
* 同步操作: store.commit方法提交mutation
* 异步操作: store.dispatch方法触发action

### element-ui 清空所有表单项的验证信息

在优雅的使用Vuex时，考虑到表单操作都要采用显式地提交 (commit) mutation来实现，通过element-ui提供的resetFields方法，会在Vuex严格模式下报错。

查看源码会发现，可以单独提出一个清空所有表单项的验证信息的方法，实践如下：

```javascript
/**
* clearValidate 用于element-ui V1.4.10 clearValidate 方法，用于清空所有表单项的验证信息
* @param  {Object} formName 传入this.$refs[formName]
* @return
*/
function clearValidate (formName) {
  if (formName) {
    let fields = formName.fields
    for (let item of fields) {
      item.validateState = ''
      item.validateMessage = ''
      item.validateDisabled = false
    }
  }
}
```

element-ui 2.0.0 已经新增此方法 [#7623](https://github.com/ElemeFE/element/pull/7623)

---

我意识到ES6了解完后需要更深入的理解JavaScript，对函数式编程的向往促使我去看React。

我认为现在还不是时机去学习node，我准备在更了解ES6，更了解JavaScript，那时，才是学习node的好时机。
