---
layout: post
title: SSH 到 Git 时 Connection timed out
tags: [Git]
categories: 文章
---

* TOC
{:toc}

# 问题出现

git仓库是使用ssh链接clone下来的，在`push`和`pull`到github时突然失效了，显示`Connection timed out`，`ssh -T git@github.com`一样不行

# 解决方案

编辑文件`~/.ssh/config`，添加以下内容

```shell
Host github.com
HostName ssh.github.com
User git
Port 443
PreferredAuthentications publickey
IdentityFile ~/.ssh/id_rsa
```

再操作就可以了。