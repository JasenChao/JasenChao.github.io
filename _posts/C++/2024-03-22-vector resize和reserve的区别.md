---
layout: post
title: vector resize和reserve的区别
tags: [C++]
categories: 文章
---

* TOC
{:toc}

在 C++ 的标准库中，resize() 和 reserve() 是用于操作 std::vector 容器的两个不同函数，它们的作用和效果有所区别。

# resize() 函数

resize() 函数用于改变 std::vector 容器的大小，即调整容器中元素的数量。

- 如果当前 vector 的大小小于指定的大小，resize() 会在容器末尾添加默认构造的元素，使得 vector 的大小达到指定大小。
- 如果当前 vector 的大小大于指定的大小，resize() 会删除多余的元素，使得 vector 的大小等于指定大小。

```cpp
std::vector<int> vec;
vec.resize(5); // 将 vec 的大小调整为 5，新增的元素值为默认值（int 类型默认为0）
```

# reserve() 函数

reserve() 函数用于为 std::vector 容器预留存储空间，但并不改变容器中元素的数量。

- 当使用 reserve() 后，vector 的容量会增加，但 vector 中元素的数量不变。这样可以避免因频繁添加元素而导致的重新分配内存和复制元素的开销。
- reserve() 只是改变了 vector 内部的容量，但不改变 vector 的大小。

```cpp
std::vector<int> vec;
vec.reserve(10); // 预留至少能容纳 10 个元素的存储空间，但 vector 的大小仍为 0
```

因此，resize() 和 reserve() 在功能上有明显区别：resize() 修改容器的大小并可能改变实际元素数量，而 reserve() 仅仅是为容器预留一定的存储空间，而不改变容器中元素的数量。根据具体需求，选择合适的函数来操作 std::vector 容器。