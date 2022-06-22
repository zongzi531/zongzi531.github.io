---
title: 借助 Webpack 实现样式手动插入
date: 2022-06-22 12:30:18
categories: "Blog"
comments: true
tags:
- Style
- CSS
- Webpack
- lazy
- import
---

<!-- no node -->

<!-- more -->

背景是需要将 `import 'example.css'` 的样式插入到 `iframe` 下。

网上有很多方案，但是都不是我想要的解决方法。

比如获得到样式的内容，然后插入到 `iframe` 下，但是我希望实现的是类似于 `import` 的实现方法。

那么查阅了相关资料发现，`style-loader` 有一个配置项 `injectType: 'lazyStyleTag'` ，支持后动插入 `<style>` 标签。

```ts
import styles from 'example.css'

styles.use()
```

是在调用 `use` 方法后对应的 `<style>` 标签才会插入到 `document.head` 中。由于我们的需求是插入到 `iframe` 下的 `head` 中去，所以我们还需要进行少许配置。

我们需要配置 `insert` 函数，让 `use` 方法的调用可以插入到指定的位置。

但是你会发现 `use` 方法不接受参数，所以我们需要使用一些手段来实现，当调用 `use` 方法时，能够顺利插入到指定位置。

我们需要看到 `node_modules/style-loader/dist/index.js` 的源码位置，可以看到 `use` 方法调用时会调用来自 `./runtime/injectStylesIntoStyleTag.js` 的默认导出函数。并且传入上下文和 `options` 。

当然这不是重点，这是入口，持续执行后会看到进入到 `insertStyleElement` 函数。如果配置 `insert` 为函数，相当于调用 `use` 方法时逻辑自行处理。

回到 `index.js` 的文件位置，这里很关键的内容出现了，我们可以在 `insert` 函数定义时通过外部的环境来定义需要将样式插入的具体位置。

```ts
options({
  injectType: 'lazyStyleTag',
  insert: function (style: HTMLStyleElement) {
    var doc = globalThis.__example__ || document
    if (doc.head) {
      doc.head.appendChild(style)
    }
  }
})
```

当然，你仔细观察源码可以发现，其他还可以在 `insert` 函数中获取到 `exported` 这个变量，至于是为什么，相信你去看下 `index.js` 便会明白。我们可以按照我们的需求，声明一下 `d.ts`：

```ts
declare module '*.css?lazy' {
  interface LazyCSS {
    locals: Record<string, any>
    use: () => void
    unuse: () => void
    __appendTo__?: Document
  }
  const lazycss: LazyCSS
  export default lazycss
}
```

我们就可以这样使用：

```ts
import styles from 'example.css?lazy'

styles.__appendTo__ = iframeDoc
styles.use()
```

当然， `insert` 函数也需要做稍许调整：

```ts
options({
  injectType: 'lazyStyleTag',
  insert: function (style: HTMLStyleElement) {
    var doc = exported.__appendTo__ || document
    if (doc.head) {
      doc.head.appendChild(style)
    }
  }
})
```

这里肯定会奇怪，为何后面加了一个参数 `?lazy` ，是为了区分普通 `import` 样式和手动插入的参数。

结合完整的在 webpack-chain 中的配置如下：

```ts
const insert = function (style: HTMLStyleElement) {
  var doc = exported.__appendTo__ || document
  if (doc.head) {
    doc.head.appendChild(style)
  }

  // 可以加入任何你想加入的逻辑
}

config.module.rule('css')
  .oneOf('lazy-css')
  .before('normal')
  .resourceQuery(/lazy/)
  .test(/\.css$/)
  .use('style-loader')
  .loader('style-loader')
  .options({ injectType: 'lazyStyleTag', insert })
  .end()
  .use('css-loader')
  .loader('css-loader')
  .end()
  .use('postcss-loader')
  .loader('postcss-loader')

// 同样的，你也可以配置 module.css 也使用手动插入的形式
// 记住需要在原有的 loader 校验前加入
```

到这里，我们其实已经实现了我们的需求。也算是大功告成！

不过这里针对 CSS Module 这种形式记得注意 `hash` 值是否能够保持一致，若你的 npm 包需要提供这类能力，可以将样式在构建产出时将 `hash` 值编译成一个非 CSS Module 的样式文件，然后进行引入使用即可。

有关 `style-loader` 的源码可以[点击查看](https://github.com/webpack-contrib/style-loader/tree/master/src)。
