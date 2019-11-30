---
title: 走进CSS世界
date: 2018-06-10 10:12:02
categories: "CSS"
comments: true
tags:
- CSS
- Vue
---

<!-- no node -->

<!-- more -->

>五月天，工作忙也好，双休日忙也好，总之，《CSS世界》并没有在五月份读完 :)。
>也好，算是给自己一个短暂的放松吧。
>《CSS世界》也快读完了，可以说，我对CSS2.1的了解在这本书中更加深入了，然而CSS3却是一片全新的世界……
>在学习前端之初，我读了《CSS揭秘》，我相信我会反复读这两本书以及去实践。
>当然啦，下一本书安排是《深入浅出node.js》，也在近期，node之父ry启动了新的项目deno。
>对我而言，饭当然是要一口一口吃啦，不着急，脚下的砖一块一块砌。
>接下来，来分享下这一个多月的收获：

## margin合并

之前写移动端项目的时候，一直不理解子元素的`margin-top`属性会影响到父元素这一现象，一直是使用`padding-top`解决的。当然这也可以解决问题，但是如果我将子元素封装成组件，在某些视觉场景上确实有些不妥，`margin`属性确实应该在子元素上。

margin合并的3种场景：
1. 相邻兄弟元素margin合并。
2. 父级和第一个/最后一个子元素。
3. 空块级元素的margin合并。

我遇到的是场景2，我采用的解决方案是将父元素设置为块状格式化上下文元素，设置`overflow: hidden;`。

正确的理解margin合并现象，减少了我不少困扰。

```html
<div class="wrapper">
	<div class="child"></div>
</div>
```

```css
.wrapper {
	overflow: hidden;
}
.child {
	margin-top: 10px;
}
```

## 与众不同的min-height

移动端开发，我会在最外层设置一个`.wrapper`父元素，用于包裹页面。关于`height: 100%;`这个问题，要涉及到需要设置`html, body`的`height`属性值，真的要讲要涉及到文档流，建议还是看书吧，所以我选择使用`min-height: 100vh;`来实现容器高度为屏幕高度。

```css
.wrapper {
	overflow: hidden;
	min-height: 100vh;
}
```

## 神奇的box-sizing

移动端开发中会有很多情况，按钮固定在屏幕底部或是底部固定着其他的视觉元素。多数采用`position: fixed;`，会造成如果内容超过屏幕高度，会被该固定元素遮挡。我这里采用的解决方案是为`.wrapper`父元素添加`box-sizing: border-box; padding-bottom: ?px;`属性来解决此问题。利用属性的天然优势，可以解决当内容不超过屏幕高度时，不会有`padding-bottom: ?px;`超过屏幕高度。

```html
<div class="wrapper">
	<div class="bottom-btn"></div>
</div>
```

```css
.wrapper {
	overflow: hidden;
	min-height: 100vh;
	box-sizing: border-box;
	padding-bottom: 50px;
}
.bottom-btn {
	position: fixed;
	left: 0;
	bottom: 0;
	width: 100vw;
	height: 50px;
}
```

## padding和border提升用户体验

有些icon在设计时会有些小，在移动端上用户点击时，会很难点，这时使用`padding: ?px`来扩大元素尺寸来提升用户体验，说到这里，其实还有另外一种实现方式是设置`border`属性，要让其和`padding`一样“隐形”的话，可以设置`border-color: transparent;`。

## 两栏——单栏自适应

这样的视觉场景无处不在，书中提及的CSS2.1解决方案，使用`float`浮动属性来实现，但是我使用`flex`属性来实现。

## 更多的利用伪元素实现页面视觉效果

比如`1px`的`border`，在这次移动端项目开发中，我使用`::after`和`::before`，实现凹槽效果，为父元素添加`overflow: hidden;`，伪元素利用绝对定位`position: absolute;`属性实现覆盖。

## background的奥秘

利用`background`可以接受多参数的特性实现有背景色下的icon居中视觉效果。

```html
<i class="icon"></i>
```

```css
.icon {
	display: block;
	width: 42px;
	height: 42px;
	background: url('./icon.png'),
	            rgba(250,251,252,1);
	background-repeat: no-repeat;
	background-size: 18px;
	background-position: 12px;
}

```

## 制作圆点+圆环小图标

```html
<i class="icon"></i>
```

```css
.icon {
    display: block;
    width: 8px;
    height: 8px;
    padding: 2px;
    border: 2px solid rgba(7, 92, 247, 1);
    border-radius: 50%;
    background-color: rgba(7, 92, 247, 1);
    background-clip: content-box;
}
```

## animation的乐趣

这里是Vue组件相关代码，逻辑没有什么亮点之处，只是记录下当页面进入时，我将`animation`动画延时设置为0的一段代码内容。

```html
<template>
  <div :class="{ 'cc-detail': true, 'cc-detail-s': !visable, 'cc-detail-b': visable }">
    <div class="click-wrapper" @click="handleClick">
      <span>{ {CarCellDetailTitle} }</span>
      <img :class="{ 'cc-unfold': true, 'cc-ani-off': first }" v-if="!visable" :src="carCellDetailIconDown" />
      <img :class="{ 'cc-packup': true, 'cc-ani-off': first }" v-if="visable" :src="carCellDetailIconUp" />
      <!-- if you want anything -->
    </div>
  </div>
</template>
```

```javascript
import { CarCellDetailTitle, carCellDetailIconUp, carCellDetailIconDown } from '@/config'

export default {
  data () {
    return {
      first: true,
      visable: false,
      CarCellDetailTitle,
      carCellDetailIconUp,
      carCellDetailIconDown
    }
  },
  methods: {
    handleClick () {
      this.visable = !this.visable
      this.first = false
    }
  }
}
```

```css
.cc-detail {
	box-sizing: border-box;
	overflow: hidden;
	margin: -30px 20px 0;
	padding-top: 20px;
	background: #fff;
	box-shadow: 0px 2px 6px 0px rgba(0,0,0,0.04);
	border-radius: 4px;
	font-size: 12px;
	text-align: center;
	color: rgba(177, 174, 181, 1);
	transition: all 0.4s ease-in-out;
}
.cc-detail-s {
	transition: all 0.4s ease-out !important;
	max-height: 60px;
}
.cc-detail-b {
	max-height: 100vh;
}
.click-wrapper {
	height: 40px;
	line-height: 40px;
}
.click-wrapper img {
	margin-left: 8px;
	vertical-align: -4px;
	width: 18px;
	height: 18px;
}
.cc-small-cell  {
	margin: 0 !important;
	border: none !important;
}
.cc-ani-on {
	animation-play-state: running;
}
.cc-ani-off {
	/* animation-play-state: paused !important; */
	/* hank rotate */
	animation: 0s unfold !important;
}
.cc-unfold {
	animation: 0.4s unfold;
}
.cc-packup {
	animation: 0.4s packup;
}
@keyframes unfold {
	from { transform: rotate(180deg); }
	to   { transform: rotate(0deg); }
}
@keyframes packup {
	from { transform: rotate(-180deg); }
	to   { transform: rotate(0deg); }
}
```


## Vue开发相关

1. 组件化发送短信按钮，将内部封装的倒计时函数使用`$emit`传入回调函数参数，这样可以通过组件外层调用时，来判断是否需要启用这个倒计时函数，需要注意的是，倒计时函数需要`bind`前组件的`this`。
2. 组件化单选框按钮，利用`this.$emit('update:value', !this.value)`更新`props`的`value`值，在使用组件时，需要使用`:value.sync`即可。
3. 学习开源项目，约定`type`或`small`（`large`）属性来控制组件样式。
4. 此次移动端项目开发，我未使用vuex，而是将一些数据存至sessionStorage，考虑到用户体验上的话还是有区别的，我会认为vuex会更好。
5. 项目中多次涉及到Vue组件生命周期问题，问题在于子组件与父组件的生命周期，未来开发切记，便于发现问题。


## 适配相关

涉及到移动端开发，考虑到适配，我选用rem解决适配问题，之前项目都是采用编辑器插件，直接在编写代码的时候转换为rem单位，但是问题在于转换后的rem单位在代码审查或者修改时，可读性极低，根本没办法直接知道值是多少，所以本次采用webpack打包时自动编译成rem单位技术。

```html
<script src="./static/js/amfe-flexible.min.js"></script>
```

```javascript
const webpack = require('webpack')
// const px2rem = require('postcss-px2rem')
// 因为无法兼容vux，所以选用下面这个
const px2rem = require('postcss-plugin-px2rem')


const webpackConfig = {
  ...,
  module: {
    rules: [
	  ...,
      {
        test: /\.(css|less|scss)(\?.*)?$/,
        loader: 'style-loader!css-loader!sass-loader!less-loader!postcss-loader'
      }
    ]
  },
  plugins: [
    new webpack.LoaderOptionsPlugin({
      vue: {
        postcss: [
          px2rem({ rootValue: 37.5, propWhiteList: [] })
        ]
      },
    })
  ]
}
```

## PR贡献之路

1. [node-abc #2](https://github.com/liuxing/node-abc/pull/2)
2. [v-charts #338](https://github.com/ElemeFE/v-charts/pull/338)
