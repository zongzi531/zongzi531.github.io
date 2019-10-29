---
title: 进阶ES6
date: 2017-10-12 21:57:25
categories: "JavaScript"
comments: true
thumbnail: /gallery/进阶es6/pic.jpeg
tags:
- JavaScript
- ECMAScript 6
---

<!-- no node -->

<!-- more -->

依然是好久不见，因为之前安排的工作及学习进度比较紧，所以选择在国庆后进行一些总结。

>**概要**
>8月中旬至9月中旬开发公司移动端项目（vue + vux）[以下简称：项目A]
>9月底至今开发公司PC端管理系统（vue + element-ui）[以下简称：项目B]
>一些常用的技术栈我这里就不在提了
>那么这一期间，我已经阅读完《ECMAScript 6 标准入门》（第2版）以及《深入理解ES6》，其实读完并不代表什么，必须得学以致用嘛
>9月底《ECMAScript 6 标准入门》（第3版）也到货了，他比之前厚了200多页，这使我感到很高兴，他包含了ES2015、ES2016、ES2017这3个版本，我准备重新再撸一遍，爽一爽

**以下，我将对这期间我认为好的内容进行总结：**

## 实践组件化、模块化的开发模式

### 组件化

从之前的项目经验中，我意识到，要让项目解耦是必要的，这样一来是的你的项目更易维护、逻辑更清晰、提升复用性等等好处，这些是不容置疑的。

我开始尝试，将vue文件中的重复或者细小可以复用的内容，进行组件化。

这些小的被我组件化的文件很方便使用，在需要的时候直接引入即可，如果需要从父组件向子组件传递数据（`props`）或者触发子组件的事件（`vm.$emit`），但是我还未尝试组件与CSS进行剥离。在我看来，在项目并不大的情况，或者公司项目不多的情况下，不同项目之间的组件复用情况，我暂时还未遇到，但我相信这是存在的，并且我一直准备着。

### 模块化

在项目A中，使用`Selector`或者`Picker`，会使用到大量单列数据、二级联动数据，这些数据会使得组件体积变大，并且不易维护，我将其数据部分提出放入JSON文件中，在需要引用时进行引入即可，并且如需对数据进行修改或者调整，只需要单独维护JSON文件即可，这大大提高了项目的可维护性。

但是JSON文件不能加注释，这是我感到很困惑。

在项目B中，我使用ES6标准推荐的ES6 Module来实现我的模块化，比如将element-ui Table组件按照我的需求进行个性化的定制，提取配置文件至`/libs`，并且此类文件可以不断扩充，并且适合于与其他组件进行组合使用，和进行统一维护管理。

可以查看我的源码：

1. [element-ui 表单验证方法集合对象](https://github.com/zongzi531/ZGadget/blob/master/element/element-ui-form-validator.js)

2. [element-ui Table 个性化定制](https://github.com/zongzi531/management_platform/tree/master/src/libs)

## 实践撰写良好的注释

由于采用了组件化、模块化的开发模式，撰写注释我当其为一种乐趣，它使我的代码更易被更多开发者读懂，我认为这很优美。

那么我撰写注释会遵循以下规则：

### 在组件顶部或者公共函数上方

```javascript
/**
 * [description]
 * @param  {[type]}   param  [description]
 * @return {[type]}          [description]
 */
```

### 在必要的段落中

```javascript
//  ...
```

## 优化方面的一些实践

在开发项目A的过程中，我意识到，复杂的项目需要进行优化。

### vue-router

[vue-router 路由懒加载](https://github.com/vuejs/vue-router/blob/dev/docs/zh-cn/advanced/lazy-loading.md) ，因为项目不大，使用此功能只会让页面加载速度较之前相比缓慢些，并且增加HTTP请求，十分不划算，但是我认为在大项目中，这是必不可少的功能。

### 减轻vender文件大小

打包时不打包vue、vuex、vue-router、axios等，将其换用CDN直接引入到index.html中，在webpack里有个`externals`，可以忽略不需要打包的库：

```javascript
externals: {
  'vue': 'Vue',
  'vue-router': 'VueRouter',
  'vuex': 'Vuex',
  'axios': 'axios'
}
```

网上可以搜索到很多关于webpack打包优化方案，可以进行参考。

### 减少服务器请求

尝试使用将图片文件转换成Base64格式，但是这会使页面响应时间变长，影响效率。其实服务器响应头中设置缓存也是一种不错的选择（`js`、`css`、`image`等等），虽然用户在第一次进入页面时，可能时间会稍长，但是用户再次进入时，将直接从本地缓存中读取缓存文件（服务器未更新文件的前提下），这也大大的提高了进入页面的速度，并且减轻了服务器的负担。

```
Response Headers:
Cache-Control: public, max-age=31536000
```

### 兼容性优化

1. 软垫片 `polyfill`

2. 兼容new URLSearchParams () `url-search-params-polyfill`

## 实践一些ES6的新特性

开发过程中，我会经常使用到 `let`、`const`、`解构赋值`、`...`、`箭头函数`、`for ...of`、`Module`。

提到`for ...of`，有趣的是，我和好友讨论了遍历数组到底哪个更快，我们测试遍历100000长度的数组，并进行赋值操作。

结果是，古老的`for`循环当然是最快的，在20组测试样例中， `for ...of` < `forEach` < `for ...in`。

还有`箭头函数`需要注意的是`this`的作用域哟，这个书中都有详细的介绍。

---

国庆长假，我去了日本，也领悟了不少，看到了不少。接下来我将忙于工作，学习！