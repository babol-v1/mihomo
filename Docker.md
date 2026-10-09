# Ubuntu Docker & 网络服务高阶部署手册

这份手册记录了在 Ubuntu 系统下，围绕 Docker 基础安装、国内镜像源配置、常用维护命令，以及利用 Macvlan 网络部署 AdGuard Home、SmartDNS、Portainer、Sun-Panel 等家庭网络/服务器应用的完整实操流程。

---

## 一、 Docker 基础安装与配置

### 1.1 Docker Engine 纯净安装
适用于长期服务器使用（强烈建议避开 Snap 版本以获得更好的性能与兼容性）。

```bash
# 1. 卸载可能存在的旧版本
sudo apt remove -y docker.io docker-doc docker-compose podman-docker containerd runc

# 2. 安装基础依赖
sudo apt update && sudo apt install -y ca-certificates curl

# 3. 添加 Docker 官方 GPG 密钥
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://docker.com -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 4. 添加 Docker 官方软件源
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://docker.com \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. 更新索引并安装 Docker 核心组件
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 6. 检查基础状态
sudo systemctl status docker --no-pager
docker compose version

# 7. 让当前非 root 用户无需 sudo 使用 Docker (以普通用户 babol 为例)
sudo usermod -aG docker babol   # 执行后请重新登录或切换该用户生效
```

### 1.2 配置国内镜像加速源
为了防止 Docker Hub 拉取镜像慢或超时，建议配置以下当前依然有效的国内主流镜像加速站：

```bash
# 1. 创建目录并将镜像源写入配置文件
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://xuanyuan.me",
    "https://1ms.run",
    "https://daocloud.io",
    "https://daocloud.io",
    "https://dockerpull.com"
  ]
}
EOF

# 2. 重启 Docker 服务使配置生效
sudo systemctl daemon-reload
sudo systemctl restart docker

# 3. 验证是否添加成功（检查输出底部的 Registry Mirrors 字段）
docker info
```

---

## 二、 网络基础：创建 Macvlan 网络
为了让每个容器在局域网内拥有**独立的硬件 MAC 地址和专属静态 IP**，避免 Docker 默认网桥（docker0）的 DNS 端口与宿主机上的 AdGuard Home 或 SmartDNS（53端口）发生冲突，我们需要先构建一个 Macvlan 网络（注意根据你的宿主机实际网卡名 `parent=ens33` 和子网段自行调整）：

```bash
docker network create -d macvlan \
  --subnet=192.168.50.0/24 \
  --gateway=192.168.50.1 \
  -o parent=ens33 \
  AGH-LAN
```

---

## 三、 容器应用部署实战

### 3.1 AdGuard Home (DNS 防护与过滤)
直接让容器加入已创建的 `AGH-LAN` 网络，并指定未被占用的静态 IP（192.168.50.72）：

```bash
docker run -d \
  --name AdGuard-Home1 \
  --restart unless-stopped \
  --mac-address 4E:A4:3D:52:C4:2E \
  --network AGH-LAN \
  --ip 192.168.50.72 \
  -v /data/adguardhome/work:/opt/adguardhome/work \
  -v /data/adguardhome/conf:/opt/adguardhome/conf \
  adguard/adguardhome:latest
```

### 3.2 SmartDNS (DNS 加速与高并发优化)

#### 步骤 1：一键创建并写入规则配置文件
直接复制并执行下方整段命令，系统会自动创建目录并写入性能最优的并发解析及生命周期缓存规则：

```bash
# 创建配置目录
sudo mkdir -p /etc/smartdns

# 自动写入核心配置文件
sudo tee /etc/smartdns/smartdns.conf <<-'EOF'
# --- 基础设置 ---
bind [::]:53
bind-tcp [::]:53
cache-size 4096
prefetch-domain yes

# --- 缓存生命周期优化（覆盖 TTL 与 24小时乐观缓存生存期） ---
rr-ttl-min 600
rr-ttl-max 3600
serve-expired yes
serve-expired-ttl 86400

# --- 测速与双栈优化 ---
response-mode fastest-ip
dualstack-ip-selection yes

# --- 专属引导 DNS (仅用于解析上游加密 DNS 的域名本身) ---
bootstrap-dns 223.5.5.5
bootstrap-dns 114.114.114.114

# --- 运营商DNS ---
server 202.99.224.68
server 202.99.224.67

# --- 阿里 DNS (DoQ, 支持 ECS) ---
server-quic 223.5.5.5:853 -subnet 1.2.5.208.0/24
server-quic 223.6.6.6:853 -subnet 1.2.5.208.0/24

# --- 腾讯 DNS (DoH, 带域名证书握手, 支持 ECS) ---
server 119.29.29.29
server 119.28.28.28
server-https https://1.12.12 -host-name doh.pub -subnet 1.2.5.208.0/24
server-https https://120.53.53 -host-name doh.pub -subnet 1.2.5.208.0/24

# --- 百度 DNS (UDP, 支持 ECS) ---
server 180.76.76.76 -subnet 1.2.5.208.0/24

# --- 114 DNS (UDP) ---
server 114.114.114.114

# --- 启用 WebUI 官方仪表盘插件 (IPv4 通配监听) ---
plugin smartdns_ui.so
smartdns-ui.ip http://0.0.0
EOF
```

#### 步骤 2：使用 Docker Compose 一键编排并拉起
执行以下整段命令，系统将自动创建 `~/smartdns` 部署目录，生成编排脚本并直接在后台拉起容器（静态 IP 为 192.168.50.73）：

```bash
# 创建并进入专属的部署目录
mkdir -p ~/smartdns && cd ~/smartdns

# 自动生成 docker-compose.yml 编排文件
cat <<-'EOF' > docker-compose.yml
version: '3'

services:
  smartdns:
    image: pymumu/smartdns:latest
    container_name: smartdns
    restart: unless-stopped
    mac_address: "4e:a4:3d:52:c4:2d"
    networks:
      AGH-LAN:
        ipv4_address: 192.168.50.73
    volumes:
      - /etc/smartdns:/etc/smartdns

networks:
  AGH-LAN:
    external: true
EOF

# 一键拉起全新的 SmartDNS 容器
docker compose up -d
```

### 3.3 Portainer (Docker 网页端图形化管理界面)
```bash
# 1. 创建独立的持久化数据卷
sudo docker volume create portainer_data

# 2. 运行 Portainer 容器 (IP 为 192.168.50.71)
sudo docker run -d \
  --name=portainer \
  --restart=always \
  --mac-address 4E:A4:3D:52:C4:2F \
  --network AGH-LAN \
  --ip 192.168.50.71 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest # 提示：如需汉化版可将其变更为 6053537/portainer-ce:latest

# 3. 提取首次登录重置密码所需的后台日志 Token
sudo docker logs portainer
```

### 3.4 Sun-Panel / Ange-Panel (个人主页导航面板)
```bash
docker run -d \
  --name ange-panel \
  --restart=unless-stopped \
  --mac-address 4E:A4:3D:52:C4:2C \
  --network AGH-LAN \
  --ip 192.168.50.74 \
  -v /root/ange-data:/data \
  ghcr.io/liandu2024/ange-panel:latest
```

---

## 四、 Docker 常用运维命令速查表

| 分类 | 常用命令 | 功能描述 |
| :--- | :--- | :--- |
| **容器控制** | `docker ps` | 查看当前正在运行的容器列表 |
| | `docker ps -a` | 查看本地所有容器（包含已停止和报错退出的容器） |
| | `docker stop <容器名/ID>` | 优雅地停止一个正在运行的容器 |
| | `docker restart <容器名/ID>` | 重启指定的容器 |
| | `docker rm -f -v <容器名/ID>` | **强行停止并彻底删除容器**，并同步清理关联的匿名数据卷 |
| **镜像操作** | `docker images` | 列出本地主机上存储的所有镜像 |
| | `docker pull <镜像名>:<标签>` | 从远程镜像源仓库拉取/下载指定的镜像 |
| | `docker rmi <镜像ID>` | 删除本地指定的镜像（需先删除依赖该镜像的容器） |
| **调试排查** | `docker logs -f <容器名/ID>` | 实时追踪并查看容器的流输出日志（排错核心） |
| | `docker exec -it <容器名/ID> bash` | 进入正在运行的容器内部交互终端 |
| | `docker stats` | 实时动态显示容器的 CPU、内存、网络及磁盘 I/O 占用 |
| | `docker inspect <容器名/ID>` | 查看容器、镜像或网络底层的完整 JSON 配置信息 |
| **网络查看** | `docker network ls` | 列出当前 Docker 引擎中存在的所有网络 |
| | `docker network inspect <网络名>` | 查看特定网络（如 AGH-LAN）的详细信息（含子网、网关） |
| | `docker inspect --format='{{json .NetworkSettings.Networks}}' <ID>` | **精准一键提取指定容器的硬件 MAC 地址和 IP 地址** |
| | `docker exec -it <容器名> ip link` | 在容器网络内部直接查看网络链路层状态 |
| **系统清理** | `docker system prune` | 深度清理系统：一键释放所有停止的容器、未使用网络及悬空镜像 |

---

## 五、 部署后续验证

1. **容器运行状态核对**：执行 `docker ps`，确保部署的各个容器其状态列（STATUS）均显示为 `Up` 且没有高频的 `Restarting`。
2. **DNS 功能交叉测试**：在同局域网的任意一台电脑终端（或做过 Macvlan 双向通信的宿主机自身）上，执行以下命令指定 SmartDNS 的 IP 进行解析测试：
   ```bash
   nslookup baidu.com 192.168.50.73
   ```
   若能正确返回百度的解析 IP，说明整套私有化 DNS 与 Macvlan 网络环境全部部署成功。
