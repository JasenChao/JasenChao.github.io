---
layout: post
title: ROS中创建python包供脚本文件调用
tags: [ROS, python]
categories: 文章
---

* TOC
{:toc}

# 问题描述

在某个ros package中，我在scripts目录下创建了3个py文件，其中一个是执行创建ros node的`a.py`，其他两个是封装了类的`b.py`和`c.py`，我在a.py中写了：
```python
import b
import c
```
然后将3个文件都放进了CMakeLists中：
```cmake
catkin_install_python(PROGRAMS
  scripts/a.py
  scripts/b.py
  scripts/c.py
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
```
然后在rosrun运行a.py时遇到了import失败的问题。

# 原因排查

在工作空间的`devel/lib`目录下，我找到了安装后的python脚本，发现ROS是通过`exec`调用源文件执行脚本的，所以b.py和c.py也是通过exec调用，因而a.py找不到import之后的内容，所以这样的作法肯定是不对的。

# 解决方法

1. 在CMakeLists中只需要安装要执行的脚本文件，其他文件封装为python包安装，需要取消`catkin_python_setup()`的注释：
   ```cmake
   catkin_python_setup()

   catkin_install_python(PROGRAMS
   scripts/a.py
   DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
   )
   ```
2. 在ros package的src目录下新建一个文件夹，命名为要封装的python包名，例如`my_package`，然后将b.py和c.py都移动到my_package目录下。
3. 在ros package目录下新建`setup.py`文件：
   ```python
   from distutils.core import setup
   from catkin_pkg.python_setup import generate_distutils_setup

   setup_args = generate_distutils_setup(
      packages=['my_package'],
      package_dir={'my_package': 'src/my_package'},
   )

   setup(**setup_args)
   ```
   packages里面放所有要封装的python包名，package_dir里面放包名和对应的目录。
4. 在a.py中正确import：
   ```python
   from my_package import b
   from my_package import c
   ```

此时再rosrun，就会发现import失败的问题解决了。