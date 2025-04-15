---
title: OS Yosemite下更新iMovie产生两个图标的解决方法
date: 2017-04-09 21:41:22
categories: "MAC"
comments: true
tags:
- Apple
- MAC OS
---

<!-- no node -->

<!-- more -->

在Yosemite的Preview版中，更新iMovie总是更新失败。

虽然最后更新成功了，但是LaunchPad中却有2个iMovie的图标，并且删不掉，在LaunchPad中一直有一个下载中的空进度条，的确很不爽。

那么下面有一个解决方法，可以尝试：

1\. 在`/private/var/folders/`目录下找到名字为`com.apple.dock.launchpad`的目录，删掉！

```bash
find /private/var/folders/ -d -name "com.apple.dock.launchpad" -type d -exec rm -rf '{}' \;
### sudo 获取最高权限
```

2\. 重启Dock

```bash
killall Dock 
```

好了，多余的iMovie拜拜了。