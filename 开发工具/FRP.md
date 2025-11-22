
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

**INI格式（经典）**：
```ini
[common]
bind_port = 7000
dashboard_port = 7500
dashboard_user = admin
dashboard_pwd = password
token = frp-secret
```

**TOML格式（推荐用于新版frp）**：
```toml
[common]
bindPort = 7000
dashboardPort = 7500
dashboardUser = "admin"
dashboardPwd = "password"
token = "frp-secret"
```

### 客户端（frpc.ini）配置示例

**INI格式（经典）**：
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

**TOML格式（推荐用于新版frp）**：
```toml
[common]
serverAddr = "x.x.x.x"
serverPort = 7000
token = "frp-secret"

[[proxies]]
name = "web"
type = "http"
localPort = 80
customDomains = ["yourdomain.com"]
```

---

# 🧰 三、核心功能与操作

## 3.1 功能A：基础 TCP/HTTP 穿透

### ① 服务端配置 `frps.ini`

**INI格式**：
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

**TOML格式**：
```toml
[common]
bindPort = 7000
dashboardPort = 7500
dashboardUser = "admin"
dashboardPwd = "123456"
token = "mysecret"
logFile = "./frps.log"
logLevel = "info"
```

### ② 客户端配置 `frpc.ini`

**INI格式**：
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

**TOML格式**：
```toml
[common]
serverAddr = "203.0.113.1"
serverPort = 7000
token = "mysecret"

[[proxies]]
name = "ssh"
type = "tcp"
localAddr = "127.0.0.1"
localPort = 22
remotePort = 6000

[[proxies]]
name = "web"
type = "http"
localPort = 8080
customDomains = ["web.myserver.com"]
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

**INI格式**：
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

**TOML格式**：
```toml
[common]
serverAddr = "203.0.113.1"
serverPort = 7000
token = "mysecret"

[[proxies]]
name = "ssh-server"
type = "stcp"
sk = "mykey"
localAddr = "127.0.0.1"
localPort = 22
```

* 机器 B（client）：

**INI格式**：
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

**TOML格式**：
```toml
[common]
serverAddr = "203.0.113.1"
serverPort = 7000
token = "mysecret"

[[proxies]]
name = "ssh-visitor"
type = "stcp"
role = "visitor"
serverName = "ssh-server"
sk = "mykey"
bindAddr = "127.0.0.1"
bindPort = 6000
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

**/opt/frp/frps/frps.toml**（推荐）

**TOML格式**：
```toml
[common]
bindPort = 7000
dashboardPort = 7500
dashboardUser = "admin"
dashboardPwd = "password"
token = "frp-token"
```

**INI格式**（兼容旧版）：
```ini
[common]
bind_port = 7000
dashboard_port = 7500
dashboard_user = admin
dashboard_pwd = password
token = frp-token
```

**/opt/frp/frpc/frpc.toml**（推荐）

**TOML格式**：
```toml
[common]
serverAddr = "your-server-ip"
serverPort = 7000
token = "frp-token"

[[proxies]]
name = "web"
type = "http"
localPort = 80
customDomains = ["example.com"]
```

**INI格式**（兼容旧版）：
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
  -v /opt/frp/frps/frps.toml:/etc/frp/frps.toml \
  -p 7000:7000 -p 7500:7500 \
  snowdreamtech/frps:latest -c /etc/frp/frps.toml
```

**frpc 客户端**

```bash
docker run -d --name frpc \
  -v /opt/frp/frpc/frpc.toml:/etc/frp/frpc.toml \
  snowdreamtech/frpc:latest -c /etc/frp/frpc.toml
```

> 💡 注意：
> - 使用 `-c` 参数显式指定配置文件路径
> - 若需使用 INI 格式，请修改挂载路径为 `.ini` 文件并相应调整 `-c` 参数

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

## 4.6 Ollama本地LLM服务穿透实战

下面给你一份**面向实战、可直接复制运行**的详尽指南：如何在客户端用 **frpc** 把本地运行的 **Ollama（本地 LLM 服务，默认监听 `127.0.0.1:11434`）** 安全地穿透到公网。

> 先说明关键事实（已验证）：
>
> * Ollama 默认在 `127.0.0.1:11434` 提供 HTTP REST API（`http://localhost:11434/api/...`）。我们只需把该端口曝光到 frps 所在的公网服务器即可。

### 4.6.1 步骤总览（高层）

1. 在公网服务器上运行并配置好 `frps`（frps 必须可被公网访问；`vhost_http_port`/`vhost_https_port` 配置见第 4 节）。
2. 在本地（Ollama 运行机）安装 `frpc` 并写好 `frpc` 配置文件。
3. 根据需求选择穿透方式（TCP 直接端口 / HTTP 虚拟主机 / HTTPS + BasicAuth / STCP 点对点），启动 `frpc`。
4. 在公网或域名上访问（或用 curl / 客户端应用调用），验证通过。

### 4.6.2 先决条件检查（在本地机器上）

```bash
# 确认 Ollama 服务在本地可访问（在 Ollama 机器上运行）
curl -sS http://127.0.0.1:11434/api  # 或访问你常用的 API 路径
# 如果返回 404/200/类似 JSON，那么 Ollama 本地可用
```

如果 Ollama 不是绑定 `127.0.0.1`，你可以用环境变量修改（见 Ollama 文档），但建议 **让 Ollama 继续只监听本地**，由 frpc 从 localhost 读取并转发（更安全）。

### 4.6.3 可选的穿透方案 —— 优缺点速览

1. **TCP 端口映射（最简单）**

   * 原理：把本地 `127.0.0.1:11434` 映射到 frps 的某个 `remote_port`（例如 `60034`）。
   * 优点：配置最简单、对 Ollama 无需改动（透明传输 HTTP）。
   * 缺点：无法使用域名复用端口（每个 TCP 服务需独占端口）；没有 HTTP 层特殊功能（BasicAuth、路径路由等）。

2. **HTTP 虚拟主机（vhost，推荐用于 Web/API）**

   * 原理：frps 在 `vhost_http_port`（例如 80/7080）监听 HTTP，根据请求 Host 自动转发到对应的 frpc 映射。
   * 优点：可用域名路由（`ollama.example.com`），允许 HTTP BasicAuth、路径限制、X-Forwarded-For 等；可复用同一端口托管多个服务。
   * 缺点：需域名解析到 frps 的公网 IP，并在 frps 侧配置 vhost 端口。

3. **HTTPS（vhost + TLS） + BasicAuth（推荐生产/公网暴露）**

   * 在 HTTP 方案上再加 TLS（在 frps 上配证书或用 reverse-proxy），同时启用 frp 的 HTTP BasicAuth（在 frpc 配置里设置用户名/密码）。这样即便域名被猜到也需要认证。

4. **STCP / XTCP（点对点/私有链接）**

   * 用于更严格的点对点场景或穿透复杂 NAT。配置更加复杂，适合需要更高安全性或点对点低延迟的场景。

### 4.6.4 `frpc` 配置详解（示例 + 字段说明）

> 下面我给出 **三套** 常见、可直接运行的 `frpc` 配置文件（你可以选其中之一，或同时保留多个代理条目）：
>
> * A. **TCP 映射（快速）**
> * B. **HTTP 虚拟主机 + BasicAuth（推荐用于 Ollama API）**
> * C. **HTTPS（通过 frps 的 vhost_https_port）示例 + Proxy Protocol 获取真实 IP**

> 先给出 **INI** 格式（很多人使用），再给出等价的 **TOML** 片段（frp 新版官方文档示例往往用 TOML）。

#### 4.6.4.1 共有 `frpc` `[common]`（必填）

将下面放到 `frpc.ini` 顶部（或 `frpc.toml` 对应字段）：

```ini
[common]
server_addr = 203.0.113.10        # 替换为你的 frps 公网 IP 或域名
server_port = 7000                # frps 的 bind_port
token = your-frp-token            # 如果 frps 配置了 token，请填一致
# 可选：如果你的 frps 要求 TLS/加密层，这里设置 transport.useEncryption（TOML）
# http_proxy = http://proxy:8080   # 如果 frpc 需通过 HTTP 代理访问 frps（可选）
```

#### A — 快速方案：**TCP 端口映射**（最简单）

适合：只想快速把本地 `11434` 暴露为公网端口（比如 `60034`）。

**INI格式**：
```ini
# frpc.ini
[common]
server_addr = 203.0.113.10
server_port = 7000
token = your-frp-token

[ollama-tcp]
type = tcp
local_ip = 127.0.0.1
local_port = 11434
remote_port = 60034
# 可选安全：启用传输加密/压缩（某些 frp 版本使用 transport.* 配置，见 TOML 例子）
# use_encryption = true
# use_compression = true
```

**TOML格式**：
```toml
# frpc.toml
[common]
serverAddr = "203.0.113.10"
serverPort = 7000
token = "your-frp-token"

[[proxies]]
name = "ollama-tcp"
type = "tcp"
localAddr = "127.0.0.1"
localPort = 11434
remotePort = 60034

# 传输层加密与压缩（提高安全，但占用 CPU）
[transport]
useEncryption = true
useCompression = true
```

**访问方法（外部）**：

```bash
# 直接对 frps IP 的 remote_port 发起 HTTP 请求（或让应用指向该 IP:port）
curl http://203.0.113.10:60034/api/models
```

**注意**：若要用域名并复用 80/443，请使用下方的 HTTP vhost 方案。

#### B — 推荐：**HTTP 虚拟主机（vhost） + BasicAuth**（更安全，支持域名）

前提（frps 上）：

* `frps.ini` 中设置了 `vhost_http_port = 80`（或其他端口）并将域名 `ollama.example.com` 的 DNS 指向 frps IP。

`frpc.ini`（INI 格式）：

```ini
[common]
server_addr = frps.example.com           # 或直接用 IP
server_port = 7000
token = your-frp-token

[ollama-http]
type = http
local_ip = 127.0.0.1
local_port = 11434
# 使用域名路由（需要你的域名解析到 frps 的 IP）
custom_domains = ollama.example.com
# 保护接口（HTTP Basic Auth），在访问时浏览器/客户端需要用户名密码
http_user = ollamauser
http_pwd = strongpassword123
# 可选：启用 proxy protocol 版本（若需要让本地服务看到真实客户端 IP）
# proxy_protocol_version = v2
```

**访问方法（外部）**：

```
# 通过域名访问（浏览器或应用）
http://ollama.example.com/api/models
# 若启用了 BasicAuth，浏览器会弹窗或 curl 需要加 -u ollamauser:strongpassword123
curl -u ollamauser:strongpassword123 http://ollama.example.com/api/models
```

**优点**：

* 可复用端口（vhost），适合多个服务共享同一公网端口；
* 支持 BasicAuth，客户端访问更安全；
* frps 可以额外在 vhost 层做 TLS（见 C）。

#### C — 生产级：**HTTPS（vhost_https_port）+ BasicAuth + 强加密**

在 `frps` 服务器上：

* 配置 `vhost_https_port = 443`（或者 7443），并在 `frps` 或前端 Nginx 上配置有效 TLS 证书（Let's Encrypt / 手动证书）。
* 如果 `frps` 本身没有内置证书或你想用 Nginx 反代：把域名 DNS 指向该服务器，用 Nginx 做反向代理到 frps 的 vhost_http_port 或 vhost_https_port。

**INI格式**：
```ini
[common]
server_addr = frps.example.com
server_port = 7000
token = your-frp-token

[ollama-https]
type = https
local_ip = 127.0.0.1
local_port = 11434
custom_domains = ollama.example.com
http_user = ollamauser
http_pwd = strongpassword123
# 可选：启用 proxy protocol 版本
# proxy_protocol_version = v2
```

**TOML格式**：
```toml
[common]
serverAddr = "frps.example.com"
serverPort = 7000
token = "your-frp-token"

[[proxies]]
name = "ollama-https"
type = "https"
localAddr = "127.0.0.1"
localPort = 11434
customDomains = ["ollama.example.com"]
httpUser = "ollamauser"
httpPassword = "strongpassword123"
proxyProtocolVersion = "v2"

# 传输层加密与压缩
[transport]
useEncryption = true
useCompression = true
```

客户端配置基本与HTTP部分相同，但**强烈建议**在 `frpc`/`frps` 之间启用传输加密（transport.useEncryption）来保护隧道内数据。

#### 4.6.4.2 TOML（新式）配置示例（推荐用于新版 frp）

如果你用 `frpc.toml`，配置更结构化且可以设置 transport 子项：

```toml
[common]
serverAddr = "203.0.113.10"
serverPort = 7000
token = "your-frp-token"

[[proxies]]
name = "ollama-http"
type = "http"
localAddr = "127.0.0.1"
localPort = 11434
customDomains = ["ollama.example.com"]
httpUser = "ollamauser"
httpPassword = "strongpassword123"
# 真实客户端 IP 支持（可选）
proxyProtocolVersion = "v2"

# 传输层加密与压缩（提高安全，但占用 CPU）
[transport]
useEncryption = true
useCompression = true
```

**说明**：

* `customDomains` 用于 vhost（域名）路由；若使用 `subdomain` 模式，请在 frps.ini 配置 `subDomainHost` 并在这里设置 `subdomain = "ollama"`。
* `httpUser/httpPassword`（TOML）或 `http_user/http_pwd`（INI）用于 HTTP BasicAuth。

### 4.6.5 安全硬化建议（必须考虑）

1. **永远不要把 Ollama 直接暴露为无认证的公网 HTTP**。至少使用 BasicAuth + HTTPS。
2. **在 frps 上启用 TLS**（`vhost_https_port` + 有效证书），或者把 frps 放在 Nginx 后面由 Nginx 终止 TLS。
3. **frpc 与 frps 之间启用传输加密**（`transport.useEncryption = true` / `use_encryption=true`），以防中间人。
4. **使用 strong token 或 OIDC**（frp 支持 token 与 OIDC 两种认证方式）保证客户端/服务端身份验证。
5. **限制 Dashboard 的访问**（仅内网管理 IP 或放在 VPN 里）。
6. **审计日志与速率限制**：监控 `frps` 日志，必要时在 nginx 层做速率限制 / IP 黑名单。
7. **不要在不受信任的网络上保存明文凭证**（把 `frpc.ini` 权限设为 600）。

### 4.6.6 启动与测试（典型命令）

在 Ollama 机器上先启动 Ollama：

```bash
ollama serve   # 默认 127.0.0.1:11434
```

然后启动 frpc（假设 frpc.ini 在当前目录）：

```bash
# 前台查看日志
frpc -c ./frpc.ini

# 或后台启动（Linux / macOS）
nohup frpc -c ~/frp/frpc.ini > frpc.log 2>&1 &
```

测试（根据方案）：

* TCP 方案：

```bash
curl http://203.0.113.10:60034/api   # public_ip:remote_port
```

* HTTP vhost（带 BasicAuth）：

```bash
curl -u ollamauser:strongpassword123 http://ollama.example.com/api/models
```

* HTTPS（若已配置 TLS）：

```bash
curl -k https://ollama.example.com/api/models    # -k 若是自签名，生产应有有效证书
```

### 4.6.7 常见故障与排查步骤

1. **`frpc` 无法连上 `frps`**

   * 检查 `server_addr` / `server_port` 是否正确；测试 `telnet frps 7000` 或 `nc -vz frps 7000`。
   * 检查防火墙 / 云安全组是否放行端口（尤其是 7000、vhost_http_port、vhost_https_port）。

2. **访问时出现 502 / 504 或 连接被重置**

   * 查看 `frpc` 日志（`frpc.log`）和 `frps` 日志，寻找 `proxy start failed`、`handshake timeout` 等信息。
   * 确认 Ollama 本地是否仍在监听 `127.0.0.1:11434` 并能响应（`curl http://127.0.0.1:11434/api`）。

3. **HTTP 请求返回 401（认证失败）**

   * 确认你是否为 HTTP 类型代理启用了 `http_user` / `http_pwd`（或 TOML 中 `httpUser/httpPassword`），curl 时是否使用了 `-u user:pwd`。

4. **域名访问没有正确路由到 Ollama**

   * 确认域名 DNS A 记录指向 `frps` 的公网 IP。
   * 确认 frps 的 `vhost_http_port` 被正确配置且未被防火墙阻挡。

5. **要在请求中获取真实客户端 IP**

   * 启用 `proxy_protocol_version = v2`（frpc 端）并在本地 Web/NGINX 上启用 proxy protocol 支持，或使用 `X-Forwarded-For` 头。（frp 文档说明）

### 4.6.8 推荐的 `frpc` 模板（拷贝即可）

#### INI（HTTP vhost + BasicAuth，推荐）

```ini
[common]
server_addr = frps.example.com
server_port = 7000
token = ReplaceWithYourToken

# Ollama via HTTP vhost with BasicAuth
[ollama-http]
type = http
local_ip = 127.0.0.1
local_port = 11434
custom_domains = ollama.example.com
http_user = ollamauser
http_pwd = superStrongPassword!
# proxy_protocol_version = v2   # 可选：如果你要把真实 IP 传给后端
# use_encryption = true         # 某些旧版 INI 支持；新版建议用 TOML transport.useEncryption
# use_compression = true
```

#### TOML（带 transport 加密）

```toml
# frpc.toml
[common]
serverAddr = "frps.example.com"
serverPort = 7000
token = "ReplaceWithYourToken"

[[proxies]]
name = "ollama-http"
type = "http"
localAddr = "127.0.0.1"
localPort = 11434
customDomains = ["ollama.example.com"]
httpUser = "ollamauser"
httpPassword = "superStrongPassword!"
proxyProtocolVersion = "v2"  # 可选：如果你要把真实 IP 传给后端

[transport]
useEncryption = true
useCompression = true
```

## 4.7 TOML 与 INI 格式转换说明与最佳实践

### 4.7.1 为什么选择 TOML？

TOML（Tom's Obvious, Minimal Language）是一种现代化的配置文件格式，相比传统的INI格式，它具有以下优势：

- **更好的类型支持**：TOML原生支持字符串、整数、浮点数、布尔值、数组等多种数据类型
- **更清晰的层次结构**：支持嵌套表，便于组织复杂配置
- **数组语法**：对于多值配置（如 `custom_domains`）更直观
- **注释支持**：更灵活的注释语法
- **新版推荐**：frp 新版官方更推荐使用 TOML 格式

### 4.7.2 INI 与 TOML 核心差异对比

| 特性 | INI 格式 | TOML 格式 |
|------|---------|----------|
| 配置分组 | `[section]` | `[table]` |
| 键值对 | `key = value` | `key = value` |
| 字符串 | `key = value` | `key = "value"` (建议用引号) |
| 代理定义 | `[proxy_name]` | `[[proxies]]`<br>`name = "proxy_name"` |
| 数组 | `key = a,b,c` | `key = ["a", "b", "c"]` |
| 传输配置 | 散布在各节中或 `use_encryption = true` | 集中在 `[transport]` 表中 |
| 多域名 | `custom_domains = example.com,test.com` | `customDomains = ["example.com", "test.com"]` |

### 4.7.3 转换示例：从 INI 到 TOML

#### 基本配置转换

INI格式：
```ini
[common]
server_addr = 1.2.3.4
server_port = 7000
token = 123456

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000
```

对应TOML格式：
```toml
[common]
serverAddr = "1.2.3.4"
serverPort = 7000
token = "123456"

[[proxies]]
name = "ssh"
type = "tcp"
localAddr = "127.0.0.1"
localPort = 22
remotePort = 6000
```

#### HTTP代理转换

INI格式：
```ini
[web]
type = http
local_ip = 127.0.0.1
local_port = 80
custom_domains = web.example.com,www.example.com
http_user = admin
http_pwd = password
```

对应TOML格式：
```toml
[[proxies]]
name = "web"
type = "http"
localAddr = "127.0.0.1"
localPort = 80
customDomains = ["web.example.com", "www.example.com"]
httpUser = "admin"
httpPassword = "password"
```

### 4.7.4 TOML 最佳实践

1. **总是使用引号包裹字符串值**：虽然TOML允许不加引号的字符串，但为了一致性和避免特殊字符问题，建议总是使用双引号
2. **使用驼峰命名法**：TOML配置推荐使用驼峰命名法（`serverAddr` 而非 `server_addr`）
3. **合理组织传输配置**：将加密、压缩等传输层配置集中放在 `[transport]` 表中
4. **为多值配置使用数组**：如 `customDomains` 应使用数组语法
5. **版本兼容性**：
   - frp v0.38.0+ 完全支持 TOML 格式
   - 对于代理配置，使用 `[[proxies]]` 数组语法而非旧版的命名表语法
6. **配置文件扩展名**：使用 `.toml` 扩展名（`frps.toml`, `frpc.toml`）
7. **使用注释**：在复杂配置中添加清晰的注释，提高可维护性

### 4.7.5 从 INI 迁移到 TOML 的步骤

1. 创建新的 `.toml` 文件（保留原INI作为备份）
2. 复制 `[common]` 部分，注意将键名转换为驼峰命名法
3. 将每个命名代理部分（如 `[ssh]`）转换为 `[[proxies]]` 表，并添加 `name = "ssh"`
4. 调整所有键名为驼峰命名法
5. 将逗号分隔的多值配置转换为数组语法
6. 集中配置传输层选项到 `[transport]` 表
7. 使用 `frpc verify -c frpc.toml` 验证配置有效性
8. 测试运行确保功能正常

### 4.6.9 进阶建议（如果你需要更专业部署）

* 把 `frps` 和 TLS（Let's Encrypt）与 Nginx 反代结合：Nginx 做 TLS 终止 + WAF/限流，frps 做隧道转发。
* 对于团队共享：用 `subdomain` + wildcard DNS 管理多个客户，或为每个用户分配不同 `token` / OIDC。
* 若追求低延迟与 P2P：研究 STCP / XTCP 模式（需要额外的 `sk`/secret 配置，适合内网点对点连接）。
* 结合 CI/CD：把 `frpc` 配置放在加密的配置仓库，用 Ansible / Salt / Puppet 部署并托管 `frpc` 服务（systemd / LaunchAgent）。

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


