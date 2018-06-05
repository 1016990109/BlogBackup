---
title: git-flow 学习
date: 2018-06-05 09:35:26
tags:
    - git
categories:
    - 学习
---

这里只是针对 `Git` 中的 `git-flow` 做一次学习记录，更加详细系统地学习 `Git` 请移步[这里](https://www.git-tower.com/learn/git/ebook)。

首先，`git-flow` 并不会为 `Git` 扩展任何新的功能，它仅仅使用了脚本来捆绑了一系列 `Git` 命令来完成一些特定的工作流程。

其次，定义一个固定的工作流程会使得团队协作更加简单容易。无论是一个 “版本控制的新手” 还是 “Git 专家”，每一个人都知道如何来正确地完成某个任务。

<!-- more -->

本文使用的工具是比较常用的 [gitflow-avh](https://github.com/petervanderdoes/gitflow/)。

## 分支的模式

`git-flow` 模式会预设两个主分支在仓库中：

* `master`:正式发布的产品代码
* `develop`:开发用分支

## 开发流程

看下图：

![git-flow](/assets/img/git-flow.png)

1. 先开发功能，可能是一个也可能是多个，功能分支为 `feature`，功能开发完并合并后会删除。
2. 功能开发完后都合并到 `develop` 分支进行汇总。
3. 所有功能开发完后，需要发布一个版本。开出一个 `release` 分支，进行分支的最后修改，如代码中某些版本号等等。然后将 `release` 分支同时合并到 `master` 和 `develop` 分支，并打上相应的 `tag`，删除该 `release` 分支，完成一次迭代。
4. 代码运行在 `master` 上一段时间后可能会有 `bug`，这时候开出 `hotfix` 分支对 `bug` 进行修复，修复完成后将代码合并到 `master` 和 `develop` 分支，打上修复的 `tag`，删除 `hotfix` 分支，一次在已发布版本上的修复就完成了。

> 注意：操作完后记得 `push` 哦！(`tag` 通过 `push` 是不会推送到远端仓库的，需要 `git push orign --tags` 推送所有 `tag`。)

## 功能开发

让我们开始开发一个新功能 “rss-feed”：

```zsh
$ git flow feature start rss-feed
Switched to a new branch 'feature/rss-feed'

Summary of actions:
- A new branch 'feature/rss-feed' was created, based on 'develop'
- You are now on branch 'feature/rss-feed'
```

经过一段时间艰苦地工作和一系列的聪明提交，我们的新功能终于完成了：

```zsh
$ git flow feature finish rss-feed
Switched to branch 'develop'
Updating 6bcf266..41748ad
Fast-forward
    feed.xml | 0
    1 file changed, 0 insertions(+), 0 deletions(-)
    create mode 100644 feed.xml
Deleted branch feature/rss-feed (was 41748ad).
```

## 管理 Releases

当你认为现在在 “develop” 分支的代码已经是一个成熟的 `release` 版本时，这意味着：第一，它包括所有新的功能和必要的修复；第二，它已经被彻底的测试过了。如果上述两点都满足，那就是时候开始生成一个新的 `release` 了：

```zsh
$ git flow release start 1.1.5
Switched to a new branch 'release/1.1.5'
```

进行最后的编辑，然后完成：

```zsh
git flow release finish 1.1.5
```

## hotfix

很多时候，仅仅在几个小时或几天之后，当对 `release` 版本作做全面测试时，可能就会发现一些小错误。
在这种情况下，`git-flow` 提供一个特定的 “hotfix” 工作流程（因为在这里不管使用 “功能” 分支流程，还是 “release” 分支流程都是不恰当的）。

创建 `hotfix`：

```zsh
$ git flow hotfix start missing-link
```

修复完 `bug` 后就该完成了：

```zsh
$ git flow hotfix finish missing-link
```

## 总结

`git-flow` 只是捆绑了一些命令来帮助用户来走这么一套通用的流程，当你能正确地理解工作流程的基本组成部分和目标的之后也可以不再使用这些工具了，可以根据自己的需要自定义流程。
