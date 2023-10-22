---
layout: post
title: ROS利用PX4和gazebo仿真offboard
tags: [ROS, PX4, gazebo]
categories: 文章
---

* TOC
{:toc}

1. 前提：安装好 ros 和 gazebo

2. 安装 PX4：

   ```shell
   git clone https://github.com/PX4/PX4-Autopilot.git
   ```

3. offboard 环境部署：

   ```shell
   # 建立工作空间
   mkdir -p catkin_ws/src
   cd catkin_ws
   catkin init
   wstool init src
   
   # 安装所需的 python 工具
   sudo apt-get install python-catkin-tools python-rosinstall-generator -y
   
   # 安装 mavlink
   rosinstall_generator --rosdistro kinetic mavlink | tee /tmp/mavros.rosinstall
   
   # 安装 mavros
   rosinstall_generator --upstream-development mavros | tee -a /tmp/mavros.rosinstall
   # 上面两步需要打开代理
   
   # 创建 deps
   wstool merge -t src /tmp/mavros.rosinstall	#这一步会报错 TypeError: load() missing 1 required positional argument: 'Loader'，需要修改 /usr/lib/python3/dist-packages/wstool/config_yaml.py 中 74 行为 yamldata = yaml.load(stream, Loader=yaml.FullLoader)
   wstool update -t src -j4	#这一步需要关闭代理
   rosdep install --from-paths src --ignore-src -y
   
   # 安装GeographicLib
   sudo ./src/mavros/mavros/scripts/install_geographiclib_datasets.sh
   
   # build
   catkin build
   
   # 更新环境变量
   source devel/setup.bash
   ```

4. 编写 offboard 文件：

   ```shell
   # 建立 offboard 包，依赖于 roscpp mavros geometry_msgs
   cd catkin_ws/src
   catkin_create_pkg offboard roscpp mavros geometry_msgs
   
   # 新建 cpp 文件，将官网提供的例程复制进去：https://docs.px4.io/main/zh/ros/mavros_offboard_cpp.html
   cd offboard/src
   gedit offb_node.cpp
   ```

   修改 CMakeList.txt 文件，需要修改 3 处：

   ```cmake
   ###################################
   ## catkin specific configuration ##
   ###################################
   ## The catkin_package macro generates cmake config files for your package
   ## Declare things to be passed to dependent projects
   ## INCLUDE_DIRS: uncomment this if your package contains header files
   ## LIBRARIES: libraries you create in this project that dependent projects also need
   ## CATKIN_DEPENDS: catkin_packages dependent projects also need
   ## DEPENDS: system dependencies of this project that dependent projects also need
   catkin_package(
    INCLUDE_DIRS include
   #  LIBRARIES offboard
    CATKIN_DEPENDS geometry_msgs mavros roscpp
    DEPENDS system_lib
   )
   
   ## Declare a C++ executable
   ## With catkin_make all packages are built within a single CMake context
   ## The recommended prefix ensures that target names across packages don't collide
   add_executable(offb_node src/offb_node.cpp)
   
   ## Specify libraries to link a library or executable target against
   target_link_libraries(offb_node
     ${catkin_LIBRARIES}
   )
   ```

5. 新建终端，回到`catkin_ws`工作区：

   ```shell
   cd catkin_ws
   catkin build
   source devel/setup.bash
   ```

6. 运行，记得每个终端都要`source devel/setup.bash`：

   ```shell
   # 打开一个新终端，启动 gazebo
   cd PX4-Autopilot
   make px4_sitl_default gazebo
   # 根据错误提示安装缺少的依赖，此处安装了 3 个
   pip3 install kconfiglib
   pip3 install --user jsonschema
   pip3 install future
   
   # 打开一个新终端，运行 mavros，连接到本地 ROS
   roslaunch mavros px4.launch fcu_url:="udp://:14540@127.0.0.1:14557"
   
   # 打开一个新终端，运行 offboard 实例
   rosrun offboard offb_node
   ```
   
7. 通过发布消息实现控制

   ```shell
   # body frame 下的控制，设置 velocity 和 yaw_rate 字段来分别控制无人机的线速度和角速度
   rostopic pub -r 20 /mavros/setpoint_raw/local mavros_msgs/PositionTarget "{
   header: { frame_id: 'base_footprint' },
   coordinate_frame: 1,
   type_mask: 4088,
   position: { x: 0.0, y: 10.0, z: 12.0 },
   velocity: { x: 0.0, y: 0.0, z: 0.0 },
   acceleration_or_force: { x: 0.0, y: 0.0, z: 0.0 },
   yaw: 0.0,
   yaw_rate: 0.0
   }"
   
   # 设置线速度和角速度控制
   rostopic pub -r 20 /mavros/setpoint_velocity/cmd_vel_unstamped geometry_msgs/Twist "linear:
     x: 1.0
     y: 0.0
     z: 0.0
   angular:
     x: 0.0
     y: 0.0
     z: 0.5"
   ```
   