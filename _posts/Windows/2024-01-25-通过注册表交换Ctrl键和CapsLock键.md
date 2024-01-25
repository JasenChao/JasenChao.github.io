---
layout: post
title: 通过注册表交换Ctrl键和CapsLock键
tags: [Windows]
categories: 文章
---

* TOC
{:toc}

频繁地使用左下角的Ctrl键对我的小拇指产生了非常大的负担，想把它和不常用但很容易按的CapsLock键交换。

1. 先打开注册表，导航到`HKEY_LOCAL_MACHINE -> System -> CurrentControlSet -> Control -> KeyBoard Layout`。
2. 右键`新建 -> 二进制值`，命名为`Scancode Map`。
3. 右键新建的`Scancode Map`，选择`修改`，如果交换左Ctrl键，输入

    ```shell
    00 00 00 00 00 00 00 00
    03 00 00 00 1D 00 3A 00
    3A 00 1D 00 00 00 00 00
    ```

    如果交换右Ctrl键，输入

    ```shell
    00 00 00 00 00 00 00 00
    03 00 00 00 3A 00 1D E0
    1D E0 3A 00 00 00 00 00
    ```

4. 重启或者注销重新登陆生效。