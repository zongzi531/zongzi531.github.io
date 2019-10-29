---
title: 参加FCC上海前端技术群线下Meetup
date: 2018-09-19 20:48:59
categories: "CSS"
comments: true
thumbnail: /gallery/参加FCC上海前端技术群线下Meetup/fcc-meetup4.JPG
tags:
- CSS
- FCC
---

<!-- no node -->

<!-- more -->

> 2018,09,15 in Shanghai
>
> 第一次参加 FCC 线下聚会，将一些收获记录至此 :)

## [1、Layers: a case study of CSS optimization - 🐑🐑🐑(吴名扬)；](https://docs.google.com/presentation/d/e/2PACX-1vRIsRF2Kz0to_0OJ_vrS4MODvj__SLtRWlXGdqt38VbahxXMHmGHyd5QvsxSCRlxM0ZWjV-szjI3qEy/pub?start=false&loop=false&delayms=3000&slide=id.p7)

![pic1-1](/gallery/参加FCC上海前端技术群线下Meetup/pic1-1.png)

### CSS 文件大小

尽可能的利用工具去减少文件体积即可，说白了就是代码压缩。

### 加载优化 (@import rule, critical path)

减少加载阻塞，减少使用`@import` (浏览器不能并行下载样式，使用`@import`会导致页面增加额外的往返开销)，可通过使用`<link>`标签（可以并行下载 CSS 文件）替代`@import`，需要注意的是一个页面中的 CSS 文件不宜过多，否则应简化和合并外部 CSS 文件以节省请求时间，从而提升页面加载速度。

### 选择器性能

下面可以结合实际应用来介绍一些 Best Practice：

1. 优先使用`class`选择器，可替代如多层标签选择器规则，增加浏览器匹配效率。
2. 谨慎使用`id`选择器，`id`选择器在页面中是唯一的，不利于团队协作和代码维护。
3. 利用选择器的继承性，避免过分限制选择器导致浏览器工作效率降低。
4. 避免 CSS 正则表达式规则。

### Reflow

减少重排（Reflow），重排意味着元素位置发生改变，对于交互性较强的 web 应用来说，重排是在所难免的，但仍然可以从以下几方面进行优化：

* 不要一条一条地修改 DOM 的样式，预先定义好 class，然后修改 DOM 的 className。
* DOM 离线后修改，如先把 DOM 元素 给 `display:none;` (有一次 Reflow)，然后修改100次，最后再把它显示出来。
* 尽可能不要修改影响范围较大的 DOM 元素。
* 为动画元素使用绝对定位 `absolute` / `fixed` 。
* 不使用 table 布局，可能很小的一个小改动会造成整个 table 的重排。

### Repaint

减少重绘（Repaint），重绘意味着元素位置不变，浏览器仅仅根据新的样式重绘该元素（如`border-color`, `background-color`，`visibility`等）。

### 优化动画，启用GPU硬件加速

GPU 加速可以不仅应用于3D，而且也可以应用于2D，通常可以应用于Canvas2D，布局合成（Layout Compositing）, CSS3转换（transitions），CSS3 3D变换（transforms），WebGL和视频(video)。

> 回顾一下这些 CSS性能优化 方案，但是本次讲的主角是 Layer，那么说到什么是 Layer，以及对症下药的问题。PPT中，杨总以及写的很直白了，所以我就直接搬过来了。接下来请看图：

再次提一下浏览器渲染流程：JavaScript -> Style -> **Layout** -> Paint -> Composite

![pic1-2](/gallery/参加FCC上海前端技术群线下Meetup/pic1-2.png)
![pic1-3](/gallery/参加FCC上海前端技术群线下Meetup/pic1-3.png)
![pic1-4](/gallery/参加FCC上海前端技术群线下Meetup/pic1-4.png)

如何对症下药：减少Layer的生成和扩散

* position relative/absolute
* transform reduction
* contain: content
* CSS filter

![pic1-5](/gallery/参加FCC上海前端技术群线下Meetup/pic1-5.png)
![pic1-6](/gallery/参加FCC上海前端技术群线下Meetup/pic1-6.png)
![pic1-7](/gallery/参加FCC上海前端技术群线下Meetup/pic1-7.png)
![pic1-8](/gallery/参加FCC上海前端技术群线下Meetup/pic1-8.png)
![pic1-9](/gallery/参加FCC上海前端技术群线下Meetup/pic1-9.png)

要解决性能问题，先理解问题根源。

## [2、webRTC的场景创新和体验优化 - 韦躐晟；](https://drive.google.com/file/d/1t4rfAeYmmpdPO-f4vSgJUzA5J8HKq5a6/view)

## [3、点融CIS基础设施 - 林选伟；](https://docs.google.com/presentation/d/e/2PACX-1vTielVNY-uGXQduStugzXq4jPepTDns66AbtgyL3DNKmzx48W36Ngx_2QI438XUJFQ2C35aH7UWZF-Z/pub?start=false&loop=false&delayms=3000&slide=id.p1)

## [4、函数式语言: ClosureScript 在前端开发的体验 - 题叶；](https://gist.github.com/jiyinyiyong/561cd06ad1a1537dc8bcc15109bcf1cc)

关于题叶老师分享的 ClojureScript 我觉得我还是被题叶老师的键盘手速给深深的吸引，看着他在讲台上流畅的操作着他的 MacBook ，真的是强的不谈。关于 ClojureScript 的一些信息，直接点击标题前往查看即可。

## [5、在错误中寻找正确的方向: 应用升级和重构之路 - WiWi；](https://drive.google.com/file/d/0ByUlCDydqkHLcWNDY0R3N0FJMDZkRmxaWDRwdUJfQ3I1WDJZ/view)

## 相关链接

* [CSS性能优化](https://juejin.im/entry/59c3bf815188257e8b36b70a)
* [CSS性能优化的8个技巧](https://juejin.im/post/5b6133a351882519d346853f)

---

[FCC Shanghai](http://shanghai.freecodecamp.one)
