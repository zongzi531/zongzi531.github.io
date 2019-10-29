---
title: 浅析 jQuery （一）
date: 2018-09-22 00:51:18
categories: "JavaScript"
comments: true
featured_image: jQuery.png
tags:
- JavaScript
- jQuery
---

<!-- no node -->

<!-- more -->

> 经过之前博客提到的一些关于 jQuery 的事情，以及在设计模式中感受到 jQuery 的惊艳，我试着在这个月阅读 jQuery 源码。
>
> 其实从我开始学习 JavaScript 至工作至今，使用 jQuery 的是少之又少，对于其 API 的熟悉程度是极其的低，但是 jQuery 毕竟是那么流行的库，它又涵盖了那么多经典而又优雅的设计模式思想，所以值得我去阅读，学习。
>
> 可能是我接触 JavaScript 比较晚吧，2016年8月，那时 Vue 已经兴起，我从那时起，我虽然很简单的实践了 jQuery ，但是我仍然把主要的精力花在了学习 Vue, React 这样的框架上。当然我还是很痴迷学习 JavaScript ，所以当学习到 JavaScript 设计模式时，我发现我需要从 jQuery 中汲取一些我想要的知识。
>
> 那么动手起来，我 Clone 了 jQuery 3.3.1 版本，从此开始阅读起来。

## 关于 undefined 的奇淫技巧

虽然 jQuery 源码中并没有这样这块部分，但是我在阅读文章时，看到这个有意思的事情。好比说下面一段代码：

```javascript
(undefined => console.log('undefined:', undefined))('TEST')
```

你会发现在这个 IIFE 中 `undefined` 被声明成了变量，并且值也同时被改变了，那么当你需要将某些值与你认为的 `undefined` 去比较时，结果就会令人意想不到。

奇淫技巧就在此出现了，直接看实现代码：

```javascript
((window, undefined) => {
  console.log('undefined:', undefined)
})(window)
```

点到为止，我相信你看到这段代码就能理解，如何防止 `undefined` 被覆盖的问题。

总结一下：

1. `window` 和 `undefined` 都是为了减少变量查找所经过的 scope 作用域。当 `window` 通过传递给闭包内部之后，在闭包内部使用它的时候，可以把它当成一个局部变量，显然比原先在 `window` scope 下查找的时候要快一些。
2. `undefined` 也是同样的道理，其实这个 `undefined` 并不是 JavaScript 数据类型的 `undefined`，而是一个普普通通的变量名。只是因为没给它传递值，它的值就是 `undefined` ，`undefined` 并不是 JavaScript 的保留字。

## jQuery 巧妙的把 new 给藏了起来

首先，我们来介绍下 `new` 运算符

创建一个用户自定义的对象需要两步：

1. 通过编写函数来定义对象类型。
2. 通过 `new` 来创建对象实例。

创建一个对象类型，需要创建一个指定其名称和属性的函数；对象的属性可以指向其他对象，看下面的例子：

当代码 `new Foo(...)` 执行时，会发生以下事情：

1. 一个继承自 `Foo.prototype` 的新对象被创建。
2. 使用指定的参数调用构造函数 `Foo` ，并将 `this` 绑定到新创建的对象。`new Foo` 等同于 `new Foo()`，也就是没有指定参数列表，`Foo` 不带任何参数调用的情况。
3. 由构造函数返回的对象就是 `new` 表达式的结果。如果构造函数没有显式返回一个对象，则使用步骤1创建的对象。（一般情况下，构造函数不返回值，但是用户可以选择主动返回对象，来覆盖正常的对象创建步骤）

当你每次使用 `$(...)` 或者 `jQuery(...)` 时，返回的对象就是 jQuery 的实例， jQuery 实例的特征很明显，是类数组对象结构的。

但是在创建 jQuery 实例的时候，开发者却不需要使用到 `new` 操作符，这是因为 jQuery 已经帮你把这一步给省略了，确实奇淫技巧。

让我们来看一下相关核心源码：

```javascript
// jquery/src/core.js
var
  ...
  // Define a local copy of jQuery
  jQuery = function( selector, context ) {

    // The jQuery object is actually just the init constructor 'enhanced'
    // Need init if jQuery is called (just allow error to be thrown if not included)
    return new jQuery.fn.init( selector, context );
  }

...

jQuery.fn = jQuery.prototype = {
  ...
  constructor: jQuery,
  ...
};
```

这里你会发现，在调用 jQuery 函数时，返回的就是 `jQuery.fn.init` 函数使用 `new` 操作符实例化的实例。但是这里会牵扯到一个问题。 `jQuery.fn.init` 函数的实例并不会是 jQuery 函数的实例，是时候展示奇淫技巧的代码了：

```javascript
// jquery/src/core/init.js
var rootjQuery,
  ...
  init = jQuery.fn.init = function( selector, context, root ) {

    ...

    return jQuery.makeArray( selector, this );
  };

// Give the init function the jQuery prototype for later instantiation
init.prototype = jQuery.fn;
```

这里你会发现， `jQuery.fn.init.prototype` 是指向 `jQuery.prototype` 的，这就是他的过人之处，这样`jQuery.fn.init` 函数的实例就是 jQuery 函数的实例。

## jQuery.extend 和 jQuery.fn.extend

**jQuery.extend()** : 将两个或更多对象的内容合并到第一个对象。

**jQuery.fn.extend()** : 一个对象的内容合并到jQuery的原型，以提供新的jQuery实例方法。

在源码中，这两个函数指向的是同一个函数，但是为什么功能上会有差异呢，我们直接上核心源码部分：

```javascript
// jquery/src/core.js
jQuery.extend = jQuery.fn.extend = function() {
  ...

  // Extend jQuery itself if only one argument is passed
  if ( i === length ) {
    target = this;
    i--;
  }

  ...

  // Return the modified object
  return target;
};
```

看到 `target = this;` 这段代码，你应该能明白，执行函数时 `this` 的指向。回顾 `jQuery.fn = jQuery.prototype`，我想你应该就能明白上面2个方法的解释为何会存在差异了。

1. 在 **jQuery.extend()** 中，`this` 的指向是 jQuery 对象（或者说是 jQuery 类），所以这里扩展在 jQuery 上；
2. 在 **jQuery.fn.extend()** 中，`this` 的指向是 fn 对象，前面有提到 `jQuery.fn = jQuery.prototype` ，也就是这里增加的是原型方法，也就是对象方法。

## jQuery 的 链式调用 和 回溯处理

链式调用其实很好理解，就是返回当前实例，就是 `return this;` 这么简单。

回溯处理却比这个有意思了，我们直接先上源码：

```javascript
// jquery/src/core.js
jQuery.fn = jQuery.prototype = {
  ...

  // Take an array of elements and push it onto the stack
  // (returning the new matched element set)
  pushStack: function( elems ) {

    // Build a new jQuery matched element set
    var ret = jQuery.merge( this.constructor(), elems );

    // Add the old object onto the stack (as a reference)
    ret.prevObject = this;

    // Return the newly-formed element set
    return ret;
  }
  ...

  eq: function( i ) {
    var len = this.length,
      j = +i + ( i < 0 ? len : 0 );
    return this.pushStack( j >= 0 && j < len ? [ this[ j ] ] : [] );
  },

  end: function() {
    return this.prevObject || this.constructor();
  }
  ...
};
```

在 jQuery 中凡是改变 jQuery 类数组的操作，都会调用 `pushStack` 方法，而回溯处理当然也是建立在 `pushStack` 方法之上的，因为在执行 `pushStack` 方法时，有这么一行代码 `ret.prevObject = this;`，它就是实现回溯处理的根源。

可以说是相当的精妙，总结下来就是：

* jQuery 链式调用：`return this;`
* jQuery 增栈处理：`return this.pushStack(...);`
* jQuery 回溯处理：`return this.prevObject;`

---

虽然，我把 jQuery 3.3.1 版本 初略的读完了，但是写到这里发现 jQuery 还有很多内容没有提到，我也还暂时没有很好的理解透彻，暂时先到这里，我把标题改成了《浅析jQuery（一）》，希望可以激励到我。还有就是关于剩余的部分，可以先查阅相关链接。

## 相关链接

* [jQuery API](https://api.jquery.com/)
* [jQuery API 中文文档](https://www.jquery123.com/)
* [[chokcoco/jQuery-] jQuery- v1.10.2 源码解读](https://github.com/chokcoco/jQuery-)
* [[慕课网] jQuery源码解析（架构与依赖模块）](http://www.imooc.com/learn/172)
* [[慕课网] jQuery源码解析（DOM与核心模块）](http://www.imooc.com/learn/222)
* [jQuery源码分析系列 - 【艾伦】 - 博客园](http://www.cnblogs.com/aaronjs/p/3279314.html)
* [jQuery架构设计与实现（2.1.4版本）](https://github.com/JsAaron/jQuery)
* [zongzi531/read-jquery](https://github.com/zongzi531/read-jquery)
