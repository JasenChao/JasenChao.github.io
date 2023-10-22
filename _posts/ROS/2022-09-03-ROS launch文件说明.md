---
layout: post
title: ROS launch文件说明
tags: [ROS]
categories: 文章
---

* TOC
{:toc}

一次性启动多个ROS节点：

1. 功能包文件夹下面新建launch文件夹

2. launch文件夹下面新建xxx.launch文件

3. 编辑 launch 文件内容

   ```xml
   <launch>
       <node pkg="helloworld" type="demo_hello" name="hello" output="screen" />
       <node pkg="turtlesim" type="turtlesim_node" name="t1"/>
       <node pkg="turtlesim" type="turtle_teleop_key" name="key1" />
   </launch>
   ```

   - node ---> 包含的某个节点
   - pkg -----> 功能包
   - type ----> 被运行的节点文件
   - name --> 为节点命名
   - output-> 设置日志的输出目标

4. 执行

   ```shell
   source ./devel/setup.bash
   # roslaunch 包名 launch文件名
   roslaunch hello_node xxx.launch
   ```
