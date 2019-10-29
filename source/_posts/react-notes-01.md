---
title: React学习笔记（一）
date: 2017-05-22 21:06:26
categories: "React"
comments: true
featured_image: pic.jpg
tags:
- React
---

<!-- no node -->

<!-- more -->

## 1.全局安装React

```bash
$ npm install -g create-react-app
```

## 2.创建一个React-app

```bash
$ create-react-app my-app
```

## 3.启动一个简单的React-app

```bash
$ cd my-app
$ npm start
```

使用浏览器访问`http://localhost:3000/`

>为什么是 `$ npm start`？
>因为package.json里面有这么一段配置信息：

```json
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --env=jsdom",
    "eject": "react-scripts eject"
  }
```

>OK，我现在还不是很看的懂，但是基本可以猜测`build`就是发布，`test`就是测试。

## 4.Hello World

修改`./src/index.js`

```javascript
ReactDOM.render(<App />, document.getElementById('root'));
registerServiceWorker();

//将以下内容替换以上内容

ReactDOM.render(
  <h1>Hello, world!</h1>,
  document.getElementById('root')
);
```

OK，页面自动更新了。

## 5.无需npm安装使用React
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>Hello World</title>
    <script src="https://unpkg.com/react@latest/dist/react.js"></script>
    <script src="https://unpkg.com/react-dom@latest/dist/react-dom.js"></script>
    <script src="https://unpkg.com/babel-standalone@6.15.0/babel.min.js"></script>
  </head>
  <body>
    <div id="root"></div>
    <script type="text/babel">

      ReactDOM.render(
        <h1>Hello, world!</h1>,
        document.getElementById('root')
      );

    </script>
  </body>
</html>
```

注意**关键**的一句话`<script type="text/babel">`。

[HTML file 下载](https://facebook.github.io/react/downloads/single-file-example.html)

那么就先到这里吧，React之旅启程。

## 链接

* [React Docs](https://facebook.github.io/react/docs/installation.html)
* [React 中文社区](http://react-china.org/)
* [React中文索引](http://nav.react-china.org/)

## Sublime Text 3 插件安装建议

1. ReactJS
2. React Templates
3. React IDE

>安装步骤可以参考[Sublime Text 3 安装Vue语法高亮插件](http://zongzi531.com/2017/04/15/sublime-vue%E6%8F%92%E4%BB%B6/)

## 来自原单位的同事的鼓励

很高兴可以收到这样的鼓励，我会努力。

* 卡部损失一名悍将啊
* 有缘相识，加油粽子。人生还是非常有趣的
* 你那么优秀，一定是康庄大道