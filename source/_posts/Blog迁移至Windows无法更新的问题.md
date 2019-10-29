---
title: Blog迁移至Windows无法部署的解决方案
date: 2017-05-07 17:53:33
categories: "Blog"
comments: true
tags:
- hexo
- Blog
---

<!-- no node -->

<!-- more -->

在使用`hexo generate -d`命令的时候，git Bash（建议不要使用cmd.exe）一直报错：

```bash
bash: /dev/tty: No such device or address
error: failed to execute prompt script (exit code 1)
fatal: could not read Username for 'https://github.com': No error
```

直接修改_config.yml即可解决。（前提已经在Windows上配置好git，[设置git](https://help.github.com/articles/set-up-git/)）

```
deploy:
  type: git
  repository: ssh://git@github.com/zongzi531/zongzi531.github.io
  branch: master
```

那么就大功告成了！

参考：[hexo deploy #1503](https://github.com/hexojs/hexo/issues/1503)