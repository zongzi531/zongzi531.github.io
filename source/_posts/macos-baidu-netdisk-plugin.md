---
title: macOS Mojave 系统中启动程序变卡
date: 2019-07-02 09:34:48
categories: "MAC"
comments: true
featured_image: Apple-OS-Mojave1020x660.jpg
tags:
- Apple
- MAC OS
---

<!-- no node -->

<!-- more -->

安装 [BaiduNetdiskPlugin-macOS](https://github.com/CodeTips/BaiduNetdiskPlugin-macOS)（这项目有毒） 后， macOS 发生了奇奇怪怪的问题：

- 所有的程序启动的时候都变"卡"了
- 启动时系统的界面会"不动"
- 可能会伴随鼠标指针停滞，诸如此类。

很符合CPU特别高导致的散热进程侵占或就是特别高的问题，但经检查并非如此

尝试解决步骤：

- 清除 `/Library/Keychains/`

重启解决。

---

参考链接

- [mac mojave 系统中启动用户程序"卡住" 例如: office idea atom等](https://my.oschina.net/u/1995274/blog/2252475/)

---

*日志*

**2019年07月05日** 又发生此情况，尝试[macOS Mojave 10.14.1更新后，重启进入桌面假死](https://discussionschinese.apple.com/thread/140139134)教程无法解决，仍是通过清除解决问题，但是是什么引发的此问题呢？ `Keychains` 里又是什么？
