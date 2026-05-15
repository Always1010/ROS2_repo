# Phase 11: Mini Project 整合

## 学习目标

- 将之前所有模块整合为完整系统
- 一条 launch 命令启动整个差速小车仿真
- 理解 ROS2 项目的完整结构和组件关系

## 最终系统架构

```
                        ┌──────────────────┐
                        │  joy / keyboard  │ (外部输入)
                        └────────┬─────────┘
                                 │ /cmd_vel_raw
                                 ▼
                        ┌──────────────────┐
                        │  cmd_vel_filter  │ (Phase 8)
                        └────────┬─────────┘
                                 │ /cmd_vel (过滤后)
                                 ▼
                        ┌──────────────────┐
                        │    odom_node     │ (Phase 9)
                        │ 读取 /cmd_vel    │
                        │ 发布 /odom       │
                        │ 广播 odom→base   │
                        └────────┬─────────┘
                                 │ TF
                                 ▼
┌──────────────┐      ┌──────────────────┐
│  URDF file   │─────▶│robot_state_pub   │─────▶ TF (Phase 10)
└──────────────┘      └──────────────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │      RViz2       │ (可视化)
                        └──────────────────┘
```

## 操作步骤

### Step 1: 升级 odom_node 支持订阅 /cmd_vel

之前的 `odom_node` 用参数模拟轮速，现在让它**订阅真实的 `/cmd_vel`** 来计算里程计。

在 `my_robot_cpp/src/` 下创建新文件 `odom_from_cmd_vel.cpp`：

```cpp
#include <chrono>
#include <cmath>
#include "rclcpp/rclcpp.hpp"
#include "nav_msgs/msg/odometry.hpp"
#include "geometry_msgs/msg/twist.hpp"
#include "geometry_msgs/msg/transform_stamped.hpp"
#include "tf2/LinearMath/Quaternion.h"
#include "tf2_ros/transform_broadcaster.h"

using namespace std::chrono_literals;

class OdomFromCmdVel : public rclcpp::Node
{
public:
  OdomFromCmdVel()
  : Node("odom_from_cmd_vel"), x_(0.0), y_(0.0), theta_(0.0),
    linear_x_(0.0), angular_z_(0.0)
  {
    this->declare_parameter("wheel_base", 0.3);

    // 订阅过滤后的 /cmd_vel
    cmd_vel_sub_ = this->create_subscription<geometry_msgs::msg::Twist>(
      "cmd_vel", 10,
      std::bind(&OdomFromCmdVel::cmd_vel_callback, this,
                std::placeholders::_1));

    // 发布 /odom
    odom_pub_ = this->create_publisher<nav_msgs::msg::Odometry>("odom", 10);

    // TF 广播器
    tf_broadcaster_ = std::make_shared<tf2_ros::TransformBroadcaster>(this);

    // 50ms 更新一次位姿
    timer_ = this->create_wall_timer(
      50ms, std::bind(&OdomFromCmdVel::timer_callback, this));

    last_time_ = this->now();
    RCLCPP_INFO(this->get_logger(), "OdomFromCmdVel started (subscribed to /cmd_vel).");
  }

private:
  void cmd_vel_callback(const geometry_msgs::msg::Twist & msg)
  {
    linear_x_ = msg.linear.x;
    angular_z_ = msg.angular.z;
    last_cmd_time_ = this->now();
  }

  void timer_callback()
  {
    auto now = this->now();
    double dt = (now - last_time_).seconds();
    last_time_ = now;
    if (dt <= 0.0) return;

    // 超时停车
    if ((now - last_cmd_time_).seconds() > 0.5) {
      linear_x_ = 0.0;
      angular_z_ = 0.0;
    }

    // 更新位姿
    double dx = linear_x_ * std::cos(theta_) * dt;
    double dy = linear_x_ * std::sin(theta_) * dt;
    x_ += dx;
    y_ += dy;
    theta_ += angular_z_ * dt;

    // 发布 /odom
    auto odom = nav_msgs::msg::Odometry();
    odom.header.stamp = now;
    odom.header.frame_id = "odom";
    odom.child_frame_id = "base_link";
    odom.pose.pose.position.x = x_;
    odom.pose.pose.position.y = y_;
    tf2::Quaternion q;
    q.setRPY(0, 0, theta_);
    odom.pose.pose.orientation.x = q.x();
    odom.pose.pose.orientation.y = q.y();
    odom.pose.pose.orientation.z = q.z();
    odom.pose.pose.orientation.w = q.w();
    odom.twist.twist.linear.x = linear_x_;
    odom.twist.twist.angular.z = angular_z_;
    odom_pub_->publish(odom);

    // 广播 TF
    auto tf = geometry_msgs::msg::TransformStamped();
    tf.header.stamp = now;
    tf.header.frame_id = "odom";
    tf.child_frame_id = "base_link";
    tf.transform.translation.x = x_;
    tf.transform.translation.y = y_;
    tf.transform.rotation.x = q.x();
    tf.transform.rotation.y = q.y();
    tf.transform.rotation.z = q.z();
    tf.transform.rotation.w = q.w();
    tf_broadcaster_->sendTransform(tf);
  }

  double x_, y_, theta_;
  double linear_x_, angular_z_;
  rclcpp::Time last_time_, last_cmd_time_;

  rclcpp::Subscription<geometry_msgs::msg::Twist>::SharedPtr cmd_vel_sub_;
  rclcpp::Publisher<nav_msgs::msg::Odometry>::SharedPtr odom_pub_;
  std::shared_ptr<tf2_ros::TransformBroadcaster> tf_broadcaster_;
  rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<OdomFromCmdVel>());
  rclcpp::shutdown();
  return 0;
}
```

### Step 2: 更新 CMakeLists.txt 和 package.xml

在 `CMakeLists.txt` 中添加：

```cmake
add_executable(odom_from_cmd_vel src/odom_from_cmd_vel.cpp)
ament_target_dependencies(odom_from_cmd_vel rclcpp nav_msgs geometry_msgs tf2 tf2_ros)

# 更新 install(TARGETS ...) 加入 odom_from_cmd_vel
```

### Step 3: 创建最终整合 launch 文件

在 `my_robot_bringup/launch/` 下创建 `bringup.launch.py`：

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
import os


def generate_launch_description():
    bringup_dir = get_package_share_directory('my_robot_bringup')
    desc_dir = get_package_share_directory('my_robot_description')

    # URDF 文件
    urdf_path = os.path.join(desc_dir, 'urdf', 'my_robot.urdf')
    with open(urdf_path, 'r') as f:
        robot_desc = f.read()

    # RViz 配置
    rviz_config = os.path.join(bringup_dir, 'config', 'bringup.rviz')

    # YAML 参数
    params_file = os.path.join(bringup_dir, 'config', 'bringup_params.yaml')

    return LaunchDescription([
        # 1. 速度过滤器
        Node(
            package='my_robot_cpp',
            executable='cmd_vel_filter',
            name='cmd_vel_filter',
            output='screen',
            parameters=[params_file],
        ),

        # 2. 里程计（订阅 /cmd_vel）
        Node(
            package='my_robot_cpp',
            executable='odom_from_cmd_vel',
            name='odom_from_cmd_vel',
            output='screen',
            parameters=[params_file],
        ),

        # 3. Robot State Publisher（URDF → TF）
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            name='robot_state_publisher',
            parameters=[{'robot_description': robot_desc}],
        ),

        # 4. Joint State Publisher（提供关节角度）
        Node(
            package='joint_state_publisher',
            executable='joint_state_publisher',
            name='joint_state_publisher',
        ),

        # 5. RViz
        Node(
            package='rviz2',
            executable='rviz2',
            name='rviz2',
            arguments=['-d', rviz_config],
        ),
    ])
```

### Step 4: 创建整合用参数文件

创建 `my_robot_bringup/config/bringup_params.yaml`：

```yaml
/cmd_vel_filter:
  ros__parameters:
    max_linear: 1.0
    max_angular: 2.0
    timeout_ms: 500

/odom_from_cmd_vel:
  ros__parameters:
    wheel_base: 0.3
```

### Step 5: 更新 bringup 包的 CMakeLists.txt

添加 `config/bringup.rviz`（可以先复制 `display.rviz`，打开后通过 GUI 调整保存）：

```cmake
# 安装新增的 launch 和 config 文件
install(DIRECTORY
  launch
  config
  DESTINATION share/${PROJECT_NAME}
)
```

确保 `package.xml` 中的依赖完整：

```xml
<exec_depend>my_robot_cpp</exec_depend>
<exec_depend>my_robot_description</exec_depend>
<exec_depend>robot_state_publisher</exec_depend>
<exec_depend>joint_state_publisher</exec_depend>
<exec_depend>rviz2</exec_depend>
```

### Step 6: 构建完整系统

```bash
cd ~/wrk/ROS2/ros2_ws

# 构建所有包
colcon build
source install/setup.bash
```

### Step 7: 一键启动！

```bash
ros2 launch my_robot_bringup bringup.launch.py
```

此时 RViz 打开，一切都已就绪。

### Step 8: 发送控制指令测试

打开新终端：

```bash
source ~/wrk/ROS2/ros2_ws/install/setup.bash

# 控制小车前进
ros2 topic pub /cmd_vel_raw geometry_msgs/msg/Twist \
  "{linear: {x: 0.5}, angular: {z: 0.0}}"

# 控制小车左转
ros2 topic pub /cmd_vel_raw geometry_msgs/msg/Twist \
  "{linear: {x: 0.3}, angular: {z: 1.0}}"

# 控制超速 → 观察 filter 限幅效果
ros2 topic pub /cmd_vel_raw geometry_msgs/msg/Twist \
  "{linear: {x: 100.0}, angular: {z: 100.0}}"

# 停车
ros2 topic pub /cmd_vel_raw geometry_msgs/msg/Twist \
  "{linear: {x: 0.0}, angular: {z: 0.0}}"
```

在 RViz 中你应该能看到：
- 小车 3D 模型
- 里程计轨迹（Odometry display）
- TF 坐标轴
- 小车随指令移动

## 最终项目结构回顾

```
~/wrk/ROS2/ros2_ws/src/
├── my_robot_interfaces/        # 自定义接口
│   ├── CMakeLists.txt
│   ├── package.xml
│   ├── msg/MotorCommand.msg
│   └── srv/SetSpeed.srv
├── my_robot_cpp/               # 核心功能节点
│   ├── CMakeLists.txt
│   ├── package.xml
│   ├── include/my_robot_cpp/
│   └── src/
│       ├── hello_node.cpp              # 第一个节点
│       ├── hello_pub.cpp               # Publisher
│       ├── hello_sub.cpp               # Subscriber
│       ├── param_node.cpp              # Parameter
│       ├── add_two_ints_server.cpp     # Service Server
│       ├── add_two_ints_client.cpp     # Service Client
│       ├── motor_test.cpp              # 自定义接口测试
│       ├── cmd_vel_filter.cpp          # 速度过滤器
│       ├── odom_node.cpp               # 里程计（参数驱动）
│       └── odom_from_cmd_vel.cpp       # 里程计（cmd_vel驱动）
├── my_robot_description/       # 机器人模型
│   ├── CMakeLists.txt
│   ├── package.xml
│   ├── urdf/my_robot.urdf
│   ├── rviz/display.rviz
│   └── launch/display_robot.launch.py
└── my_robot_bringup/           # 启动配置
    ├── CMakeLists.txt
    ├── package.xml
    ├── launch/
    │   ├── robot_launch.py
    │   └── bringup.launch.py
    └── config/
        ├── robot_params.yaml
        └── bringup_params.yaml
```

## 数据流全景图

```
                    ┌──────────────────────┐
pub cmd_vel_raw ───▶│   /cmd_vel_raw       │
                    │   (geometry_msgs/    │
                    │    Twist)            │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   cmd_vel_filter     │
                    │   - 速度限幅         │
                    │   - 超时停车         │
                    └──────────┬───────────┘
                               │ /cmd_vel
                               ▼
                    ┌──────────────────────┐
                    │  odom_from_cmd_vel   │
                    │  - 位姿积分          │
                    │  - 发布 /odom        │
                    │  - 广播 odom→base    │
                    └──────────┬───────────┘
                               │ TF + /odom
         ┌─────────────────────┼─────────────────────┐
         ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ robot_state_    │  │  Odometry       │  │  TF display     │
│ publisher       │  │  display        │  │  (坐标轴)        │
│ (URDF→TF)       │  │  (轨迹)          │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │       RViz2          │
                    │    (可视化一切)       │
                    └──────────────────────┘
```

## 验证清单

- [ ] `ros2 launch my_robot_bringup bringup.launch.py` 一键启动所有节点
- [ ] RViz 显示机器人 3D 模型
- [ ] 发送 `/cmd_vel_raw` 指令，小车在 RViz 中移动
- [ ] 超速指令被 cmd_vel_filter 限幅
- [ ] 停止发指令后，odom_node 自动停车
- [ ] Odometry display 能看到运动轨迹

## 学完下一步做什么？

1. **键盘遥控**: 用 `teleop_twist_keyboard` 替代手动 pub
   ```bash
   sudo apt install ros-jazzy-teleop-twist-keyboard
   ros2 run teleop_twist_keyboard teleop_twist_keyboard
   ```
2. **Gazebo 仿真**: 在物理引擎中运行你的机器人
3. **Nav2 导航**: 自动路径规划 + 避障
4. **SLAM**: 同步定位与建图

## 课程完成！

你已经从零搭建了一个完整的 ROS2 C++ 差速小车项目，掌握了 ROS2 开发的核心技能。
