# Phase 6: Custom Interfaces (自定义接口)

## 学习目标

- 创建自定义消息 (`.msg`) 和服务 (`.srv`) 接口包
- 在 C++ 节点中使用自定义接口
- 掌握 `ros2 interface` CLI

## 背景知识

### 为什么需要自定义接口？

之前的 `std_msgs/msg/String`、`example_interfaces/srv/AddTwoInts` 是 ROS2 内置的通用接口。实际项目中你需要定义自己的：

- `MotorCommand.msg` — 包含左右轮速度
- `SetSpeed.srv` — 设置速度并返回是否成功

### 接口包 vs 功能包

ROS2 推荐把接口定义**单独放在一个 package** 中，这样其他包可以独立依赖它。接口包里只有 `.msg` 和 `.srv` 文件，没有代码。

### .msg 和 .srv 文件格式

`.msg`（消息，用于 Topic）：
```
field_type field_name
field_type field_name
...
```

`.srv`（服务，用于 Service）：
```
# request 部分 (上面)
field_type field_name
---
# response 部分 (下面)
field_type field_name
```

## 操作步骤

### Step 1: 创建接口包

```bash
cd ~/wrk/ROS2/ros2_ws/src
ros2 pkg create my_robot_interfaces --build-type ament_cmake
cd my_robot_interfaces

# 删除自动生成的目录（接口包不需要这些）
rm -rf include src

# 创建 msg 和 srv 目录
mkdir msg srv
```

### Step 2: 定义 MotorCommand.msg

创建 `msg/MotorCommand.msg`：

```
# 电机控制指令
float64 left_speed    # 左轮速度 (rad/s)
float64 right_speed   # 右轮速度 (rad/s)
```

### Step 3: 定义 SetSpeed.srv

创建 `srv/SetSpeed.srv`：

```
# 请求
float64 max_linear   # 最大线速度 (m/s)
float64 max_angular  # 最大角速度 (rad/s)
---
# 响应
bool success         # 是否设置成功
string message       # 附加信息
```

### Step 4: 配置 CMakeLists.txt（接口包）

把自动生成的 `CMakeLists.txt` **替换**为：

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_robot_interfaces)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

# 声明消息文件
set(msg_files
  "msg/MotorCommand.msg"
)

# 声明服务文件
set(srv_files
  "srv/SetSpeed.srv"
)

# 生成接口代码
rosidl_generate_interfaces(${PROJECT_NAME}
  ${msg_files}
  ${srv_files}
)

ament_export_dependencies(rosidl_default_runtime)
ament_package()
```

**关键差异**：接口包用 `rosidl_default_generators`，功能包用 `rclcpp`。

### Step 5: 配置 package.xml（接口包）

在 `package.xml` 中添加：

```xml
<buildtool_depend>rosidl_default_generators</buildtool_depend>
<exec_depend>rosidl_default_runtime</exec_depend>
<member_of_group>rosidl_interface_packages</member_of_group>
```

`<member_of_group>rosidl_interface_packages</member_of_group>` 这一行**必须加**，否则 `ros2 interface` 命令找不到你的接口。

### Step 6: 构建接口包

```bash
cd ~/wrk/ROS2/ros2_ws
colcon build --packages-select my_robot_interfaces
source install/setup.bash
```

### Step 7: 用 CLI 验证接口

```bash
# 列出所有接口（你的应该在里面）
ros2 interface list | grep my_robot

# 查看消息定义
ros2 interface show my_robot_interfaces/msg/MotorCommand
# 期望输出:
#   float64 left_speed
#   float64 right_speed

# 查看服务定义
ros2 interface show my_robot_interfaces/srv/SetSpeed
# 期望输出:
#   float64 max_linear
#   float64 max_angular
#   ---
#   bool success
#   string message
```

### Step 8: 在 C++ 节点中使用自定义接口

创建 `~/wrk/ROS2/ros2_ws/src/my_robot_cpp/src/motor_test.cpp`：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "my_robot_interfaces/msg/motor_command.hpp"

class MotorTest : public rclcpp::Node
{
public:
  MotorTest()
  : Node("motor_test")
  {
    pub_ = this->create_publisher<my_robot_interfaces::msg::MotorCommand>(
      "motor_cmd", 10);

    // 发布一条测试指令
    auto cmd = my_robot_interfaces::msg::MotorCommand();
    cmd.left_speed = 1.5;
    cmd.right_speed = 1.5;
    pub_->publish(cmd);
    RCLCPP_INFO(this->get_logger(),
      "Published: left=%.1f, right=%.1f",
      cmd.left_speed, cmd.right_speed);
  }

private:
  rclcpp::Publisher<my_robot_interfaces::msg::MotorCommand>::SharedPtr pub_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<MotorTest>());
  rclcpp::shutdown();
  return 0;
}
```

### Step 9: 更新 my_robot_cpp 的依赖

**package.xml** 添加：

```xml
<depend>my_robot_interfaces</depend>
```

**CMakeLists.txt** 添加：

```cmake
find_package(my_robot_interfaces REQUIRED)

add_executable(motor_test src/motor_test.cpp)
ament_target_dependencies(motor_test rclcpp my_robot_interfaces)

# 更新 install(TARGETS ...)
install(TARGETS
  # ... 之前的 ...
  motor_test
  DESTINATION lib/${PROJECT_NAME}
)
```

### Step 10: 构建并测试

```bash
cd ~/wrk/ROS2/ros2_ws
# 先构建接口包，再构建功能包
colcon build --packages-select my_robot_interfaces my_robot_cpp
source install/setup.bash

ros2 run my_robot_cpp motor_test
```

## ros2 interface CLI 速查

| 命令 | 作用 |
|------|------|
| `ros2 interface list` | 列出所有可用接口 |
| `ros2 interface show Type` | 查看接口详细定义 |
| `ros2 interface package my_robot_interfaces` | 列出某个包的所有接口 |

## 验证清单

- [ ] `ros2 interface show my_robot_interfaces/msg/MotorCommand` 能看到字段
- [ ] `ros2 interface show my_robot_interfaces/srv/SetSpeed` 能看到请求/响应
- [ ] `motor_test` 节点编译运行成功
- [ ] 理解接口包和功能包的分离原因

## 完成后

进入 [Phase 7: Launch + YAML Config](../phase7-launch/GUIDE.md)
