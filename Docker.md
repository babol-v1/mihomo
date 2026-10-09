# Ubuntu Docker & 网络服务部署手册

这份手册记录了在 Ubuntu 系统下，围绕 Docker 基础安装、国内镜像源配置、常用维护命令，以及利用 Macvlan 网络部署 AdGuard Home、SmartDNS、Portainer、Sun-Panel 等家庭网络/服务器应用的具体实操流程。

---

## 一、 Docker 基础安装与配置

### 1.1 Docker Engine 干净安装
适用于长期服务器（不推荐 Snap 版）。

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

# 6. 让当前非 root 用户无需 sudo 使用 Docker (以普通用户 babol 为例)
sudo usermod -aG docker babol   # 执行后请重新登录该用户
```

### 1.2 配置国内镜像加速源
为了防止 Docker Hub 拉取镜像慢或超时，建议配置以下依然有效的国内加速站：

```bash
# 1. 直接写入配置文件
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

# 2. 重启服务使配置生效
sudo systemctl daemon-reload
sudo systemctl restart docker

# 3. 验证是否配置成功（检查输出底部的 Registry Mirrors 字段）
docker info
```

---

## 二、 网络基础：创建 Macvlan 网络
为了让每个容器在局域网内拥有**独立的硬件 MAC 地址和专属静态 IP**（避免与宿主机 53 端口等发生冲突），先创建一个 Macvlan 网络（注意根据你的网卡名 `parent=ens33` 和子网段自行调整）：

```bash
docker network create -d macvlan \
  --subnet=192.168.50.0/24 \
  --gateway=192.168.50.1 \
  -o parent=ens33 \
  AGH-LAN
```

---

## 三、 容器应用部署

### 3.1 AdGuard Home (DNS 防护)
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

### 3.2 SmartDNS (DNS 加速优化)
#### 步骤 1：创建并配置 `/etc/smartdns/smartdns.conf`
```ini
bind [::]:53
bind-tcp [::]:53
cache-size 4096
prefetch-domain yes

# 缓存生命周期优化
rr-ttl-min 600
rr-ttl-max 3600
serve-expired yes
serve-expired-ttl 86400

# 测速与双栈优化
response-mode fastest-ip
dualstack-ip-selection yes

# 上游引导与公共 DNS
bootstrap-dns 223.5.5.5
bootstrap-dns 114.114.114.114
server 202.99.224.68
server 202.99.224.67

# 阿里 DNS (DoQ) / 腾讯 DNS (DoH) / 百度及114 (UDP)
server-quic 223.5.5.5:853 -subnet 1.2.5.208.0/24
server-quic 223.6.6.6:853 -subnet 1.2.5.208.0/24
server 119.29.29.29
server 119.28.28.28
server-https https://1.12.12 -host-name doh.pub -subnet 1.2.5.208.0/24
server-https https://120.53.53 -host-name doh.pub -subnet 1.2.5.208.0/24
server 180.76.76.76 -subnet 1.2.5.208.0/24
server 114.114.114.114

# 启用 WebUI 官方仪表盘
plugin smartdns_ui.so
smartdns-ui.ip http://0.0.0
```
#### 步骤 2：启动容器（推荐使用 Docker Compose）
在 `~/smartdns/docker-compose.yml` 中写入：
```yaml
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
```
一键拉起容器：`cd ~/smartdns && docker compose up -d`。

### 3.3 Portainer (Web UI 管理面板)
```bash
# 1. 创建持久化数据卷
sudo docker volume create portainer_data

# 2. 运行 Portainer 容器
sudo docker run -d \
  --name=portainer \
  --restart=always \
  --mac-address 4E:A4:3D:52:C4:2F \
  --network AGH-LAN \
  --ip 192.168.50.71 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest  # 如果想用汉化版可以更换为 6053537/portainer-ce:latest

# 3. 提取首次登录所需的初始化 Token
sudo docker logs portainer
```

### 3.4 Sun-Panel / Ange-Panel (主页导航面板)
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
| **容器控制** | `docker ps` / `docker ps -a` | 查看正在运行的容器 / 查看全部容器（含已停止） |
| | `docker start / stop / restart <名>` | 启动 / 停止 / 重启容器 |
| | `docker rm -f -v <名>` | **强行停止并删除容器**，同时删除关联的匿名卷 |
| **镜像操作** | `docker images` | 列出本地所有镜像 |
| | `docker pull <镜像名>` | 从仓库下载镜像 |
| | `docker rmi <镜像ID>` | 删除本地镜像 |
| **调试排查** | `docker logs -f <名>` | 实时追踪并查看容器输出日志 |
| | `docker exec -it <名> bash` | 进入容器内部交互终端 |
| | `docker inspect <名>` | 查看容器、镜像底层的详细 JSON 信息 |
| **网络查看** | `docker network ls` | 查看所有 Docker 网络 |
| | `docker network inspect <网络名>` | 查看特定网络的详细信息 (包含子网、网关) |
| | `docker inspect --format='{{json .NetworkSettings.Networks}}' <名/ID>` | **精准查看容器的 MAC 地址和 IP 地址** |
| | `docker exec -it <容器名> ip link` | 在容器内部查看网络链路状态 |
| **系统清理** | `docker system prune` | 清理所有已停止的容器、未使用的网络和悬空镜像 |

---

## 五、 服务验证与测试
* **基础状态检查**：`sudo systemctl status docker` 和 `docker compose version`。
* **DNS 服务测试**：在同局域网设备上使用 `nslookup baidu.com 192.168.50.73` 测试 SmartDNS 容器是否能正常响应解析。
