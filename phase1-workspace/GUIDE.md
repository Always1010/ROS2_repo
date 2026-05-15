# Phase 1: Workspace 初始化 + 第一个 Package

## 学习目标

- 理解 ROS2 工作空间结构 (`src/`, `build/`, `install/`, `log/`)
- 掌握 `colcon build` 构建流程
- 理解 `package.xml` 和 `CMakeLists.txt` 的基本结构

## 背景知识

### ROS2 工作空间

```
ros2_ws/
├── src/        ← 你写代码的地方（所有 package 放这里）
├── build/      ← colcon build 生成的中间文件（不用手动改）
├── install/    ← colcon build 生成的最终可运行产物
└── log/        ← 构建日志
```

`build/`, `install/`, `log/` 都是 `colcon build` 自动生成的，你只需要关心 `src/`。

### Package 是什么？

package 是 ROS2 的最小组织单元。每个 package 有自己的 `package.xml`（元信息）和 `CMakeLists.txt`（编译规则）。一个 workspace 可以有多个 package。

### ament_cmake vs ament_python

- C++ 包用 `ament_cmake`（本课程全用这个）
- Python 包用 `ament_python`

## 操作步骤

### Step 1: 创建工作空间

```bash
cd ~/wrk/ROS2
mkdir -p ros2_ws/src
```

### Step 2: 创建 C++ Package

```bash
cd ~/wrk/ROS2/ros2_ws/src
ros2 pkg create my_robot_cpp --build-type ament_cmake --dependencies rclcpp
```

这会生成以下结构：

```
my_robot_cpp/
├── CMakeLists.txt       # 编译规则
├── package.xml          # 包元信息
├── include/             # 头文件（自动创建的目录结构）
│   └── my_robot_cpp/
└── src/                 # 源文件目录（需要创建）
```

### Step 3: 查看生成的文件

用编辑器打开 `my_robot_cpp/package.xml`，看看里面有什么：

```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>my_robot_cpp</name>
  <version>0.0.0</version>
  <description>TODO</description>
  <maintainer email="you@email.com">your_name</maintainer>
  <license>TODO</license>

  <buildtool_depend>ament_cmake</buildtool_depend>
  <depend>rclcpp</depend>

  <test_depend>ament_lint_auto</test_depend>
  <test_depend>ament_lint_common</test_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

再看 `CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_robot_cpp)

# 默认会检查编译警告
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)  # 因为我们 --dependencies rclcpp

# 后续添加 executable 和 install 规则的位置

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
```

关键理解：
- `find_package(rclcpp REQUIRED)` — 引入 rclcpp 库
- `ament_package()` — 必须放在**最后一行**

### Step 4: 首次构建

```bash
cd ~/wrk/ROS2/ros2_ws

# 只编译 my_robot_cpp 这个包（加快速度）
colcon build --packages-select my_robot_cpp
```

构建成功后你会看到：
```
Starting >>> my_robot_cpp
Finished <<< my_robot_cpp [1.23s]

Summary: 1 package finished [1.45s]
```

### Step 5: Source 工作空间

构建完成后，需要 source 工作空间的 setup 文件：

```bash
source install/setup.bash
```

这样 `ros2` 命令才能找到你写的包。

> 提示：以后可以加到 ~/.bashrc：
> ```bash
> echo "source ~/wrk/ROS2/ros2_ws/install/setup.bash" >> ~/.bashrc
> ```

### Step 6: 验证包可被发现

```bash
ros2 pkg list | grep my_robot_cpp
# 期望输出: my_robot_cpp

ros2 pkg executables my_robot_cpp
# 目前没有输出（还没写节点）
```

## 理解 source 的叠加机制

ROS2 采用 **overlay / underlay** 模式：

```
你的 ros2_ws/install (overlay — 上层)
        ↓ 叠加在
/opt/ros/jazzy/    (underlay — 底层)
```

source 你工作空间的 setup.bash 后，你的包会"覆盖"底层同名的包。正常开发时，先 source `/opt/ros/jazzy/setup.bash`，再 source 你自己的 `install/setup.bash`。

## 验证清单

- [ ] `ros2 pkg list | grep my_robot_cpp` 能找到你的包
- [ ] `ls ros2_ws/build/` 不为空
- [ ] `ls ros2_ws/install/` 不为空
- [ ] 理解 `src/` vs `build/` vs `install/` 的关系

## 完成后

进入 [Phase 2: 第一个 rclcpp Node](../phase2-first-node/GUIDE.md)
