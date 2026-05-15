# Phase 3: Publisher / Subscriber + ros2 topic CLI

## 学习目标

- 理解 ROS2 Topic（话题）通信模型
- 编写 Publisher（发布者）和 Subscriber（订阅者）节点
- 掌握 `ros2 topic` 系列命令进行调试

## 背景知识

### Topic 通信模型

```
Publisher 节点                Subscriber 节点
    │                              │
    │  publish("topic", msg)       │  subscribe("topic", callback)
    │                              │
    └───────────┬──────────────────┘
                │
        ROS2 Middleware (DDS)
```

- **解耦**: Pub 和 Sub 互不知道对方存在，只知道 topic 名称
- **多对多**: 一个 topic 可以有多个 Pub 和多个 Sub
- **消息类型**: 每条 topic 有固定的消息类型（如 `std_msgs/msg/String`）

### std_msgs/msg/String

ROS2 内置的标准消息类型，包含一个 `data` 字段（string）。

## 操作步骤

### Step 1: 编写 Talker (Publisher)

创建 `~/wrk/ROS2/ros2_ws/src/my_robot_cpp/src/hello_pub.cpp`：

```cpp
#include <chrono>
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

using namespace std::chrono_literals;

class HelloPub : public rclcpp::Node
{
public:
  HelloPub()
  : Node("hello_pub"), count_(0)
  {
    // 创建 publisher
    // 模板参数: 消息类型
    // 参数1: topic 名称
    // 参数2: QoS(队列深度)，10 表示缓存最近 10 条消息
    pub_ = this->create_publisher<std_msgs::msg::String>("chatter", 10);

    // 创建定时器，每 500ms 调用一次 timer_callback
    timer_ = this->create_wall_timer(
      500ms, std::bind(&HelloPub::timer_callback, this));
  }

private:
  void timer_callback()
  {
    auto msg = std_msgs::msg::String();
    msg.data = "Hello ROS2! Count: " + std::to_string(count_++);
    RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", msg.data.c_str());
    pub_->publish(msg);
  }

  rclcpp::Publisher<std_msgs::msg::String>::SharedPtr pub_;
  rclcpp::TimerBase::SharedPtr timer_;
  size_t count_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<HelloPub>());
  rclcpp::shutdown();
  return 0;
}
```

### Step 2: 编写 Listener (Subscriber)

创建 `~/wrk/ROS2/ros2_ws/src/my_robot_cpp/src/hello_sub.cpp`：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

class HelloSub : public rclcpp::Node
{
public:
  HelloSub()
  : Node("hello_sub")
  {
    // 创建 subscriber
    // 模板参数: 消息类型
    // 参数1: topic 名称
    // 参数2: QoS(队列深度)
    // 参数3: 回调函数（收到消息时被调用）
    sub_ = this->create_subscription<std_msgs::msg::String>(
      "chatter", 10,
      std::bind(&HelloSub::topic_callback, this, std::placeholders::_1));
  }

private:
  void topic_callback(const std_msgs::msg::String & msg)
  {
    RCLCPP_INFO(this->get_logger(), "I heard: '%s'", msg.data.c_str());
  }

  rclcpp::Subscription<std_msgs::msg::String>::SharedPtr sub_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<HelloSub>());
  rclcpp::shutdown();
  return 0;
}
```

### Step 3: 更新 CMakeLists.txt

在 CMakeLists.txt 中添加两个 executable（在 `ament_package()` 之前）：

```cmake
# 新增 std_msgs 依赖
find_package(std_msgs REQUIRED)

# hello_node (之前的)
add_executable(hello_node src/hello_node.cpp)
ament_target_dependencies(hello_node rclcpp)

# hello_pub (新增)
add_executable(hello_pub src/hello_pub.cpp)
ament_target_dependencies(hello_pub rclcpp std_msgs)

# hello_sub (新增)
add_executable(hello_sub src/hello_sub.cpp)
ament_target_dependencies(hello_sub rclcpp std_msgs)

install(TARGETS
  hello_node
  hello_pub
  hello_sub
  DESTINATION lib/${PROJECT_NAME}
)
```

### Step 4: 更新 package.xml

在 `my_robot_cpp/package.xml` 中添加 `std_msgs` 依赖（在 `<depend>rclcpp</depend>` 下面）：

```xml
<depend>std_msgs</depend>
```

### Step 5: 构建

```bash
cd ~/wrk/ROS2/ros2_ws
colcon build --packages-select my_robot_cpp
source install/setup.bash
```

### Step 6: 运行 & 验证

打开**3 个终端**，每个都要 source：

```bash
source ~/wrk/ROS2/ros2_ws/install/setup.bash
```

终端1 — 运行 publisher：
```bash
ros2 run my_robot_cpp hello_pub
```

终端2 — 运行 subscriber：
```bash
ros2 run my_robot_cpp hello_sub
```

终端3 — 用 CLI 工具观察：
```bash
# 列出所有 topic
ros2 topic list
# 应该看到 /chatter 和 /parameter_events, /rosout

# 查看 /chatter 的实时数据
ros2 topic echo /chatter

# 查看 /chatter 的消息类型
ros2 topic info /chatter

# 查看发布频率
ros2 topic hz /chatter
```

## ros2 topic CLI 速查

| 命令 | 作用 |
|------|------|
| `ros2 topic list` | 列出所有活跃的 topic |
| `ros2 topic echo /topic_name` | 实时打印 topic 数据 |
| `ros2 topic info /topic_name` | 查看 topic 类型和 pub/sub 数量 |
| `ros2 topic hz /topic_name` | 查看发布频率 |
| `ros2 topic pub /topic_name msg_type "{data: hello}"` | 手动发布一条消息 |
| `ros2 topic bw /topic_name` | 查看带宽 |

## 验证清单

- [ ] 终端1 看到 Publishing 日志
- [ ] 终端2 看到 I heard 日志
- [ ] `ros2 topic list` 看到 `/chatter`
- [ ] `ros2 topic echo /chatter` 看到实时数据
- [ ] Ctrl+C 关闭 pub 后，sub 不再收到消息

## 完成后

进入 [Phase 4: Parameter + YAML](../phase4-param/GUIDE.md)
