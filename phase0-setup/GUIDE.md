# Phase 0: ROS2 环境安装

## 学习目标

在你的 Ubuntu 24.04 (arm64) 上安装 ROS2 Jazzy 及全套开发工具，并能运行 `ros2 --version`。

## 背景知识

- **ROS2 发行版**: 每2年发布一次，和 Ubuntu 版本绑定。Ubuntu 24.04 → ROS2 Jazzy Jalisco。
- **ros-jazzy-desktop**: 包含 ROS2 核心 + RViz + rqt 等可视化工具的完整安装包。
- **colcon**: ROS2 的编译构建工具（类比 CMake 之上的一层）。
- **rosdep**: 自动安装系统依赖的工具。

## 操作步骤

### Step 1: 启用 Universe 仓库

```bash
sudo apt update
sudo apt install -y curl software-properties-common
sudo add-apt-repository universe -y
```

### Step 2: 添加 ROS2 官方 apt 源

```bash
# 下载 ROS2 GPG 密钥
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

# 添加 apt 源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

> 解释：`$(dpkg --print-architecture)` 在你的机器上会输出 `arm64`，`$(. /etc/os-release && echo $UBUNTU_CODENAME)` 输出 `noble`。

### Step 3: 安装 ROS2

```bash
sudo apt update
sudo apt install -y ros-jazzy-desktop
```

这个包大约 **2-3GB**，包含 ROS2 核心库、CLI 工具、RViz、rqt 等。

### Step 4: 安装开发工具

```bash
sudo apt install -y ros-dev-tools
```

这会安装 `colcon`、`rosdep`、`vcstool` 等开发必备工具。

### Step 5: 配置环境变量

ROS2 不会自动加入 PATH，你需要每次打开终端时 source 一下：

```bash
# 临时生效（每次新终端都要执行）
source /opt/ros/jazzy/setup.bash
```

建议写入 bashrc，避免每次手动 source：

```bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
echo "source /usr/share/colcon_argcomplete/hook/colcon-argcomplete.bash" >> ~/.bashrc
source ~/.bashrc
```

## 验证

```bash
# 1. 检查 ROS2 版本
ros2 --version
# 期望输出: 类似 "ROS 2 Jazzy Jalisco ..."

# 2. 检查 ROS2 环境变量
echo $ROS_DISTRO
# 期望输出: jazzy

# 3. 检查 colcon
colcon --version
# 期望有版本号输出

# 4. 试运行一个 demo（ROS2 自带的 talker/listener）
# 终端1:
ros2 run demo_nodes_cpp talker
# 终端2:
ros2 run demo_nodes_cpp listener
```

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `ros2: command not found` | 没 source setup.bash | `source /opt/ros/jazzy/setup.bash` |
| `Unable to locate package ros-jazzy-desktop` | apt 源没配好 | 重新执行 Step 2 |
| GPG key 报错 | 密钥下载失败 | 检查网络，重新执行 Step 2 的 curl 命令 |

## 完成后

进入 [Phase 1: Workspace + Package](../phase1-workspace/GUIDE.md)
