---
layout: post
title: Linux使用sudo命令时免去输入密码
tags: [linux]
categories: 文章
---

* TOC
{:toc}

在终端中输入命令：

```shell
sudo visudo
```

此时会打开一个文件，在文件末尾添加以下内容：

```
<username>     ALL=(ALL) NOPASSWD: ALL
```

其中`<username>`要替换为需要免密的用户名。