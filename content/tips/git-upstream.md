---
title: GitHub 上 Fork 的仓库如何与原仓库同步并提交内容
date: 2019-03-24
categories: ["其他"]
---

# GitHub 上 Fork 的仓库如何与原仓库同步并提交内容

我们可能有时会遇到需要向 GitHub 上的开源的项目提交代码。在 GitHub 中，由于你不是那个开源项目中有权限提交代码的人，你只能 Fork 后再发起 Pull Request。

## Fork 仓库

你可以在 GitHub 网页上点击 Fork，之后你可以在你的个人主页的 Repository 中找到你 Fork 的仓库。

![](/images/git-upstream/1.png)

然后你就可以将其 clone 到本地。

## Fork 的本地仓库与源仓库连接

参考：[Syncing a fork - User Documentation](https://help.github.com/articles/syncing-a-fork/)

首先在命令行工具中输入

```bash
$ git remote -v
```

此时应该显示如下：

```
origin  https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
origin  https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)
```

因为我们没有将其连接到源仓库，所以需要用命令去连接。

```bash
$ git remote add upstream https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git
```

这里的地址是**源仓库**的地址。接着我们再输入

```bash
$ git remote -v
```

这时应该能看到：

```
origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)
upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (fetch)
upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (push)
```

这样我们就完成了与与源仓库的连接。

## Fork 的仓库与源仓库同步代码

首先：

```
$ git fetch upstream
```

获取源仓库更新。

```
$ git checkout master
$ git merge upstream/master
```

这样就可以将本地 master 分支更新到最新。我们就可以在此基础上创建新分支提交 PR。
