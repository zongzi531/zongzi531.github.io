---
title: Chrome 浏览器性能排查指南
date: 2023-01-12 13:45:21
categories: "Performance"
comments: true
tags:
- Chrome
- Performance
- 浏览器
- Devtools
- 性能优化
---

<!-- no node -->

<!-- more -->

# 使用 Chrome DevTools Memory 排查问题

1. 打开 Chrome DevTools 选择「Memory」选项卡，可以现在 JavaScript VM instance 下实时观察 JavaScript 的内存变化。

2. 那么我们结合 Chrome 任务管理器配合进行排查内存变化，通常浏览器会在空闲时间进行 GC 操作，如果存在内存泄露的话，可以从几次 GC 操作后发现内存是有上升，如果一直攀升并未得到正常下降的话，我们需要有规律的打印堆快照，并进行对比。

3. 我们默认选择「Heap snapshot」打印堆快照，尝试打印 2-3 次 GC 操作后的堆快照信息，并选择「Comparison」与上一份快照或者其他快照进行比对，从中排查问题。

4. 可以通过 #New 列和 #Deleted 列观察内存创建和销毁的数量，按照 #Delta 列进行降序排序，观察堆快照的使用情况从中排查问题。

# 常见案例

## 1. 发现 (closure) 出现问题

1. 比如轮询回调函数，未在调用结束后释放；

2. 重复的事件监听，未释放过期的事件监听；

3. 模块文件内容临时变量存储未被清理；

## 2. 发现 (array) 出现问题

1. 比如 Vue 下的监听导致数组重复监听；

# 使用 Chrome 任务管理器协助排查问题

1. 点击 Chrome 右侧的「自定义及控制 Google Chrome」按钮，选择「更多工具」，打开「任务管理器」。

2. 可以右键点击顶部表头对任务管理器表头进行筛选显示，如下图。

![](pic1.png)

3. 建议开启「内存占用空间」、「CPU」、「网络」、「GPU内存」、「JavaScript使用的内存」来观察当前 Chrome Tab 标签页的使用情况。

# 改善方案

1. 养成良好的编码习惯

2. 注重代码审查

3. 沉淀最佳实践及排查指南
