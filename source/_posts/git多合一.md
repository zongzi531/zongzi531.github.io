---
title: git多合一
date: 2023-08-01 09:09:25
categories: "Git"
comments: true
tags:
- Git
- git
- rebase
- commit
- cherry-pick
---

<!-- no node -->

<!-- more -->

我们都会遇到分支 Feature 合并的协作场景吧，接下来我将总结一个。

假设我们在某个 Feature 分支提交了许多 commit 信息，我们需要将近期研发的 Feature 合并至主分支，或者将近期的某部分 Feature 合并至主分支。

操作这个的前提是，需要团队养成平时研发时保持良好的纯净的 commit 信息习惯，我们只做 git 操作。

首先我们需要操作 `git rebase -i` 指令，将需要合并的 Feature 合成一条 commit 记录。

在此之后，我们需要修改此条 commit 记录的时间，可以执行 `date -R` 取得当前时间，如 `Tue, 01 Aug 2023 09:15:01 +0800` 。

然后通过 `git commit --amend --date="Tue, 01 Aug 2023 09:15:01 +0800"` 命令修改提交时间。

记住此条 commit 的 hash 值后回到主分支操作 `git cherry-pick` 命令（如果有冲突请自行本地处理）。

> 如果额外产生了一条 merge 信息，请尝试使用 `git cherry-pick -m` 进行解决。

那么，现在就实现了分支 Feature 合并到主分支的纯净操作。

分支多条 commit 合一后，同步到主分支为一次 commit。
