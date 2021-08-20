---
title: 《进阶 TypeScript 之初见泛型》
date: 2021-08-20 09:28:10
categories: "Share"
comments: true
tags:
- TypeScript
---

<!-- no node -->

<!-- more -->

> [链接](https://github.com/zongzi531/daily-learning/blob/master/share/%E8%BF%9B%E9%98%B6%20TypeScript%20%E4%B9%8B%E5%88%9D%E8%A7%81%E6%B3%9B%E5%9E%8B.pdf)

大家晚上好，特别为大家做这次分享，因为现在几个团队都开始使用 TypeScript 了，那么除了在日常的 TypeScript 基础使用外，我想给大家带来一些 TypeScript 的进阶使用，那么本次为大家分享有关泛型的进阶入门技巧。

那我们现在就从为什么会有泛型来说什么是泛型，让我们看到这段代码，我们定义了一个 `identity` 函数，他用来处理 `number` 类型的数据，并返回 `number` 类型的数据。再来看到下一段代码，当这个函数要处理另外的类型的时候，我们不得不定义另外的类型，比如在 `number` 的基础上添加 `string | Array<string>` 等等类型，当然这里可以用到函数重载的概念，不过我这里要硬生生的引出泛型。

最后我们来看到这段代码，我们可以在使用的时候借助 TypeScript 的类型推断自动来使用，或者说我们可以传入类型来定义我们想要的类型，比如 `identity<boolean>(true)` ，这就是泛型。

我给大家摘抄了官方提供的全局泛型，见 [Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html) 。

官方提供的这些泛型可以说使用起来非常的方便，并且大家在日常开发当中也用到了许多，那么如何自己书写泛型呢？从这里开始将是本次分享的重点。

我们来看到例子1，这里的 `isString` 并没有 `typeof value === 'string'` 来的好用，因为 `isString` 不能进行类型推断，有同学可能会说，我在下面对 `el` 使用断言，从代码运行上来说，这确实没什么问题。但是既然使用了 TypeScript ，就应该让类型推断变的更自然，而不是用断言来解决问题，这样和写 JavaScript 又有什么区别呢？那么其实很简单，我们将函数返回的 `boolean` 修改成 `value is string` 这样就可以实现类型推断了。

是不是很有趣，再进阶一步讲，我们是不是可以将这个 `isString` 函数改造一下，是他变成一个更通用的泛型。

```typescript
declare function isSomething <T>(value: T): value is T

const num = isSomething(1) // isSomething<number>
const str = isSomething('') // isSomething<string>
```

非常的妙是不是。再来看到例子2，假设我们要写一个样式对象，但是我们需要自己考虑兼容性，比如会加入前缀 `webkit` ，那么我们自然会一个个把他们写出来，但是在维护类型的时候会带来很多繁琐的问题，比如说我修改 `flex` 的类型时，我还需要顺带修改 `webkit` 前缀的对应的类型，当这种场景变得多又复杂时，代码的维护成本就会逐渐提升。

那么当然是有办法解决解决这样的问题的，首先我们需要利用一些手段将 `flex` 转换成 `webkitFlex` ，然后将 `interface` 的 key 换成我们想要的内容。

到目前为止，我们已经可以实现 key 的替换，但是原来的 key 却不见了，所以我们这里需要借助 `| P` 来完成，来实现我们的需求。到目前为止我们已经完成了我们想要的，我们再来进阶一步讲，把问题变得有趣，我们这里希望经过转换后， `lineHeight` 是不会加 `webkit` 前缀的，怎么实现呢。

```typescript
type WebkitKey<K extends string> = K extends 'lineHeight' ? never : `webkit${Capitalize<K>}`
```

这样，我们就可以实现新的需求，当然了这里还是可以结合例子1进行进阶。

```typescript
type ExcludeWebkitKey = 'lineHeight' | 'something you need.'
type WebkitKey<K extends string, E = ExcludeWebkitKey> = K extends E ? never : `webkit${Capitalize<K>}`
```

哈哈，是不是也非常的有趣。确实，就是这么有趣。

那么我们来看到例子3，我们如何可以在 `methods` 下的方法中获得到 `data` 返回的类型呢，我这边摘抄了官网的例子，大家可以看一下。

其实我们的例子和官网的区别就是官网是个对象，而我们是个函数，我们只要把 `D` 修改成 `() => D` 就可以了。

那么本次的分享就到这里，后面我也会尝试总结更多的技巧分享到大家。
