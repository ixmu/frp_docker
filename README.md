### 1. 准备配置文件

在宿主机上创建配置目录，并编写 `frps.toml` 配置文件。

```
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
```

### 2. 启动 frps 容器

运行以下 Podman 命令启动服务端。如果你在配置文件中开启了 Web 面板或特定的端口，请确保通过 `-p` 参数将其映射到宿主机。

```
podman run -d \
  --name frps \
  --restart on-failure \
  -p 7000:7000 \
  -p 7500:7500 \
  -v /etc/frp/frps.toml:/etc/frp/frps.toml:ro \
  docker.io/xlousp/frps:latest
```

## 客户端部署 (frpc)

### 1. 准备配置文件

在需要内网穿透的机器上准备 `frpc.toml` 文件。

```
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
```

### 2. 启动 frpc 容器

客户端通常推荐使用 `--network host` 模式，这样可以更方便地代理宿主机上运行的其他服务（如 `127.0.0.1` 的端口），避免容器网络隔离带来的端口转发问题。

```
podman run -d \
  --name frpc \
  --network host \
  --restart on-failure \
  -v /etc/frp/frpc.toml:/etc/frp/frpc.toml:ro \
  docker.io/xlousp/frpc:latest
```
