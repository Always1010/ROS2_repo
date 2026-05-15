# ROS 2 C++ 快速上手清单

## 一、主线学习路径
# 建议掌握顺序

如果目标是快速上手 ROS 2 C++，可以按下面顺序学习：

1. `ros2 CLI`
2. `colcon`
3. `ros2 run`
4. `ros2 launch`
5. `ros2 topic / node / service / param`
6. `ros2 interface`
7. `CMakeLists.txt / package.xml`
8. `rclcpp node`
9. `publisher / subscriber`
10. `parameter`
11. `launch + YAML`
12. `service / client`
13. `custom msg / srv`
14. `rqt_graph`
15. `ros2 bag`
16. `TF2`
17. `RViz`
18. `URDF / Xacro`
19. `robot_state_publisher`
20. `bringup package`
21. `差速小车 mini project`

按这个顺序推进，目标是快速具备 ROS 2 C++ 项目开发能力。

1. **Workspace + colcon**
   - 创建 ROS 2 工作空间
   - 熟悉 `src/`、`build/`、`install/`、`log/`
   - 掌握 `colcon build`

2. **ament_cmake Package**
   - 创建 C++ package
   - 理解 `ament_cmake` 构建方式

3. **rclcpp Node**
   - 编写第一个 C++ 节点
   - 理解 `rclcpp::Node`、`spin`、logger

4. **Publisher / Subscriber**
   - 编写发布者和订阅者
   - 熟悉 topic 通信模型

5. **CMakeLists.txt / package.xml**
   - 掌握依赖声明
   - 掌握 executable 安装规则
   - 解决 `ros2 run` 找不到节点的问题

6. **CLI 调试**
   - 熟悉 `ros2 node`
   - 熟悉 `ros2 topic`
   - 熟悉 `ros2 service`
   - 熟悉 `ros2 param`
   - 熟悉 `ros2 interface`

7. **Parameter**
   - 声明和读取参数
   - 命令行修改参数
   - 为后续 YAML 配置做准备

8. **Launch + YAML**
   - 使用 launch 启动多个节点
   - 使用 YAML 管理参数
   - 减少手动启动和硬编码配置

9. **Service / Client**
   - 编写 service server
   - 编写 service client
   - 理解 request / response 模式

10. **Custom msg / srv**
    - 创建自定义接口包
    - 定义 `.msg` 和 `.srv`
    - 在 C++ 节点中使用自定义接口

11. **cmd_vel_filter_node**
    - 订阅 `/cmd_vel_raw`
    - 发布 `/cmd_vel`
    - 实现速度限幅、超时停车等逻辑

12. **QoS**
    - 理解 depth
    - 理解 reliable / best_effort
    - 理解 transient_local
    - 能排查 QoS 不匹配问题

13. **rosbag**
    - 录制 topic 数据
    - 回放数据
    - 用于复现和离线调试问题

14. **TF2**
    - 理解坐标系关系
    - 熟悉 `map`、`odom`、`base_link`
    - 使用 `tf2_echo` 和 `view_frames`

15. **odom_node**
    - 根据轮速计算里程计
    - 发布 `/odom`
    - 广播 `odom -> base_link`

16. **RViz**
    - 可视化 TF
    - 可视化 Odometry
    - 可视化 RobotModel
    - 保存 RViz 配置

17. **URDF / Xacro**
    - 描述机器人模型
    - 定义 link 和 joint
    - 使用 Xacro 简化模型描述

18. **bringup Package**
    - 集中管理 launch、config、rviz
    - 一条命令启动完整系统

19. **完成差速小车 Mini Project**
    - 整合接口、控制、里程计、TF、RViz、launch
    - 形成一个最小可运行的 ROS 2 C++ 项目


# ROS 2 常用工具优先级

## 第一优先级：必须先熟

这些工具是 ROS 2 开发的基础，几乎每天都会用。

1. **ros2 CLI**
   - ROS 2 命令行总入口

2. **colcon**
   - 工作空间构建工具

3. **ros2 run**
   - 运行单个节点

4. **ros2 launch**
   - 启动多个节点或完整系统

5. **ros2 topic / node / service / param**
   - 查看和调试节点、话题、服务、参数

6. **ros2 interface**
   - 查看 msg、srv、action 的接口定义

---

## 第二优先级：项目必备

这些工具用于可视化、调试、数据记录和机器人坐标系统。

7. **RViz**
   - 机器人三维可视化工具

8. **rqt_graph**
   - 查看节点和 topic 的连接关系

9. **ros2 bag**
   - 录制和回放 topic 数据

10. **TF2 工具**
    - `tf2_echo`
    - `view_frames`

11. **YAML 参数**
    - 管理节点配置
    - 避免硬编码参数

12. **robot_state_publisher**
    - 根据 URDF 发布机器人 TF
    - 支持 RViz 显示 RobotModel

---

## 第三优先级：机器人系统

这些工具用于构建更完整的机器人模型、仿真和工程环境。

13. **URDF / Xacro**
    - 描述机器人结构和关节关系

14. **joint_state_publisher**
    - 发布 `/joint_states`
    - 辅助测试机器人模型

15. **Gazebo**
    - 机器人仿真环境

16. **rosdep**
    - 安装 ROS 2 package 的系统依赖

17. **vcs**
    - 管理多仓库 workspace

---

## 第四优先级：方向专项

根据后续方向选择学习，不需要一开始全部掌握。

### 移动机器人方向

- **Nav2**
- **SLAM Toolbox**
- **map_server**
- **AMCL**
- **costmap**

### 机械臂方向

- **MoveIt 2**
- **ros2_control**
- **joint_trajectory_controller**