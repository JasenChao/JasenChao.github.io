---
layout: post
title: 【XV6】 page tables
tags: [xv6, OS]
categories: 文章
---

* TOC
{:toc}

# 快速获取pid-ugetpid

题目要求参考已实现的`ugetpid()`使用`USYSCALL`快速获取`pid`。

实现的思路是在每一个进程中增加一个共享页面，通过USYSCALL指定的虚拟地址，找到指定的页面。参考进程中的`Trampoline页`和`Trapframe页`。Trampoline页保存进出内核的代码，Trapframe页保存寄存器的数据。

在`proc.h`文件中的`struct proc`中增加成员`struct usyscall *sc`。

在`proc.c`文件中的`allocproc()`函数中，模仿`struct trapframe`，加入以下代码，申请内存并把`pid`保存下来：

```c
if((p->sc = (struct usyscall *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
}
p->sc->pid = p->pid;
```

对应地修改`freeproc`函数，释放内存空间：

```c
if(p->sc)
    kfree((void*)p->sc);
```

在`proc.c`文件中的`proc_pagetable`函数中，增加`USYSCALL`的`PTE`：

```c
if(mappages(pagetable, USYSCALL, PGSIZE,
            (uint64)(p->sc), PTE_R | PTE_U) < 0){
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
}
```

每个 PTE 都包含一些标志位，说明分页硬件对应的虚拟地址的使用权限：
- PTE_V：表示页表项的有效性（valid），即该页表项是否有效。
- PTE_R：表示可读权限（read），指示对应的页面是否可以被读取。
- PTE_W：表示可写权限（write），指示对应的页面是否可以被写入。
- PTE_X：表示可执行权限（execute），指示对应的页面是否可以被执行。
- PTE_U：表示用户访问权限（user），指示该页表项是否可以被用户级别的程序访问。

对应地修改`proc_freepagetable`函数，释放内存：

```c
uvmunmap(pagetable, USYSCALL, 1, 0);
```

# 打印页表

题目要求可视化RISC-V页表，在启动XV6的时候打印第一个进程的页表。

根据提示在`vm.c`中实现打印页表功能的函数，可以参考同文件中的`freewalk`函数，只遍历页表，不要将页表清空即可。代码如下：

```c
static void vmprint_func(pagetable_t pagetable, int level)
{
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V){
      uint64 pa = PTE2PA(pte);
      for(int j = 0; j < level; ++j) printf(".. ");
      printf("..%d: pte %p pa %p\n", i, pte, pa);
      if (level < 2) vmprint_func((pagetable_t)pa, level + 1);
    }
  }
}

void
vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  vmprint_func(pagetable, 0);
}
```

为了让`exec`函数可以调用以上函数，在`defs.h`中进行声明：

```c
void            vmprint(pagetable_t pagetable);
```

在`exec.c`文件中的`exec`函数返回前增加代码，判断如果`pid`为1则调用打印页表的功能：

```c
if(p->pid == 1){
    vmprint(p->pagetable);
}
```

使用`make GRADEFLAGS=printout grade`命令测试是否通过。

# 检测已访问的页面

题目要求实现一个系统调用，通过bit位检测页面是否被访问，测试程序`pgaccess_test`已经给出，用`unsigned int abits`的32位检测32个页面被访问的情况，置1即为被访问，下面对第1、2和30个页面的内容进行了写操作，正常应该返回`abits`为`40000006`。

先在`riscv.h`文件中增加`#define PTE_A (1L << 6)`，一开始我按顺序设的5，程序测试不通过，后来发现手册里面必须写6...

再完善`sysproc.c`文件中的`sys_pgaccess`函数，函数框架已经给出来了，代码如下：

```c
int
sys_pgaccess(void)
{
  struct proc *p = myproc();
  uint64 addr, mask, va;
  int n;
  unsigned int abits = 0;

  argaddr(0, &addr);
  argint(1, &n);

  va = addr;
  for(int i = 0; i < n; ++i){
    pte_t* pte = walk(p->pagetable, va, 0); // 获取PTE
    if(*pte & PTE_A){
      abits |= 1 << i;
      *pte ^= PTE_A;                        // 如果PTE_A已经置1，需要置为0，避免第二次访问时混淆
    }
    va += PGSIZE;
  }

  argaddr(2, &mask);
  if(copyout(p->pagetable, mask, (char*)&abits, sizeof(abits)) < 0) return -1;
  return 0;
}
```

使用`make GRADEFLAGS=pgaccess grade`命令测试结果是否正确。

# 测试结果

使用`make grade`测试，结果如下：

```shell
== Test   pgtbltest: ugetpid ==
  pgtbltest: ugetpid: OK
== Test   pgtbltest: pgaccess ==
  pgtbltest: pgaccess: OK
== Test pte printout ==
$ make qemu-gdb
pte printout: OK (0.9s)
```