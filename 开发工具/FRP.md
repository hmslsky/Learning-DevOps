
# 🌐 FRP 内网穿透系统深度讲解（含 Docker 容器化部署）

---

## 一、概述与价值

### 1.1 简介

**FRP（Fast Reverse Proxy）** 是由开源作者 *fatedier* 开发的一款高性能反向代理工具，主要用于**内网穿透、远程访问与多协议反向代理**。
其核心理念是通过一台拥有公网 IP 的服务器（称为 **frps**）作为中继，将内网主机上的服务（由 **frpc** 客户端注册）转发到公网，从而实现：

> 🌍 "让没有公网 IP 的内网设备，也能被外部安全访问。"

🔹 **核心定位**：

* 内网穿透（NAT / 防火墙后的访问）
* 公网服务反向代理
* 动态隧道转发

🔹 **目标用户群**：

| 用户类型     | 使用目标                      |
| -------- | ------------------------- |
| 个人用户     | 远程访问家中电脑、NAS、路由器或摄像头      |
| 开发者      | 在无公网环境下，向客户/同事演示本地 Web 项目 |
| 企业运维     | 构建安全的跨网访问通道或统一远程接入中心      |
| SaaS 提供商 | 为客户提供自助穿透隧道服务（自建版 ngrok）  |

---

### 1.2 核心优势

FRP 的优势体现在性能、安全性、灵活性和部署生态四个维度：

| 维度             | 说明                                                      |
| -------------- | ------------------------------------------------------- |
| ⚡ **性能**       | 基于 Go 语言实现，支持高并发与长连接复用（multiplexing），大幅降低延迟与资源占用。       |
| 🔐 **安全性**     | 提供 token 验证、TLS 加密、STCP/XTCP 安全通道、ACL 访问控制、身份认证机制。      |
| ⚙️ **灵活性**     | 同时支持 TCP、UDP、HTTP、HTTPS、STCP、XTCP、SUDP 等协议，覆盖绝大多数网络场景。  |
| 🧩 **生态集成**    | 可轻松与 Docker、Kubernetes、Systemd、CI/CD 流程集成，支持自动重连与多隧道配置。 |
| 🛡️ **自控与私有化** | 与 ngrok 等 SaaS 服务不同，FRP 完全自建，数据与权限由用户自行掌控。              |

🔸 **性能对比参考：**

| 工具         | 是否自建 | 协议支持              | 性能   | 安全机制              | 商用可控性 |
| ---------- | ---- | ----------------- | ---- | ----------------- | ----- |
| FRP        | ✅    | TCP/UDP/HTTP/STCP | ⭐⭐⭐⭐ | Token + TLS + ACL | ✅     |
| ngrok (免费) | ❌    | HTTP              | ⭐⭐   | 弱认证               | ❌     |
| Tailscale  | ❌    | VPN (WireGuard)   | ⭐⭐⭐  | 强加密               | 部分受限  |
| Zerotier   | ✅/❌  | 局域网虚拟化            | ⭐⭐⭐⭐ | 控制节点依赖            | 中等    |
| NPS        | ✅    | TCP/HTTP          | ⭐⭐⭐  | 基础认证              | ✅     |

结论：
👉 FRP 是目前 **自建穿透方案的性能与自由度兼优之选**。

FRP 的独特设计理念：

* **客户端 frpc**（位于内网）
* **服务端 frps**（位于公网）

它们通过反向连接建立安全隧道，使得 **公网 -> 内网** 成为可能。

---

### 1.3 典型应用场景

FRP 的应用广泛，覆盖从个人到企业级网络穿透需求：

| 场景                       | 描述                                       | 协议          |
| ------------------------ | ---------------------------------------- | ----------- |
| 🧑‍💻 **远程开发环境访问**       | 在家访问公司内网的 IDE / GitLab / Jenkins         | TCP/HTTP    |
| 🖥️ **远程桌面控制 (RDP/VNC)** | 公网安全访问内网 Windows 桌面                      | STCP / XTCP |
| 🌐 **内网 Web 服务外网暴露**     | 在公网访问内网网站 (localhost:8080 → web.frp.com) | HTTP/HTTPS  |
| 📦 **Docker 容器服务穿透**     | 暴露容器内端口给公网                               | TCP         |
| 🧱 **多云/多节点通信**          | 不同云服务间的安全互联                              | XTCP        |
| 📹 **摄像头 / NAS 外网访问**    | 低功耗设备数据上云                                | UDP/TCP     |

此外，FRP 还常被用于：

* 🌍 内网应用公网演示（如临时网站）
* 🧰 DevOps 架构中的内网代理桥接
* 🚀 边缘设备远程管理（IoT 场景）
* 🕹️ 结合 RustDesk、自建远程桌面穿透系统

---

### 1.4 文档阅读指南

| 章节            | 内容                | 学习收获     |
| ------------- | ----------------- | -------- |
| **一、概述与价值**   | 理解 FRP 的设计理念与应用范围 | 是否适合你的业务 |
| **二、基础环境与配置** | 安装、配置文件结构、系统依赖    | 能够独立部署   |
| **三、核心功能与操作** | 学习最关键的 80% 操作     | 快速上手穿透   |
| **四、进阶应用与案例** | 跨机部署、混合协议、性能优化    | 具备实战能力   |
| **五、总结与资源**   | 后续版本、社区与扩展        | 持续更新与学习  |

---

# ⚙️ 二、基础环境与配置

## 2.1 准备工作

### ✅ 系统要求

| 类型  | 最低要求                    | 推荐配置              |
| --- | ----------------------- | ----------------- |
| OS  | Linux / Windows / macOS | Ubuntu 22.04+     |
| CPU | 1 核                     | 2 核以上             |
| 内存  | 512MB                   | ≥1GB              |
| 网络  | 服务端需公网 IP               | 建议开放 7000/7500 端口 |

### ✅ 网络端口规划（建议）

| 作用             | 默认端口              | 建议   |
| -------------- | ----------------- | ---- |
| frps 控制端口      | 7000              | 固定   |
| Dashboard 管理界面 | 7500              | 可改   |
| 用户服务端口         | 自定义（如 6000, 8080） | 按需配置 |

### ✅ 安装方式

**手动安装**

```bash
wget https://github.com/fatedier/frp/releases/download/v0.60.0/frp_0.60.0_linux_amd64.tar.gz
tar -zxvf frp_0.60.0_linux_amd64.tar.gz
cd frp_0.60.0_linux_amd64
```

**Docker 安装**

```bash
docker pull snowdreamtech/frps
docker pull snowdreamtech/frpc
```

---

## 2.2 界面导览与配置文件结构

FRP 无 GUI，核心由两个可执行文件与 INI 配置文件组成：

```
frp/
├── frps               # 服务端程序
├── frps.ini           # 服务端配置文件
├── frpc               # 客户端程序
├── frpc.ini           # 客户端配置文件
└── systemd/frps.service (可选)
```

📘 **Dashboard 管理界面（可选）**

* 启用后访问：`http://<服务器IP>:7500`
* 提供实时连接状态、在线代理、带宽流量监控等
* 支持用户身份认证（username/password）

### 服务端（frps.ini）配置示例

```ini
[common]
bind_port = 7000
dashboard_port = 7500
dashboard_user = admin
dashboard_pwd = password
token = frp-secret
```

### 客户端（frpc.ini）配置示例

```ini
[common]
server_addr = x.x.x.x
server_port = 7000
token = frp-secret

[web]
type = http
local_port = 80
custom_domains = yourdomain.com
```

---

# 🧰 三、核心功能与操作

## 3.1 功能A：基础 TCP/HTTP 穿透

### ① 服务端配置 `frps.ini`

```ini
[common]
bind_port = 7000
dashboard_port = 7500
dashboard_user = admin
dashboard_pwd = 123456
token = mysecret
log_file = ./frps.log
log_level = info
```

### ② 客户端配置 `frpc.ini`

```ini
[common]
server_addr = 203.0.113.1
server_port = 7000
token = mysecret

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000

[web]
type = http
local_port = 8080
custom_domains = web.myserver.com
```

启动：

```bash
./frps -c frps.ini
./frpc -c frpc.ini
```

然后：

```bash
ssh -p 6000 user@203.0.113.1
```

或访问：

```
http://web.myserver.com
```

---

## 3.2 功能B：高级代理与安全通道

### 🔹 STCP / XTCP

* **STCP**：安全内网互访，仅同 token 的 frpc 之间可连接。
* **XTCP**：点对点直连，延迟更低，但 NAT 要求高。

示例（STCP 双客户端）：

* 机器 A（server）：

```ini
[common]
server_addr = 203.0.113.1
server_port = 7000
token = mysecret

[ssh-server]
type = stcp
sk = mykey
local_ip = 127.0.0.1
local_port = 22
```

* 机器 B（client）：

```ini
[common]
server_addr = 203.0.113.1
server_port = 7000
token = mysecret

[ssh-visitor]
type = stcp
role = visitor
server_name = ssh-server
sk = mykey
bind_addr = 127.0.0.1
bind_port = 6000
```

B 上执行：

```bash
ssh -p 6000 user@127.0.0.1
```

即可访问 A 的 SSH。

### 🔹 高级特性与插件支持

* **STCP（安全内网互连）**

  * 无需公网 IP，通过共享密钥访问
  * 适合点对点通讯、内网联机
* **Dashboard 面板**

  * 访问 `http://公网IP:7500` 查看实时状态
* **插件机制**

  * 支持 HTTP Basic Auth、Header 修改、域名路由
* **UDP 穿透**

  * 游戏服务器、实时通信等 UDP 应用

---

# 🚀 四、进阶应用与案例

## 4.1 企业级典型部署：跨网远程办公中心

目标：
总部服务器部署 frps，分支机构客户端部署 frpc
→ 支持统一访问内网 GitLab、ERP、RDP 服务

部署结构图：

```
员工电脑 → 公网 → frps (云服务器)
                    ↑
        分支1(frpc)    分支2(frpc)
```

**配合 Systemd 自动启动：**

```bash
sudo nano /etc/systemd/system/frps.service
```

```ini
[Unit]
Description=FRP Server
After=network.target

[Service]
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.ini
Restart=always

[Install]
WantedBy=multi-user.target
```

启动：

```bash
systemctl enable frps
systemctl start frps
```

---

## 4.2 Docker 容器化部署 FRP

### ① 准备配置文件

在主机目录 `/opt/frp`：

```bash
mkdir -p /opt/frp/{frps,frpc}
```

**/opt/frp/frps/frps.ini**

```ini
[common]
bind_port = 7000
dashboard_port = 7500
dashboard_user = admin
dashboard_pwd = password
token = frp-token
```

**/opt/frp/frpc/frpc.ini**

```ini
[common]
server_addr = your-server-ip
server_port = 7000
token = frp-token

[web]
type = http
local_port = 80
custom_domains = example.com
```

### ② 启动容器

**frps 服务端**

```bash
docker run -d --name frps \
  -v /opt/frp/frps/frps.ini:/etc/frp/frps.ini \
  -p 7000:7000 -p 7500:7500 \
  snowdreamtech/frps:latest
```

**frpc 客户端**

```bash
docker run -d --name frpc \
  -v /opt/frp/frpc/frpc.ini:/etc/frp/frpc.ini \
  snowdreamtech/frpc:latest
```

> ✅ 优点：
>
> * 容器隔离配置，不影响主机
> * 可通过 `docker logs` 查看状态
> * 轻松用 `docker-compose` 管理多节点部署

---

## 4.3 使用 Docker Compose 管理

**docker-compose.yml**

```yaml
version: "3"
services:
  frps:
    image: snowdreamtech/frps
    container_name: frps
    volumes:
      - ./frps.ini:/etc/frp/frps.ini
    ports:
      - "7000:7000"
      - "7500:7500"

  frpc:
    image: snowdreamtech/frpc
    container_name: frpc
    volumes:
      - ./frpc.ini:/etc/frp/frpc.ini
    depends_on:
      - frps
```

启动：

```bash
docker compose up -d
```

---

## 4.4 性能优化与安全配置

* 启用压缩与加密：

  ```ini
  use_compression = true
  use_encryption = true
  ```
* 限制客户端：

  ```ini
  privilege_mode = true
  ```
* 开启系统优化：

  ```bash
  sysctl -w net.core.rmem_max=52428800
  sysctl -w net.core.wmem_max=52428800
  ```

---

## 4.5 常见问题与优化

| 问题                   | 原因               | 解决方案                                 |
| -------------------- | ---------------- | ------------------------------------ |
| `Connection refused` | 防火墙或端口未放行        | 检查安全组、iptables                       |
| 频繁断连                 | NAT 超时、token 不匹配 | 修改 `heartbeat_timeout`               |
| Dashboard 无法访问       | 端口被占用或未配置用户      | 检查 `dashboard_port`                  |
| CPU 占用高              | 过多长连接            | 调整 `max_pool_count`、`log_level=warn` |
| 需要更高安全性              | 默认传输为明文          | 启用 `tls_enable = true`               |
| Docker 容器重启后配置丢失 | 未挂载配置卷         | 使用 `-v /opt/frp:/etc/frp`            |

---

# 📘 五、总结与资源

## 5.1 总结与未来

FRP 是目前国内外最成熟、稳定的开源内网穿透工具之一。
它以**高性能、灵活架构、完全自控**著称，可从家庭远控延伸到企业混合云架构。

FRP 是一款 **轻量级、高性能、完全可自托管** 的内网穿透解决方案。
它的 Docker 化部署使得多节点快速上线成为可能，是 **自建远程访问系统** 的首选工具。

**学习路径建议：**

1. 📗 基础：理解 TCP/HTTP 穿透；
2. ⚙️ 进阶：掌握 STCP/XTCP 点对点连接；
3. 🧩 集成：结合 Docker、Systemd、RustDesk；
4. ☁️ 企业：集中管理多用户隧道、TLS 加密、ACL 限制。

未来方向：

* 集成 Web 管理界面（如 frp-panel）
* Kubernetes 集群支持
* 自动证书与域名分配

---

## 5.2 学习与参考资源

**推荐资源：**

* 官方文档：[https://gofrp.org/docs/](https://gofrp.org/docs/)
* GitHub 项目：[https://github.com/fatedier/frp](https://github.com/fatedier/frp)
* Docker 镜像：[https://hub.docker.com/r/snowdreamtech/frps](https://hub.docker.com/r/snowdreamtech/frps)
* 国内镜像下载：[https://mirror.ghproxy.com/github.com/fatedier/frp/releases](https://mirror.ghproxy.com/github.com/fatedier/frp/releases)
* 社区管理面板：
  🔗 `https://github.com/Zo3i/frp-admin`

