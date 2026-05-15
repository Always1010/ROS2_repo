# Phase 10: URDF + robot_state_publisher + RViz

## 学习目标

- 理解 URDF (Unified Robot Description Format) 的概念
- 编写差速小车的 URDF 模型
- 使用 `robot_state_publisher` 将 URDF 发布为 TF
- 在 RViz 中可视化机器人 3D 模型

## 背景知识

### 什么是 URDF？

URDF 是描述机器人**物理结构**的 XML 格式文件。包含：
- **link** (连杆) — 机器人的刚性部件（如车身、轮子）
- **joint** (关节) — link 之间的连接关系（如轮子绕轴旋转）

### robot_state_publisher

`robot_state_publisher` 是一个标准 ROS2 节点，作用：
```
URDF 文件 + /joint_states (关节角度)
        ↓
    发布所有 frame 之间的 TF 变换
```

这样 RViz 就能根据 TF 显示整个机器人的 3D 模型了。

## 操作步骤

### Step 1: 创建 description 包

```bash
cd ~/wrk/ROS2/ros2_ws/src
ros2 pkg create my_robot_description --build-type ament_cmake
cd my_robot_description

# 创建 urdf, rviz, launch 目录
mkdir urdf rviz launch
```

### Step 2: 编写 URDF 文件

创建 `urdf/my_robot.urdf`：

```xml
<?xml version="1.0"?>
<robot name="my_robot">

  <!-- ===== 材料定义 ===== -->
  <material name="black">
    <color rgba="0.0 0.0 0.0 1.0"/>
  </material>
  <material name="blue">
    <color rgba="0.0 0.0 0.8 1.0"/>
  </material>
  <material name="gray">
    <color rgba="0.6 0.6 0.6 1.0"/>
  </material>

  <!-- ===== 车身 ===== -->
  <link name="base_link">
    <visual>
      <geometry>
        <box size="0.3 0.2 0.1"/>  <!-- 长30cm 宽20cm 高10cm -->
      </geometry>
      <material name="blue"/>
    </visual>
  </link>

  <!-- ===== 左轮 ===== -->
  <link name="left_wheel">
    <visual>
      <geometry>
        <cylinder radius="0.05" length="0.02"/>  <!-- 半径5cm 宽2cm -->
      </geometry>
      <material name="black"/>
    </visual>
  </link>

  <joint name="left_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="left_wheel"/>
    <origin xyz="0.0 0.12 0.0" rpy="1.5708 0 0"/>  <!-- 轮子旋转90度水平放置 -->
    <axis xyz="0 1 0"/>  <!-- 绕Y轴旋转 -->
  </joint>

  <!-- ===== 右轮 ===== -->
  <link name="right_wheel">
    <visual>
      <geometry>
        <cylinder radius="0.05" length="0.02"/>
      </geometry>
      <material name="black"/>
    </visual>
  </link>

  <joint name="right_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="right_wheel"/>
    <origin xyz="0.0 -0.12 0.0" rpy="1.5708 0 0"/>
    <axis xyz="0 1 0"/>
  </joint>

  <!-- ===== 万向轮（前） ===== -->
  <link name="caster_front">
    <visual>
      <geometry>
        <sphere radius="0.025"/>
      </geometry>
      <material name="gray"/>
    </visual>
  </link>

  <joint name="caster_front_joint" type="fixed">
    <parent link="base_link"/>
    <child link="caster_front"/>
    <origin xyz="0.15 0.0 -0.025"/>
  </joint>

</robot>
```

**urdf 语法快速参考**：

| 元素 | 含义 |
|------|------|
| `<link>` | 连杆，机器人的一个部件 |
| `<joint>` | 关节，连接两个 link |
| `<visual>` | 可视化几何形状 |
| `<origin>` | 位置偏移 (xyz) + 姿态 (rpy) |
| `<axis>` | 关节旋转轴 |
| joint type=`"fixed"` | 固定不动 |
| joint type=`"continuous"` | 可无限旋转 |
| joint type=`"revolute"` | 有限角度旋转 |

### Step 3: 编写 display launch 文件

创建 `launch/display_robot.launch.py`：

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.substitutions import Command
from ament_index_python.packages import get_package_share_directory
import os


def generate_launch_description():
    # URDF 文件路径
    urdf_path = os.path.join(
        get_package_share_directory('my_robot_description'),
        'urdf',
        'my_robot.urdf'
    )

    with open(urdf_path, 'r') as f:
        robot_desc = f.read()

    # robot_state_publisher: URDF → TF + TF_static
    robot_state_pub = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        name='robot_state_publisher',
        output='screen',
        parameters=[{'robot_description': robot_desc}],
    )

    # joint_state_publisher: 发布假关节角度（方便测试）
    joint_state_pub = Node(
        package='joint_state_publisher',
        executable='joint_state_publisher',
        name='joint_state_publisher',
        output='screen',
    )

    # RViz
    rviz_config = os.path.join(
        get_package_share_directory('my_robot_description'),
        'rviz',
        'display.rviz'
    )
    rviz = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        arguments=['-d', rviz_config],
        output='screen',
    )

    return LaunchDescription([
        robot_state_pub,
        joint_state_pub,
        rviz,
    ])
```

### Step 4: 创建 RViz 配置文件

创建 `rviz/display.rviz`（后面通过 RViz GUI 保存会自动生成完整内容，先写一个最小配置）：

```
Panels:
  - Class: rviz_common/Displays
    Name: Displays
  - Class: rviz_common/Views
    Name: Views
Visualization Manager:
  Displays:
    - Class: rviz_default_plugins/TF
      Name: TF
      Enabled: true
    - Class: rviz_default_plugins/RobotModel
      Name: RobotModel
      Enabled: true
  Global Options:
    Fixed Frame: base_link
  Tools:
    - Class: rviz_default_plugins/MoveCamera
```

### Step 5: 更新 CMakeLists.txt 和 package.xml

**CMakeLists.txt**：
```cmake
cmake_minimum_required(VERSION 3.8)
project(my_robot_description)

find_package(ament_cmake REQUIRED)

install(DIRECTORY
  urdf
  rviz
  launch
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

**package.xml** 添加：
```xml
<exec_depend>robot_state_publisher</exec_depend>
<exec_depend>joint_state_publisher</exec_depend>
<exec_depend>rviz2</exec_depend>
```

### Step 6: 构建 & 运行

```bash
cd ~/wrk/ROS2/ros2_ws
colcon build --packages-select my_robot_description
source install/setup.bash

# 启动 RViz 查看机器人模型
ros2 launch my_robot_description display_robot.launch.py
```

你应该能在 RViz 中看到一个蓝色车身 + 两个黑色轮子的差速小车！

### Step 7: RViz 基本操作

| 操作 | 方法 |
|------|------|
| 旋转视角 | 鼠标左键拖拽 |
| 平移视角 | 鼠标中键拖拽 |
| 缩放 | 滚轮 |
| 聚焦物体 | 在左侧 Displays 面板选中目标 |
| 添加 Display | 左下角 Add 按钮 → 选择类型 |
| 保存配置 | File → Save Config As → 覆盖 `display.rviz` |

保存 RViz 配置后，下次启动直接用你调好的视角。

### Step 8: 整合 odom + URDF（可选挑战）

试试在一个 launch 文件中同时启动：
- `odom_node`（Phase 9 的里程计）
- `robot_state_publisher`（URDF → TF）
- `rviz2`（可视化）

提示：此时 RViz 的 Fixed Frame 应该设为 `odom`（因为 TF 树的根是 odom）。

## 验证清单

- [ ] `ros2 launch my_robot_description display_robot.launch.py` 能打开 RViz
- [ ] RViz 中看到蓝色车身 + 两个黑色轮子 + 一个灰色万向轮
- [ ] 添加 RobotModel display 后看到完整模型
- [ ] TF display 中能看到 base_link → left_wheel, right_wheel 的坐标轴

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| RViz 中看不到模型 | Fixed Frame 不匹配 | Global Options → Fixed Frame 设为 `base_link` |
| RobotModel 灰色不可用 | 没收到 robot_description 参数 | 检查 launch 文件中参数传递 |
| 轮子位置不对 | `<origin>` 的 xyz 或 rpy 不对 | 调整 URDF 中的 origin 值 |
| `joint_state_publisher` 报错 | 需要安装 | `sudo apt install ros-jazzy-joint-state-publisher` |

## 完成后

进入 [Phase 11: Mini Project 整合](../phase11-integration/GUIDE.md)
