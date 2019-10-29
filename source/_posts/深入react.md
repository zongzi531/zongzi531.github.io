---
title: 深入React
date: 2018-02-05 21:53:17
categories: "React"
comments: true
featured_image: logo.svg
tags:
- React
- JavaScript
---

<!-- no node -->

<!-- more -->

>《深入React技术栈》 => 完成
>《你不知道的JavaScript 上卷》 => 进行中
>购入《你不知道的JavaScript 中卷》、《你不知道的JavaScript 下卷》、《CSS世界》
>管理系统PC端及微信小程序端应用上线

## React

先谈谈读后感，目前对于Virtual DOM，函数式编程，专注视图层的开发思想，有了更深入的理解。

简单来说，Virtual DOM配合diff算法，可以实现高效的DOM操作，从而让开发者专注于视图、专注于数据的开发体验。

模块化的开发模式，当前盛为流行，组件化的概念也是同理，那么了解组件的生命周期，以及React无状态组件也是至关重要。

认识了Flux架构模式，像是追溯到了Redux，Vuex之前。感受了这股历史浪潮。

也意识到SSR的存在的重要性，关于为什么使用SSR，我建议可以去阅读[Vue的SSR完全指南](https://ssr.vuejs.org/zh/)

目前，我的项目开发模式已将组件模块和业务模块解耦，组件模块会更专注于页面视图层面的代码逻辑，业务模块则是组合组件模块专注于业务逻辑。

这样的开发模式，更利于代码的迭代、组件的复用、清晰的业务逻辑、易维护的代码等等……，总之就是好。

### 新项目使用到的技术栈：

* `React v16.2.0`
* `React Router v4.2.2` [(react-router-dom)](https://github.com/ReactTraining/react-router/tree/master/packages/react-router-dom)
* [`history v4.7.2`](https://github.com/ReactTraining/history): 配合(react-router-dom)使用，实现路由之间切换等功能
* [`prop-types v15.6.0`](https://reactjs.org/docs/typechecking-with-proptypes.html): 用于类型检查
* [`ant-design-mobile v2.1.3`](https://github.com/ant-design/ant-design-mobile): UI组件库
* `CSS Modules`: 顾名思义，CSS模块，好用没话说
* [`postcss-modules-values v1.3.0`](https://www.npmjs.com/package/postcss-modules-values): 配合CSS模块，可以在CSS文件内部定义变量
* `ESLint`
* `Jest`
* ……

新项目ESlint配置如下：

```javascript
module.exports = {
  root: true,
  parser: 'babel-eslint',
  parserOptions: {
    sourceType: 'module'
  },
  env: {
    browser: true,
    node: true
  },
  extends: 'standard',
  'globals': {
    React: true
  },
  parserOptions: {
    ecmaFeatures: {
      jsx: true
    }
  },
  plugins: [
    'react',
    'import',
    'jsx-a11y'
  ],
  //  https://github.com/yannickcr/eslint-plugin-react
  'rules': {
    //  https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-uses-react.md
    'react/jsx-uses-react': 1,
    //  https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-uses-vars.md
    'react/jsx-uses-vars': 1
  }
}
```

有些遗憾的是，没有实践出像Vue那样的ESlint浏览器提示，只是在VSCode中提示，我查看Vue配置文件找到相关插件[Friendly-errors-webpack-plugin](https://github.com/geowarin/friendly-errors-webpack-plugin)，待后续再试。

有关Jest，暂时未在React实践，只是暂时在自己的项目[vue-to-do-list](https://github.com/zongzi531/vue-to-do-list)进行尝试，但是由于配置问题。输出关于统计的内容并不理想，待后续再试。

### 特别要提到的setState

我相信大家都很熟悉`setState()`方法，但是他的工作机制确实比较有意思，因为他不是同步更新的，而是异步的。

每次使用`setState()`方法更新数据时，其实是将本次要更新的内容提交至一个事务，然后进行批量更新，也就是说是异步的，而非同步。

那么问题来了，为什么要搞懂这个呢，因为他和组件的生命周期息息相关，否则可能导致你的代码产生意想不到的问题，所以知其然，更要知其所以然。

来看一段书中的示例代码：

```javascript
class Example extends Component {
  constructor (props) {
    super(props)
    this.state = { val: 0 }
  }

  componentDidMount () {
    this.setState({ val: this.state.val + 1 })
    console.log(this.state.val)  // 第一次输出

    this.setState({ val: this.state.val + 1 })
    console.log(this.state.val)  // 第二次输出

    setTimeout(() => {
      this.setState({ val: this.state.val + 1 })
      console.log(this.state.val)  // 第三次输出

      this.setState({ val: this.state.val + 1 })
      console.log(this.state.val)  // 第四次输出
    }, 0)
  }
}
```

最终输出的结果，我相信大家已经实践出来了，确实很出乎意料。

我建议可以阅读一下[setState()](https://reactjs.org/docs/react-component.html#setstate)。

这段内容我相信对你理解代码会有很大的帮助：

>This form of setState() is also asynchronous, and multiple calls during the same cycle may be batched together. For example, if you attempt to increment an item quantity more than once in the same cycle, that will result in the equivalent of:

```javascript
Object.assign(
  previousState,
  {quantity: state.quantity + 1},
  {quantity: state.quantity + 1},
  ...
)
```

当然官方也提供了解决方案：

>Subsequent calls will override values from previous calls in the same cycle, so the quantity will only be incremented once. If the next state depends on the previous state, we recommend using the updater function form, instead:

```javascript
this.setState((prevState) => {
  return {quantity: prevState.quantity + 1};
});
```

[FAQ State](https://reactjs.org/docs/faq-state.html)

## 重新认识JavaScript

截止目前，《你不知道的JavaScript 上卷》学习到的内容，我认为我正在重新认识JavaScript。

简单提一下，目前我认为有必要掌握的知识点：

1. 编译器、引擎和作用域之间的关系
2. 词法作用域和动态作用域的工作原理

了解以上两点，会让你对闭包有新的认识！

## 附：

[管理系统PC端 IE9 input标签 实现placeholder属性的解决方案](https://segmentfault.com/a/1190000011172434)