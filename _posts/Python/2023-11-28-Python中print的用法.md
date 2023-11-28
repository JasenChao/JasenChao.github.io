---
layout: post
title: Python 中 print 函数的用法
tags: [Python]
categories: 文章
---

* TOC
{:toc}

在 Python 中，可以使用`print`函数来打印一个变量或者一个字符串：

```python
print("My name is Alice")
print(i)
```

如果需要字符串格式化来打印一句话中包含变量的内容，有几种常用的方法：

1. 使用格式化字符串（f-string）：在字符串前面加上字母"f"，然后在字符串中使用大括号{}包裹变量名。示例代码如下：

    ```python
    name = "Alice"
    age = 25
    print(f"My name is {name} and I am {age} years old.")
    ```

2. 使用字符串的format()方法：在字符串中使用一对花括号{}作为占位符，并调用format()方法传入变量值。示例代码如下：

    ```python
    name = "Alice"
    age = 25
    print("My name is {} and I am {} years old.".format(name, age))
    ```

3. 使用百分号（%）进行格式化：在字符串中使用百分号作为占位符，并使用%运算符将变量与占位符关联起来。示例代码如下：

    ```python
    name = "Alice"
    age = 25
    print("My name is %s and I am %d years old." % (name, age))
    ```

输出结果均为：

My name is Alice and I am 25 years old.