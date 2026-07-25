服务端部署 (frps)
1. 准备配置文件
在宿主机上创建配置目录，并编写 frps.toml 配置文件。

Bash
mkdir -p /etc/frp
cat <<EOF> /etc/frp/frps.toml
bindPort = 7000
auth.method = "token"
auth.token = "your_secure_token"

# web 面板（可选）
webServer.port = 7500
webServer.user = "admin"
webServer.password = "admin"
EOF
2. 启动 frps 容器
运行以下 Podman 命令启动服务端。如果你在配置文件中开启了 Web 面板或特定的端口，请确保通过 -p 参数将其映射到宿主机。

Bash
podman run -d \\
  --name frps \\
  --restart on-failure \\
  -p 7000:7000 \\
  -p 7500:7500 \\
  -v /etc/frp/frps.toml:/etc/frp/frps.toml:ro \\
  <YOUR_DOCKERHUB_USERNAME>/frps:latest
客户端部署 (frpc)
1. 准备配置文件
在需要内网穿透的机器上准备 frpc.toml 文件。

Bash
mkdir -p /etc/frp
cat <<EOF> /etc/frp/frpc.toml
serverAddr = "x.x.x.x"
serverPort = 7000
auth.method = "token"
auth.token = "your_secure_token"

[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
EOF
2. 启动 frpc 容器
客户端通常推荐使用 --network host 模式，这样可以更方便地代理宿主机上运行的其他服务（如 127.0.0.1 的端口），避免容器网络隔离带来的端口转发问题。

Bash
podman run -d \\
  --name frpc \\
  --network host \\
  --restart on-failure \\
  -v /etc/frp/frpc.toml:/etc/frp/frpc.toml:ro \\
  <YOUR_DOCKERHUB_USERNAME>/frpc:latest
配置 Systemd 守护进程 (推荐)
在 Podman 环境中，推荐将容器转换为 Systemd 服务进行系统级管理，确保开机自启和更稳定的运行环境。

以 frpc 为例，生成并启用 systemd 服务：

Bash
# 生成 systemd 配置文件
cd /etc/systemd/system/
podman generate systemd --new --name frpc -f

# 重新加载 systemd 守护进程
systemctl daemon-reload

# 启用并启动服务
systemctl enable --now container-frpc.service

# 查看服务状态
systemctl status container-frpc.service
镜像更新机制
本项目配置了 GitHub Actions 定时任务（每 7 天）。即使 FRP 官方没有发布新版本，也会自动拉取最新的 alpine 基础镜像重新构建，并覆盖 Docker Hub 上的相同标签。

要更新宿主机上的容器，只需执行：

Bash
# 拉取最新镜像
podman pull <YOUR_DOCKERHUB_USERNAME>/frpc:latest

# 如果你是用 Podman CLI 启动的：
podman stop frpc
podman rm frpc
# 重新执行 run 命令...

# 如果你是用 Systemd 管理的，只需重启服务即可（如果你在 service 文件中配置了自动拉取）：
systemctl restart container-frpc.service