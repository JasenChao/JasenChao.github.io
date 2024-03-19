---
layout: post
title: Linux下生成核心转储core
tags: [Linux]
categories: 文章
---

* TOC
{:toc}

为了方便进行分析调试，希望当程序发生崩溃或者收到 SIGSEGV、SIGABRT 等信号时，系统会生成相应的核心转储文件。

# 核心转储大小限制

首先，要检查核心转储的大小限制。可以使用 ulimit 命令来查看当前用户的核心转储大小限制：

```bash
ulimit -c
```

如果输出为 0，则表示不生成核心转储文件。可以使用以下命令来设置核心转储大小限制为无限制：

```bash
ulimit -c unlimited
```

# 核心转储文件的位置和命名规则

核心转储文件默认会生成在当前进程的工作目录下，文件名通常为 core。如果需要更改核心转储文件的生成位置和命名规则，可以通过修改 /proc/sys/kernel/core_pattern 文件来实现。

```bash
echo "/tmp/core-%e-%s-%u-%g-%p-%t" > /proc/sys/kernel/core_pattern
```

上述命令将核心转储文件命名规则修改为 /tmp/core-可执行文件名-信号编号-用户ID-组ID-进程ID-时间戳。

# 永久生效

如果想要在系统重启后仍然保持核心转储设置，可以修改 /etc/sysctl.conf 文件，在文件末尾添加以下内容：

```bash
kernel.core_pattern = /tmp/core-%e-%s-%u-%g-%p-%t
```

# 立即生效

使用 sysctl 命令使设置立即生效：

```bash
sysctl -p
```