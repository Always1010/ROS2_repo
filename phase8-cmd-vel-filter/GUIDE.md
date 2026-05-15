# Phase 8: cmd_vel_filter_node（实用节点）

## 学习目标

- 写一个**有实际意义的 ROS2 节点**
- 实践 Topic 的订阅-处理-发布模式
- 掌握 QoS（Quality of Service）配置
- 实现超时安全逻辑

## 背景知识

### 为什么需要 cmd_vel_filter？

在差速机器人中，控制指令通过 `/cmd_vel` topic 发送（类型 `geometry_msgs/msg/Twist`）。实际工程中需要：

1. **速度限幅** — 防止指令超过机器人物理极限（如 max_linear = 1.0 m/s）
2. **超时保护** — 如果遥控端掉线/卡顿，超过一定时间没收到新指令，自动停车

这个节点就是中间层：
```
/joy → /cmd_vel_raw → [cmd_vel_filter] → /cmd_vel → robot_base
```

### geometry_msgs/msg/Twist

```cpp
// Twist.msg 定义
Vector3 linear         // 线速度 (m/s)
  float64 x            // 前进方向
  float64 y            // 横向（差速小车通常为 0）
  float64 z            // 上下（通常为 0）
Vector3 angular        // 角速度 (rad/s)
  float64 x
  float64 y
  float64 z            // 绕 Z 轴旋转（转向）
```

对于差速小车，实际使用的字段是：`linear.x` 和 `angular.z`。

## 操作步骤

### Step 1: 编写 cmd_vel_filter 节点

创建 `~/wrk/ROS2/ros2_ws/src/my_robot_cpp/src/cmd_vel_filter.cpp`：

```cpp
#include <chrono>
#include "rclcpp/rclcpp.hpp"
#include "geometry_msgs/msg/twist.hpp"

using namespace std::chrono_literals;

class CmdVelFilter : public rclcpp::Node
{
public:
  CmdVelFilter()
  : Node("cmd_vel_filter")
  {
    // 声明参数（可在 launch 或 YAML 中覆盖）
    this->declare_parameter("max_linear", 1.0);     // 最大线速度 m/s
    this->declare_parameter("max_angular", 2.0);    // 最大角速度 rad/s
    this->declare_parameter("timeout_ms", 500);     // 超时时间 ms

    // 读取参数
    max_linear_ = this->get_parameter("max_linear").as_double();
    max_angular_ = this->get_parameter("max_angular").as_double();
    timeout_ms_ = this->get_parameter("timeout_ms").as_int();

    // 订阅 /cmd_vel_raw
    sub_ = this->create_subscription<geometry_msgs::msg::Twist>(
      "cmd_vel_raw", 10,
      std::bind(&CmdVelFilter::cmd_vel_callback, this, std::placeholders::_1));

    // 发布 /cmd_vel
    pub_ = this->create_publisher<geometry_msgs::msg::Twist>("cmd_vel", 10);

    // 定时器：每 100ms 检查一次超时
    timer_ = this->create_wall_timer(
      100ms, std::bind(&CmdVelFilter::timer_callback, this));

    last_cmd_time_ = this->now();

    RCLCPP_INFO(this->get_logger(),
      "CmdVelFilter started. max_linear=%.1f, max_angular=%.1f, timeout=%ldms",
      max_linear_, max_angular_, timeout_ms_);
  }

private:
  void cmd_vel_callback(const geometry_msgs::msg::Twist & msg)
  {
    last_cmd_time_ = this->now();

    auto filtered = geometry_msgs::msg::Twist();

    // 速度限幅（clamp）
    filtered.linear.x = clamp(msg.linear.x, -max_linear_, max_linear_);
    filtered.angular.z = clamp(msg.angular.z, -max_angular_, max_angular_);

    // 检查是否做了限幅
    if (filtered.linear.x != msg.linear.x ||
        filtered.angular.z != msg.angular.z) {
      RCLCPP_WARN(this->get_logger(),
        "Speed limited! Raw: (%.2f, %.2f) → Filtered: (%.2f, %.2f)",
        msg.linear.x, msg.angular.z,
        filtered.linear.x, filtered.angular.z);
    }

    pub_->publish(filtered);
  }

  void timer_callback()
  {
    // 检查超时
    auto elapsed = (this->now() - last_cmd_time_).seconds() * 1000.0;
    if (elapsed > timeout_ms_) {
      // 超时 → 发布停车指令
      auto stop = geometry_msgs::msg::Twist();
      stop.linear.x = 0.0;
      stop.angular.z = 0.0;
      pub_->publish(stop);
    }
  }

  double clamp(double val, double min, double max)
  {
    if (val < min) return min;
    if (val > max) return max;
    return val;
  }

  double max_linear_;
  double max_angular_;
  int timeout_ms_;
  rclcpp::Time last_cmd_time_;

  rclcpp::Subscription<geometry_msgs::msg::Twist>::SharedPtr sub_;
  rclcpp::Publisher<geometry_msgs::msg::Twist>::SharedPtr pub_;
  rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<CmdVelFilter>());
  rclcpp::shutdown();
  return 0;
}
```

### Step 2: 更新依赖和 CMakeLists.txt

**package.xml** 添加：
```xml
<depend>geometry_msgs</depend>
```

**CMakeLists.txt** 添加：
```cmake
find_package(geometry_msgs REQUIRED)

add_executable(cmd_vel_filter src/cmd_vel_filter.cpp)
ament_target_dependencies(cmd_vel_filter rclcpp geometry_msgs)

# 更新 install(TARGETS ...)
install(TARGETS
  # ... 之前的 ...
  cmd_vel_filter
  DESTINATION lib/${PROJECT_NAME}
)
```

### Step 3: 构建 & 运行

```bash
cd ~/wrk/ROS2/ros2_ws
colcon build --packages-select my_robot_cpp
source install/setup.bash
```

终端1 — 运行 filter：
```bash
ros2 run my_robot_cpp cmd_vel_filter
```

终端2 — 测试：
```bash
source ~/wrk/ROS2/ros2_ws/install/setup.bash

# 测试1: 正常速度 → 应该通过
ros2 topic pub /cmd_vel_raw geometry_msgs/msg/Twist "{linear: {x: 0.5}, angular: {z: 1.0}}"

# 测试2: 超速 → 观察 filter 终端的 WARN 日志
ros2 topic pub /cmd_vel_raw geometry_msgs/msg/Twist "{linear: {x: 100.0}, angular: {z: 100.0}}"

# 测试3: 验证超时停车 → 等待 500ms 不发送指令，观察 /cmd_vel 是否收到 0
ros2 topic echo /cmd_vel
```

## 关键知识点

### 1. clamp 限幅模式

```cpp
// 对称限幅：输入范围 [-limit, limit]
filtered = clamp(raw, -limit, limit);
```

### 2. 超时保护模式

这是实际机器人开发中的**安全关键**逻辑：
- 用一个 timer 定期检查距离上次收到命令的时间
- 超过阈值 → 强制停车
- 防止机器人"失控"

### 3. QoS 配置

Publisher 和 Subscriber 的 QoS 必须匹配才能通信。上面的代码用了默认值（reliable），更多 QoS 选项：

```cpp
// Best effort: 丢包也不重传，适合高频传感器数据
auto qos = rclcpp::QoS(rclcpp::KeepLast(10)).best_effort();
pub_ = this->create_publisher<MsgType>("topic", qos);
```

## 验证清单

- [ ] 正常速度指令能通过 filter
- [ ] 超速指令被限幅，终端能看到 WARN 日志
- [ ] 停止发送指令 500ms 后，`/cmd_vel` 收到全零停车指令
- [ ] `ros2 topic echo /cmd_vel` 能观察到过滤后的输出

## 完成后

进入 [Phase 9: TF2 + odom_node](../phase9-tf2-odom/GUIDE.md)
