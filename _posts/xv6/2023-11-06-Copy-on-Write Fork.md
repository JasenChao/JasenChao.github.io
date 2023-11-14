---
layout: post
title: 【XV6】 Copy-on-Write Fork for xv6
tags: [xv6, OS]
categories: 文章
---

* TOC
{:toc}

代码：https://github.com/JasenChao/xv6-labs.git

# Copy-on-Write Fork

系统调用`fork()`会复制一个父进程的用户空间到子进程，一方面如果进程较大，复制需要很长的时间，另一方面复制的内存的大部分会被丢弃，造成浪费。

题目要求实现写时复制`COW`来延迟fork的物理内存复制，子进程只创建了一个页表，指向父进程的内存，此时内存无论对谁都标记为只读，当有进程要对复制来的内存进行写操作时，CPU强制页面错误，内核检测到故障之后再分配物理内存，在页面错误的进程中使用新分配的内存，此时PTE标记为可写。简而言之就是在需要写时再创建对应的副本。

在xv6中可以相对较为简单地通过标记页面的引用来完成这个功能，当引用减到1之后页面才可写，减到0之后才会释放。

题目在`user/cowtest.c`中提供了测试程序。

## 引用计数

在`kalloc.c`中，在结构体`kmem`中增加数组用作引用计数，同时加入锁，对多进程操作数组进行保护。

```c
struct {
  struct spinlock lock;
  struct run *freelist;
  struct spinlock reflock; // 页面引用计数锁
  // 直接使用数组保存引用计数，简单
  char ref_count[PHYSTOP/PGSIZE];
} kmem;
```

在`kinit`函数中对锁进行初始化：

```c
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&kmem.reflock, "kmemref");
  freerange(end, (void*)PHYSTOP);
}
```

`freerange`函数也需要对应地修改被释放的内存空间：

```c
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  int i;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(i=0; i<(uint64)p/PGSIZE; i++) {
    kmem.ref_count[i] = 0;
  }
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE) {
    kmem.ref_count[(uint64)p/PGSIZE] = 1;
    kfree(p);
  }
}
```

这里`kmem.ref_count[(uint64)p/PGSIZE] = 1`是因为紧接着调用的`kfree`函数也需要修改，会进行`-1`的操作：

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  char refcnt = ksubref((void*)pa);
  if (refcnt > 0)
    return;
  
  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```

`kalloc`函数也需要对应地将分配的页面的引用次数置为1：

```c
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r){
    kmem.freelist = r->next;
    acquire(&kmem.reflock);
    // 初始引用次数记为1
    kmem.ref_count[(uint64)r / PGSIZE] = 1;
    release(&kmem.reflock);
  }
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

在`kalloc.c`中封装几个引用计数所需要的函数：

```c
inline void
acquire_refcnt(){
    acquire(&kmem.reflock);
}

inline void
release_refcnt(){
    release(&kmem.reflock);
}

char
kgetref(void *pa){
  acquire_refcnt();
  char refcnt = kmem.ref_count[(uint64)pa/PGSIZE];
  release_refcnt();
  return refcnt;
}

void
kaddref(void *pa){
  acquire_refcnt();
  kmem.ref_count[(uint64)pa/PGSIZE]++;
  release_refcnt();
}

char
ksubref(void *pa){
  acquire_refcnt();
  char refcnt = --kmem.ref_count[(uint64)pa/PGSIZE];
  release_refcnt();
  return refcnt;
}
```

需要其他地方调用的函数需要在`defs.h`中声明：

```c
void            kaddref(void*);
char            ksubref(void*);
char            kgetref(void*);
```

## fork时不再分配内存

题目有提示，在PTE中标记COW映射，因此在`riscv.h`中增加：

```c
#define PTE_C (1L << 8) // cow page
```

修改`fork`函数中分配内存的代码，发现在`uvmcopy`函数中完成，在uvmcopy函数中删去分配内存相关的代码，修改为清除父进程内存的可写标记，设置COW标记，并将引用+1，子进程的页表指向父进程，PTE同样是不可写，修改后为：

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    
    // 清除父进程的PTE_W位，设置PTE_C位
    // 如果父进程是只读页，无需设置PTE_C位
    if(*pte & PTE_W) {
      *pte = (*pte & (~PTE_W)) | PTE_C;
    }
    pa = PTE2PA(*pte);

    kaddref((void *)pa); // 引用加1

    // 子进程不需要内存，指向pa，页表属性为flags
    flags = PTE_FLAGS(*pte);
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

此时再有进程对COW标记的页面进行写操作会引起页面错误，在 RISC-V 架构中，r_scause 寄存器用于保存导致异常或中断的原因。当 r_scause() == 15 时，表示发生了一个Environment Call异常，也就是 ecall 指令。因此在`usertrap`函数中增加对`r_scause() == 15`的错误处理：

```c
else if(r_scause() == 15) { // 写页面错
  uint64 va0 = r_stval();
  if(va0 > p->sz || cowhandler(p->pagetable,va0) !=0 || va0 < PGSIZE) {
    p->killed = 1;    
  }
}
```

其中`cowhandler`函数需要单独实现：

```c
int
cowhandler(pagetable_t pagetable, uint64 va)
{
    char *mem;
    if (va >= MAXVA)
      return -1;
    pte_t *pte = walk(pagetable, va, 0);
    if (pte == 0)
      return -1;
    // check the PTE
    if ((*pte & PTE_C) == 0 || (*pte & PTE_U) == 0 || (*pte & PTE_V) == 0) {
      return -1;
    }
    uint64 pa = PTE2PA(*pte);

    char refcnt = kgetref((void *)pa);

    if(refcnt == 1) {                         // 引用计数为1时，清除PTE_W，增加PTE_C
       *pte = (*pte & (~PTE_C)) | PTE_W;
       return 0;
    }
    if(refcnt > 1) {                          // 引用计数大于1时，申请内存，把原来页面的内容复制过来
      if ((mem = kalloc()) == 0) {
        return -1;
      }
      // copy old data to new mem
      memmove((char*)mem, (char*)pa, PGSIZE);
      kfree((void*)pa);                       // 创建好副本后释放原来的引用
      uint flags = PTE_FLAGS(*pte);
      *pte = (PA2PTE(mem) | flags | PTE_W);   // 设置副本页为可写
      *pte &= ~PTE_C;                         // 清除副本页的COW标记
      return 0;
    }
    return -1;
}
```

## 处理copyout

在`copyout`函数中也要增加页面错误处理，因为copyout是在内核中调用的，缺页不会进入usertrap。因此在copyout中判断缺页错误，如果是因为COW则分配新的页：

```c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  pte_t *pte;

  while(len > 0){
    // 查找虚拟地址dstva对应的物理地址pa0
    va0 = PGROUNDDOWN(dstva);
    // walkaddr调用了walk
    pa0 = walkaddr(pagetable, va0); 
    if(pa0 == 0) // 查找失败
      return -1;
    
    if(va0 >= MAXVA || va0 < PGSIZE)
      return -1;
    struct proc *p = myproc();
    if((pte = walk(pagetable, va0, 0))==0) {
      p->killed = 1;
      return -1;
    }
    // check
    if ((va0 < p->sz) && (*pte & PTE_V) &&
            (*pte & PTE_C)&&(*pte & PTE_U)) {

      char refcnt = kgetref((void *)pa0);

      if(refcnt == 1) {
         *pte = (*pte &(~PTE_C)) | PTE_W;
      }else if(refcnt > 1){
        char *mem;

        ksubref((void *)pa0);

        if ((mem = kalloc()) == 0) {
          p->killed = 1;        
          return -1;
        } 
        memmove(mem, (char*)pa0, PGSIZE);
        uint flags = PTE_FLAGS(*pte);
        *pte = (PA2PTE(mem) | flags | PTE_W);
        *pte &= ~PTE_C;
        pa0 = (uint64)mem;          
      }
    }
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

对copyout的修改需要增加两个头文件：

```c
#include "spinlock.h"
#include "proc.h"
```

# 测试结果

使用`make grade`测试，结果如下：

```shell
== Test running cowtest == 
$ make qemu-gdb
(4.4s) 
== Test   simple == 
  simple: OK 
== Test   three == 
  three: OK 
== Test   file == 
  file: OK 
== Test usertests == 
$ make qemu-gdb
(32.0s) 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyout == 
  usertests: copyout: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK
```
