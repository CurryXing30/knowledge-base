## ✅ Docker 网络配置

### 1. 配置 `daemon.json`（Docker 加速器或官方源）

- 配置文件路径：`/etc/docker/daemon.json`
- **若使用官方 Docker Hub**，建议 **清空所有镜像加速器** 避免出错：

```json
{
  "registry-mirrors": []
}
```

- **若希望使用国内加速器**（不推荐长远依赖），可填写：

```json
{
  "registry-mirrors": [
    "https://mirror.baidubce.com",
    "https://docker.nju.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
```

- 修改完后记得：

```bash
sudo systemctl daemon-reexec
sudo systemctl restart docker
```

------

### 2. 为 Docker 服务单独配置代理（让 daemon 独立走代理）

- 创建配置目录：

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
```

- 创建代理配置文件：

```bash
sudo vim /etc/systemd/system/docker.service.d/http-proxy.conf
```

- 示例内容（根据你的代理替换 IP 和端口）：

```ini
[Service]
Environment="HTTP_PROXY=http://10.216.164.165:10811"
Environment="HTTPS_PROXY=http://10.216.164.165:10811"
Environment="NO_PROXY=localhost,127.0.0.1"
```

- 重载配置并重启 Docker：

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart docker
```

> ✅ 这一步确保 **只有 Docker 服务走代理**，不会影响整个服务器的其它流量。

------

### 3. 配置当前用户使用 Docker 命令权限（免 sudo）

- 把当前用户加入 `docker` 用户组：

```bash
sudo usermod -aG docker $USER
```

- 要让权限生效，必须：
  - **重新登录当前 shell**
  - 或者 **彻底重启服务器**（推荐）

```bash
reboot
```

- 验证是否生效：

```bash
docker run hello-world
```

------

### 4. 补充

#### ✅ 验证 Docker 网络是否通畅

需要先在终端进行代理配置，让服务器走主机的流量代理，export修改下

```bash
curl -I https://registry-1.docker.io/v2/
```

返回 `401 Unauthorized` 是正常的，说明网络可达。

------

#### ✅ 查看 Docker 是否走了代理

```bash
sudo systemctl show --property=Environment docker
```

确认里面有你配置的 `HTTP_PROXY` 和 `HTTPS_PROXY`。

------

#### ✅ 确保 docker.sock 权限正确（如果有异常）

```bash
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
```

------

## ✅ 汇总

| 操作                    | 作用                     | 是否必须 |
| ----------------------- | ------------------------ | -------- |
| 清空/配置 `daemon.json` | 控制是否用加速器         | ✅        |
| 设置 systemd 代理       | 让 Docker 服务单独走代理 | ✅        |
| 用户加入 `docker` 组    | 免 sudo 使用 Docker 命令 | ✅        |
| 重启 shell / 系统       | 权限更新生效             | ✅        |
| 其他网络验证命令        | 排查问题时使用           | ⚠️ 建议   |

------

## ✅ docker登录配置

```
docker login
```

然后根据提示输入：

- 用户名（Docker Hub 或仓库账号）
- 密码或 Access Token（推荐使用 token）

登录后凭据会保存在 `~/.docker/config.json`，Docker 会自动使用。

# 🚀 一键式部署脚本

```
#!/bin/bash
PROXY_URL="http://10.216.164.165:10811"
DOCKER_USER=$(whoami)
DAEMON_JSON="/etc/docker/daemon.json"
PROXY_CONF_DIR="/etc/systemd/system/docker.service.d"
PROXY_CONF_FILE="$PROXY_CONF_DIR/http-proxy.conf"

# =============================

echo "🧱 [1/5] 配置 Docker daemon.json，清空 registry-mirrors..."
sudo mkdir -p $(dirname $DAEMON_JSON)
sudo tee $DAEMON_JSON >/dev/null <<EOF
{
  "registry-mirrors": []
}
EOF

echo "✅ 清空加速器完成"

echo "🌐 [2/5] 配置 Docker daemon 使用代理: $PROXY_URL"
sudo mkdir -p $PROXY_CONF_DIR
sudo tee $PROXY_CONF_FILE >/dev/null <<EOF
[Service]
Environment="HTTP_PROXY=$PROXY_URL"
Environment="HTTPS_PROXY=$PROXY_URL"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF

echo "✅ 设置代理完成"

echo "👤 [3/5] 添加当前用户 '$DOCKER_USER' 到 docker 用户组..."
sudo usermod -aG docker "$DOCKER_USER"
echo "✅ 添加成功（重启系统后生效）"

echo "🔄 [4/5] 重载 systemd 并重启 Docker 服务..."
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart docker
echo "✅ Docker 服务重启完成"

echo "🔍 [5/5] 状态检查:"
echo "当前用户所属组："
groups
echo ""
echo "Docker 代理配置："
sudo systemctl show --property=Environment docker

echo ""
echo "📌 请重新登录或执行 'reboot' 使 docker 用户组权限生效"
```

1. 将上述内容保存为 `setup_docker_network.sh` 文件：

```
vim setup_docker_network.sh
```

2. 赋予执行权限：

```
chmod +x setup_docker_network.sh
```

​	3. 运行脚本：

```
./setup_docker_network.sh
```