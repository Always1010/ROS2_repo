# Phase 7: Launch + YAML Config

## 学习目标

- 理解 ROS2 Launch 系统的作用
- 编写 Python launch 文件同时启动多个节点
- 使用 YAML 文件管理参数
- 掌握 `ros2 launch` 命令

## 背景知识

### 为什么需要 Launch？

之前每个节点都要手动开一个终端 + `ros2 run`。实际项目有十几个节点，手动启动不现实。

Launch 文件可以：
- **一次启动多个节点**
- **统一管理参数**（YAML 文件）
- **设置节点间启动顺序**
- **支持条件启动、传参**

### Python Launch vs XML Launch

ROS2 支持两种 launch 格式：
- **Python launch** — 主流，灵活，动态逻辑
- **XML launch** — 简单声明式

本课程用 Python launch。

### Launch Package

Launch 文件可以放在任何包里，但推荐单独一个 `bringup` 包来管理，把所有 launch 和 config 集中在一起。

## 操作步骤

### Step 1: 创建 bringup 包

```bash
cd ~/wrk/ROS2/ros2_ws/src
ros2 pkg create my_robot_bringup --build-type ament_cmake
cd my_robot_bringup

# 创建 launch 和 config 目录
mkdir launch config
```

### Step 2: 创建 YAML 参数文件

创建 `config/robot_params.yaml`：

```yaml
# /param_node 的参数
/param_node:
  ros__parameters:
    robot_name: "cool_bot"
    max_speed: 3.0
    enable_log: true

# /hello_pub 的参数（如果有的话）
# /hello_pub:
#   ros__parameters:
#     publish_rate: 2.0
```

### Step 3: 编写 Launch 文件

创建 `launch/robot_launch.py`：

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
import os


def generate_launch_description():
    # 获取 config 目录路径
    config_dir = os.path.join(
        get_package_share_directory('my_robot_bringup'),
        'config'
    )
    params_file = os.path.join(config_dir, 'robot_params.yaml')

    # 声明要启动的节点
    hello_pub_node = Node(
        package='my_robot_cpp',       # 包名
        executable='hello_pub',       # executable 名
        name='hello_pub',             # 节点名（可自定义，会和 executable 不同）
        output='screen',              # 日志输出到终端
    )

    hello_sub_node = Node(
        package='my_robot_cpp',
        executable='hello_sub',
        name='hello_sub',
        output='screen',
    )

    param_node = Node(
        package='my_robot_cpp',
        executable='param_node',
        name='param_node',
        output='screen',
        parameters=[params_file],     # 加载 YAML 参数
    )

    # 返回 LaunchDescription，包含所有节点
    return LaunchDescription([
        hello_pub_node,
        hello_sub_node,
        param_node,
    ])
```

### Step 4: 更新 bringup 包的 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_robot_bringup)

find_package(ament_cmake REQUIRED)

# 安装 launch 和 config 目录到正确位置
install(DIRECTORY
  launch
  config
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

### Step 5: 更新 bringup 包的 package.xml

添加依赖声明，确保 launch 节点依赖的包存在：

```xml
<exec_depend>my_robot_cpp</exec_depend>
<exec_depend>rclcpp</exec_depend>
<exec_depend>std_msgs</exec_depend>
```

**注意**：launch 文件的依赖用 `<exec_depend>`（运行时依赖），不是 `<depend>`。

### Step 6: 构建 & 运行

```bash
cd ~/wrk/ROS2/ros2_ws
colcon build --packages-select my_robot_bringup
source install/setup.bash
```

一条命令启动所有节点：

```bash
ros2 launch my_robot_bringup robot_launch.py
```

期望看到 3 个节点的日志输出混在一起（可添加参数过滤）：

```
[hello_pub] Publishing: 'Hello ROS2! Count: 0'
[hello_sub] I heard: 'Hello ROS2! Count: 0'
[param_node] robot=cool_bot, max_speed=3.0, enable_log=true
...
```

### Step 7: Launch 的高级调试

```bash
# 启动时传入自定义参数覆盖
ros2 launch my_robot_bringup robot_launch.py max_speed:=10.0

# 查看 launch 文件预览（不真正启动）
ros2 launch my_robot_bringup robot_launch.py --show-args
```

### Step 8: Launch 文件中的参数覆盖

你也可以在 launch 文件中直接设置参数（不依赖 YAML）：

```python
param_node = Node(
    package='my_robot_cpp',
    executable='param_node',
    name='param_node',
    output='screen',
    parameters=[{
        'robot_name': 'launch_bot',
        'max_speed': 5.0,
    }],
)
```

## 项目当前状态

```
ros2_ws/src/
├── my_robot_cpp/            # 功能节点包
│   ├── CMakeLists.txt
│   ├── package.xml
│   ├── src/
│   │   ├── hello_node.cpp
│   │   ├── hello_pub.cpp
│   │   ├── hello_sub.cpp
│   │   ├── param_node.cpp
│   │   ├── add_two_ints_server.cpp
│   │   ├── add_two_ints_client.cpp
│   │   └── motor_test.cpp
│   └── include/my_robot_cpp/
├── my_robot_interfaces/     # 自定义接口包
│   ├── CMakeLists.txt
│   ├── package.xml
│   ├── msg/MotorCommand.msg
│   └── srv/SetSpeed.srv
└── my_robot_bringup/        # Launch 和 Config 包
    ├── CMakeLists.txt
    ├── package.xml
    ├── launch/robot_launch.py
    └── config/robot_params.yaml
```

## 验证清单

- [ ] `ros2 launch my_robot_bringup robot_launch.py` 一次性启动 3 个节点
- [ ] param_node 使用的是 YAML 文件中的参数值
- [ ] Ctrl+C 能同时关闭所有节点
- [ ] 理解 `exec_depend` 的作用

## 完成后

进入 [Phase 8: cmd_vel_filter_node](../phase8-cmd-vel-filter/GUIDE.md)
