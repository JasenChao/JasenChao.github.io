---
layout: post
title: 【XV6】 Multithreading
tags: [xv6, OS]
categories: 文章
---

* TOC
{:toc}

代码：https://github.com/JasenChao/xv6-labs.git

# 用户级线程切换

题目要求完成用户级线程系统，提示程序要在`uthread.c`和`uthread_switch.S`中补充完成。

用户级线程调度和进程的机制是类似的，因此`uthread_switch.S`可以复制`swtch.S`中的内容：

```
	.globl thread_switch
thread_switch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
	/* YOUR CODE HERE */
	ret    /* return to ra */
```

在`uthread.c`中也需要保存上下文的结构体，和`proc.h`中类似：

```c
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

在`thread`结构体中加入`context`：

```c
struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context context;
};
```

同时在`thread_schedule`和`thread_create`中有注释提示，在`YOUR CODE HERE`处增加代码，`thread_schedule`需要调用上下文切换函数`thread_switch`：

```c
thread_switch((uint64)&t->context, (uint64)&next_thread->context);
```

`thread_create`函数需要设置上下文的栈指针和函数入口，其中栈指针应当指向栈顶（高地址）：

```c
t->context.sp = (uint64)t->stack + STACK_SIZE;
t->context.ra = (uint64)func;
```

使用`make GRADEFLAGS=uthread grade`测试结果是否正确。

# 使用线程

注意题目要求使用的线程是在`Unix`而非`vx6`中的，需要安装好`gcc`。测试程序在`notxv6/ph.c`中，多线程去访问哈希表，可能会在插入过程中引起资源争夺从而导致有的插入其实是失败的（但不会报错），因此只需要按照提示在`put`函数的插入过程加锁：

```c
pthread_mutex_t lock;

static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  pthread_mutex_lock(&lock);
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&lock);
}
```

题目提示不要忘记init锁，在`main`函数开头加上：

```c
pthread_mutex_init(&lock, NULL);
```

使用`make GRADEFLAGS=ph_fast grade`测试结果是否正确。

# Barrier

这道题目仍然是在`Unix`中的，在`notxv6/barrier.c`中实现`barrier`函数，思路就是先上锁，然后判断到达barrier的线程数是否足够，不够则到达的线程休眠，够则最后到达的线程处理圈数和线程数，然后唤醒所有线程：

```c
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
  if (bstate.nthread != nthread) pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  else {
    bstate.round++;
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

使用`make GRADEFLAGS=barrier grade`测试结果是否正确。

# 测试结果

使用`make grade`测试，结果如下：

```shell
== Test uthread == 
$ make qemu-gdb
uthread: OK (1.8s) 
== Test answers-thread.txt == 
answers-thread.txt: FAIL 
    Cannot read answers-thread.txt
== Test ph_safe == make[1]: Entering directory '/home/why/Workspace/xv6-labs-2023'
gcc -o ph -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/ph.c -pthread
make[1]: Leaving directory '/home/why/Workspace/xv6-labs-2023'
ph_safe: OK (7.4s) 
== Test ph_fast == make[1]: Entering directory '/home/why/Workspace/xv6-labs-2023'
make[1]: 'ph' is up to date.
make[1]: Leaving directory '/home/why/Workspace/xv6-labs-2023'
ph_fast: OK (17.2s) 
== Test barrier == make[1]: Entering directory '/home/why/Workspace/xv6-labs-2023'
gcc -o barrier -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/barrier.c -pthread
make[1]: Leaving directory '/home/why/Workspace/xv6-labs-2023'
barrier: OK (2.4s)
```
