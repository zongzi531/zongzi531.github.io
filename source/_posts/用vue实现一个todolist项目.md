---
title: 用Vue实现一个To Do List项目
date: 2017-06-02 22:49:10
categories: "Vue"
comments: true
featured_image: demo.gif
tags:
- Vue
---

<!-- no node -->

<!-- more -->

看再多不如多动手，那么先从简单的项目做起。

走，抄起键盘干。

今天给大家带来就是用Vue2.3 实现一个To Do List项目。

在这里只分享学习过程中的接触到的点，其他请看源码。可以直接点击访问到官网API节点，以下提到的API官网会有更详细的介绍。

## API

### [data](https://cn.vuejs.org/v2/api/#data)

Vue 实例的数据对象

### [methods](https://cn.vuejs.org/v2/api/#methods)

methods 将被混入到 Vue 实例中。可以直接通过 VM 实例访问这些方法，或者在指令表达式中使用。方法中的 `this` 自动绑定为 Vue 实例。

### [v-model](https://cn.vuejs.org/v2/api/#v-model)

在data中定义变量，监听`<input>`标签的值。

### [v-on](https://cn.vuejs.org/v2/api/#v-on)

绑定事件监听器。

可以用`@`代替。

`v-on:keyup.enter`：指按回车键事件

`@click`：指鼠标点击事件

值得一提的是我在码的过程中遇到了冒泡，Vue提供了一个很好的解决方案，那就是`.stop`，直接跟在后面使用即可。

### [v-show](https://cn.vuejs.org/v2/api/#v-show)

根据表达式之真假值，切换元素的 `display` CSS 属性。

`v-show`和`v-if`的区别和使用场景官网有介绍，有兴趣可以去阅读。[传送门](http://v1-cn.vuejs.org/guide/conditional.html#v-if-vs-v-show)

### [v-for](https://cn.vuejs.org/v2/api/#v-for)

此指令之值，必须使用特定语法 alias in expression ，为当前遍历的元素提供别名。

### [v-bind](https://cn.vuejs.org/v2/api/#v-bind)

用于动态地绑定一个或多个特性，或一个组件 prop 到表达式。

## 源码

### template

```html
  <div id="app">
    <h1 class="title">{ To Do List }<small class="by">by Zong</small></h1>
    <div class="list-wrapper">
      <div class="input-group">
        <input class="form-control" type="text" name="list" v-model="newTodo" v-on:keyup.enter="addTodo">
        <span class="input-group-addon" @click="addTodo"><span class="glyphicon glyphicon-plus btn-add"></span></span>
      </div>
      <p class="help-block help-note">来添加你的备忘录吧！</p>
      <ul class="nav nav-pills list-title" role="tablist" @click="reversetodo(todoflag)"><li role="presentation" class="active"><a href="#">未完成 <span class="badge">{{todos.length}}</span></a></li></ul>
      <ul v-show="todoflag">
        <li v-for="(todo, index) in todos" class="list-item" @click.stop="haveDo(index)">
          <p class="list-text" v-bind:title="todo.text">{{todo.text}}<span class="glyphicon glyphicon-remove btn-del" @click.stop="removeTodo(index)"></span></p>
        </li>
      </ul>
      <ul class="nav nav-pills list-title" role="tablist" @click="reversehavedo(havedoflag)"><li role="presentation" class="active"><a href="#">已完成 <span class="badge">{{havedos.length}}</span></a></li></ul>
      <ul v-show="havedoflag">
        <li v-for="(todo, index) in havedos" class="list-item" @click.stop="unDo(index)">
          <p class="list-text" v-bind:title="todo.text"><span class="through">{{todo.text}}</span><span class="glyphicon glyphicon-remove btn-del" @click.stop="removeUndo(index)"></span></p>
        </li>
      </ul>
    </div>
  </div>
```

### JavaScript
```javascript
export default {
  data () {
    return {
      newTodo: '',
      things: '',
      todoflag: true,
      havedoflag: false,
      todos: [],
      havedos: []
    }
  },
  methods: {
    addTodo: function () {
      let text = this.newTodo.trim()
      if (text) {
        this.todos.push({text: text})
        this.newTodo = ''
      }
    },
    removeTodo: function (index) {
      this.todos.splice(index, 1)
    },
    removeUndo: function (index) {
      this.havedos.splice(index, 1)
    },
    haveDo: function (index) {
      this.things = this.todos[index].text
      this.todos.splice(index, 1)
      this.havedos.push({text: this.things})
      this.things = ''
    },
    unDo: function (index) {
      this.things = this.havedos[index].text
      this.havedos.splice(index, 1)
      this.todos.push({text: this.things})
      this.things = ''
    },
    reversetodo: function (val) {
      this.todoflag = !val
    },
    reversehavedo: function (val) {
      this.havedoflag = !val
    }
  }
}
```

### CSS

```css
@import '../static/normalize-css/normalize.css';
@import '../static/bootstrap/dist/css/bootstrap.min.css';

...
```

这里引入CSS并不是使用`<style>`标签，而是依赖于css-loader，直接使用`@import`即可。

代码真的很节俭，很干净易懂。

**[GitHub仓库地址](https://github.com/zongzi531/vue-to-do-list)**