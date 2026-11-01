# Docker 基础

## 概念

- **镜像**：系统用户空间、软件和依赖的模板。
- **容器**：由镜像创建的运行环境，共享宿主机内核。
- **进入容器**：在容器内打开终端，使用其中的软件和文件环境。
- **目录挂载**：宿主机和容器读写同一目录，删除容器后文件仍保留。

## 服务与权限

以下命令均在宿主机执行。

```bash
sudo docker version                      # 客户端和服务端版本
systemctl is-active docker               # 服务状态
sudo systemctl enable --now docker       # 启动并设置开机启动
sudo docker run --rm hello-world          # 运行测试，结束后删除容器
```

免 sudo（可选）：

```bash
sudo usermod -aG docker "$USER"
```

注销桌面、重新登录并重启相关应用后生效。Docker 组具有接近 root 的权限。

## 下载代理

编辑 `/etc/docker/daemon.json`，与已有配置合并：

```json
{
  "proxies": {
    "http-proxy": "http://127.0.0.1:7897",
    "https-proxy": "http://127.0.0.1:7897",
    "no-proxy": "localhost,127.0.0.1,::1"
  }
}
```

```bash
sudo nano /etc/docker/daemon.json
sudo systemctl restart docker
```

- 换节点但端口不变：无需修改或重启。
- 地址、端口变化：修改两处代理地址后重启。
- 停用代理：删除 `proxies` 字段后重启；无其他配置时保留 `{}`。
- 下载时保持代理运行；重启 Docker 可能影响运行中的容器。
- APT 软件源和 Docker 镜像仓库是不同渠道，换 APT 源不加速 `docker pull`。

## 常用命令

将 `IMAGE:TAG`、`NAME`、`/host/path` 替换为实际值。

```bash
sudo docker pull IMAGE:TAG                # 下载镜像
sudo docker image ls                      # 查看镜像
sudo docker run -it --name NAME IMAGE:TAG bash  # 创建并进入容器
sudo docker ps                            # 运行中的容器
sudo docker ps -a                         # 全部容器
sudo docker start -ai NAME                 # 启动已停止容器，进入主终端
sudo docker exec -it NAME bash             # 在运行中容器里另开终端
sudo docker exec -u 0 -it NAME bash        # 以容器 root 打开终端
sudo docker stop NAME                     # 停止容器
sudo docker logs --tail 50 NAME            # 主进程日志，不含所有 exec 输出
sudo docker inspect NAME                  # 配置、状态和挂载
sudo docker stats                         # 资源占用，Ctrl+C 退出查看
sudo docker rm NAME                       # 删除已停止容器及其可写层
sudo docker image rm IMAGE:TAG             # 删除未被容器引用的镜像
```

`run` 创建新容器；`start` 启动已有容器；`exec` 在运行中的容器内执行新命令。不要重复创建同名容器。

## run 参数

| 参数 | 含义 |
| --- | --- |
| `-it` | 交互式终端 |
| `--name NAME` | 容器名称 |
| `--init` | 处理信号、回收子进程 |
| `--network host` | 共享宿主机网络 |
| `--user "$(id -u):$(id -g)"` | 使用宿主机 UID/GID，避免共享文件归 root |
| `-e KEY=VALUE` | 设置容器环境变量 |
| `-v /host/path:/container/path` | 绑定挂载目录，默认可读写 |
| `-w /container/path` | 默认工作目录 |
| `--rm` | 退出后自动删除容器，适合临时任务 |

行末 `\` 表示续行；`$HOME`、`$(id -u)` 等在宿主机展开。

## 模板与配置文件

Dockerfile、Compose 使用标准格式，可复制模板后修改。辅助脚本和环境变量名由项目自定义。

| 文件 | 用途 | 来源 |
| --- | --- | --- |
| `Dockerfile` | 基础镜像、用户、软件和默认命令 | 手写，可复用 |
| `compose.yaml` | 容器名称、网络、挂载和环境变量 | 手写，可复用 |
| `compose.*.yaml` | 图形、GPU 等可选启动配置 | 手写，可复用 |
| `.env` | 本机路径、UID/GID、代理等参数 | 从模板复制后手动填写 |
| `.env.example` | 环境变量模板 | 手写 |
| `.dockerignore` | 排除构建不需要的文件 | 手写 |
| `.gitignore` | 排除不应提交 Git 的文件 | 手写 |
| 启动脚本、Shell 配置 | 封装命令、加载环境 | 手写，非 Docker 必需 |
| `.runtime/` | 本项目脚本生成的临时认证文件 | 自动生成，通常不改 |

重点关注：**Dockerfile 决定镜像里有什么，Compose 决定容器怎么运行，.env 填本机参数。** 修改配置文件不会自动改变已有容器。

流程：选基础镜像 → 写 Dockerfile → 构建镜像 → 配置 Compose → 创建容器 → 验证。

```bash
# 通用 Compose 项目，在配置目录执行
docker compose build       # 构建镜像
docker compose up -d       # 创建/启动；配置或镜像变化时可能重建
docker compose exec SERVICE bash  # 进入，SERVICE 为服务名
docker compose stop        # 停止，保留容器
docker compose down        # 删除容器和项目网络，绑定挂载的主机文件保留
```

`up` 替代手写长串 `docker run` 参数；具体项目可能还需指定附加配置或使用配套脚本。

## 退出与保存

- 主 Bash 中 `exit`：容器通常停止；额外 exec 终端中 `exit`：仅关闭该终端。
- 附着主终端时依次按 `Ctrl+P`、`Ctrl+Q`：脱离，容器继续运行。
- `Exited (0)` 通常表示正常结束。
- 停止、重启同一容器：文件和已安装软件保留。
- 删除容器：可写层丢失，宿主机挂载目录保留。
- 依赖安装步骤写入 Dockerfile，可重新构建；临时安装不会修改原镜像。

`I have no name!` 或找不到组名：数字 UID/GID 在容器用户表中没有对应名称。可通过 Dockerfile 创建对应用户；不代表程序启动失败。

参考：[安装](https://docs.docker.com/engine/install/ubuntu/)、[代理](https://docs.docker.com/engine/daemon/proxy/)、[目录挂载](https://docs.docker.com/engine/storage/bind-mounts/)。
