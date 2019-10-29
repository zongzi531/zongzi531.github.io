---
title: this & setTimeout 
date: 2017-06-20 22:05:55
categories: "JavaScript"
comments: true
thumbnail: /gallery/thisandsetTimeout/pic.jpeg
tags:
- JavaScript
---

<!-- no node -->

<!-- more -->

首先关于上一篇博客[《Vue实战经验分享》](http://zongzi531.com/2017/06/17/vue%E5%AE%9E%E6%88%98%E7%BB%8F%E9%AA%8C%E5%88%86%E4%BA%AB/)最后提到的问题进行解答，`exampleData`的结果是：`'example 1'`。

OK，那是什么样的业务场景会让我遇到这样的问题呢，当时我想将一段代码延迟执行，但是一开始我直接使用this，导致这句话并没有延迟执行。

那么之后我看到了这么一段很有趣的话：

>inside function called by setTimeout this is NOT VueJs object (is setTimeout global object), but self, also called after 2 seconds, is still VueJs Object.

也就是说，在`setTimeout`方法中的`this`指向的是一个全局对象，而非Vue对象。简而言之就是此`this`非彼`this`。

好，那么我们来认真认识一下他们吧！

---

## 关于this

在绝大多数情况下，函数的调用方式决定了`this`的值。`this`不能在执行期间被赋值，在每次函数被调用时`this`的值也可能会不同。

### this在全局上下文

在全局运行上下文中（在任何函数体外部），`this`指代全局对象，无论是否在严格模式下。

比如在浏览器中指`window`，Node中指`global`。

以下例子运行在浏览器中：

```javascript
console.log(this.document === document); // true

// 在浏览器中，全局对象为 window 对象：
console.log(this === window); // true

this.a = 37;
console.log(window.a); // 37
```

### this在函数上下文

在函数内部，`this`的值取决于函数是如何调用的。

在非严格模式下执行，此时的`this`的值会默认设置为全局对象。然而，在严格模式下，`this`将保持他进入执行环境时的值，所以下面的`this`将会默认为`undefined`，如果`this`未被执行的上下文环境定义，那么它将会默认为`undefined`。

```javascript
function f1(){
  return this;
}

f1() === window; // true

function f2(){
  "use strict"; // 这里是严格模式
  return this;
}

f2() === undefined; // true
```

## 关于setTimeout

>这里只提到关于`this`的问题

当你向 `setTimeout()` (或者其他函数也行)传递一个函数时，该函数中的`this`会指向一个错误的值。这个问题在 [JavaScript reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this#As_an_object_method) 中进行了详细解释。

### 解释

由`setTimeout()`调用的代码运行在与所在函数完全分离的执行环境上。这会导致，这些代码中包含的 `this` 关键字会指向 `window` (或全局)对象，这和所期望的`this`的值是不一样的。查看下面的例子：

```javascript
myArray = ["zero", "one", "two"];
myArray.myMethod = function (sProperty) {
    alert(arguments.length > 0 ? this[sProperty] : this);
};

myArray.myMethod(); // prints "zero,one,two"
myArray.myMethod(1); // prints "one"
setTimeout(myArray.myMethod, 1000); // prints "[object Window]" after 1 second
setTimeout(myArray.myMethod, 1500, "1"); // prints "undefined" after 1,5 seconds
// let's try to pass the 'this' object
setTimeout.call(myArray, myArray.myMethod, 2000); // error: "NS_ERROR_XPC_BAD_OP_ON_WN_PROTO: Illegal operation on WrappedNative prototype object"
setTimeout.call(myArray, myArray.myMethod, 2500, 2); // same error
```

正如你所看到的一样，我们没有任何方法将`this`对象传递给回调函数。

一个可用的解决 "this" 问题的方法是使用两个非原生的`setTimeout()` 和 `setInterval()` 全局函数代替原生的。该非原生的函数通过使用`Function.prototype.call` 方法激活了正确的作用域。下面的代码显示了应该如何替换：

```javascript
// Enable the passage of the 'this' object through the JavaScript timers
 
var __nativeST__ = window.setTimeout, __nativeSI__ = window.setInterval;
 
window.setTimeout = function (vCallback, nDelay /*, argumentToPass1, argumentToPass2, etc. */) {
  var oThis = this, aArgs = Array.prototype.slice.call(arguments, 2);
  return __nativeST__(vCallback instanceof Function ? function () {
    vCallback.apply(oThis, aArgs);
  } : vCallback, nDelay);
};
 
window.setInterval = function (vCallback, nDelay /*, argumentToPass1, argumentToPass2, etc. */) {
  var oThis = this, aArgs = Array.prototype.slice.call(arguments, 2);
  return __nativeSI__(vCallback instanceof Function ? function () {
    vCallback.apply(oThis, aArgs);
  } : vCallback, nDelay);
};
```

>注：JavaScript 1.8.5 引入了 `Function.prototype.bind()` 方法，该方法允许显式地指定函数调用时 `this` 所指向的值 。该方法可以帮助你解决 `this` 指向不确定的问题。

上面这段代码中最重点的一句话就是`var oThis = this`。

其实说了这么多，也很容易把这个问题延伸出去，那就可以聊一聊：**JavaScript上下文与作用域**。

我只能说目前我学习到的内容还是很浅的，还做不到很好的总结。

待我再揣摩揣摩吧~

## 参考与学习

* [Vue equivalent of setTimeOut? - Stack Overflow](https://stackoverflow.com/questions/38399050/vue-equivalent-of-settimeout)
* [this - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)
* [window.setTimeout - API | MDN](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout)