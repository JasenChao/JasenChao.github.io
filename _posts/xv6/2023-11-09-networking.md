---
layout: post
title: 【XV6】 networking
tags: [xv6, OS]
categories: 文章
---

* TOC
{:toc}

# E1000网络设备驱动

题目已经在`kernel/e1000.c`中给出了E1000的初始化函数和发送接收函数，要求完善发送和接收的功能。

其他相关的代码，上层的网络协议在`kernel/net.c`和`kernel/net.h`中。

PCI总线上搜索网卡的代码在`kernel/pci.c`中：

```c
// tell the e1000 to reveal its registers at
// physical address 0x40000000.
base[4+0] = e1000_regs;

e1000_init((uint32*)e1000_regs);
```

`e1000_init()`函数配置了E1000为DMA模式，预备了多个缓冲区组成了两个环，用来存储需要发送和接收的消息，相当于一个队列，两个环配置为大小为 `RX_RING_SIZE` 和 `TX_RING_SIZE`。

## 发送函数

当 `net.c` 中的网络堆栈需要发送数据包时，`e1000_transmit`函数使用要发送的数据包`mbuf`。发送完成的数据包需要确保mbuf最终被释放，同时在传输数据包完成后，mbufs的status标志位会变为E1000_TXD_STAT_DD。

```c
int
e1000_transmit(struct mbuf *m)
{
  //
  // Your code here.
  //
  // the mbuf contains an ethernet frame; program it into
  // the TX descriptor ring so that the e1000 sends it. Stash
  // a pointer so that it can be freed after sending.
  //
  acquire(&e1000_lock);
  int idx = regs[E1000_TDT];                              // 取E1000_TDT寄存器的值作为队列尾部
  if ((tx_ring[idx].status & E1000_TXD_STAT_DD) == 0) {   // 如果发送已完成，则退出
    release(&e1000_lock);
    return -1;
  }

  if (tx_mbufs[idx]) mbuffree(tx_mbufs[idx]);             // 如果mbuf非空，则释放掉

  tx_mbufs[idx] = m;                                      // mbuf指向需要发送的m
  tx_ring[idx].length = m->len;
  tx_ring[idx].addr = (uint64)m->head;
  tx_ring[idx].css = tx_ring[idx].cso = tx_ring[idx].special = 0;
  tx_ring[idx].cmd = E1000_TXD_CMD_RS | E1000_TXD_CMD_EOP;
  regs[E1000_TDT] = (idx + 1) % TX_RING_SIZE;             // 更新寄存器

  release(&e1000_lock);
  return 0;
}
```

## 接收函数

接收函数应该扫描队列，将新收到的mbuf传送到上层的网络协议中，也就是`net_rx()`，然后用空的mbuf代替。

```c
static void
e1000_recv(void)
{
  //
  // Your code here.
  //
  // Check for packets that have arrived from the e1000
  // Create and deliver an mbuf for each packet (using net_rx()).
  //
  int idx = (regs[E1000_RDT] + 1) % RX_RING_SIZE;
  while(rx_ring[idx].status & E1000_RXD_STAT_DD) {
    rx_mbufs[idx]->len = rx_ring[idx].length;
    net_rx(rx_mbufs[idx]);                            // mbuf送到上层协议

    // 清空mbuf
    rx_mbufs[idx] = mbufalloc(0);
    rx_ring[idx].status = 0;
    rx_ring[idx].addr = (uint64)rx_mbufs[idx]->head;
    regs[E1000_RDT] = idx;
    idx = (idx + 1) % RX_RING_SIZE;
  }
}
```

# 测试结果

直观的测试可以用两个xv6目录的终端，分别运行`make server`和`make qemu`，在`make qemu`的xv6中运行`nettests`，然后可以在`make server`的终端里看到测试结果。

也可以使用`make grade`测试，结果如下：

```shell
== Test running nettests == 
$ make qemu-gdb
(3.7s) 
== Test   nettest: ping == 
  nettest: ping: OK 
== Test   nettest: single process == 
  nettest: single process: OK 
== Test   nettest: multi-process == 
  nettest: multi-process: OK 
== Test   nettest: DNS == 
  nettest: DNS: OK 
```
