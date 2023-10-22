---
layout: post
title: ROS利用PX4和gazebo进行多机仿真
tags: [ROS, PX4, gazebo]
categories: 文章
---

需要具备[单机仿真](https://jasenchao.github.io/2022-09-04-ROS利用PX4和gazebo仿真offboard/)的前置知识。

1. 修改 launch 文件：`PX4-Autopilot/launch/multi_uav_mavros_sitl.launch`，仿照已有内容，增加`uav3`，其中`fcu_url`、`mavlink_udp_port`和`mavlink_tcp_port`都要 +1 避免重复，上限是10台。（注意起始坐标）

2. 在单机 offboard 的环境下，src 中新建 `offb_node_4_circle.cpp`，每架无人机都要定义 NodeHandle 以及设置`Offboard enabled`和`Vehicle armed`。

3. 启动 gazebo：

   ```shell
   cd PX4-Autopilot
   git submodule update --init --recursive
   DONT_RUN=1 make px4_sitl_default gazebo
   roslaunch px4 multi_uav_mavros_sitl.launch
   ```

4. 运行 offboard 节点：

   ```shell
   rosrun offboard offb_node
   ```

5. 主程序为无人机画圆的过程，启动节点后无人机开始画圆

6. 如果无人机出现碰撞，则可以调整 launch 文件中的起始位置和 cpp 文件中的 offset。
