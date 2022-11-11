---
title: <img> 使用 svg 后如何改变颜色
date: 2020-12-16 17:06:56
categories: "CSS"
comments: true
featured_image: pic1.jpeg
tags:
- CSS
intro: '一次知难而上的尝试'
---

<!-- no node -->

<!-- more -->

在一次开发中，使用 `<img src="some.svg" />` 的形式引入了 svg 文件，考虑到 svg 图片会在鼠标移入后进行变色操作，想想还是挺麻烦的。

不像是直接使用 svg 标签那样方便，这就开始了这次知难而上的尝试了。

接下来，我将为大家介绍我在解决问题的时候尝试的方法以及最终选择的方案。

## 方案一

```html
<svg style="display: none;">
  <defs>
    <filter id="turnIntoRed">
      <feColorMatrix
        in="SourceGraphic"
        type="matrix"
        values="1 1 1 1 0
                0 0 0 0 0
                0 0 0 0 0
                0 0 0 1 0" />
    </filter>
  </defs>
</svg>
<img style="filter: url('#turnIntoRed')" src="some.svg" />
```

这种方式可以实现红色的渲染，但是会隐约出现原来的颜色，效果不行。

## 方案二

```html
<svg style="opacity: 0; position: fixed; left: -1000px; top: -1000px;">
  <defs>
    <filter id="turnIntoRed">
      <feFlood flood-color="#00" flood-opacity="1" result="color" />
      <feComposite in="color" in2="SourceGraphic" operator="in" />
    </filter>
  </defs>
</svg>
<img style="filter: url('#turnIntoRed')" src="some.svg" />
```

这种方法实现效果达成了要求，可以完美显示红色，并且原来的颜色看不见。

但是方案一和方案二都有一个致命问题，需要定义一个存在全局唯一的 ID ，如果这个 ID 被覆盖的话那么就会超出预期显示内容。

## 方案三

```html
<img style="filter: drop-shadow(1000px 0 0 red); transform: translate(-1000px);" src="some.svg" />
```

这种方案不需要再定义晦涩难懂的 svg 标签，而是通过显示阴影的方式来解决问题，完美的解决了上面的问题，但是这个方案有一个注意点，需要将本身显示的图片移动至看不见的地方，不然就会有 2 个噢。

## 结论

最终我选择方案三来解决我的问题。

## 附：方案四

[《svg 文件自动化转组件》](https://zongzi531.com/2022/11/11/svg%E6%96%87%E4%BB%B6%E8%87%AA%E5%8A%A8%E5%8C%96%E8%BD%AC%E7%BB%84%E4%BB%B6/)
