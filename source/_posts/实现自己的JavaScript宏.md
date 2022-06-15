---
title: 《实现自己的 JavaScript 宏》
date: 2022-06-15 12:55:30
categories: "Share"
comments: true
tags:
- JavaScript
- Babel
- macro
---

<!-- no node -->

<!-- more -->

**宏**（英语：Macro），是一种批量处理的称谓。

之所以会想做这次分享一是因为想继续多的操作一些 AST 相关的知识，二是也是受到学习 Rust 的影响。

## 什么是宏

我觉得百科中这段解释比较合理：

> 计算机科学里的宏是一种抽象（Abstraction），它根据一系列预定义的规则替换一定的文本模式。解释器或编译器在遇到宏时会自动进行这一模式替换。对于编译语言，宏展开在编译时发生，进行宏展开的工具常被称为宏展开器。宏这一术语也常常被用于许多类似的环境中，它们是源自宏展开的概念，这包括键盘宏和宏语言。绝大多数情况下，“宏”这个词的使用暗示着将小命令或动作转化为一系列指令。
> 
> 宏的用途在于自动化频繁使用的序列或者是获得一种更强大的抽象能力。

当然，更多的解释可以前往百科中学习。

## 初识宏

我最早知道宏的时候是在小学，那时候对宏其实产生了很大的恐惧，那时候流行的是宏病毒，以至于在学习 Office 软件时一直以来不敢接触里面的宏功能。

> 在计算机技术的历史中，宏病毒（英语：Macro virus）是一种使得应用软件的相关应用文档内含有被称为宏的可执行代码的病毒。一个电子表格程序可能允许用户在一个文档中嵌入“宏命令”，使得某种操作得以自动运行；同样的操作也就可以将病毒嵌入电子表格来对用户的使用造成破坏。

那时候的我，看到这些还是会感到惧怕，担心电脑因此中病毒……当然现在的我也不会去特意接触这些。

## 为什么 JavaScript 没有宏编程

学习 JavaScript 至今，你会发现， JavaScript 没有宏编程的概念，我想其实这与 JavaScript 的运行环境有关系，当然 JavaScript 设计之初就没有设计宏的概念。

我会认为在 JavaScript 中要想实现类似于宏编程的能力就是函数本身，可以自己实现一个具备抽象的函数处理方法，而宏编程则是向上面百科中提到的内容一样，是一种预定义的规则替换模式。

## 如何借助 Babel 实现自己的宏

本文将不会侧重什么是 AST 、如何编写 AST 等知识，我们可以借助 Babel 提供的宏插件来按照要求实现自己的宏能力。

我们会以 [kentcdodds/babel-plugin-macros](https://github.com/kentcdodds/babel-plugin-macros) 提供的教程实现自己的宏。

我们按照文档要求编写代码如下：

```ts
// index.ts
import { createMacro } from 'babel-plugin-macros'

export default createMacro(({ references, state, babel }) => {
  // do something
})
```

并且，我们需要这样去使用：

```ts
// example
import { somemacro } from '@zong/js.macro'

somemacro`any your want`
```

由于使用 TypeScript 编写，为了提供更好的类型推导，我们需要在 `index.ts` 定义我们需要被导出的宏方法，并定义对应的逻辑即可。

```ts
// index.ts
export declare const define: Function
```

目前，我设计的是想实现一个属于我自己的声明变量的宏。

我期望实现的宏代码如下：

```ts
import { define } from '@zong/js.macro'

define`
  @a:1;
  @b:2;
  @c:'cde';
  @d:"def";
  @e:true;
  @f:false;
`
```

最终经过 Babel 编译后的结果是：

```ts
// Compiled
const a = 1;
const b = 2;
const c = "cde";
const d = "def";
const e = true;
const f = false;
```

我们需要从 `references` 对象中获取对应的宏名称使用：

```ts
// index.ts
import { createMacro } from 'babel-plugin-macros'
import transformDefine from './transforms/define'

export default createMacro(({ references, state, babel }) => {
  // 这里的 define 即我们期望使用的宏名称
  transformDefine(references.define, state, babel)
})

export declare const define: Function
```

接下来辛苦由 `transformDefine` 函数进行 AST 操作获得我们想要的结果。

## 总结

至此，我们了解了什么是宏以及 JavaScript 该如何借助 Babel 实现自己的宏。

其中的折腾劲十足，明明可以通过抽象一个函数得到的结果，我们非要绕这么大一圈。

也并不是没有收获，我们收获上面的知识，很开心。

可以查看 [@zong/js.macro](https://www.npmjs.com/package/@zong/js.macro) 学习 AST 处理操作。

目前来看，我不仅实现了上面的静态宏使用方法，还实现了：

```ts
import { define } from '@zong/js.macro'

const var1 = 2

define`
  @va:${1};
  @vb:${var1};
`

// Compiled
const var1 = 2;
const va = 1;
const vb = var1;
```

当然，未来我还考虑加入更多的宏能力来锻炼 AST 处理能力。

1. 支持 `let` 声明 (`mut@`/`@@`) ；
2. `;` 结尾为可选项；
3. 支持没有初始值；
4. 若没有初始值则必须以 `let` 进行声明的校验；
5. 初始值支持变量等其他 JavaScript 使用方式；
6. ESLint 插件 —— 上下文声明变量重复检查；
