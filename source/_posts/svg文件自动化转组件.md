---
title: svg 文件自动化转组件
date: 2022-11-11 08:52:11
categories: "JavaScript"
comments: true
featured_image: pic1.jpeg
tags:
- JavaScript
---

<!-- no node -->

<!-- more -->

这是介于之前发布的[《<img> 使用 svg 后如何改变颜色》](https://zongzi531.com/2020/12/16/img%E6%A0%87%E7%AD%BE%E4%BD%BF%E7%94%A8svg%E5%90%8E%E5%A6%82%E4%BD%95%E6%94%B9%E5%8F%98%E9%A2%9C%E8%89%B2/) 的第四种解决方案。

由于方案三使用的是 CSS 解决，总的来说在部分场景下会不太方便，并且 `transform: translate(-1000px);` 的值始终不好控制，以至于在某些场景下会导致图片“消失”，就比较诡异。

## 方案四

避免使用 `<img>` 标签，而是直接使用脚本自动化解析 svg 文件，通过规律生成一个新的 JSX 组件子集，以此来解决问题。

首先，我们的 svg 文件来自 iconfont ，也以此为例子。

可以看到 svg 文件格式规律如下：

```xml
<?xml version="1.0" standalone="no"?><!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd"><svg class="icon" width="200px" height="200.00px" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg">/* svg content */</svg>
```

由此来扫描 svg 文件夹：

```js
const fs = require('fs-extra')
const path = require('path')

const svgs = path.resolve(__dirname, './svgs') // 存放 svg 文件夹

const files = fs.readdirSync(svgs) // svg 下的 svg 文件
```

那么我们现在可以获得到 svg 文件，接下来我们需要设计一下生成的 JSX 组件子集：

```typescript
export const svgname = <>/* svg content */</>
```

我们再来设计一下所需的 svg 组件（以 Vue 为例）：

```typescript
import * as svgMap from './svgs'

export default defineComponent({
  props: {
    name: {
      type: String,
      required: true,
    },
  },
  setup(props) {
    return () => <svg
      viewBox="0 0 1024 1024"
      version="1.1"
      xmlns="http://www.w3.org/2000/svg"
    >
      { svgMap[props.name] }
    </svg>
  },
})
```

到此，我们的 svg 组件已设计完成，那么我们只需要实现脚本，自动扫描 svg 文件夹即可实现我们的组件。

```js
const fs = require('fs-extra')
const path = require('path')

const svgs = path.resolve(__dirname, './svgs') // 存放 svg 文件夹
const newFilePath = path.resolve(__dirname, './svgs.tsx') // JSX 组件子集文件位置

const files = fs.readdirSync(svgs) // svg 下的 svg 文件

const svgContentReg = /<svg([\s\S]*?)>([\s\S]*?)<\/svg>/ // 正则分析 svg 文件内容，可以自行换成其他库，如 magic-string 之类的

const newFileContent = [] // JSX 组件子集内容，如需要避开 eslint ，可设置初始值为 ['/* eslint-disable */']

for (const file of files) {
  const filePath = path.resolve(svgs, './', file)
  const res = fs.readFileSync(filePath) // 读取 svg 文件内容
  const content = res.toString()
  const svgContent = content.match(svgContentReg).pop() // 获取 svg 中 svg 标签内的内容
  newFileContent.push(`export const ${file.replace(/-/, '').replace('.svg', '')} = <>${svgContent}</>`)
  // 文件名的解析这边只做了简单处理，可以根据自己的规范调整文件名到命名的转换， 同理 svg 文件内容
}

fs.writeFileSync(newFilePath, newFileContent.join('\n'))
```

到此，我们可以快乐的执行自动化脚本，无惧 svg 文件的变更，畅快更新 svg 组件资源，也能更好的改变 svg 的颜色咯～

但是，目前唯一的缺点就是手动执行的自动化脚本，如果有需要可以更抽象化成 webpack vite rollup 插件。
