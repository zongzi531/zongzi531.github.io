---
title: VUE·CONF HANGZHOU 2018
date: 2018-11-24 17:45:28
update: 2018-11-30 19:55:37
categories: "Vue"
comments: true
thumbnail: /gallery/vueconf-hangzhou-2018/logo.svg
tags:
- Vue
---

<!-- no node -->

<!-- more -->

>11.24 / 杭州
>第二届Vue.js开发者大会由Vue.js官方举办，将于2018年11月24日在杭州金逸影视中心IMAX店正式开启。

## Vue 3.0 进展

>演讲者：尤雨溪（远程参与）
>Vue.js 作者，前 Google Creative Lab 成员。

在场听尤大讲时非常震撼，所以会后回顾[视频](https://www.bilibili.com/video/av36787459/)内容进行相应的整理。

[Vue 3.0 进展分享](https://img.w3ctech.com/Vue3.0Updates.pdf)

{% iframe //player.bilibili.com/player.html?aid=36787459&cid=64599487&page=1 %}

- 更多编译时的优化，以减少运行时的开销

简单来说，就是通过对目前框架的现状分析，对模板编译进行优化，来减少运行时（runtime）的开销，也就是利用空间换取时间的概念。

比如说在`<template>...</template>`（以下称：模板）中会有很多不会变动的地方，但是在 Virtual DOM 他不可避免的要重新生成这些对应的节点，然后对他们进行对比，这些操作在很多情况下实际上是不必要的。那么就可以从这个点对模板编译进行优化，关键点在于将模板编译成 Virtual DOM 渲染函数时，对模板中不会变动的地方做了更多的处理，比如对组件和原生HTML元素在编译时进行处理，生成对应的运行时代码，来减少运行时开销，这就是所谓的 Component Fast Path。

同样的，就是在生成这些运行时的渲染函数时，我们尽量的保持这些函数参数个数的一致（我们称作：Monomorphic calls），这样更易于被 JavaScript 引擎优化，这属于比较底层的优化技巧。

再是我们可以做的是在模板中分析这个元素是否包含子元素以及这个子元素的类型，为运行时留下一些 hint “提示”（Children type detection）。比如直接告诉渲染函数，这个元素只有1个子元素，那么渲染函数中在对应的算法分支，来提升运行时的执行效率，可以跳过很多其他不必要的判断，积少成多效果也就明显了。

Component Fast Path + Monomorphic calls + Children type detection

```html Template
<Comp></Comp>
<div>
  <span></span>
</div>
```

```javascript Compiler output
render() {
  const Comp = resloveComponent('Comp', this)
  return createFragment([
    createComponentVNode(Comp, null, null, 0 /* no children */),
    createElementVNode('div', null, [
      createElementVNode('span', null, null, 0 /* no children */)
    ], 2 /* single vnode child */)
  ], 8 /* multiple non-keyed children */)
}
```

- 优化 Slots 生成

目前，每次子组件变动的时候，父组件也会进行变动，有时候为了更新子组件内容，父组件其余内容并没有变化，而让父组件进行不必要的更新，浪费运行时开销。这次优化将 slot 生成为一个函数，这个函数的特点是 lazy 的，当你把函数传给子组件时，由子组件来决定什么时候来调用这个函数，这样就只需要重新渲染子组件即可，这样呢在整个应用中就能够得到一个非常精确的组件级的依赖收集，可以进一步的避免不必要的组件渲染。

```html Template
<Comp>
  <div>{{ hello }}</div>
</Comp>
```

```javascript Compiler output
render() {
  return h(Comp, null, {
    default: () => [h('div', this.hello)]
  }, 16 /* compiler generated slots */)
}
```

- 对于静态内容、静态属性、内联事件的函数进行提取，在编译时检查模板是不会变的，把他提取出来，在之后的更新可以复用，也就是把这些存在 cache 里，从而直接跳过比对树，以实现运行时优化。在此展示内联事件的函数示例：

```html Template
<Comp @event="count++" />
```

```javascript Compiler output
import { getBoundMethod } from 'vue'

function __fn1() {
  this.count++
}

render() {
  return h(Comp, {
    onEvent: getBoundMethod(__fn1, this)
  })
}
```

- 基于 Proxy 的新数据监听系统（Lazy by default）

利用 Proxy 减少组件实例初始化开销 和 Object.defineProperty 说再见，因为大量的 Object.defineProperty 是一个相对昂贵的操作。

- 利用 tree-shaking 来减小打包体积

将 Vue 包的体积变小，利用 tree-shaking 来减小打包体积，移除不必要的功能。将各种工具函数做成按需引入。

- 内部模块解耦

  - 易于维护
  - 易于阅读源码（TypeScript类型信息提示）
  - 易于PR

- 更好的为各种IDE工具链铺路

编译器重构，实现插件化，将各种逻辑做成一个个解耦的小插件， parser 用来提供位置信息的，生成 source map ，报错的时候，能够直接指向模板。

- 更好的多端渲染支持

```javascript Custom Renderer API
import { createRenderer } from '@vue/runtime-core'

const { render } = createRenderer({
  nodeOps,
  patchData
})
```

- 响应式的数据监听 API （[有点像mobx？](https://github.com/SangKa/MobX-Docs-CN)）

```javascript 响应式的数据监听 API
import { observable, effect } from 'vue'

const state = observable({
  count: 0
})

effect(() => {
  console.log(`count is: ${state.count}`)
}) // count is: 0

state.count++ //  count is: 1
```

- 轻松排查组件更新的触发原因

```javascript renderTriggered
const Comp = {
  render(props) {
    return h('div', props.count)
  },
  renderTriggered(event) {
    debugger
  }
}
```

- 更好的 TypeScript 支持，包括原生的 Class API 和 TSX

不需要再去依赖 Vue Class Component 这个库了。

```typescript TSX
interface HelloProps {
  text: string
}

class Hello extends Component<HelloProps> {
  count = 0

  render() {
    return <div>
      {this.count}
      {this.$props.text}
    </div>
  }
}
```

- Time Slicing Support

这个概念很有意思，相当于将 JavaScript 计算切成一块一块的，一帧一帧的去执行，每执行完一帧 `yield` 给浏览器，让浏览器接受用户事件等等。

实现此功能的 API：[window.requestAnimationFrame](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestAnimationFrame)

根据官网 DEMO 稍适修改，可以发现一帧在 16 - 17 ms 之间。

```javascript
var start = null;

function step(timestamp) {
  if (!start) start = timestamp;
  var progress = timestamp - start;
  console.log(Math.floor(progress));
  window.requestAnimationFrame(step);
}

window.requestAnimationFrame(step);
```

那么为什么会出现这样的情况其实也很有意思，因为 JavaScript 是单线程的，当 JavaScript 进行大量的计算时，会导致 CPU 完全使用的情况，发生这种情况时，浏览器会被 block ，也就是没有响应。所以框架采用这个思路解决方案真的很 NICE。

其实在平常业务当中也有相应的解决方案，是另外一种思路，大量的 JavaScript 计算交由子线程（子进程）去完成，在计算完成后通知主线程（主进程），这在浏览器（Web Worker）和 Node.js（child_process）都有相应的实现。

---

>还有一些视频中提到的点，这里并未列出，详细请看视频。

## 挖掘Vue的声明式交互能力

>演讲者：程劭非（Winter）
>“计算机之子”
>
>声明式与命令式设计是Vue和React的核心区别之一，我的分享中将会从几个方面来探讨如何挖掘声明式编程的优势，分别包括：声明式与双向绑定，声明式与交互，声明式与递归

[挖掘Vue的声明式交互能力](https://img.w3ctech.com/vueconf-winter.pdf)

{% iframe //player.bilibili.com/player.html?aid=37345007&cid=65636472&page=1 %}

可以说 Winter 带我学习了“另一方面”的计算机知识，他在演讲中有提到声明式和命令式（过程式），我当场并没有理解，在观看视频后（看的仍然犯困），简单来介绍下：

- 命令式编程的主要思想是关注计算机执行的步骤，即一步一步告诉计算机先做什么再做什么。
- 声明式编程是以数据结构的形式来表达程序执行的逻辑。它的主要思想是告诉计算机应该做什么，但不指定具体要怎么做。

这么说起来，命令式编程确实很像过程式编程，告诉计算机一步一步做什么，在看到声明式编程其实也很像函数式编程。

- 函数式编程和声明式编程是有所关联的，因为他们思想是一致的：即只关注做什么而不是怎么做。但函数式编程不仅仅局限于声明式编程。函数式编程最重要的特点是“函数第一位”，即函数可以出现在任何地方，比如你可以把函数作为参数传递给另一个函数，不仅如此你还可以将函数作为返回值。

Winter 利用 Vue Template 实现了一些图灵完备性的特征，使用组件调用自身实现例如费波那奇数列，其实说的实在点就是函数自调。

那么关于声明式和命令式其实，在视图层面，声明式的优势在于，你可以很快的找到这块视图区域对应的代码。

谈到交互，把用户输入的内容以 UI 的形式呈现时，这就是交互，基于 Vue，可以通过 Vue Directive 来实现用户输入到 UI 呈现中间的过程，他可以比作他们中间的桥梁。

Winter 介绍了一个关于 UI 性能提升的例子 recycle-list ，值得学习。

## What I Learned from Maintaining Vue CLI

>演讲者：蒋豪群
>Vue.js 官方团队成员，Vue CLI 维护者。曾在阿里巴巴、蘑菇街就职。
>
>本次演讲将给大家分享 Vue CLI 的一些设计思路、Vue 生态圈的 tooling 发展和如何更好地跟进这些新技术。

[What I Learned from Maintaining Vue CLI](https://img.w3ctech.com/What-I-Learned-from-Maintaining-Vue-CLI.pdf)

{% iframe //player.bilibili.com/player.html?aid=37343145&cid=65632951&page=1 %}

先说说 CLI 的职责：创建项目，本地开发，打包。

开场先介绍了 CLI 3，以及之所以开发 CLI 3 是为了解决 CLI 2(1) 存在的一些问题。本质上 CLI 2(1) 都是基于 webpack 配置出来的脚手架而已，因为 Zero-config 的概念诞生以后，也因为 webpack 学习成本高的问题，Zero-config 得以流行起来，但是话说回来，要让 CLI 去实现 Zero-config 其实是简单的，只不过把配置隐藏起来即可，但是难就难在 Best Default Configs 这个问题上，毕竟重口难调，那么选用社区的最佳实践是不是一个不错的方案，但是由于前段工程化体系的庞大，将多数的最佳实践（Best Practices）集成在一个 CLI 中又是一大难题，所以在 CLI 3 中诞生了插件（plugin）的解决方案，可以说这个方案确确实实能解决掉这种过程中的问题，他比 CLI 2(1) 这种 template 的形式灵活性更高，而且可以局部更新，使得工程化的配置更为简单。

关于最佳实践，可以参阅[《Front-End Performance CheckList 2018》](https://www.dropbox.com/s/8h9lo8ee65oo9y1/front-end-performance-checklist-2018.pdf?dl=0)

但是我认为不管是 CLI 3 或是 CLI 2(1) 也好，都同样存在这版本更新的问题，在多数不同的场景下，其实 CLI 是很难控制更新这个问题的，不管是官方插件或是第三方插件也好，都有这样的问题，如果说开发者破坏性或者重度的修改了与插件预设模板出入较大的文件时，这样还不如手动让开发者去更新。并且，这种情况我认为是普遍存在的。

CLI 3 在配置文件选择了 `config.js` 的文件格式，因为 JavaScript 表现力强。但是目前看来 `config.js` 也只是解决了表面问题，在一些简单的配置选项时，你可以去官网学习，学习难度可以说比 webpack 真的简单不少，因为他们已经将常用的配置项提炼出来了。但是在复杂的问题上，其实也并不比 webpack 简单，至少 webpack 是开发自由修改的，但是配置文件却需要按照官方来进行单独配置，因为菜的关系，我还是会去看 issues。

最后介绍了版本管理，目前 CLI 3 用的是 lerna。在我看来，管理 CLI 3 和插件以及上游依赖的版本问题确实是难事，这点会不会还是不如 CLI 2(1) template 的形式要好，但是官方既然这么做了，我还是选择支持。包括提到的大版本更新的解决方法，还是蛮期待的。舔一口，支持一下。

## <闪电>燃烧你的 CLI

>演讲者：韦元悦
>前新浪微博前端工程师
>
>Vue CLI 3.0 为开发者赋能，它已经不再是一个普通的命令行脚手架，从可定制的插件到 GUI，开发者可以发挥想象充分利用这些新能力来做更多有意思的事情。让我们一起来看 Vue CLI 能玩出什么更高级的花样。

[<闪电>燃烧你的 CLI](https://slides.com/miccycn/vuecli)

秀的很，利用 CLI 插件的特性直接破坏性的将整个 Vue 配置替换成了 React，具体直接看仓库吧[miccycn/vue-cli-plugin-react](https://github.com/miccycn/vue-cli-plugin-react)

## <闪电>JS class fields & Vue

>演讲者：Hax
>贺师俊，网名 [@hax](https://github.com/hax)，现就职于百姓网架构部；十多年来一直活跃在 Web 标准、前端开发和 JavaScript 社区，对 HTML 标准有微小的贡献。

<!-- [YouTube](https://www.youtube.com/watch?v=WDsEvlXE-BE) -->
{% youtube WDsEvlXE-BE %}

[今天在VueConf杭州上的闪电分享](https://zhuanlan.zhihu.com/p/50781272)
[<闪电>JS class fields & Vue](http://johnhax.net/2018/field-vue/slide#0)

反手就是一套转载，关于演讲中提到的 Vue 中不要使用 private fields 这点，我还得再学习一下。

精通 JavaScript，早在 ES4 时代就通过 [es-discuss](https://esdiscuss.org/) 邮件列表参与标准讨论并提交 issue，近年来则通过 GitHub 关注了几乎所有 ECMAScript 新草案的进展和讨论。尤其是最近富有争议的 [optional chaining](https://github.com/tc39/proposal-optional-chaining) 和 [class fields](https://github.com/tc39/proposal-class-fields) 提案，深度参与了讨论。Hax 给 Babel、ESLint、Webpack 等多个 JavaScript 生态中的重要项目提交过 issue 和 pull request，写过多个针对 ES 新特性的 Babel 转换插件，并是 Atom 编辑器 js-refactor 插件的维护者。Hax 做过大量 JavaScript 相关的分享，包括题为「JavaScript — The World’s Best Programming Language」的演讲。

**内容**

早在去年7月，[tc39](https://github.com/tc39) 已经批准 class field 提案到达 Stage 3，但浏览器厂商一直没有实现该提案，Babel 也只实现了 public field 而没有实现 private field。其中一个原因也许是因为争议性的 “#priv” 语法。最近，Babel 7 和 Chrome 终于实现了该提案，但是争议并没有因此停止。自从 ES Harmony 以来，我们还是第一次见到如此激烈的分歧。

作为中国 JS 社区的活跃分子，我通常都是向大家介绍 JS 新特性如何能更好的帮助我们开发者；我很不情愿将提案讨论中的争议性内容作为话题呈现给开发者，因为这对我们开发者来说没有什么意义，也并不能帮助 [tc39](https://github.com/tc39) 解决争议，还影响“和谐”。但是作为本次争议提案的反对者之一，我认为形势已经非常严峻 —— 这份提案已经接近 Stage 4，也就是正式标准；同时 [tc39](https://github.com/tc39) 最近的会议也已经拒绝所有的竞争提案，并决议停止寻求其他替代性方案；引擎厂商也即将实现和默认开启该特性。当使用该新特性的代码进入 production 环境，就意味着再也没有回头路。它很可能会成为 JS 永远无法摆脱的新的 “Bad Part”。而且本提案涉及语言的核心设施之一 class，影响烈度并非其他局部特性可比，我认为可能影响整个 JavaScript 生态。因此，我不得不将这场争议呈现给社区：

* 无论是寻求更广泛的社区反馈以提交给 [tc39](https://github.com/tc39) 和引擎厂商，还是说在最坏的情况下，让开发者做好准备;
* 至少我已经尽力了；

注意，在本次分享中，我会尽量保持客观，但作为提案的反对者，我不可能以全然中立立场叙述争议双方的观点，并且本次分享将涉及许多 JS 语法语义中的细节问题和一些对普通开发者来说相当陌生的概念。本次分享对于大家很可能将是一场痛苦的旅行。You have been warned!

> 一句话总结：https://github.com/tc39/proposal-class-fields/issues

## Making Your Vue App Accessible

>演讲者：勾三股四
>Vue.js 官方团队成员，阿里云前端工程师
>
>让自己的 Web 应用具备较高的可访问性 (a11y) 是一件非常有意义的事情，希望通过本次分享和大家一起探讨开发 Vue 应用会遇到的典型场景和问题，并提供相应的最佳实践。

[Making Your Vue App Accessible](https://img.w3ctech.com/Making-You-Vue-App-Accessible.pdf)

{% iframe //player.bilibili.com/player.html?aid=37464789&cid=65856927&page=1 %}

主题非常的明确：**Accessbility（信息无障碍化）**。

很支持这样的推广，详细的建议看视频。再是可以学习以下两个仓库，体会 Accessbility 。

- [Jinjiang/vue-a11y-utils](https://github.com/Jinjiang/vue-a11y-utils)
- [Jinjiang/vue-a11y-examples](https://github.com/Jinjiang/vue-a11y-examples)

同时，会后老师发布了篇博客，提到对技术会议的一些看法，可以直接阅读学习：[我对技术会议的一些看法](https://github.com/Jinjiang/jinjiang.github.io/issues/7)

### Background

As the [(WIP) Vue accessibility guide page](https://github.com/vuejs/vuejs.org/pull/1002) says:

> The World Health Organization estimate that 15% of the world's population has some form of disability, 2-4% of them severely so ... which can be divided roughly into four categories: _visual impairments_, _motor impairments_, _hearing impairments_ and _cognitive impairments_.

_table: issues for different impairments_

| visual  | motor              | hearing | cognitive                    |
| ------- | ------------------ | ------- | ---------------------------- |
| 🖥 🔎 🎨 | 🖱 📱 ⌨️ 🕹 🎮 🎙 🖊 🎛 | 🔈      | content, layout, interaction |

Or there are also some accessibility issues for a normal person in such a situation like driving a car, having a meeting, using a mobile device with a bluetooth keyboard etc.

So actually accessibility is not just for the "less amount of people", but for almost everyone.

But some mistakes we often make in a real project like:

- Mouse-only in a desktop app
- Touch-only in a mobile app
- Remote-control-only in a TV app
- Operation through keyboard only is not possible or with low efficiency
- No text alternative for non-text content
- Have no fallback way for the creative interaction like e-pencil, audio input, face ID, touch ID, NFC etc.
- The color contrast is not enough

Each of them might make user confused, block the user flow or lead user to a no-way-out trap in some certain cases.

### Web Standards

However, there are already some web standards and best practice to follow which let developers do it better.

In W3C there are 3 main parts of accessibility standards:

![WAI standard overview](https://www.w3.org/WAI/content-images/wai-std-gl-overview/specs.png)  
via: [W3C Accessibility Standards Overview](https://www.w3.org/WAI/standards-guidelines/)

- [WCAG](https://www.w3.org/TR/WCAG20/): about web content, targeting websites.
- [UAAG](https://www.w3.org/TR/UAAG20/): about user agent, targeting browsers, screen readers etc.
- [ATAG](https://www.w3.org/TR/ATAG20/): about authoring tools, targeting CMS, WYSIWYG editor etc.

and a technical spec which is commonly used:

- [WAI-ARIA](https://w3c.github.io/aria/): targeting web app.

For web developers, we may pay more attention on WCAG and WAI-ARIA. At the same time, we should know which user agents people use most and how about their support and compatibility to the standard.

Here is a survey about most common screen reader and browser combinations table:

| Screen Reader & Browser     | # of Respondents | % of Respondents |
| --------------------------- | ---------------- | ---------------- |
| JAWS with Internet Explorer | 424              | 24.7%            |
| NVDA with Firefox           | 405              | 23.6%            |
| JAWS with Firefox           | 260              | 15.1%            |
| VoiceOver with Safari       | 172              | 10.0%            |
| JAWS with Chrome            | 112              | 6.5%             |
| NVDA with Chrome            | 102              | 5.9%             |
| NVDA with IE                | 40               | 2.3%             |
| VoiceOver with Chrome       | 24               | 1.4%             |
| Other combinations          | 180              | 10.5%            |

_via [Screen Reader User Survey by webaim.org](https://webaim.org/projects/screenreadersurvey7/#browsercombos)_

## 探索Vue的高级应用(Ant Design Vue里的那些黑科技)

>演讲者：唐金州
>Ant-Design-Vue作者，一点资讯数据组前端团队负责人
>
>Antd是蚂蚁金服推出的一门优秀的设计语言，但官方仅提供了React版本，虽然社区(个人、团队)也一直有不少前辈投入到Vue版本中，但由于Antd的功能之强大，实现之复杂，大多都已弃坑。那么我们又是如何完成这么一件看似不可能的事情的呢？本主题将分享ant-design-vue里的那些黑科技，挖掘Vue的更多高级应用，如何将不可能变成可能。少一点套路，多一点干货。

[探索Vue的高级应用(Ant Design Vue里的那些黑科技)](https://img.w3ctech.com/VueConf_%E5%94%90%E9%87%91%E5%B7%9E.pdf)

{% iframe //player.bilibili.com/player.html?aid=37534336&cid=65979920&page=1 %}

介绍了无阉割版的 Ant Design for Vue 的诞生，虽然表示没有用过 Ant Design Vue 。组件的业务逻辑层面其实是可以直接复用的，但是本身 React 和 Vue 的区别，所以需要分别一一去实现相应的功能。那么在此分享了在开发过程中遇到的一些问题：

### JSX vs SFC （模板引擎）

JSX 编程能力更强，语法灵活，但学习成本⾼，逻辑、视图、样式混合，不清晰。

SFC (Single File Component) 将 Template 、 Style 、 Script 分离，分离后结构简洁清晰，更亲切和易上手的 Vue 写法。

### 组件生命周期

比如 `componentWillReceiveProps` 和 `watch` 这两个周期。周期阶段是相同的，只不过 `componentWillReceiveProps` 是进行统一管理而 `watch` 是进行单一管理，要实现类似于 React 这样的统一管理，需要再度封装一层来实现，至于怎么封装实现，可以观看下视频，视频中有介绍，这属于业务方面。

### Ref 引用

再如 Ref 引用，之前 React 是使用 `String` 的形式来实现 Ref 的引用的，现在已经变为 `CallBack` 的形式。功能性更强，但是 Vue 本身并没有实现这样的功能，所以作者自己实现了一个，参阅[vue-ref](https://github.com/vueComponent/vue-ref)

---

所以值得我们思考的是，在将一个成熟的 UI 库从一个框架移植到另一个框架的时候，我们需要充分的了解这两个框架，并考虑他们之间的区别，并选择以最优的方式去解决移植过程中发生的问题，尽量来告别阉割版。

## 基于Electron-Vue的桌面应用开发实践

>演讲者：赵帅
>美团酒旅事业群 前端工程师
>
>企业内部系统开发同质化严重，重复开发成本过高，为了减少工程师的重复开发成本。我们开发了一款桌面应用，提供项目快速搭建、业务模板生成、可视化发布等一体化解决方案，提升研发效率，目标实现Devless，本主题分享如何使用Electron和Vue结合设计桌面应用，以及核心能力的实现。

[基于Electron-Vue的桌面应用开发实践](https://img.w3ctech.com/%E5%9F%BA%E4%BA%8EElectron-vue%E7%9A%84%E6%A1%8C%E5%BA%94%E7%94%A8%E5%AE%9E%E6%88%982.pdf)

{% iframe //player.bilibili.com/player.html?aid=37320192&cid=65592298&page=1 %}

对 Electron 是有所耳闻，知道 VSCode 、 GitHub Desktop 是使用 Electron 开发的。借机先学习下 Electron 的原理：

![Electron-Vue-1](/gallery/vueconf-hangzhou-2018/Electron-Vue-1.png)

### 1.如何解耦业务逻辑
  
- ⻚⾯代码与业务逻辑混写，功能重复
- 升级 Electron 框架，修改成本⼤

单独封装一层 bridge 来解决 NodeAPI 或者 NativeAPI 的更新问题，当遇到 API 更新时，无需关注业务逻辑代码，只需关注 bridge 即可。

### 2.如何实现登录认证

看不懂的时候就做个勤劳的搬运工吧

![Electron-Vue-2](/gallery/vueconf-hangzhou-2018/Electron-Vue-2.png)

### 3.如何完成签名、⾃动更新

在发布 Electron 应用时遇到的问题：

- ⽤户每次都要重新下载
- 下载的应⽤不被信任
- ⽹络请求被询问……

关于这一系列的问题，采用 AutoUpdater 来解决。

- 每次新版本上线，⼀旦线上出现严重bug
- 影响**所有⽤户**使⽤

提供一个*版本管理平台*来实现版本控制，添加灰度版本，得以控制版本更新，避免上述问题。除了灰度发布，还有指定⼈员发布，指定版本发布功能。

### 4.如何定位和收集问题

其实就是在应用中设置埋点，上报监控系统

![Electron-Vue-3](/gallery/vueconf-hangzhou-2018/Electron-Vue-3.png)

---

比较值得一提的是 Electron 开发和 Web 开发差异

|   | Electron 应⽤              | Web 应⽤ |
| ------- | ------------------ | ------- |
| 设计 | 单窗⼝、多窗⼝ | 单⻚⾯、多⻚⾯ |
| 开发 | html CSS JS **Node.js OS API** | html CSS JS  |
| 调试 | **主进程**，渲染进程 | 渲染进程 |
| 构建 | 资源⽂件，**安装包** | 资源⽂件 |
| 发布 | **Mac / Window / Linux** | **Nginx / CDN** |
| 关注点 | 进程通信，内存管理，版本管理，性能及Crash监控…… | 兼容，DOM，组件、性能，…… |

## 多端统一方案 Hippy-Vue 是如何设计实现的

>演讲者：旷旭卿
>腾讯 SuperTeam 高级前端工程师，Hippy 核心开发者
>
>目前多端统一的实践方案，很多都是以移动端开发角度进行设计的，前端开发在使用这些方案时往往需要重新学习所有组件和参数，而腾讯即将开源的 Hippy-Vue，在移动端中实现了网页的运行时，在降低前端开发的学习曲线的同时还可以复用大量的 Vue 生态，那么 Hippy-Vue 是如何做到这点的？我将在本次演讲中告诉你！

[多端统一方案 Hippy-Vue 是如何设计实现的](https://img.w3ctech.com/Hippy-VueConf.pdf)

{% iframe //player.bilibili.com/player.html?aid=37464898&cid=65857401&page=1 %}

多端统一方案背景介绍

![Hippy-Vue-1](/gallery/vueconf-hangzhou-2018/Hippy-Vue-1.png)

Hippy SDK 采⽤三层设计，其中：

- JavaScript 层：提供业务代码运⾏时的前端上下⽂环境；
- Native Framework 层：负责前终端通讯与 JavaScript VM，并提供 Native 相关模块；
- Portable UI 层：提供基础 UI 组件与布局计算框架，并负责渲染⾄⽬标平台；

Hippy Buffer：⼆进制传输协议，编解码性能更好。
X5 V8：X5 团队特供 V8 引擎。
Hippy Layout：iOS、Android 共享布局引擎，纯 C 开发，只有 50kb。

![Hippy-Vue-2](/gallery/vueconf-hangzhou-2018/Hippy-Vue-2.png)

## 应用多端统一的实践

>演讲者：卓凌
>阿里巴巴高级前端工程师，淘宝小程序核心开发者
>
>小程序应用技术如火如荼，越来越多的公司在打造自己的小程序应用，但小程序的平台也是各自不同，本主提将介绍基于 Vue 语法（Simple File Component）角度，包括 Rax 无线端应用同时支持运行在 weex 环境，淘宝小程序，支付宝小程序，微信小程序等，践行 Write Once, Run everywhere!

[应用多端统一的实践](https://img.w3ctech.com/%E5%BA%94%E7%94%A8%E5%A4%9A%E7%AB%AF%E7%BB%9F%E4%B8%80%E7%9A%84%E5%AE%9E%E8%B7%B5.pdf)

{% iframe //player.bilibili.com/player.html?aid=37534477&cid=65980180&page=1 %}

看起来多端统一方案都基本差不多，就是具体实现上有差异，接下来看看阿里的

![rax-1](/gallery/vueconf-hangzhou-2018/rax-1.png)
![rax-2](/gallery/vueconf-hangzhou-2018/rax-2.png)
![rax-3](/gallery/vueconf-hangzhou-2018/rax-3.png)
![rax-4](/gallery/vueconf-hangzhou-2018/rax-4.png)

介绍几个阿里已经开源的仓库：

- [sfc2mp](https://github.com/alibaba/rax/tree/master/packages/sfc2mp)
- [rax](https://github.com/alibaba/rax)
- [ice](https://github.com/alibaba/ice)

## 再谈Vue SSR -- 响应式数据流在快手游戏直播中的应用

>演讲者：天翔
>快手前端架构师，游戏直播团队前端负责人。
>
>快手游戏直播第一版本上线以后，随着业务迭代，逐渐发现了一些问题，比如：
>1、同构应用中是否能够更好的方式来明确Service边界；
>2、对于持续性数据的封装、转换，状态的聚合、分发，是否有其他的手段来管理
>基于以上问题，我们选择通过xstream与Apollo GraphQL对现有项目进行重构，本次演讲中会以此为切入点，通过讲解我们的架构迁移中的思考，希望与会者们对响应式数据的应用有一个新的启发

[再谈Vue SSR -- 响应式数据流在快手游戏直播中的应用](https://img.w3ctech.com/vue-conf-tianxiang.pdf)

上一个演讲结束后，我就抗不出提前溜了，没有现场听，看了下视频，首先上次关于 SSR 的内容我肯定是没有听过的。

关于 SSR 的优缺点，我从官网先拿来：

### 什么是服务器端渲染(SSR)？

Vue.js 是构建客户端应用程序的框架。默认情况下，可以在浏览器中输出 Vue 组件，进行生成 DOM 和操作 DOM。然而，也可以将同一个组件渲染为服务器端的 HTML 字符串，将它们直接发送到浏览器，最后将这些静态标记"激活"为客户端上完全可交互的应用程序。

服务器渲染的 Vue.js 应用程序也可以被认为是"同构"或"通用"，因为应用程序的大部分代码都可以在**服务器**和**客户端**上运行。

### 为什么使用服务器端渲染(SSR)？

与传统 SPA（Single-Page Application - 单页应用程序）相比，服务器端渲染(SSR)的优势主要在于：

- 更好的 SEO，由于搜索引擎爬虫抓取工具可以直接查看完全渲染的页面。

  请注意，截至目前，Google 和 Bing 可以很好对同步 JavaScript 应用程序进行索引。在这里，同步是关键。如果你的应用程序初始展示 loading 菊花图，然后通过 Ajax 获取内容，抓取工具并不会等待异步完成后再行抓取页面内容。也就是说，如果 SEO 对你的站点至关重要，而你的页面又是异步获取内容，则你可能需要服务器端渲染(SSR)解决此问题。

- 更快的内容到达时间(time-to-content)，特别是对于缓慢的网络情况或运行缓慢的设备。无需等待所有的 JavaScript 都完成下载并执行，才显示服务器渲染的标记，所以你的用户将会更快速地看到完整渲染的页面。通常可以产生更好的用户体验，并且对于那些「内容到达时间(time-to-content)与转化率直接相关」的应用程序而言，服务器端渲染(SSR)至关重要。

使用服务器端渲染(SSR)时还需要有一些权衡之处：

- 开发条件所限。浏览器特定的代码，只能在某些生命周期钩子函数(lifecycle hook)中使用；一些外部扩展库(external library)可能需要特殊处理，才能在服务器渲染应用程序中运行。

- 涉及构建设置和部署的更多要求。与可以部署在任何静态文件服务器上的完全静态单页面应用程序(SPA)不同，服务器渲染应用程序，需要处于 Node.js server 运行环境。

- 更多的服务器端负载。在 Node.js 中渲染完整的应用程序，显然会比仅仅提供静态文件的 server 更加大量占用 CPU 资源(CPU-intensive - CPU 密集)，因此如果你预料在高流量环境(high traffic)下使用，请准备相应的服务器负载，并明智地采用缓存策略。

在对你的应用程序使用服务器端渲染(SSR)之前，你应该问的第一个问题是，是否真的需要它。这主要取决于内容到达时间(time-to-content)对应用程序的重要程度。例如，如果你正在构建一个内部仪表盘，初始加载时的额外几百毫秒并不重要，这种情况下去使用服务器端渲染(SSR)将是一个小题大作之举。然而，内容到达时间(time-to-content)要求是绝对关键的指标，在这种情况下，服务器端渲染(SSR)可以帮助你实现最佳的初始加载性能。

上面这段官网的介绍其实已经提到了 SSR 的优点，解决首屏白屏和 SEO 的问题。那么天翔遇到的问题是， Component 与 Model 的边界是否可以更加明确？ OK ，然后先讲了几个我听不懂的，毕竟我没有实践过 SSR ，那么谈论到组件 LifeCycle VS 状态 LifeCycle 时，你会发现，组件中的 state 是跟随着组件的生命周期存在而存在的，但是状态中的 state 比如（vuex）却不是这样，那么如何让开发者去管理这个问题呢？说来说去还是集中处理。

数据处理边界这个点比较有意思了，那么从 Service 给过来的数据是在 node.js 中处理呢还是到 client 去处理。这个问题需要根据场景去决定，然后天翔发现 API Proxy 和 Service 是有一定的重叠性的。所以天翔选用了又一个新的东西 GraphQL 。最终，node.js 实现数据的声明 + 转换， SSR 实现数据的聚合 / 过滤。

关于提到 Observable 也很好理解，其实 Observable 就是一个数组，要理解为什么 Observable 是一个数组其实就要理解弹珠图。我对这部分的概念呢感觉是似懂非懂……

>“状态管理，等同于状态结构及其变化过程，你可以从描述状态⼊⼿，以某个时刻的状态为侧重点，淡化其变迁过程，也可以从变迁过程⼊⼿，侧重某个过程对数据的变动”

![Vue-SSR-1](/gallery/vueconf-hangzhou-2018/Vue-SSR-1.png)

利用 Observable 实现：

- 数据分发 filter
- 数据聚合 merge

很像在操作数组，然后以 Observable 的特性去实现响应式。

Vue 帮我们解决了同步响应式， Observable 帮我们解决了异步响应式

![Vue-SSR-2](/gallery/vueconf-hangzhou-2018/Vue-SSR-2.png)

## how CRDT improves Vue apps

>演讲者：Andrey Sitnik
>Evil Martians前端工程师，PostCSS、Autoprefixer作者
>
>What technologies could replace GraphQL in the future? How can we mix ideas from Vuex and long-standing ideas from the 80s to make the better user experience for our web applications?
>Andrey Sitnik, the creator of Autoprefixer and PostCSS, will explain why I switched from making improvements to CSS tooling to pursuing CRDT ideas.

[how CRDT improves Vue apps](http://slides.com/ai/crdt-vue-cn#/)

{% iframe //player.bilibili.com/player.html?aid=37324888&cid=65600614&page=1 %}

看视频感觉大佬做的 slides 挺好懂的。大佬介绍了 CRDT 与其他 AJAX 的体验区别，猜测 CRDT 是趋势，但是这和 Vue 有什么关系，我在 slides 中也没看出来。 CRDT 的全称是 [Conflict-free replicated data type](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) 。接下来我分享几张通俗易懂的 slides。

![crdt-1](/gallery/vueconf-hangzhou-2018/crdt-1.png)
![crdt-2](/gallery/vueconf-hangzhou-2018/crdt-2.png)
![crdt-3](/gallery/vueconf-hangzhou-2018/crdt-3.png)
![crdt-4](/gallery/vueconf-hangzhou-2018/crdt-4.png)
![crdt-5](/gallery/vueconf-hangzhou-2018/crdt-5.png)
![crdt-6](/gallery/vueconf-hangzhou-2018/crdt-6.png)
![crdt-7](/gallery/vueconf-hangzhou-2018/crdt-7.png)
