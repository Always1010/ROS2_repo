# Phase 4: Parameter + YAML

## 学习目标

- 理解 ROS2 Parameter（参数）系统
- 在 C++ 节点中声明和读取参数
- 掌握 `ros2 param` CLI 动态修改参数

## 背景知识

### 什么是 Parameter？

Parameter 是节点的**配置变量**，可以在运行时读取和修改。比如：
- `max_speed: 2.0` — 最大速度
- `robot_name: "my_robot"` — 机器人名字
- `use_sim_time: false` — 是否使用仿真时间

参数值是**键值对**，支持类型：`int`, `double`, `string`, `bool`, `array`。

### 为什么要用 Parameter？

不用 parameter 的话，你只能**硬编码**常量，改一次就要**重新编译**。用了 parameter，可以在运行时动态调整。

## 操作步骤

### Step 1: 编写参数节点

创建 `~/wrk/ROS2/ros2_ws/src/my_robot_cpp/src/param_node.cpp`：

```cpp
#include <chrono>
#include "rclcpp/rclcpp.hpp"

using namespace std::chrono_literals;

class ParamNode : public rclcpp::Node
{
public:
  ParamNode()
  : Node("param_node")
  {
    // 声明参数: 参数名, 默认值
    this->declare_parameter("robot_name", "my_robot");
    this->declare_parameter("max_speed", 2.0);
    this->declare_parameter("enable_log", true);

    // 定时器，每秒打印当前参数值
    timer_ = this->create_wall_timer(
      1s, std::bind(&ParamNode::timer_callback, this));
  }

private:
  void timer_callback()
  {
    // 读取参数值
    std::string robot_name = this->get_parameter("robot_name").as_string();
    double max_speed = this->get_parameter("max_speed").as_double();
    bool enable_log = this->get_parameter("enable_log").as_bool();

    RCLCPP_INFO(this->get_logger(),
      "robot=%s, max_speed=%.1f, enable_log=%s",
      robot_name.c_str(), max_speed,
      enable_log ? "true" : "false");
  }

  rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<ParamNode>());
  rclcpp::shutdown();
  return 0;
}
```

### Step 2: 更新 CMakeLists.txt

添加 executable（在 `ament_package()` 之前）：

```cmake
add_executable(param_node src/param_node.cpp)
ament_target_dependencies(param_node rclcpp)

# 同时更新 install(TARGETS ...) ，把 param_node 加进去
install(TARGETS
  hello_node
  hello_pub
  hello_sub
  param_node
  DESTINATION lib/${PROJECT_NAME}
)
```

### Step 3: 构建 & 运行

```bash
cd ~/wrk/ROS2/ros2_ws
colcon build --packages-select my_robot_cpp
source install/setup.bash
ros2 run my_robot_cpp param_node
```

你应该看到每秒打印一次参数值（都是默认值）。

### Step 4: 用 CLI 操作参数

在**另一个终端**（已 source）：

```bash
# 列出所有节点
ros2 node list

# 列出 /param_node 的所有参数
ros2 param list /param_node
# 输出:
#   enable_log
#   max_speed
#   robot_name
#   use_sim_time (ROS2 自动给每个节点加的参数)

# 读取某个参数
ros2 param get /param_node robot_name
# 输出: String value is: my_robot

# 动态修改参数
ros2 param set /param_node max_speed 5.0
# 回到运行节点的终端，你会看到输出中 max_speed 变成 5.0 了！

ros2 param set /param_node robot_name "cool_bot"

# 导出所有参数到 YAML 文件
ros2 param dump /param_node
# 会在当前目录生成 param_node.yaml
```

### Step 5: 查看导出的 YAML

```bash
cat ./param_node.yaml
```

内容类似：

```yaml
/param_node:
  ros__parameters:
    enable_log: true
    max_speed: 5.0
    robot_name: cool_bot
    use_sim_time: false
```

### Step 6: 从 YAML 文件加载参数

修改 YAML 文件后，重新运行节点并加载：

```bash
# 编辑 YAML 文件，把 max_speed 改成 10.0

# 运行时加载 YAML
ros2 run my_robot_cpp param_node --ros-args --params-file ./param_node.yaml
```

你会看到节点启动时 max_speed 就是 10.0。

## ros2 param CLI 速查

| 命令 | 作用 |
|------|------|
| `ros2 param list /node` | 列出节点的所有参数 |
| `ros2 param get /node param_name` | 读取参数值 |
| `ros2 param set /node param_name value` | 动态修改参数 |
| `ros2 param dump /node` | 导出参数到 YAML 文件 |
| `ros2 param load /node --params-file file.yaml` | 从 YAML 加载参数 |

## 验证清单

- [ ] 节点运行后能看到参数默认值
- [ ] `ros2 param set` 修改参数后节点输出同步变化
- [ ] `ros2 param dump` 能导出 YAML
- [ ] 用 `--params-file` 加载 YAML 参数生效

## 完成后

进入 [Phase 5: Service / Client](../phase5-service/GUIDE.md)
