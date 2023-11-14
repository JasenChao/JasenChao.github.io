---
layout: post
title: 【XV6】 file system
tags: [xv6, OS]
categories: 文章
---

* TOC
{:toc}

代码：https://github.com/JasenChao/xv6-labs.git

# 支持大文件

XV6目前只支持268个blocks大小的文件，一个block（BSIZE）为1024，文件块inode包含12个一级地址和1个二级地址，二级地址指向另一个block，其中存放了256个一级地址，因此一共是268个。

题目要求支持大文件（65803个blocks），提示通过三级地址来实现，将一级地址调整为11个，多出来的一个指向三级地址（三级地址中包含256个二级地址），最大就可以支持`256*256+256+11=65803`个blocks大小的文件。

首先在`kernel/fs.h`中把一级地址的数量从12改为11，`NINDIRECT`是256，那么新定义`NDINDIRECT`为256*256，是三级地址可以包含的blocks数量，从而`MAXFILE`应为`NDIRECT + NINDIRECT + NDINDIRECT`，结构体`inode`和`dinode`也需要修改使得包含三级地址：

```c
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define NDINDIRECT (NINDIRECT * NINDIRECT)
#define MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT)

// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+2];   // Data block addresses
};

// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+2];
};
```

修改`fs.c`中的`bmap`函数，函数中只包含了一级和二级地址的处理过程，仿照完成三级地址的部分，加在二级地址的代码后面：

```c
 bn -= NINDIRECT;

  if(bn < NDINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT + 1]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT + 1] = addr;
    }
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    uint c = bn / NINDIRECT;
    bn %= NINDIRECT;
    if((addr = a[c]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[c] = addr;
        log_write(bp);
      }
    }
    brelse(bp);

    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }
```

对应地，释放文件的`itrunc`函数也要增加对于三级地址的处理：

```c
 if(ip->addrs[NDIRECT + 1]){
    bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j]){
        struct buf *bp1 = bread(ip->dev, a[j]);
        uint *a1 = (uint*)bp1->data;
        for(i = 0; i < NINDIRECT; ++i)
          if(a1[i])
            bfree(ip->dev, a1[i]);
        brelse(bp1);
        bfree(ip->dev, a[j]);
        a[j] = 0;
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  }
```

通过`make GRADEFLAGS=bigfile grade`测试代码是否通过。

# 支持符号链接

题目要求实现一个系统调用`symlink`创建符号链接，新增系统调用之前做过很多次：

```c
# user/user.h
int symlink(const char*, const char*);

# user/usys.pl
entry("symlink");

# kernel/syscall.h
#define SYS_symlink 22

# kernel/syscall.c
extern uint64 sys_symlink(void);

static uint64 (*syscalls[])(void) = {
...
[SYS_symlink]   sys_symlink,
};
```

根据提示在`kernel/stat.h`中增加新的文件类型：

```c
#define T_SYMLINK 4
```

之前已经创建了系统调用`symlink`，但是还没有实现具体的函数，在`kernel/sysfile.c`中实现，先创建一个inode，然后将目标文件的路径写入其中第一块block，其中需要注意`creat`会对申请的inode上锁但不会解锁：

```c
uint64
sys_symlink(void)
{
  struct inode *ip;
  char target[MAXPATH], path[MAXPATH];

  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0) return -1;

  begin_op();

  if((ip = create(path, T_SYMLINK, 0, 0)) == 0){
    end_op();
    return -1;
  }

  if(writei(ip, 0, (uint64)target, 0, strlen(target)) < 0){
    iunlockput(ip);
    end_op();
    return -1;
  }

  iunlockput(ip);
  end_op();

  return 0;
}
```

根据提示在`kernel/fcntl.h`中增加新的flag，用于系统调用`open`，这里的标志位是按位运算的，不要与已经定义的标志重叠：

```c
#define O_NOFOLLOW 0x800
```

在`Makefile`中加入已经提供的测试程序：

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
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_symlinktest\
```

修改`kernel/sysfile.c`中的`sys_open`函数，增加对符号链接文件类型的处理，循环读取inode，直到找到目标文件或者达到深度为10为止：

```c
int depth = 0;
while(!(omode & O_NOFOLLOW) && ip->type == T_SYMLINK)
{
  if(readi(ip, 0, (uint64)path, 0, MAXPATH) == -1)
  {
    iunlockput(ip);
    end_op();
    return -1;
  }
  iunlockput(ip);
  if((ip = namei(path)) == 0)
  {
    end_op();
    return -1;
  }
  
  if(++depth == 10)
  {
    end_op();
    return -1;
  }
  ilock(ip);
}
```

通过`make GRADEFLAGS=symlinktest grade`测试代码是否通过。

# 测试结果

使用`make grade`测试，结果如下：

```shell
== Test running bigfile == 
$ make qemu-gdb
running bigfile: OK (53.6s) 
== Test running symlinktest == 
$ make qemu-gdb
(0.5s) 
== Test   symlinktest: symlinks == 
  symlinktest: symlinks: OK 
== Test   symlinktest: concurrent symlinks == 
  symlinktest: concurrent symlinks: OK 
== Test usertests == 
$ make qemu-gdb
usertests: OK (100.7s) 
```
