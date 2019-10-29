---
title: 简单探索text-overflow
date: 2017-04-19 19:46:39
categories: "CSS"
comments: true
thumbnail: /gallery/简单探索text-overflow/pic.jpg
tags:
- HTML/CSS
- CSS3
---

<!-- no node -->

<!-- more -->

# 认识

" text-overflow" 属性用于确定告知用户此处有因溢出而无法显示的内容的方式。它可以截断（clipped）掉溢出的内容，也可以显示一个省略号（ '…', U+2026 HORIZONTAL ELLIPSIS，水平省略号），或者显示一个自定义的字符串。 

# 遇到问题

在练习vue-eleme时，遇到了这么一个奇妙的问题，设置这段CSS后，PC端显示内容与移动端显示内容稍有不符。

```css
white-space: nowrap
overflow: hidden
text-overflow: ellipsis
```

我将一段内容设置了文字溢出后截断溢出内容，并显示省略号，以下为出入：

![](/gallery/简单探索text-overflow/test1.jpeg)

溢出的省略号内容的出入的确很让人头痛。

# 尝试解决

然后去MDN仔细看了看[text-overflow](https://developer.mozilla.org/en-US/docs/Web/CSS/text-overflow)

![](/gallery/简单探索text-overflow/text-overflow.png)

看到了一个很有意思的参数：Custom string ""。但是很遗憾的是大部分浏览器**不支持**。

带着测试的心态，我修改了下代码内容：

```css
white-space: nowrap
overflow: hidden
text-overflow: "···"
```

用Firefox试了一试：
>移动端Firefox版本未达到9.0

![](/gallery/简单探索text-overflow/test2.jpeg)

# 总结

最终因为浏览器兼容性问题，没有采用这个解决办法。但是在尝试解决问题的过程中对`text-overflow`有了更进一步的认识，这很值得。

