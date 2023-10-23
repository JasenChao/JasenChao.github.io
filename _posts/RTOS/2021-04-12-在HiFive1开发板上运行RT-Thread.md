---
layout: post
title: 在HiFive1开发板上运行RT-Thread
tags: [RT-Thread, HiFive]
categories: 文章
---

* TOC
{:toc}

开发板：HiFive1

芯片：Freedom E310

# 编译

开发环境：Ubuntu18.04（Windows同理）

编译工具链：riscv64-unknown-elf-toolchain

编译工具链在SiFive官网上可以下载：https://www.sifive.com/software

下载后解压至/opt/目录下。

终端打开rt-thread/bsp/hifive1目录，修改rtconfig.py文件中的`EXEC_PATH`指定编译工具链，修改为解压的`bin`文件夹路径。

使用scons编译，生成bin文件。

# 烧录

烧录需要使用JLink工具，安装JLink步骤不再赘述。

```shell
cd /opt/SEGGER/JLink
sudo ./JLinkExe
```

打开JLink，依次执行：

```shell
connect（connect的所有选项直接回车）
r
h
erase
loadbin rtthread.bin 0x20000000
loadbin rtthread.bin 0x20400000
setpc 0x20400000
go
q
```

如果loadbin和setpc都成功，那么烧录完成，串口工具打开/dev/ttyACM0（也有可能是ACM1，这个板子插上会出现两个串口），即可出现终端。

# 复位

这块开发板复位后pc指针会回到0x20000000，需要再进入JLink，执行：

```shell
connect（还是回车处理）
r
h
setpc 0x20400000
go
q
```

串口即可打印输出。

# 注意

这块板子比较奇怪，必须从0x20400000开始烧录，在此之前还必须从0x20000000开始烧录一次，而且每次复位都得设置pc指针，属实吐了。