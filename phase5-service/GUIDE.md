# Phase 5: Service / Client

## 学习目标

- 理解 ROS2 Service（服务）的 Request/Response 模型
- 编写 Service Server 和 Service Client
- 掌握 `ros2 service` CLI 调试

## 背景知识

### Topic vs Service

| | Topic | Service |
|------|-------|---------|
| 通信模式 | 发布/订阅 (单向流) | 请求/响应 (一问一答) |
| 类比 | 收音机广播 | HTTP 请求 |
| 时机 | 持续数据流 | 偶尔的 RPC 调用 |

什么时候用 Service？
- "给我当前电量" → Service
- "把相机角度设为 30 度" → Service
- 激光雷达扫描数据 → Topic

### 标准接口

ROS2 内置了 `example_interfaces/srv/AddTwoInts`，我们用这个来学习，不需要自定义接口。

## 操作步骤

### Step 1: 编写 Service Server

创建 `~/wrk/ROS2/ros2_ws/src/my_robot_cpp/src/add_two_ints_server.cpp`：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "example_interfaces/srv/add_two_ints.hpp"

class AddTwoIntsServer : public rclcpp::Node
{
public:
  AddTwoIntsServer()
  : Node("add_two_ints_server")
  {
    // 创建 service server
    // 模板参数: 服务类型
    // 参数1: service 名称
    // 参数2: 回调（收到请求时调用）
    srv_ = this->create_service<example_interfaces::srv::AddTwoInts>(
      "add_two_ints",
      std::bind(&AddTwoIntsServer::handle_request, this,
                std::placeholders::_1, std::placeholders::_2));

    RCLCPP_INFO(this->get_logger(), "AddTwoInts service is ready.");
  }

private:
  void handle_request(
    const std::shared_ptr<example_interfaces::srv::AddTwoInts::Request> request,
    std::shared_ptr<example_interfaces::srv::AddTwoInts::Response> response)
  {
    // 处理请求：把两个数加起来
    response->sum = request->a + request->b;
    RCLCPP_INFO(this->get_logger(),
      "Received: a=%ld, b=%ld → sum=%ld",
      request->a, request->b, response->sum);
  }

  rclcpp::Service<example_interfaces::srv::AddTwoInts>::SharedPtr srv_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<AddTwoIntsServer>());
  rclcpp::shutdown();
  return 0;
}
```

### Step 2: 编写 Service Client

创建 `~/wrk/ROS2/ros2_ws/src/my_robot_cpp/src/add_two_ints_client.cpp`：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "example_interfaces/srv/add_two_ints.hpp"

using namespace std::chrono_literals;

class AddTwoIntsClient : public rclcpp::Node
{
public:
  AddTwoIntsClient()
  : Node("add_two_ints_client")
  {
    // 创建 service client
    client_ = this->create_client<example_interfaces::srv::AddTwoInts>(
      "add_two_ints");

    // 等待 service 上线（最多等 2 秒）
    if (!client_->wait_for_service(2s)) {
      RCLCPP_ERROR(this->get_logger(),
        "Service /add_two_ints not available!");
      return;
    }

    // 构造请求
    auto request =
      std::make_shared<example_interfaces::srv::AddTwoInts::Request>();
    request->a = 41;
    request->b = 1;

    // 异步发送请求
    auto future = client_->async_send_request(
      request,
      std::bind(&AddTwoIntsClient::response_callback, this,
                std::placeholders::_1));
  }

private:
  void response_callback(
    rclcpp::Client<example_interfaces::srv::AddTwoInts>::SharedFuture future)
  {
    auto response = future.get();
    RCLCPP_INFO(this->get_logger(),
      "Result: %ld", response->sum);
    rclcpp::shutdown();  // 拿到结果就退出
  }

  rclcpp::Client<example_interfaces::srv::AddTwoInts>::SharedPtr client_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<AddTwoIntsClient>();
  rclcpp::spin(node);  // 等待异步回调
  rclcpp::shutdown();
  return 0;
}
```

### Step 3: 更新 CMakeLists.txt 和 package.xml

在 `package.xml` 添加：

```xml
<depend>example_interfaces</depend>
```

在 `CMakeLists.txt` 添加：

```cmake
find_package(example_interfaces REQUIRED)

add_executable(add_two_ints_server src/add_two_ints_server.cpp)
ament_target_dependencies(add_two_ints_server rclcpp example_interfaces)

add_executable(add_two_ints_client src/add_two_ints_client.cpp)
ament_target_dependencies(add_two_ints_client rclcpp example_interfaces)

# 在 install(TARGETS ...) 中加入
install(TARGETS
  hello_node
  hello_pub
  hello_sub
  param_node
  add_two_ints_server
  add_two_ints_client
  DESTINATION lib/${PROJECT_NAME}
)
```

### Step 4: 构建 & 运行

```bash
cd ~/wrk/ROS2/ros2_ws
colcon build --packages-select my_robot_cpp
source install/setup.bash
```

终端1 — 启动 server：
```bash
ros2 run my_robot_cpp add_two_ints_server
```

终端2 — 运行 client：
```bash
ros2 run my_robot_cpp add_two_ints_client
# 期望输出: [INFO] [...] Result: 42
```

### Step 5: 用 CLI 手动调用 Service

**不需要 client 程序**，用 CLI 也能调用 service：

```bash
# 列出所有 service
ros2 service list

# 查看 service 接口类型
ros2 service type /add_two_ints
# 输出: example_interfaces/srv/AddTwoInts

# 查看接口定义
ros2 interface show example_interfaces/srv/AddTwoInts
# 输出:
#   int64 a
#   int64 b
#   ---
#   int64 sum

# 手动调用 service
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 10, b: 20}"
# 输出: sum=30
```

**注意**：YAML 参数中 `a:` 和 `10` 之间**必须有空格**，`{a: 10, b: 20}` 而不是 `{a:10, b:20}`！

## ros2 service CLI 速查

| 命令 | 作用 |
|------|------|
| `ros2 service list` | 列出所有活跃的 service |
| `ros2 service type /svc_name` | 查看 service 的接口类型 |
| `ros2 service call /svc_name Type "{args}"` | 手动调用 service |
| `ros2 service find Type` | 查找提供某个类型的 service |

## 验证清单

- [ ] server 启动后 `ros2 service list` 能看到 `/add_two_ints`
- [ ] client 运行后得到正确的 sum 值
- [ ] `ros2 service call` 手动调用成功

## 完成后

进入 [Phase 6: Custom Interfaces](../phase6-interfaces/GUIDE.md)
