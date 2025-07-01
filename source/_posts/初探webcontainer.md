---
title: 初探 WebContainer
date: 2025-07-01 19:09:55
categories: "JavaScript"
comments: true
featured_image: pic1.jpg
tags:
- WebContainer
- Web
- Container
---

<!-- no node  -->

<!-- more -->

因为需要结合 AI 的关系，对 WebContainer 进行学习，其实官方的文档比较完整，很容易入手，并应用到项目当中。

但是在使用的过程中会遇到一些问题，因为时间的关系也没有特别花精力去解决。

在 WebContainer 加载项目时，可以使用 `FileSystemTree` 来直接挂载项目，官方也非常贴心的提供了 Node.js 的 `@webcontainer/snapshot` 包来向浏览器推送流，让浏览器更轻松的进行挂载。

在实践的过程中，我并没有尝试推送的时候带有 `node_modules` 目录，而是直接在浏览器进行 `install` 来完成依赖的安装。

非常棒的是，在第一次安装后（带有缓存的情况）再次安装时，并不会发起网络请求，而是结合浏览器缓存，非常的快。

简单尝试过使用 `@webcontainer/api` 的 `export` 来导出 `node_modules` 的 `FileSystemTree` 的信息，但是并没有得到较好的效果。也是因为时间关系，暂时没有在第一次加载这块有更好的实践。

希望后续有时间对这块能有一个较好的优化效果，嘿嘿。

并且，当项目“较大”时，就算使用 Vite 来进行启动也会非常的缓慢，这在用户体验上其实是致命的，当然了，我选择一种避开的方式，来优化启动的速度，有了一些显著的效果。

同时，结合 AI 的使用，可以让用户只需与 AI 对话，即可完成项目的开发，算是一种较为新奇的体验吧。

当然了，重点还是后续的优化，以给用户提供更好的体验。
