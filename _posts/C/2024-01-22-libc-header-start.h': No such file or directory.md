---
layout: post
title: "bits/libc-header-start.h: No such file or directory"
tags: [C, gcc]
categories: 文章
---

* TOC
{:toc}

# 问题出现

在编译一个工程的时候，出现了报错

```shell
In file included from /usr/lib/gcc/x86_64-linux-gnu/9/include/stdint.h:9,
                 from main.c:1:                                                                                                      
/usr/include/stdint.h:26:10: fatal error: bits/libc-header-start.h: No such file or directory                           
   26 | #include <bits/libc-header-start.h>                                                                                          
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~
```

# 解决方法

发现这个工程是要分别编译32位和64位的，因此才会出现这个问题，安装`gcc-multilib`来解决。