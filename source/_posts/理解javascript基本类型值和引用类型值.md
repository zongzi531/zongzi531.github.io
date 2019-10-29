---
title: 理解Javascript基本类型值和引用类型值
date: 2017-04-13 21:51:35
updated: 2017-04-24 21:25:07
categories: "JavaScript"
comments: true
tags:
- JavaScript
---

<!-- no node -->

<!-- more -->

## 数据类型

ECMAScript中有5种简单数据类型：`Undefined`、`Null`、`Boolean`、`Number`、`String`，和一种复杂数据类型：`Object`。相信大家都对这几个数据类型有所认识，那这边就不再赘述了。


## 基本类型值和引用类型值

首先，介绍下基本的概念，**基本类型值**是指简单的数据段，**引用类型值**是指那些可能由多个值构成的对象。

```javascript
let a = undefined; 	//Undefined
let b = null;		//Null
let c = true;		//Boolean
let d = 0;		//Number
let e = "e";		//String
```

这5种数据类型是按照值访问的。

那么什么是引用类型呢？看下面的代码

```javascript
let cat = new Object();
cat.name = "tom";
console.log(cat.name);		//tom
```

大家一定很好奇那基本类型值和引用类型值到底有什么区别呢？下面给大家举个例子：

```javascript
//基本类型值
let a = 10;
let b = a;
b = 20;
console.log(a);		//这时a的值是多少？
console.log(b);		//这时b的值是多少？

//引用类型值
let x = { a: 10 };
let y = x;
y.a = 20;
console.log(x.a);	//这时x.a的值是多少？
console.log(y.a);	//这时y.a的值是多少？
```

先讲讲基本类型值的代码执行过程，`let a = 10;`的时候a的值是10，`let b = a;`执行之后，b的值也同样是10，虽然a和b的值相等，但是其实他们的值已经互相不影响了，不会影响到对方。所以当执行`b = 20;`的时候，a的值并不会受到改变。

整个执行过程见下图解：

![基本类型值](/gallery/理解javascript基本类型值和引用类型值/基本类型值1.jpeg)

**基本类型值就是保存了一个值。**

那么引用类型值与基本类型值到底有什么区别呢？引用类型值实际保存的一个指针，那么这个指针指向存储在堆中的一个对象。

提到**堆**，就能把**栈**扯进来，实际上基本类型值是保存在栈内存中的。

再讲讲引用类型值的代码执行过程，`let x = { a: 10 };`首先定义了一个x，a属性的值是10。其实在这里，`{ a: 10 };`是存放在堆内存中的，变量x保存的就是`{ a: 10 };`的指针地址，`let y = x;`执行了之后，变量y复制了变量x的指针地址。最后在执行`y.a = 20;`时，相当于把a属性的值改成了20，因此y.a发生改变时，x.a也同时发生了变化。当然啦，这里的a的值是保存在栈内存中的喔。

整个执行过程见下图解：

![引用类型值](/gallery/理解javascript基本类型值和引用类型值/引用类型值1.jpeg)

那么接下来就有个问题了，如何切断指针地址呢？那就是赋值一个空对象。

es5的话可以直接用`for ...in`遍历拷贝属性。那么这里讲一下新方法吧。

ECMAScript 6提供了一个`Object.assign()`的方法。使用方法：Object.assign方法的第一个参数是目标对象，后面的参数都是源对象。

```javascript
let x = { a: 10 };
let y = x;
let z = {};
Object.assign(z, x);
y.a = 20;
z.a = 30;
```

了解更多可以查看[《认识Object.assign以及JavaScript中的浅拷贝和深拷贝》](http://zongzi531.com/2017/04/24/%E8%AE%A4%E8%AF%86Object-assign%E4%BB%A5%E5%8F%8AJavaScript%E4%B8%AD%E7%9A%84%E6%B5%85%E6%8B%B7%E8%B4%9D%E5%92%8C%E6%B7%B1%E6%8B%B7%E8%B4%9D/)

相信通过这个例子和图解已经可以理解什么是基本类型值和引用类型值了吧！是不是很有趣！

感谢阅读至此，文中若有错误请大牛指点，可以邮件我。联系方式当然是见Github啦。
