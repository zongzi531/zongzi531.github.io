---
title: patch-package 遭私有源背刺
date: 2022-09-02 15:07:55
categories: "JavaScript"
comments: true
featured_image: pic1.png
tags:
- patch-package
- npm
- registry
---

<!-- no node -->

<!-- more -->

众所周知， `patch-package` 是一款很不错的打补丁工具包，虽然 `pnpm` 没法使用。

由于目前工作环境受限于私有源限制，想给某个包打补丁。

但是问题就是，因为私有源同步 npm 落后了，导致执行 `npx patch-package pkg_example` 失败。

报错提示：

```shell
Error Couldn't find any versions for "..." that matches "^0.0.0"
...
```

而且看报错提示就是 npm 的执行过程的提示。

当然了，最简单的办法当然是私有源及时同步 npm 最新的依赖包版本。

但是，明明我打补丁，我并不关心依赖的关系，我只是想单纯的打补丁，能不能从本地打呢？

答案是：**能！**

我们来看到源码：

```ts
// https://github.com/ds300/patch-package/blob/master/src/makePatch.ts
// 删除部分代码
import { dirSync } from "tmp"

export function makePatch() {
  // 创建临时文件夹
  const tmpRepo = dirSync({ unsafeCleanup: true })
  const tmpRepoPackagePath = join(tmpRepo.name, packageDetails.path)
  const tmpRepoNpmRoot = tmpRepoPackagePath.slice(
    0,
    -`/node_modules/${packageDetails.name}`.length,
  )

  const tmpRepoPackageJsonPath = join(tmpRepoNpmRoot, "package.json")

    if (packageManager === "yarn") {
      // 尝试使用 yarn 安装依赖
    } else {
      // 尝试使用 npm 安装依赖
    }
```

那么我们就可以从这里展开操作，自己创建一个临时文件夹来代替 `tmp` 创建的临时文件夹路径，以此为出发点，在临时文件夹内手动安装依赖，并将安装动作取消。

```ts
const tmpRepo = dirSync({ unsafeCleanup: true })
tmpRepo.name = // 你需要的本地文件夹路径
```

在完成这些操作后，我们就可以使用本地资源来打补丁了。

:tada::tada::tada:
