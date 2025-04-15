---
title: GitHub自定义域名解析
date: 2017-04-16 00:46:20
categories: "Git"
comments: true
featured_image: pic.jpg
tags:
- GitHub
---

<!-- no node -->

<!-- more -->

关于创建\*.github.io仓库以及购买域名，这里不再多说，请自行谷歌。

这里需要提到的是域名解析

首先为 GitHub Pages 设置自定义域名，简单来说，就是将你的域名写入一个文件名为 CNAME 的文件，放在根目录下并提交到 Github。然后在域名提供商里为你的域名添加两条 A 记录，分别指向 **192.30.252.153** 和 **192.30.252.154**。需要注意的是，如果开启了自定义域名支持，GitHub 提供的子域名 *.github.io 的 HTTPS 就无法生效了。这里放上 Github 官方的 Guide，[点击打开](https://help.github.com/articles/quick-start-setting-up-a-custom-domain/)