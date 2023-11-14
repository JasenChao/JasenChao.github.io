---
layout: post
title: 【XV6】 Xv6 and Unix utilities
tags: [xv6, OS]
categories: 文章
---

* TOC
{:toc}

代码：https://github.com/JasenChao/xv6-labs.git

# 运行xv6

实验环境使用的是Ubuntu 20.04，需要安装一些工具：

```shell
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

安装完成后验证`qemu`是否安装成功，成功的话应该打印`qemu`的版本：

```shell
qemu-system-riscv64 --version
```

同时还需要有`RISC-V`版本的`GCC`，以下三个至少有一个正常安装，可以打印版本：

```shell
riscv64-linux-gnu-gcc --version
riscv64-unknown-elf-gcc --version
riscv64-unknown-linux-gnu-gcc --version
```

# 下载xv6源码并编译运行

```shell
git clone git://g.csail.mit.edu/xv6-labs-2023
cd xv6-labs-2023
make qemu
```

一切正常的话会显示`xv6 kernel is booting`，以及一个可以输入命令的终端。

# 实现sleep命令

题目要求实现一个sleep命令，可以使系统睡眠指定的时间。

在`user`文件夹下新建`sleep.c`文件，要求包含判断参数是否正确：

```c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  if(argc != 2){
    fprintf(2, "usage: sleep times\n");
    exit(1);
  }

  sleep(atoi(argv[1]));

  exit(0);
}
```

修改`Makefile`中的以下内容，使得`sleep`出现在命令列表中：

```makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_sleep\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
```

使用`make GRADEFLAGS=sleep grade`命令测试是否正确。

# 实现pingpong命令

题目要求实现一个pingpong命令，父进程传给子进程一个字符，子进程读到这个字符就输出`<pid>: received ping`，再把这个字符传回给父进程，父进程读到这个字符就输出`<pid>: received pong`。

在`user`文件夹下新建`pingpong.c`文件：

```c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int p1[2], p2[2];

  pipe(p1);
  pipe(p2);

  if(fork() == 0){
    char byte;

    close(p1[1]);   // 子进程不需要p1的写端
    close(p2[0]);   // 子进程不需要p2的读端
    
    read(p1[0], &byte, sizeof(byte));
    printf("%d: received ping\n", getpid());

    write(p2[1], &byte, sizeof(byte));

    close(p1[0]);
    close(p2[1]);

    exit(0);
  }else{
    char byte = 's';

    close(p1[0]);   // 父进程不需要p1的读端
    close(p2[1]);   // 父进程不需要p2的写端
    
    write(p1[1], &byte, sizeof(byte));

    read(p2[0], &byte, sizeof(byte));
    printf("%d: received pong\n", getpid());

    close(p1[1]);
    close(p2[0]);

    exit(0);
  }
}
```

修改`Makefile`中的以下内容，使得`pingpong`出现在命令列表中：

```makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_pingpong\
	$U/_rm\
	$U/_sh\
	$U/_sleep\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
```

使用`make GRADEFLAGS=pingpong grade`命令测试是否正确。

# 实现primes命令

题目要求实现一个primes命令，筛选2到35之间的所有质数。

在`user`文件夹下新建`primes.c`文件，其实就是不断用pipe对范围内的数字进行筛选，正常可以用递归的思维不断fork，但是题目中提到xv6资源很少，也不知道是有多少，就用了3个子进程筛选，剩下的用暴力判断的方法筛选：

```c
#include "kernel/types.h"
#include "user/user.h"

int main() {
    int p1[2];
    pipe(p1);

    if (fork()) {
        // 父进程
        close(p1[0]);

        for (int i = 2; i < 36; ++i) {
            write(p1[1], &i, sizeof i);
        }

        close(p1[1]);
    } else {
        // 子进程
        close(p1[1]);

        int base;
        if (read(p1[0], &base, sizeof base) == 0) exit(1);
        printf("prime %d\n", base);

        int p2[2];
        pipe(p2);

        if (fork()) {
            // 还是子进程
            close(p2[0]);
            
            int n;
            while (read(p1[0], &n, sizeof n)) {
                if (n % base != 0) write(p2[1], &n, sizeof n);
            }

            close(p2[1]);
        } else {
            // 孙进程
            close(p2[1]);

            if (read(p2[0], &base, sizeof base) == 0) exit(1);
            printf("prime %d\n", base);

            int p3[2];
            pipe(p3);

            if (fork()) {
                // 还是孙进程
                close(p3[0]);

                int n;
                while (read(p2[0], &n, sizeof n)) {
                    if (n % base != 0) write(p3[1], &n, sizeof n);
                }

                close(p3[1]);
            } else {
                // 曾孙进程
                close(p3[1]);

                while (read(p3[0], &base, sizeof base)) {
                    int flag = 0;
                    for (int i = 2; i * i <= base; ++i) {
                        if (base % i == 0) {
                            flag = 1;
                            break;
                        }
                    }
                    if (flag == 0) printf("prime %d\n", base);
                }
            }
        }

        close(p1[0]);
    }

    wait(0);

    return 0;
}
```

修改`Makefile`中的以下内容，使得`primes`出现在命令列表中：

```makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_find\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_pingpong\
	$U/_primes\
	$U/_rm\
	$U/_sh\
	$U/_sleep\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
```

使用`make GRADEFLAGS=primes grade`命令测试是否正确。

# 实现xargs命令

题目要求实现一个xargs命令，效果相当于Unix中的`xargs -n 1`。

在`user`文件夹下新建`xargs.c`文件：

```c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
    char *cmd;                                                      // 记录xargs后面的命令
    char **new_argv;                                                // 记录xargs后面的命令需要的参数

    cmd = argv[1];                                                  // xargs后面的第一个参数就是命令
    new_argv = malloc(argc * sizeof(char *));                       // 申请argc个指针大小的内存
    for (int i = 0; i < argc - 1; ++i) new_argv[i] = argv[i + 1];   // 将除了xargs以外的内容都保存下来

    char buf[512];                                                  // 用来保存xargs前面的指令输出的内容
    char *pos = buf;                                                // 填充buf的下标指针
    while (read(0, pos, sizeof(char)) > 0) {                        // 每次读一个char，直到没有输出，也就是前面命令的输出内容全部读完了
        if (*pos == '\n' || *pos == '\0') {                         // 如果读出来是换行或者结束符，意味着一条参数读完了，将其统一修改为结束符
            *pos = '\0';
            if (fork() == 0) {                                      // 将最后一个参数补齐，由子进程来完成
                new_argv[argc - 1] = buf;
                exec(cmd, new_argv);
            } else {                                                // 父进程的buf清空，pos归位，准备再读下一条参数
                memset(buf, 0, 512);
                pos = buf;
            }
        } else pos++;                                               // 读到换行和结束符以外的内容就正常写入buf
    }
    wait(0);                                                        // 等待子进程全部结束
    free(new_argv);                                                 // 释放前面malloc的内存

    return 0;
}
```

修改`Makefile`中的以下内容，使得`xargs`出现在命令列表中：

```makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_find\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_pingpong\
	$U/_primes\
	$U/_rm\
	$U/_sh\
	$U/_sleep\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_xargs\
	$U/_zombie\
```

使用`make GRADEFLAGS=xargs grade`命令测试是否正确。

# 测试结果

使用`make grade`命令对所有内容进行测试，输出测试结果：

```shell
== Test sleep, no arguments ==
$ make qemu-gdb
sleep, no arguments: OK (1.9s)
== Test sleep, returns ==
$ make qemu-gdb
sleep, returns: OK (0.8s)
== Test sleep, makes syscall ==
$ make qemu-gdb
sleep, makes syscall: OK (1.0s)
== Test pingpong ==
$ make qemu-gdb
pingpong: OK (1.0s)
== Test primes ==
$ make qemu-gdb
primes: OK (1.1s)
== Test find, in current directory ==
$ make qemu-gdb
find, in current directory: OK (1.0s)
== Test find, recursive ==
$ make qemu-gdb
find, recursive: OK (1.1s)
== Test xargs ==
$ make qemu-gdb
xargs: OK (1.1s)
```