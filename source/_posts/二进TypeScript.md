---
title: 二进TypeScript
date: 2018-12-01 00:07:48
categories: "TypeScript"
comments: true
thumbnail: /gallery/二进TypeScript/foreground_bluprint.svg
tags:
- TypeScript
---

<!-- no node -->

<!-- more -->

其实从国庆假期前就开始慢慢二读 TypeScript 了。只不过那时因为计划安排，先学习了面试经验。之后决定二读是在11月。

在二读前，我选择先阅读了[《深入理解 TypeScript》](https://github.com/jkchao/typescript-book-chinese)，因为我很清楚的知道，自己对 TypeScript 的了解并不是很深。在阅读完后，我认为需要重新去阅读 TypeScript 。当然我的英语水平并不是那么的好，所以为了减轻我的阅读压力，我选择阅读[《TypeScript 使用手册》](https://github.com/zhongsp/TypeScript)。二读让我对 TypeScript 了解了更多。当然我相信，这两本书我会读第三次。

> 阅读过程中，相继对以上两个仓库进行了不通程度的贡献，比如：错误更正，帮助翻译。*相关PR我会列在文章尾部*

那么为了验证我二读的效果，我开始对我 GitHub 原有的 JavaScript 项目进行 TypeScript 重写。分别对以下仓库开立 `ts` 分支，进行重写：

- [x] [zongzi531/react-to-do-list](https://github.com/zongzi531/react-to-do-list)
- [x] [zongzi531/vue-to-do-list](https://github.com/zongzi531/vue-to-do-list)
- [x] [zongzi531/express-mongo-to-do-list-server](https://github.com/zongzi531/express-mongo-to-do-list-server)

那么接下来分享我的学习经验：

## 关于 `type` 关键字

其实之前我一直不理解 `type` 关键字到底是做什么用的，直到二读时，我发现它和 `interface` 很相像，但是又有细微的不同。

首先来介绍下 `type` ：

它在 TypeScript 1.4 被发布，称它为**类型别名**。

你现在可以使用 `type` 关键字来为类型定义一个“别名”：

```typescript
type PrimitiveArray = Array<string|number|boolean>;
type MyNumber = number;
type NgScope = ng.IScope;
type Callback = () => void;
```

类型别名与其原始的类型完全一致；它们只是简单的替代名。接下来看看它与 `interface` 的不同之处。

其一，接口创建了一个新的名字，可以在其它任何地方使用。 类型别名并不创建新名字—比如，错误信息就不会使用别名。 在下面的示例代码里，在编译器中将鼠标悬停在 `interfaced` 上，显示它返回的是 `Interface` ，但悬停在 `aliased` 上时，显示的却是对象字面量类型。

```typescript
type Alias = { num: number }
interface Interface {
    num: number;
}
declare function aliased(arg: Alias): Alias;
declare function interfaced(arg: Interface): Interface;
```

另一个重要区别是类型别名不能被 `extends` 和 `implements` （自己也不能 `extends` 和 `implements` 其它类型）。 因为软件中的对象应该对于扩展是开放的，但是对于修改是封闭的，你应该尽量去使用接口代替类型别名。

另一方面，如果你无法通过接口来描述一个类型并且需要使用联合类型或元组类型，这时通常会使用类型别名。

## 关于 `keyof` 关键字

这个关键字就比较厉害了，我一读的时候根本就没有注意到！！！为什么没注意到其实也有一定的原因，那时阅读的是[《TypeScript中文网》](https://www.tslang.cn/)，在该网站中直接缺失 `<T, K extends keyof T>` 。气不气！！！而且没地方提PR……捞不捞！！！所以还是推荐阅读官网文档啦。

二读时，我产生疑惑的位置在这里：

### 在泛型约束中使用类型参数

你可以声明一个类型参数，且它被另一个类型参数所约束。 比如，现在我们想要用属性名从对象里获取这个属性。 并且我们想要确保这个属性存在于对象 `obj` 上，因此我们需要在这两个类型之间使用约束。

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K) {
    return obj[key];
}

let x = { a: 1, b: 2, c: 3, d: 4 };

getProperty(x, "a"); // okay
getProperty(x, "m"); // error: Argument of type 'm' isn't assignable to 'a' | 'b' | 'c' | 'd'.
```

来介绍下 `keyof` ：

它在 TypeScript 2.1 被发布，称它为**查找类型**。

在 JavaScript 中属性名称作为参数的API是相当普遍的，但是到目前为止还没有表达在那些API中出现的类型关系。

输入索引类型查询或 `keyof` ，索引类型查询 `keyof T` 产生的类型是 `T` 的属性名称。 `keyof T` 的类型被认为是 `string` 的子类型。

```typescript
interface Person {
    name: string;
    age: number;
    location: string;
}

type K1 = keyof Person; // "name" | "age" | "location"
type K2 = keyof Person[];  // "length" | "push" | "pop" | "concat" | ...
type K3 = keyof { [x: string]: Person };  // string
```

与之相对应的是*索引访问类型*，也称为*查找类型*。在语法上，它们看起来像元素访问，但是写成类型：

```typescript
type P1 = Person["name"];  // string
type P2 = Person["name" | "age"];  // string | number
type P3 = string["charAt"];  // (pos: number) => string
type P4 = string[]["push"];  // (...items: string[]) => number
type P5 = string[][0];  // string
```

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K) {
    return obj[key];  // 推断类型是T[K]
}

function setProperty<T, K extends keyof T>(obj: T, key: K, value: T[K]) {
    obj[key] = value;
}

let x = { foo: 10, bar: "hello!" };

let foo = getProperty(x, "foo"); // number
let bar = getProperty(x, "bar"); // string

let oops = getProperty(x, "wargarbl"); // 错误！"wargarbl"不存在"foo" | "bar"中

setProperty(x, "foo", "string"); // 错误！, 类型是number而非string
```

## React To Do List

对之前的 TypeScript 版本进行了相应的优化，出于某种原因，将 `.ts` 改为 `.tsx` 文件。

将部分组件替换为无状态组件，书写方式基本如下（以 `App.tsx` 为例）：

```jsx
// https://github.com/zongzi531/react-to-do-list/blob/master/src/App.tsx

import * as React from 'react'
import Routes from './routes'

type App = () => JSX.Element

const App: App = () => (<Routes />)

export default App
```

将之前申明的静态变量替换为枚举：

```typescript
// https://github.com/zongzi531/react-to-do-list/blob/master/src/config/index.ts

export enum COLOR {
  DEFAULT = 'default',
  // ...
}

export interface ICOLOR {
  color: COLOR
  flag: COLORFLAG
}

export type COLORS = ICOLOR[]

export const COLORS: COLORS = [
  { color: COLOR.DEFAULT, flag: COLORFLAG.ACTIVE },
  // ...
]
```

更多的使用类型别名：

```typescript
// https://github.com/zongzi531/react-to-do-list/blob/master/src/interfaces/index.ts

import { RouterProps } from 'react-router'
import { FormComponentProps } from 'antd/lib/form'

export type AntdFormAndRouterProps = FormComponentProps & RouterProps
```

## [Vue To Do List](https://zongzi531.com/vue-to-do-list)

使用 `Emit` 装饰器实现以前的 `this.$emit` 方法：

```typescript
// https://github.com/zongzi531/vue-to-do-list/blob/master/src/components/Navigation.vue

/**
 * <template>
 *   <v-navigation-drawer light v-model="currentShow" />
 * </template>
 */

import { Component, Prop, Emit, Vue } from 'vue-property-decorator';

@Component
export default class Navigation extends Vue {

  // show 为组件外传入参数
  // 因为 v-model 会直接修改值
  // 所以我们定义一个 currentShow 的 getter 和 setter
  @Prop({ default: false })
  private show!: boolean;

  // getter 当然是获取外部传入的 show
  private get currentShow() { return this.show; }

  // setter 因为没有办法 return ，所以这里我们只需一个内部方法 setCurrentShow 将 value 传入
  private set currentShow(value) {
    this.setCurrentShow(value);
  }

  // 这里使用 Emit decorator 在方法内使用 return 将 value 传出
  // 这样在组件外就可以使用 @on-change 绑定方法， 方法内第一参数就是 value
  @Emit('on-change')
  private setCurrentShow(value: boolean) {
    return value;
  }
}
```

在 `.vue` 中 `import *.svg` 的解决方法：

在 `shims-vue.d.ts` 中添加以下代码：

```typescript
declare module '*.svg' {
  const value: any;
  export default value;
}
```

在使用 GitHub Pages 时遇到的问题，有关于 Vue CLI 3：

部署应用包时的基本 URL。用法和 webpack 本身的 `output.publicPath` 一致，但是 Vue CLI 在一些其他地方也需要用到这个值，所以**请始终使用 `baseUrl` 而不要直接修改 webpack 的 `output.publicPath`** 。

默认情况下，Vue CLI 会假设你的应用是被部署在一个域名的根路径上，例如 `https://www.my-app.com/` 。如果应用被部署在一个子路径上，你就需要用这个选项指定这个子路径。例如，如果你的应用被部署在 `https://www.my-app.com/my-app/`，则设置 `baseUrl` 为 `/my-app/`。

这个值也可以被设置为空字符串 (`''`) 或是相对路径 (`'./'`)，这样所有的资源都会被链接为相对路径，这样打出来的包可以被部署在任意路径，也可以用在类似 Cordova hybrid 应用的文件系统中。

>**相对 baseUrl 的限制**
>
>相对路径的 `baseUrl` 有一些使用上的限制。在以下情况下，应当避免使用相对 `baseUrl`:
>
>- 当使用基于 HTML5 `history.pushState` 的路由时；
>- 当使用 `pages` 选项构建多页面应用时。

这个值在开发环境下同样生效。如果你想把开发服务器架设在根路径，你可以使用一个条件式的值：

```javascript
// https://github.com/zongzi531/vue-to-do-list/blob/master/vue.config.js

module.exports = {
  baseUrl: process.env.NODE_ENV === 'production'
    ? '/vue-to-do-list/'
    : '/'
}
```

## 执行 `git push -u origin master` 提示 `remote: Invalid username or password.`

看这个提示意思是用户名或密码无效，那么问题来了，明明没数错啊，为什么？

前提：如果你的 GitHub 账户开启了二步验证，请直接前往[Creating a personal access token for the command line](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/)按照提示操作，即可解决。

## 自定义 `index.html` 插入内容

`html-webpack-plugin` 插件对应的数据。它包括两部分：

`htmlWebpackPlugin.files` : 此次html-webpack-plugin插件配置的chunk和抽取的css样式。该files值其实是webpack的stats对象的 `assetsByChunkName` 属性代表的值，该值是插件配置的chunk块对应的按照 `webpackConfig.output.filename` 映射的值。例如对应上面配置插件各个属性配置项例子中生成的数据格式如下：

```json
"htmlWebpackPlugin": {
  "files": {
    "css": [ "inex.css" ],
    "js": [ "common.js", "index.js"],
    "chunks": {
      "common": {
        "entry": "common.js",
        "css": [ "index.css" ]
      },
      "index": {
        "entry": "index.js",
        "css": ["index.css"]
      }
    }
  }
}
```

这样，就可以是用如下模板引擎来动态输出script脚本：

```html
<% for (var chunk in htmlWebpackPlugin.files.chunks) { %>
<script type="text/javascript" src="<%=htmlWebpackPlugin.files.chunks[chunk].entry %>"></script>
<% } %>
```

`htmlWebpackPlugin.options`: 传递给插件的配置项，具体的配置项如上面插件配置项小节所描述的 —— *参见相关链接*。

## Pull requests

**[《深入理解 TypeScript》](https://github.com/jkchao/typescript-book-chinese)**

1. [Fix typo #19](https://github.com/jkchao/typescript-book-chinese/pull/19)
2. [Fix README commits link #31](https://github.com/jkchao/typescript-book-chinese/pull/31)
3. [Fix typo #37](https://github.com/jkchao/typescript-book-chinese/pull/37)
4. [Fix typo #39](https://github.com/jkchao/typescript-book-chinese/pull/39)
5. [Fix typo #40](https://github.com/jkchao/typescript-book-chinese/pull/40)

**[《TypeScript 使用手册》](https://github.com/zhongsp/TypeScript)**

1. [Update Advanced Types.md & Enums.md #228](https://github.com/zhongsp/TypeScript/pull/228)
2. [Update Breaking changes in TypeScript 2.6 and TypeScript 2.7 #229](https://github.com/zhongsp/TypeScript/pull/229)
3. [Update Breaking changes in TypeScript 2.8 #234](https://github.com/zhongsp/TypeScript/pull/234)

## 相关链接

- [TypeScript React Starter](https://github.com/Microsoft/TypeScript-React-Starter#typescript-react-starter)
- [TypeScript Vue Starter](https://github.com/Microsoft/TypeScript-Vue-Starter#typescript-vue-starter)
- [TypeScript Node Starter](https://github.com/Microsoft/TypeScript-Node-Starter#typescript-node-starter)
- [webpack not able to import images( using express and angular2 in typescript)](https://stackoverflow.com/questions/36148639/webpack-not-able-to-import-images-using-express-and-angular2-in-typescript/36151803#36151803)
- [Vue 中使用 typescript](http://www.cnblogs.com/stone-lyl/p/9542126.html)
- [在Vue中使用TypeScript](https://www.jianshu.com/p/cd475fee2794)
- [Creating a personal access token for the command line](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/)
- [html-webpack-plugin详解](http://www.cnblogs.com/wonyun/p/6030090.html)
