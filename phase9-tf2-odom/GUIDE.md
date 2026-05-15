# Phase 9: TF2 + odom_node（坐标变换）

## 学习目标

- 理解 TF2 坐标系统的基本概念
- 编写里程计节点，发布 `/odom` 和广播 TF
- 掌握 `tf2_echo`、`view_frames` 等调试工具
- 理解 ROS2 中机器人常用的坐标系

## 背景知识

### 什么是 TF2？

TF2 (Transform Library 2) 是 ROS2 中管理**坐标系变换**的系统。它追踪各个坐标系（frame）之间的位置和姿态关系。

### 差速小车常用坐标系

```
map          ← 世界固定坐标系（全局地图）
  │
  └─ odom    ← 里程计坐标系（累积位姿，可能有漂移）
       │
       └─ base_link  ← 机器人本体坐标系（随机器人移动）
            │
            └─ laser / imu / camera  ← 传感器坐标系
```

本课我们实现 `odom → base_link` 的变换。

### nav_msgs/msg/Odometry

里程计消息包含：
- `pose` — 机器人在 odom 坐标系中的位置和姿态
- `twist` — 速度（线速度和角速度）

## 操作步骤

### Step 1: 编写里程计节点

创建 `~/wrk/ROS2/ros2_ws/src/my_robot_cpp/src/odom_node.cpp`：

```cpp
#include <chrono>
#include <cmath>
#include "rclcpp/rclcpp.hpp"
#include "nav_msgs/msg/odometry.hpp"
#include "geometry_msgs/msg/transform_stamped.hpp"
#include "tf2/LinearMath/Quaternion.h"
#include "tf2_ros/transform_broadcaster.h"

using namespace std::chrono_literals;

class OdometryNode : public rclcpp::Node
{
public:
  OdometryNode()
  : Node("odom_node"), x_(0.0), y_(0.0), theta_(0.0)
  {
    // 声明参数：轮距和轮半径
    this->declare_parameter("wheel_base", 0.3);   // 轮距 (m)
    this->declare_parameter("wheel_radius", 0.05); // 轮半径 (m)
    this->declare_parameter("left_rpm", 60.0);     // 模拟：左轮 RPM
    this->declare_parameter("right_rpm", 60.0);    // 模拟：右轮 RPM

    // 发布 /odom
    odom_pub_ = this->create_publisher<nav_msgs::msg::Odometry>("odom", 10);

    // TF 广播器
    tf_broadcaster_ = std::make_shared<tf2_ros::TransformBroadcaster>(this);

    // 定时器：每 50ms 更新一次里程计
    timer_ = this->create_wall_timer(
      50ms, std::bind(&OdometryNode::timer_callback, this));

    last_time_ = this->now();

    RCLCPP_INFO(this->get_logger(), "OdometryNode started.");
  }

private:
  void timer_callback()
  {
    auto now = this->now();
    double dt = (now - last_time_).seconds();
    last_time_ = now;

    if (dt <= 0.0) return;

    // 读取模拟轮速参数
    double left_rpm = this->get_parameter("left_rpm").as_double();
    double right_rpm = this->get_parameter("right_rpm").as_double();
    double wheel_base = this->get_parameter("wheel_base").as_double();
    double wheel_radius = this->get_parameter("wheel_radius").as_double();

    // RPM → 线速度 (m/s)
    double left_vel = left_rpm * 2.0 * M_PI / 60.0 * wheel_radius;
    double right_vel = right_rpm * 2.0 * M_PI / 60.0 * wheel_radius;

    // 差速模型：线速度和角速度
    double v = (left_vel + right_vel) / 2.0;         // 线速度
    double omega = (right_vel - left_vel) / wheel_base; // 角速度

    // 更新位姿（欧拉积分）
    double dx = v * std::cos(theta_) * dt;
    double dy = v * std::sin(theta_) * dt;
    x_ += dx;
    y_ += dy;
    theta_ += omega * dt;

    // --- 发布 /odom ---
    auto odom_msg = nav_msgs::msg::Odometry();
    odom_msg.header.stamp = now;
    odom_msg.header.frame_id = "odom";
    odom_msg.child_frame_id = "base_link";

    // 位置
    odom_msg.pose.pose.position.x = x_;
    odom_msg.pose.pose.position.y = y_;
    odom_msg.pose.pose.position.z = 0.0;

    // 姿态 (yaw → quaternion)
    tf2::Quaternion q;
    q.setRPY(0, 0, theta_);
    odom_msg.pose.pose.orientation.x = q.x();
    odom_msg.pose.pose.orientation.y = q.y();
    odom_msg.pose.pose.orientation.z = q.z();
    odom_msg.pose.pose.orientation.w = q.w();

    // 速度
    odom_msg.twist.twist.linear.x = v;
    odom_msg.twist.twist.angular.z = omega;

    odom_pub_->publish(odom_msg);

    // --- 广播 odom → base_link 变换 ---
    auto tf_msg = geometry_msgs::msg::TransformStamped();
    tf_msg.header.stamp = now;
    tf_msg.header.frame_id = "odom";
    tf_msg.child_frame_id = "base_link";
    tf_msg.transform.translation.x = x_;
    tf_msg.transform.translation.y = y_;
    tf_msg.transform.translation.z = 0.0;
    tf_msg.transform.rotation.x = q.x();
    tf_msg.transform.rotation.y = q.y();
    tf_msg.transform.rotation.z = q.z();
    tf_msg.transform.rotation.w = q.w();

    tf_broadcaster_->sendTransform(tf_msg);
  }

  double x_, y_, theta_;
  rclcpp::Time last_time_;

  rclcpp::Publisher<nav_msgs::msg::Odometry>::SharedPtr odom_pub_;
  std::shared_ptr<tf2_ros::TransformBroadcaster> tf_broadcaster_;
  rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<OdometryNode>());
  rclcpp::shutdown();
  return 0;
}
```

### Step 2: 更新依赖

**package.xml** 添加：
```xml
<depend>nav_msgs</depend>
<depend>tf2</depend>
<depend>tf2_ros</depend>
```

**CMakeLists.txt** 添加：
```cmake
find_package(nav_msgs REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)

add_executable(odom_node src/odom_node.cpp)
ament_target_dependencies(odom_node rclcpp nav_msgs tf2 tf2_ros)

# 更新 install(TARGETS ...)
```

### Step 3: 构建 & 运行

```bash
cd ~/wrk/ROS2/ros2_ws
colcon build --packages-select my_robot_cpp
source install/setup.bash

# 运行
ros2 run my_robot_cpp odom_node
```

### Step 4: 用 TF2 工具调试

在**另一个终端（已 source）**：

```bash
# 1. 查看 odom 数据
ros2 topic echo /odom

# 2. 查看 TF 树
ros2 run tf2_tools view_frames
# 会生成 frames.pdf，用 PDF 查看器打开

# 3. 实时查看 odom → base_link 的变换
ros2 run tf2_ros tf2_echo odom base_link
# 期望输出: 实时显示平移和旋转

# 4. 动态改变轮速模拟转弯
ros2 param set /odom_node left_rpm 30.0
ros2 param set /odom_node right_rpm 90.0
# 此时小车应该原地左转，观察 /odom 数据变化
```

### Step 5: 理解差速运动模型

```
v = (vl + vr) / 2         线速度 = 左右轮速度平均值
ω = (vr - vl) / wheelbase 角速度 = 速度差 / 轮距

vl = vr       → 直行
vl > vr       → 右转
vl < vr       → 左转
vl = -vr      → 原地旋转
```

### Step 6: 可视化 TF（为下一课做准备）

```bash
# 启动 RViz2
rviz2
```

在 RViz 中：
1. 添加 `TF` display → 能看到坐标轴
2. 添加 `Odometry` display → 看到里程计轨迹
3. Fixed Frame 设为 `odom`

## ros2 param set 模拟各种运动

```bash
# 直行
ros2 param set /odom_node left_rpm 60.0
ros2 param set /odom_node right_rpm 60.0

# 左转
ros2 param set /odom_node left_rpm 30.0
ros2 param set /odom_node right_rpm 90.0

# 原地右转
ros2 param set /odom_node left_rpm 60.0
ros2 param set /odom_node right_rpm -60.0

# 停车
ros2 param set /odom_node left_rpm 0.0
ros2 param set /odom_node right_rpm 0.0
```

## 验证清单

- [ ] `/odom` topic 持续发布数据
- [ ] `tf2_echo odom base_link` 能看到变换
- [ ] `view_frames` 生成的 PDF 中能看到 odom → base_link
- [ ] 修改左右轮速参数，里程计数据相应变化
- [ ] (可选) RViz 中能看到 TF 坐标轴

## 完成后

进入 [Phase 10: URDF + RViz](../phase10-urdf-rviz/GUIDE.md)
