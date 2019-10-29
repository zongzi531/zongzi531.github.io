---
title: 关于vue-router路由匹配的踩坑事件
date: 2017-06-26 21:51:47
categories: "Vue"
comments: true
tags:
- Vue
- vue-router
---

<!-- no node -->

<!-- more -->

又一个在使用Vue过程中遇到的新坑。首先踩这个坑都怪我自己看文档不够仔细

所以告诫大家，看文档真的一定要认认真真的一个个字的认真的品味。

**[vue-router 动态路由匹配](https://github.com/vuejs/vue-router/blob/dev/docs/zh-cn/essentials/dynamic-matching.md#匹配优先级)** 中有这么一段：

### 匹配优先级

有时候，同一个路径可以匹配多个路由，此时，匹配的优先级就按照路由的定义顺序：谁先定义的，谁的优先级就最高。

---

那么来说下业务场景吧，我这里的思路就是很简单的通过路由来控制显示内容和业务逻辑。

出问题的代码：

```javascript
import VueRouter from 'vue-router'

Vue.use(VueRouter)

const routes = [
  {
    path: '/app/:id'
  },
  {
    path: '/app/page1',
    meta: {title: 'page1'}
  }
]

const router = new VueRouter({
  routes
})

router.beforeEach((to, from, next) => {
  document.title = to.meta.title
  next()
})
```

问题在`meta.title`的值一直是`undefined`。

最终我尝试出三种解决方法：

### 方法一

最土的办法

```javascript
const temp = {
  '/page1': 'page1',
  '/page2': 'page2'
}

const routes = [
  {path: '/app/:id'}
]

const router = new VueRouter({
  routes
})

router.beforeEach((to, from, next) => {
  document.title = temp[to.path]
  next()
})

```

### 方法二

移除动态路径参数

```javascript
const routes = [
  {
    path: '/app/page1',
    meta: {
      title: 'page1'
    }
  },
  {
    path: '/app/page2',
    meta: {
      title: 'page2'
    }
  }
]
```

### 方法三

将动态路径参数置后，滞后匹配。

```javascript
const routes = [
  {
    path: '/app/page1',
    meta: {title: 'page1'}
  },
  {
    path: '/app/:id'
  }
]
```

---

最终，在项目中我选择使用方法二去解决，这样逻辑更清晰，并且维护成本低。

也算是更了解vue-router了吧~