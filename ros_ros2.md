# Docker 中配置 ROS / ROS 2

宿主机：Ubuntu 24.04 + zsh；容器内：Bash。Docker 操作见 [docker.md](docker.md)。

| 环境 | 镜像 | 容器 | 宿主机目录 |
| --- | --- | --- | --- |
| ROS 1 Noetic | `local/ros:noetic-dev`（基于 OSRF） | `ros` | `~/ros-workspaces/noetic` |
| ROS 2 Jazzy | `osrf/ros:jazzy-desktop` | `ros2` | `~/ros-workspaces/jazzy` |

Noetic 容器使用 Ubuntu 20.04 用户空间，可运行在 24.04 主机上；Noetic 已结束上游支持。两套环境独立，不自动互通，不共用编译输出。

## 1. 准备（宿主机）

```bash
sudo docker pull osrf/ros:noetic-desktop-full
sudo docker pull osrf/ros:jazzy-desktop

mkdir -p ~/ros-workspaces/{noetic,jazzy}/src
mkdir -p ~/ros-workspaces/{noetic,jazzy}/.home
```

`Pull complete` 仅表示单层完成；出现最终 `Status` 并返回提示符，才表示整次下载结束。

## 2. ROS 1 开发环境

配置位于 `~/ros-docker/noetic`，详见 [使用说明](../../ros-docker/noetic/README.md)。容器仍叫 `ros`。

### 配置文件

以下文件位于 `~/ros-docker/noetic`，均可作为模板复用：

| 文件 | 用途／需要修改的内容 |
| --- | --- |
| `Dockerfile` | 用户、开发工具、长期依赖 |
| `compose.yaml` | 容器名 `ros`、网络、共享目录 |
| `compose.gui.yaml` | X11 显示和 CPU 软件渲染 |
| `compose.gpu.yaml` | NVIDIA GPU 参数，需先安装 Container Toolkit |
| `.env` / `.env.example` | 本机配置／模板：UID、GID、工作目录、代理 |
| `ros-shell.bash` | 自动加载 Noetic 和工作空间 |
| `ros1` | 自定义快捷脚本，不是 Docker 内置命令 |
| `.dockerignore` / `.gitignore` | 排除构建文件／Git 文件 |
| `README.md` | 使用说明 |

`.runtime/` 由脚本生成，不手动维护。工作空间的 `build/`、`devel/` 由 catkin 生成，不直接修改。源码放在 `src/`。

### 首次构建与创建

宿主机执行：

```bash
cd ~/ros-docker/noetic
./ros1 build    # 根据 Dockerfile 构建开发镜像
./ros1 gui      # 创建/更新容器，启用 X11 软件渲染
./ros1 shell    # 进入，自动加载 ROS 和已编译工作空间
```

本机已完成创建，无需重复执行。换电脑时，复制 `.env.example` 为 `.env`，填写本机参数，再构建。只需命令行时用 `./ros1 up` 代替 `./ros1 gui`。

镜像包含 ros 用户、nano、Git、GDB、clangd、catkin tools 和 Python 工具。用户 UID/GID 与宿主机一致。

容器内：

```bash
cd /workspace
catkin_make -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
source devel/setup.bash
rosversion -d  # noetic
roscore
```

另开宿主机终端，`docker exec -it ros bash`，容器内运行 `rosnode list`，应包含 `/rosout`。

宿主机 `~/ros-workspaces/noetic` 对应 `/workspace`；其中 `.home` 对应 `/home/ros`，保存配置和日志。

`./ros1 gui` 启用 X11 软件渲染；安装 NVIDIA Container Toolkit 后可使用 `./ros1 gpu`。切换模式可能重建容器，先结束容器内任务；长期依赖写入 Dockerfile。

## 3. 创建 ROS 2（宿主机，仅首次）

```bash
sudo docker run -it --name ros2 \
  --init \
  --network host \
  --user "$(id -u):$(id -g)" \
  -e HOME=/workspace/.home \
  -v "$HOME/ros-workspaces/jazzy:/workspace" \
  -w /workspace \
  osrf/ros:jazzy-desktop \
  bash
```

在容器内验证：

```bash
echo "$ROS_DISTRO"   # 应输出 jazzy
ros2 --help
```

ROS 2 不需要 `roscore`。

## 4. 后续进入（宿主机）

日常使用已有容器，无需重新构建：

```bash
docker start ros           # 启动已停止的容器，保留现有图形配置
docker exec -it ros bash   # 进入；exit 只退出该终端
docker stop ros            # 停止
```

ROS 2 原始镜像：

```bash
sudo docker start -ai ros2 # 已停止时使用
# 已运行时另开终端
sudo docker exec -it ros2 bash -c \
  'source /opt/ros/jazzy/setup.bash; exec bash'
```

ROS 1 开发镜像已自动加载环境；原始 ROS 2 镜像的新 exec 终端仍需 source。容器内使用 `setup.bash`。

## 5. 安装依赖

容器运行时，从宿主机打开容器 root 终端：

```bash
sudo docker exec -u 0 -it ros bash
# ROS 2 则将名称改为 ros2
```

然后在容器内执行 `apt-get update` 和所需的 `apt-get install ...`。依赖装在容器内，不影响宿主机；后续应写入 Dockerfile。

长期依赖（如新增软件包）加入 Dockerfile 的安装列表后，在宿主机执行：

```bash
cd ~/ros-docker/noetic
./ros1 build
./ros1 gui
```

这会更新镜像并可能重建容器。先结束容器内任务；临时安装的软件可能丢失，宿主机代码和 `.home` 保留。重新登录桌面后运行 `./ros1 gui` 可刷新 X11 认证。

ROS 1 显示配置见开发环境使用说明。ROS 2 尚为基础命令行环境。传感器需单独开放设备或网络访问。

参考：[OSRF 镜像](https://github.com/osrf/docker_images)。
