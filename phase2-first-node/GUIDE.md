# Phase 2: 第一个 rclcpp Node

## 学习目标

- 编写第一个 C++ ROS2 节点
- 理解 `rclcpp::Node`、`RCLCPP_INFO` logger、`spin`
- 掌握 `ros2 run` 运行节点

## 背景知识

### 什么是 Node（节点）？

Node 是 ROS2 中最小的执行单元。每个节点负责一个功能：
- 一个节点负责读取激光雷达数据
- 一个节点负责控制电机
- 一个节点负责路径规划
- ...

节点之间通过 **topics**（话题）、**services**（服务）、**actions**（动作） 通信。

### rclcpp::Node

`rclcpp::Node` 是 C++ 中所有节点的基类。你的节点继承它，在构造函数中初始化。

### spin

`rclcpp::spin(node)` 让节点进入事件循环，持续处理订阅回调、服务请求等。它会阻塞当前线程直到节点被关闭（Ctrl+C）。

## 操作步骤

### Step 1: 创建源文件

创建 `~/wrk/ROS2/ros2_ws/src/my_robot_cpp/src/hello_node.cpp`：

```cpp
#include "rclcpp/rclcpp.hpp"

class HelloNode : public rclcpp::Node
{
public:
  HelloNode()
  : Node("hello_node")  // "hello_node" 是节点名称
  {
    RCLCPP_INFO(this->get_logger(), "Hello ROS2! Node has started.");
  }
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);                 // 初始化 ROS2
  auto node = std::make_shared<HelloNode>(); // 创建节点
  rclcpp::spin(node);                       // 进入事件循环
  rclcpp::shutdown();                       // 关闭 ROS2
  return 0;
}
```

### Step 2: 修改 CMakeLists.txt

在 `my_robot_cpp/CMakeLists.txt` 中，**在 `ament_package()` 之前**添加：

```cmake
add_executable(hello_node src/hello_node.cpp)
ament_target_dependencies(hello_node rclcpp)

install(TARGETS
  hello_node
  DESTINATION lib/${PROJECT_NAME}
)
```

修改后的完整 CMakeLists.txt 结构：

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_robot_cpp)

# ... 编译选项 ...

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)

# === 新增：注册可执行文件 ===
add_executable(hello_node src/hello_node.cpp)
ament_target_dependencies(hello_node rclcpp)

install(TARGETS
  hello_node
  DESTINATION lib/${PROJECT_NAME}
)
# === 新增结束 ===

if(BUILD_TESTING)
  # ... 测试配置 ...
endif()

ament_package()
```

**关键解释**：
- `add_executable(name source.cpp)` — 声明一个可执行文件
- `ament_target_dependencies(name rclcpp)` — 自动处理 include 路径和链接库
- `install(TARGETS ...)` — 把编译产物安装到正确位置，**没有这行 `ros2 run` 找不到你的节点**

### Step 3: 构建

```bash
cd ~/wrk/ROS2/ros2_ws
colcon build --packages-select my_robot_cpp
```

### Step 4: Source 并运行

```bash
source install/setup.bash
ros2 run my_robot_cpp hello_node
```

期望输出：

```
[INFO] [1234567890.123456789] [hello_node]: Hello ROS2! Node has started.
```

节点会一直运行（spin 在等待），按 `Ctrl+C` 停止。

## ros2 node CLI 调试

在节点运行期间，打开**另一个终端**（记得 source setup.bash），试试：

```bash
# 列出所有运行中的节点
ros2 node list
# 输出: /hello_node

# 查看节点详细信息（订阅了哪些 topic、提供了哪些 service 等）
ros2 node info /hello_node
```

## RCLCPP 日志级别

ROS2 有多种日志宏，按严重程度排序：

```cpp
RCLCPP_DEBUG(logger, "调试信息");
RCLCPP_INFO(logger, "普通信息");    // 最常用
RCLCPP_WARN(logger, "警告信息");
RCLCPP_ERROR(logger, "错误信息");
RCLCPP_FATAL(logger, "致命错误");
```

## 验证清单

- [ ] `colcon build` 成功
- [ ] `ros2 run my_robot_cpp hello_node` 看到日志输出
- [ ] `ros2 node list` 能看到 `/hello_node`
- [ ] `ros2 node info /hello_node` 能看到节点信息

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `ros2 run` 报 "No executable found" | 没加 `install(TARGETS ...)` | 检查 CMakeLists.txt，重新 build |
| 编译报 `rclcpp/rclcpp.hpp: No such file` | `ament_target_dependencies` 没写 | 确认 CMakeLists.txt 有这行 |
| 运行后立刻退出，看不到日志 | 没写 `rclcpp::spin()` | spin 阻塞主线程，否则 main 函数直接返回 |

## 完成后

进入 [Phase 3: Publisher / Subscriber](../phase3-pubsub/GUIDE.md)
