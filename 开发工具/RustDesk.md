# 🖥️ RustDesk 全面使用与自建指南

## 1. 概述与基础

### 1.1 什么是 RustDesk

**RustDesk** 是一款优秀的开源远程桌面软件，可作为 TeamViewer / AnyDesk 的替代方案。它支持 Windows、Linux、macOS、Android 和 iOS 多平台互通，具备自建服务器能力，可实现局域网直连，为用户提供安全、稳定、低成本的远程访问解决方案。

### 1.2 核心优势

| 对比维度  | RustDesk                   | TeamViewer / AnyDesk |
| ----- | -------------------------- | -------------------- |
| 开源    | ✅ 完全开源（Rust + Flutter）     | ❌ 闭源                 |
| 自建服务器 | ✅ 支持 Docker / Baremetal 部署 | ❌ 不支持                |
| 成本    | ✅ 免费                       | 💲 收费 / 商业限制         |
| 性能    | ✅ P2P + 中继模式，速度接近原生        | ✅ 稳定但服务器依赖厂商         |
| 安全性   | ✅ 数据端到端加密，可完全内网化           | ⚠️ 数据经第三方服务器         |
| 可定制性  | ✅ 可修改源码 / 接口               | ❌ 不可修改               |

### 1.3 典型应用场景

| 场景        | 说明                | RustDesk 如何解决      |
| --------- | ----------------- | ------------------ |
| 🏠 家庭远程控制 | 在外地控制家中电脑 / NAS   | 配置公网IP或内网穿透中继，稳定访问 |
| 🏢 企业内网运维 | 员工远程登录公司电脑        | 自建服务端，保障隐私与访问速度    |
| 💻 技术支持   | IT远程解决客户电脑问题      | 使用客户端临时连接码快速协作     |
| 🖧 教育培训   | 远程教学、屏幕共享         | 跨平台客户端轻量运行         |
| 🪟 多系统管理  | 管理混合系统（Win+Linux） | RustDesk 自带跨平台 GUI |

## 2. 快速入门

### 2.1 客户端安装

在 [RustDesk 官网](https://rustdesk.com) 下载对应平台的客户端并安装。所有平台安装过程都非常简单，按照提示完成即可。

### 2.2 快速远程连接

1. 在两台设备上安装 RustDesk
2. 目标机器打开 RustDesk，复制「ID」和「密码」
3. 控制端输入该 ID → 点击连接 → 输入密码 → 成功连接

> 📌 **小技巧：**
> * "复制"按钮可以一键复制 ID/密码
> * 如果两台设备在同一局域网，会自动直连（速度更快）

### 2.3 界面导览

* **左侧面板**：历史设备、我的设备列表
* **右侧面板**：输入远程 ID 或 IP 连接区域
* **底部按钮**：文件传输 / 远程剪贴板 / 设置等功能
* **常用快捷键**：
  * `F11`：全屏 / 退出全屏
  * `Ctrl + Alt + Q`：断开连接
  * `Ctrl + F`：打开文件传输

## 3. 核心功能详解

### 3.1 永久免密码访问（无人值守）

在被控端设置：

1. 打开 RustDesk → 设置 → 「安全」
2. 启用「允许无人值守访问」
3. 设置一个永久密码
4. 控制端输入 ID 时可勾选"记住密码"

> ✅ 这样即可随时连接，无需目标端手动确认

### 3.2 文件传输功能

* 连接成功后，顶部点击 **"文件传输"** 图标
* 支持拖放文件、双向传输
* 文件管理窗口可直接复制、粘贴、删除文件

> 📎 文件传输不依赖剪贴板，传大文件更稳定

### 3.3 剪贴板同步

* 默认情况下剪贴板（文本）自动同步
* 若复制粘贴不生效：
  * 检查「设置 → 功能 → 剪贴板同步」是否开启
  * 尝试重启 RustDesk 或重新连接

### 3.4 多显示器支持

* 连接后点击顶部「显示器」图标
* 可以选择"单屏显示"或"所有屏幕"
* 支持动态切换显示器

### 3.5 自定义显示设置

路径：**设置 → 显示 → 远程分辨率 / 帧率 / 压缩率**

> ⚙️ 调整建议：
> * 如果延迟高 → 降低帧率（如 15fps）
> * 如果网络稳定 → 提高画质或帧率

## 4. 自建服务器部署指南

### 4.1 准备工作

* **服务器要求**：Linux（Ubuntu/CentOS 任意发行版），内存 ≥ 1GB，CPU ≥ 1核，有固定公网 IP
* **软件要求**：已安装 Docker（若未装：`curl -fsSL https://get.docker.com | sh`）
* **端口需求**：需要开放 TCP `21114-21119` 和 UDP `21116`（最少需要 `21115-21117` + UDP 21116）

### 4.2 服务器组件说明

* **hbbs** = ID / 信令服务器，默认监听 TCP `21115/21116/21118` 与 UDP `21116`（NAT 打洞）
* **hbbr** = Relay（中继）服务器，默认监听 TCP `21117`（以及 `21119` 为 web-socket）

### 4.2.1 hbbs 与 hbbr 深度解析

非常好的问题，这个是很多人刚开始自建 RustDesk 时最容易混淆的地方 👏。
要想正确部署 RustDesk 服务端，必须先理解 hbbs 与 hbbr 的角色区别和协作机制。下面来详细讲清楚两者的原理、作用、通信逻辑和部署关系。

🧭 一、RustDesk 服务端整体架构概览
RustDesk 的完整通信系统分为两部分：

| 组件名称 | 主要功能 | 端口（默认） |
|--------|---------|------------|
| hbbs | Host Based Broker Service<br>负责注册设备、分配 ID、协调连接（信令服务） | TCP 21115 |
| hbbr | Host Based Relay Service<br>负责数据中继转发（当直连失败时使用） | TCP/UDP 21116 |

RustDesk 的"中继服务器"其实由这两个服务组成。
它们通常在一台服务器上运行，但逻辑上完全独立。

🧩 二、hbbs：信令 / ID 服务（Broker Server）

1️⃣ 作用

**hbbs 是整个系统的"指挥官"**，用于：
* 设备上线注册
* 分配唯一 ID
* 记录客户端的公网 / 内网地址
* 协调双方是否能建立 P2P 直连
* 如果直连失败，则引导双方通过 hbbr 进行中继

2️⃣ 类比理解
* 相当于 TeamViewer 的"ID 注册服务器"
* 相当于 WebRTC 的"信令服务器"

3️⃣ 通信流程
* 客户端启动 → 连接 hbbs
* hbbs 记录设备信息（ID、地址、状态）
* 当另一台客户端输入该 ID 连接时：
  * hbbs 通知目标端准备连接
  * 如果双方 NAT 可穿透 → 建立直连
  * 如果不行 → 交给 hbbr 中继

4️⃣ 端口与协议

| 端口 | 协议 | 用途 |
|-----|------|------|
| 21115 | TCP | 信令注册与 ID 分配 |
| 21118 | TCP | Web 控制面板（可选） |

🌉 三、hbbr：中继（Relay）服务

1️⃣ 作用

**hbbr 是实际负责数据转发的中继服务器**。
当两个客户端无法直接建立连接时，所有的屏幕、键鼠、文件数据都会通过 hbbr 转发。

2️⃣ 类比理解
* 相当于 WebRTC 的 TURN Server
* 相当于 TeamViewer 的中继节点

3️⃣ 通信流程
* hbbs 判断双方 P2P 无法直连
* 命令双方通过 hbbr 建立中继通道
* 数据流经 hbbr → 转发给对方
* hbbr 不解密内容，仅转发加密数据

4️⃣ 端口与协议

| 端口 | 协议 | 用途 |
|-----|------|------|
| 21116 | TCP/UDP | 数据中继转发（屏幕流、键鼠、文件等） |

UDP 性能更好，因此建议同时开放 21116/tcp 和 21116/udp。

⚙️ 四、两者关系与工作原理

下面是一个简化的数据流示意：
```
[客户端A] ←→ hbbs ←→ [客户端B]
   │                  │
   └─────hbbr──────────┘
```

| 阶段 | 使用组件 | 说明 |
|-----|---------|------|
| 注册与发现 | hbbs | 客户端连接 hbbs 注册 ID、查询目标状态 |
| 连接协商 | hbbs | 判断是否能直连，若失败则选择中继 |
| 数据传输 | hbbr（或直连） | 实际屏幕与控制数据传输 |

🧱 五、部署策略与建议

| 场景 | 推荐部署 |
|-----|---------|
| 单服务器（最常见） | 同时运行 hbbs 和 hbbr |
| 高并发 / 多地域 | hbbs 独立部署，多个 hbbr 做负载均衡 |
| 内网使用 | 只需要 hbbs 即可（直连足够） |
| 公网中继（家庭/中小团队） | 同机部署 hbbs + hbbr，开放 21115/21116 端口 |

🚀 六、Docker 部署（同时运行 hbbs + hbbr）

官方镜像已经集成两者，你只需：
```bash
docker run -d \
  --name rustdesk-server \
  --restart=always \
  -p 21115:21115 \
  -p 21116:21116 \
  -p 21116:21116/udp \
  -p 21118:21118 \
  rustdesk/rustdesk-server
```

容器内自动运行：
* /usr/local/bin/hbbs
* /usr/local/bin/hbbr

🧠 七、总结：核心对比表

| 项目 | hbbs | hbbr |
|-----|------|------|
| 全称 | Host Based Broker Server | Host Based Relay Server |
| 职责 | 信令、设备注册、ID 分配、连接协调 | 数据中继、转发屏幕流/文件流 |
| 是否必须 | ✅ 必需 | ⚙️ 仅在直连失败时使用 |
| 通信协议 | TCP | TCP / UDP |
| 默认端口 | 21115 / 21118 | 21116 |
| 加密 | 不处理数据内容 | 数据端到端加密，hbbr仅转发 |
| 性能影响 | 低 | 受带宽影响大 |
| 建议部署 | 公网可访问 | 与 hbbs 同机部署 |

### 4.3 使用 Docker 部署（推荐）

#### 4.3.1 使用 docker run 命令

1. 拉取镜像：
```bash
sudo docker pull rustdesk/rustdesk-server:latest
```

2. 创建数据目录：
```bash
sudo mkdir -p /opt/rustdesk/data
cd /opt/rustdesk
```

3. 启动 hbbs（信令/ID 服务）：
```bash
sudo docker run -d --name hbbs \
  -v "$(pwd)/data":/root \
  --restart unless-stopped \
  --net=host \
  rustdesk/rustdesk-server:latest hbbs
```

4. 启动 hbbr（relay 中继）：
```bash
sudo docker run -d --name hbbr \
  -v "$(pwd)/data":/root \
  --restart unless-stopped \
  --net=host \
  rustdesk/rustdesk-server:latest hbbr
```

#### 4.3.2 使用 docker-compose

创建 `docker-compose.yml` 文件：
```yaml
version: "3.8"
services:
  hbbs:
    image: rustdesk/rustdesk-server:latest
    container_name: hbbs
    command: hbbs
    volumes:
      - ./data:/root
    network_mode: "host"
    restart: unless-stopped

  hbbr:
    image: rustdesk/rustdesk-server:latest
    container_name: hbbr
    command: hbbr
    volumes:
      - ./data:/root
    network_mode: "host"
    restart: unless-stopped
```

运行命令：
```bash
docker compose up -d
```

### 4.4 详细的 Docker 部署指南

#### 4.4.1 Docker 环境准备

Ubuntu 24.04 环境中安装 Docker：

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

创建数据目录：

```bash
sudo mkdir -p /opt/rustdesk
cd /opt/rustdesk
```

#### 4.4.2 部署方式比较

##### 方式一：分别启动两个容器（推荐）

```bash
# 启动 hbbs（信令服务）
docker run -d --name rustdesk-hbbs \
  --restart=always \
  -v /opt/rustdesk:/data \
  -p 21114:21114 -p 21116:21116 \
  rustdesk/rustdesk-server hbbs -r hbbr.mydomain.com

# 启动 hbbr（中继服务）
docker run -d --name rustdesk-hbbr \
  --restart=always \
  -v /opt/rustdesk:/data \
  -p 21115:21115 -p 21117:21117 -p 21119:21119/udp \
  rustdesk/rustdesk-server hbbr
```

📌 参数说明：

| 参数                       | 说明                 |
| ------------------------ | ------------------ |
| `-p 21114:21114`         | hbbs 信令端口          |
| `-p 21115:21115`         | hbbr 中继端口          |
| `-p 21119:21119/udp`     | 中继 UDP 端口          |
| `-v /opt/rustdesk:/data` | 数据（证书、配置）持久化       |
| `-r hbbr.mydomain.com`   | hbbs 告诉客户端，中继服务器在哪 |

##### 方式二：将 hbbs 和 hbbr 放在一个容器中运行

可以，但**不推荐生产环境使用**，因为：

* 容器内两个进程管理复杂；
* 日志与异常分离不方便；
* 不易独立重启。

如果仅做测试或个人使用，可以这样运行：

```bash
docker run -d --name rustdesk-server \
  --restart=always \
  -v /opt/rustdesk:/data \
  -p 21114:21114 -p 21115:21115 -p 21116:21116 -p 21117:21117 -p 21119:21119/udp \
  rustdesk/rustdesk-server sh -c "hbbr & hbbs -r yourdomain.com"
```

### 4.5 防火墙配置

在服务器上打开必要端口（以 Ubuntu + ufw 为例）：
```bash
# 允许 SSH（如果需要）
sudo ufw allow ssh

# TCP: 21114-21119
sudo ufw allow 21114:21119/tcp

# UDP: 21116
sudo ufw allow 21116/udp

# UDP: 21119
sudo ufw allow 21119/udp

sudo ufw enable
sudo ufw status
```

> ⚠️ 注意：如果你在云服务器上，还需要在云控制台的安全组中开放相同端口

### 4.6 客户端配置与验证

1. 查看容器日志：

   ```bash
   docker logs -f rustdesk-hbbs
   docker logs -f rustdesk-hbbr
   ```

   如果出现类似 "Listening on 21114" 则表示成功。

2. 获取服务器的公钥：

   ```bash
   docker exec -it rustdesk-hbbs cat /data/id_ed25519.pub
   ```

   将这串公钥填入 RustDesk 客户端设置 → "自建服务器" → "公钥"。

3. 在客户端中填写：

   ```
   ID Server: yourserver.com:21114
   Relay Server: yourserver.com:21115
   Key: <上面那串公钥>
   ```

## 5. 高级应用与最佳实践

### 5.1 性能优化技巧

* **内网直连（P2P）优化**：
  * 保证两端网络能互相访问（同 Wi-Fi 或局域网）
  * 在设置中启用「直连优先」
  * 使用自建的 `hbbs` 和 `hbbr` 服务

* **自动启动配置**：
  * 打开 RustDesk → 设置 → 常规
  * 启用：
    * ✅ "开机启动"
    * ✅ "最小化到托盘"
    * ✅ "自动启动无人值守服务"

### 5.2 安全建议

* 使用最新版本的 RustDesk
* 为无人值守访问设置强密码
* 自建服务器时备份 `./data` 目录（内含私钥/数据库）
* 可在宿主机上做额外防火墙限制，仅允许特定 IP 段入站
* 定期更新密码和检查连接记录

### 5.3 命令行与自动化

RustDesk 支持命令行启动远程连接：
```bash
rustdesk.exe --connect 123456789 --password yourpass
```

配合批处理或脚本工具，可实现：
* 一键远控固定主机
* 定时连接任务
* 快捷键打开远控窗口

### 5.4 推荐配置模板

| 设置项   | 建议值            |
| ----- | -------------- |
| 分辨率   | 1280×720（带宽友好） |
| 帧率    | 30 FPS         |
| 压缩率   | 中等             |
| 剪贴板同步 | 开启             |
| 文件传输  | 开启             |
| 输入控制  | 开启             |
| 安全认证  | 设置强密码 + 启用无人值守 |
| 开机启动  | 开启             |

## 6. 疑难解答与常见问题

### 6.1 连接问题排查

| 问题                      | 原因         | 解决方案                  |
| ----------------------- | ---------- | --------------------- |
| 无法连接中继                  | 端口未放行      | 检查防火墙或安全组 21115-21116 |
| 客户端 ID 一直是 "Connecting" | 未设置自建服务器   | 在客户端设置自定义服务器          |
| 延迟高 / 卡顿                | 未启用 UDP 中继 | 确保 `-p 21116/udp` 开放  |
| 文件传输失败                  | 权限不足       | 检查远程端权限设置             |
| 客户端显示 `Not ready`      | 密钥不匹配      | 确保客户端正确配置了公钥 Key      |

### 6.2 常见问题解答

1. **为什么推荐使用 `--net=host` 网络模式？**
   * `--net=host` 让容器看到真实公网 IP 并接收 UDP 包（对于 NAT 打洞很关键）
   * 官方示例与社区也推荐在 Linux 上使用 host 模式
   * 若不能使用 host，则需把所有相关 TCP/UDP 端口逐一映射且更复杂

2. **没有域名能否使用 IP？**
   * 可以直接用公网 IP + 端口（例如 `203.0.113.45:21116`）
   * 唯一额外步骤是把 `id_ed25519.pub` 的 Key 填入客户端，确保加密认证通过

3. **中继和信令可以放在不同机器吗？**
   * 可以，但如果 `hbbr`（relay）不在 `hbbs` 同机或不是默认端口，需要在 hbbs 配置 `RELAY_SERVERS`
   * 大多数单机部署直接同时运行 hbbs 与 hbbr 最简单且稳定

4. **Docker 部署 hbbs 和 hbbr 的最佳实践？**
   * **推荐方式**：同主机不同容器（便于独立管理和升级）
   * **测试/个人使用**：可尝试单容器运行（但生产环境不建议）
   * **高可用性**：可在不同机器上部署多个 hbbr 实例，实现中继服务负载均衡

5. **如何优化 Docker 部署的性能？**
   * 确保启用 UDP 转发（`-p 21116:21116/udp` 和 `-p 21119:21119/udp`）
   * 使用 `--net=host` 网络模式（减少网络开销）
   * 为容器分配足够的资源（CPU 和内存限制）
   * 定期清理日志，防止磁盘空间不足

## 7. 总结与资源

### 7.1 RustDesk 学习路径

1. 安装客户端
2. 连接远程设备
3. 自建中继服务器（Docker / 二进制）
4. 高级配置（证书、Web 管理、内网穿透）
5. 企业多用户管理（RustDesk Server Web Console）

### 7.2 官方与社区资源

* **官网**：[https://rustdesk.com](https://rustdesk.com)
* **GitHub**：[https://github.com/rustdesk/rustdesk](https://github.com/rustdesk/rustdesk)
* **Docker 镜像**：[https://hub.docker.com/r/rustdesk/rustdesk-server](https://hub.docker.com/r/rustdesk/rustdesk-server)
* **中文社区论坛**：[https://bbs.rustdesk.com](https://bbs.rustdesk.com)
* **官方文档**：[https://rustdesk.com/docs](https://rustdesk.com/docs)

### 7.3 部署检查清单

1. [ ] Docker 已安装并能运行容器
2. [ ] 拉取 `rustdesk/rustdesk-server:latest` 镜像
3. [ ] 启动 `hbbs` 与 `hbbr`（推荐使用 `--net=host`）
4. [ ] 开放防火墙：TCP `21114-21119`、UDP `21116` 和 `21119`
5. [ ] 在云控制台安全组中开放相同端口
6. [ ] 读取 `id_ed25519.pub`，把公钥配置到客户端
7. [ ] 客户端配置 ID Server 和 Relay Server 地址
8. [ ] 测试远程连接功能

---

通过本指南，您已经掌握了 RustDesk 的基本使用方法和自建服务器部署技巧。无论是个人远程办公还是企业内部运维，RustDesk 都能提供安全、稳定、高效的远程访问解决方案。
