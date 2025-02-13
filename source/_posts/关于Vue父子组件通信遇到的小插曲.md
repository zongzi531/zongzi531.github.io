---
title: 关于Vue父子组件通信遇到的小插曲
date: 2018-02-12 19:12:28
categories: "Vue"
comments: true
thumbnail: https://cn.vuejs.org/images/logo.png
tags:
- Vue
---

<!-- no node -->

<!-- more -->

近期，从React重新回到Vue开发，那么选择了UI组件库Vux，那么简单来说一下遇到的问题吧。

使用到XDialog组件，可以说Vux已经将这个组件封装的很好使用了，但是我就是想把他根据项目的业务情况再封装一遍，那么有趣的是，`v-model`的值是又`props`传入的，文档中提供的方法有限，并且我需要使用到`hide-on-blur`属性。

然后控制台就报错了：
```javascript
[Vue warn]: Avoid mutating a prop directly since the value will be overwritten whenever the parent component re-renders. Instead, use a data or computed property based on the prop's value. Prop being mutated: "..."
```

当时我就愣住了，很难受吧！好的，再次逼着我去读源码，真的很OJBK。

当我看到`hide`方法的时候我意外的惊喜：

```javascript
hide () {
  if (this.hideOnBlur) {
    this.$emit('update:show', false)
    this.$emit('change', false)
    this.$emit('on-click-mask')
  }
}
```

我发现了一个隐藏的事件监听器可以用，`@on-click-mask`，但是很遗憾的是，在执行这个事件之前，`v-model`的值就被修改了！

那么很抱歉，我放弃了这个想法，查阅了一下网上的资料，找到了一丝灵感。

代码最终实现如下：

```html
<template>
  <div v-transfer-dom>
    <x-dialog v-model="currentShow" :hide-on-blur="true"></x-dialog>
  </div>
</template>

<script>
import { XDialog, TransferDomDirective as TransferDom } from 'vux'

export default {
  props: {
    show: {
      type: Boolean,
      default: false
    }
  },
  computed: {
    currentShow: {
      get () {
        return this.show
      },
      set (val) {
        this.$emit('on-Click-hide', val)
      }
    }
  },
  directives: {
    TransferDom
  },
  components: {
    XDialog
  }
}
</script>
```

确实需要`props`传入`show`，但是绑定在`v-model`上的却不能是`show`，不然就会有上面那个报错，而是自定义一个计算属性，并设置他的`get`和`set`方法。

我们修改了原有的`set`方法，将其以一个事件监听器的形式传递至外层，由外层控制`props`的值即可。OJBK，问题解决了。

其实后续可以回想，我其实可以简单的只封装XDialog中的slot内容即可，别问我为什么要把XDialog也封装进去，没有为什么，就是喜欢。

再回想我解决问题的思路：

0. 呀！报错了
1. 先搞清楚问题出在哪儿
2. 试着从源码找突破口
3. 各种搜各种试
4. 狗屎运，问题给我解决了
5. 然后看心情记一笔

---
>扯一扯环节

说到试着从源码找突破口，我就想起使用Element @1.4.10版本的时候，因为使用到vuex，所以修改数据要显示的提交`commit`，由于库提供的方法是直接清空表单数据的，会限制我使用vuex。

如果我手动提交commit方法，表单上的验证信息并不会清除。那么我就开始读源码了……，然后在源码中找到了惊喜！解决了我的需求。

解决方法：[5.1 element-ui 清空所有表单项的验证信息](http://zongzi531.github.io/2017/11/25/%E8%BF%9B%E9%98%B6es6%E4%BA%8C/)

马上要过新年了，提前祝大家新年快乐哦！
