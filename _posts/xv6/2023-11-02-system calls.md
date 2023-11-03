---
layout: post
title: 【XV6】 system calls
tags: [xv6, OS]
categories: 文章
---

* TOC
{:toc}

# 使用GDB调试

## 安装risc-v的GDB

先安装依赖：

```shell
sudo apt-get install libncurses5-dev python2 python2-dev texinfo libreadline-dev
```

再下载源码，可以从清华镜像源下载：

```shell
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/gdb/gdb-13.2.tar.xz
```

解压缩并编译安装：

```shell
tar xvf gdb-13.2.tar.xz
cd gdb-13.2/
mkdir build
cd build
../configure --prefix=/usr/local --target=riscv64-unknown-elf --enable-tui=yes
make -j$(nproc)
sudo make install
```

## 使用GDB

先用qemu运行xv6，在xv6目录下执行：

```shell
make qemu-gdb
```

此时应该打印一个端口号，例如`tcp::26000`，记住这个端口号，另起一个终端，运行gdb：

```shell
riscv64-unknown-elf-gdb
```

在GDB中连接这个端口，由于本次实验想在每次syscall中断时断点，所以`file kernel/kernel`：

```
target remote localhost:26000
file kernel/kernel
b syscall
```

用`c`让程序从断点处继续执行，输出如下：

```
(gdb) b syscall
Breakpoint 1 at 0x80002142: file kernel/syscall.c, line 243.
(gdb) c
Continuing.
[Switching to Thread 1.2]

Thread 2 hit Breakpoint 1, syscall () at kernel/syscall.c:243
243     {
(gdb) layout src
(gdb) backtrace
```

layout 命令将窗口一分为二，显示`src`，也可以`layout asm`等。

# 添加系统调用-trace

题目要求基于已经给出的`trace.c`实现系统调用跟踪。根据提示按步骤进行。

- 将`$U/_trace\`添加到`Makefile`的`UPROGS`中
- 在`user/user.h`中添加系统调用`int trace(int)`
- 在`user/usys.pl`中添加stub `entry("trace")`
- 在`kernel/syscall.h`中添加系统调用编号`#define SYS_trace  22`
- 在`kernel/sysproc.c`中添加函数`sys_trace()`，代码如下：
    ```c
    uint64
    sys_trace(void)
    {
        int n;
        argint(0, &n);
        myproc()->trace_mask = n;
        return 0;
    }
    ```
- 在`kernel/syscall.c`中添加命令名称数组，便于显示系统调用名，修改`syscall`函数，代码如下：
    ```c
    static char syscall_name[23][16] = {"fork", "exit", "wait", "pipe", "read", "kill", "exec", "fstat", "chdir", "dup", "getpid", "sbrk", "sleep", "uptime", "open", "write", "mknod", "unlink", "link", "mkdir", "close", "trace", "sysinfo"};

    void
    syscall(void)
    {
    int num;
    struct proc *p = myproc();

    num = p->trapframe->a7;
    if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        // Use num to lookup the system call function for num, call it,
        // and store its return value in p->trapframe->a0
        p->trapframe->a0 = syscalls[num]();
        if(p->trace_mask > 0 && (p->trace_mask & (1 << num)))
        {
        printf("%d: syscall %s -> %d\n", p->pid, syscall_name[num - 1], p->trapframe->a0);
        }
    } else {
        printf("%d %s: unknown sys call %d\n",
                p->pid, p->name, num);
        p->trapframe->a0 = -1;
    }
    }
    ```
- 在`kernel/syscall.c`中还需要增加系统调用的函数指针：
    ```c
    extern uint64 sys_trace(void);

    static uint64 (*syscalls[])(void) = {
    [SYS_fork]    sys_fork,
    [SYS_exit]    sys_exit,
    [SYS_wait]    sys_wait,
    [SYS_pipe]    sys_pipe,
    [SYS_read]    sys_read,
    [SYS_kill]    sys_kill,
    [SYS_exec]    sys_exec,
    [SYS_fstat]   sys_fstat,
    [SYS_chdir]   sys_chdir,
    [SYS_dup]     sys_dup,
    [SYS_getpid]  sys_getpid,
    [SYS_sbrk]    sys_sbrk,
    [SYS_sleep]   sys_sleep,
    [SYS_uptime]  sys_uptime,
    [SYS_open]    sys_open,
    [SYS_write]   sys_write,
    [SYS_mknod]   sys_mknod,
    [SYS_unlink]  sys_unlink,
    [SYS_link]    sys_link,
    [SYS_mkdir]   sys_mkdir,
    [SYS_close]   sys_close,
    [SYS_trace]   sys_trace,
    };
    ```
- 修改`kernel/proc.h`中的`struct proc`结构体，增加`int trace_mask`
- 修改`kernel/proc.c`中的`fork`函数，使得子进程可以继承父进程的`trace_mask`，增加的代码如下：
    ```c
    acquire(&np->lock);
    np->trace_mask = p->trace_mask;
    release(&np->lock);
    ```

Makefile调用perl脚本`user/usys.pl`，该脚本生成`user/usys.S`，这是实际的系统调用，它使用RISC-V ecall指令转换到内核。

使用`make GRADEFLAGS=trace grade`测试代码是否通过。

# 添加系统调用-sysinfo

题目要求基于已经给出的`sysinfotest.c`实现系统调用跟踪。根据提示按步骤进行。

- 将`$U/_sysinfotest\`添加到`Makefile`的`UPROGS`中
- 在`user/user.h`中添加系统调用：
    ```c
    struct sysinfo;
    int sysinfo(struct sysinfo*);
    ```
- 在`user/usys.pl`中添加stub `entry("sysinfo");`
- 在`kernel/syscall.h`中添加系统调用编号`#define SYS_sysinfo 23`
- 在`kernel/sysproc.c`中添加函数`sys_sysinfo()`，代码如下，`free_memory()`和`proc_num()`函数待实现：
    ```c
    #include "sysinfo.h"

    uint64
    sys_sysinfo(void)
    {
        struct sysinfo info;
        uint64 addr;

        argaddr(0, &addr);
        struct proc* p = myproc();
        info.freemem = free_memory();
        info.nproc = proc_num();
        // 将内核态中的info复制到用户态
        if (copyout(p->pagetable, addr, (char*)&info, sizeof(info)) < 0)
            return -1;
        return 0;
    }
    ```
- 在`kernel/syscall.c`中增加系统调用的函数指针：
    ```c
    extern uint64 sys_sysinfo(void);

    static uint64 (*syscalls[])(void) = {
    [SYS_fork]    sys_fork,
    [SYS_exit]    sys_exit,
    [SYS_wait]    sys_wait,
    [SYS_pipe]    sys_pipe,
    [SYS_read]    sys_read,
    [SYS_kill]    sys_kill,
    [SYS_exec]    sys_exec,
    [SYS_fstat]   sys_fstat,
    [SYS_chdir]   sys_chdir,
    [SYS_dup]     sys_dup,
    [SYS_getpid]  sys_getpid,
    [SYS_sbrk]    sys_sbrk,
    [SYS_sleep]   sys_sleep,
    [SYS_uptime]  sys_uptime,
    [SYS_open]    sys_open,
    [SYS_write]   sys_write,
    [SYS_mknod]   sys_mknod,
    [SYS_unlink]  sys_unlink,
    [SYS_link]    sys_link,
    [SYS_mkdir]   sys_mkdir,
    [SYS_close]   sys_close,
    [SYS_trace]   sys_trace,
    [SYS_sysinfo] sys_sysinfo,
    };
    ```
- 在`kernel/proc.c`中增加`proc_num()`函数，统计进程数，代码如下：
    ```c
    int
    proc_num(void)
    {
        int i;
        int n = 0;
        for (i = 0; i < NPROC; i++)
        {
            if (proc[i].state != UNUSED) n++;
        }
        return n;
    }
    ```
- 在`kernel/kalloc.c`中增加`free_memory()`函数，统计可用内存，代码如下：
    ```c
    uint64 
    free_memory(void)
    {
        struct run* p = kmem.freelist;
        uint64 num = 0;
        while (p)
        {
            num ++;
            p = p->next;
        }
        return num * PGSIZE;
    }
    ```
- 在`kernel/defs.h`中对应的区域增加函数声明，代码如下：
    ```c
    uint64          free_memory(void);
    int             proc_num(void);
    ```

使用`make GRADEFLAGS=sysinfo grade`测试代码是否通过。

题目提示`copyout()`函数的使用可以参考`filestat()`函数，这个函数的功能是从内核复制到用户，传入参数依次为当前进程的页表、地址、需要复制内容的起始地址、需要复制的长度。

# 测试结果

使用`make grade`测试，结果如下：

```shell
== Test trace 32 grep ==
$ make qemu-gdb
trace 32 grep: OK (1.5s)
== Test trace all grep ==
$ make qemu-gdb
trace all grep: OK (1.0s)
== Test trace nothing ==
$ make qemu-gdb
trace nothing: OK (0.9s)
== Test trace children ==
$ make qemu-gdb
trace children: OK (8.3s)
== Test sysinfotest ==
$ make qemu-gdb
sysinfotest: OK (1.1s)
```