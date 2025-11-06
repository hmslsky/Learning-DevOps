# 🐳 Docker 完全使用指南

## 目录

1. [Docker 在 Ubuntu 上的安装与配置](#1-docker-在-ubuntu-上的安装与配置)
   - [1.1 GPG 公钥导入](#11-gpg-公钥导入)
   - [1.2 配置 APT 源](#12-配置-apt-源)
   - [1.3 安装 Docker 组件](#13-安装-docker-组件)
   - [1.4 验证安装成功](#14-验证安装成功)
   - [1.5 安装总结](#15-安装总结)
2. [Docker 代理配置详解](#2-docker-代理配置详解)
   - [2.1 代理使用场景概述](#21-代理使用场景概述)
   - [2.2 配置 Docker 客户端代理](#22-配置-docker-客户端代理)
   - [2.3 配置 Docker 守护进程代理](#23-配置-docker-守护进程代理)
   - [2.4 配置容器内部网络代理](#24-配置容器内部网络代理)
   - [2.5 测试代理是否生效](#25-测试代理是否生效)
   - [2.6 使用国内镜像源加速](#26-使用国内镜像源加速)
   - [2.7 代理配置总结](#27-代理配置总结)

---

## 1. Docker 在 Ubuntu 上的安装与配置

### 1.1 GPG 公钥导入

Docker 官方仓库使用 GPG 密钥来确保软件包的真实性。以下是在 Ubuntu 24.04（使用新的 keyrings 机制）上导入 Docker GPG 公钥的详细步骤：

#### 确认公钥文件格式

假设你下载的公钥文件名是：
- 二进制格式：`docker.gpg`
- 文本格式：`docker.asc`

可以通过以下命令检查文件格式：
```bash
head docker.gpg
```

- 如果是二进制文件，会显示乱码（正常）
- 如果是 ASCII 格式（以 `-----BEGIN PGP PUBLIC KEY BLOCK-----` 开头），需要先转换为二进制格式

#### 导入步骤

1. **建立 keyring 目录**（仅第一次执行时需要）：
   ```bash
   sudo mkdir -p /etc/apt/keyrings
   ```

2. **ASCII 格式转换**（如果文件是 `.asc` 格式）：
   ```bash
   sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg docker.asc
   ```
   > `--dearmor` 参数用于将文本格式的 GPG 公钥转换为二进制格式

3. **直接复制二进制格式**（如果文件已经是 `.gpg` 格式）：
   ```bash
   sudo cp docker.gpg /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   ```
   > 注意：必须赋予只读权限，否则 apt 会提示权限不足

4. **验证密钥导入成功**：
   ```bash
   gpg --show-keys /etc/apt/keyrings/docker.gpg
   ```

   成功输出示例：
   ```
   pub   rsa4096 2017-02-22 [SC]
         9DC858229FC7DD38854AE2D88D81803C0EBFCD88
   uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
   ```

### 1.2 配置 APT 源

Ubuntu 24.04 推荐使用 `signed-by` 参数在源配置中指定密钥，而不是全局导入：

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

这个命令的作用是：
- 自动检测并指定系统架构
- 通过 `signed-by` 参数指定签名验证文件路径
- 添加 Docker 官方稳定版仓库源

### 1.3 安装 Docker 组件

更新软件包索引并安装 Docker 最新版：

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

安装的组件包括：
- `docker-ce`：Docker 社区版引擎
- `docker-ce-cli`：Docker 命令行工具
- `containerd.io`：容器运行时
- `docker-buildx-plugin`：Docker Buildx 插件（增强构建功能）
- `docker-compose-plugin`：Docker Compose 插件（多容器应用编排）

### 1.4 验证安装成功

检查 Docker 版本和运行状态：
```bash
docker --version
sudo systemctl status docker
```

运行测试容器：
```bash
sudo docker run hello-world
```

如果看到 "Hello from Docker!" 消息，说明安装成功 ✅

### 1.5 安装总结

| 步骤 | 命令 | 说明 |
|------|------|------|
| 1 | `sudo mkdir -p /etc/apt/keyrings` | 创建密钥存放目录 |
| 2 | `sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg docker.asc` | 转换并导入公钥（ASCII格式） |
| 3 | `sudo cp docker.gpg /etc/apt/keyrings/docker.gpg && sudo chmod a+r /etc/apt/keyrings/docker.gpg` | 复制并设置权限（二进制格式） |
| 4 | `echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null` | 添加 APT 源 |
| 5 | `sudo apt update && sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y` | 安装 Docker 组件 |
| 6 | `sudo docker run hello-world` | 验证安装成功 |

---

## 2. Docker 代理配置详解

在国内环境或内网环境中，为 Docker 配置代理是非常重要的。下面详细介绍 Docker 三个层面的代理配置方法。

### 2.1 代理使用场景概述

| 代理层面 | 描述 | 配置位置 | 重要性 |
|---------|------|----------|--------|
| **Docker 客户端（CLI）** | 执行 `docker pull`、`docker build` 等命令时使用 | Shell 环境变量 | ⭐⭐ |
| **Docker 守护进程（Daemon）** | Docker 服务在后台拉取镜像或构建时使用 | `/etc/systemd/system/docker.service.d/proxy.conf` | ⭐⭐⭐ |
| **Docker 容器内部** | 运行中的容器需要访问外部网络时使用 | Dockerfile 或 `docker run` 环境变量 | ⭐⭐ |

### 2.2 配置 Docker 客户端代理

客户端代理仅对当前终端会话生效，适用于临时需要使用代理的场景：

```bash
export HTTP_PROXY="http://127.0.0.1:7890"
export HTTPS_PROXY="http://127.0.0.1:7890"
export NO_PROXY="localhost,127.0.0.1"
```

> 提示：请根据实际使用的代理工具（如 Clash、V2Ray、Warp 等）修改端口号

#### 永久配置客户端代理

将环境变量添加到 `.bashrc` 或 `.zshrc` 文件中，使其永久生效：

```bash
echo 'export HTTP_PROXY="http://127.0.0.1:7890"' >> ~/.bashrc
echo 'export HTTPS_PROXY="http://127.0.0.1:7890"' >> ~/.bashrc
echo 'export NO_PROXY="localhost,127.0.0.1"' >> ~/.bashrc
source ~/.bashrc  # 立即生效
```

### 2.3 配置 Docker 守护进程代理

> 注意：这是最关键的配置，因为 `docker pull` 等操作实际上是由守护进程执行的

1. **创建 systemd 配置目录**：
   ```bash
   sudo mkdir -p /etc/systemd/system/docker.service.d
   ```

2. **创建代理配置文件**：
   ```bash
   sudo nano /etc/systemd/system/docker.service.d/proxy.conf
   ```

3. **添加以下内容**（根据实际代理修改端口）：
   ```ini
   [Service]
   Environment="HTTP_PROXY=http://127.0.0.1:7890"
   Environment="HTTPS_PROXY=http://127.0.0.1:7890"
   Environment="NO_PROXY=localhost,127.0.0.1,::1"
   ```

4. **重新加载 systemd 配置并重启 Docker**：
   ```bash
   sudo systemctl daemon-reexec
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```

5. **验证配置是否生效**：
   ```bash
   systemctl show --property=Environment docker
   ```

   成功输出示例：
   ```
   Environment=HTTP_PROXY=http://127.0.0.1:7890 HTTPS_PROXY=http://127.0.0.1:7890 NO_PROXY=localhost,127.0.0.1,::1
   ```

### 2.4 配置容器内部网络代理

当运行中的容器需要访问外部网络时，有两种配置方式：

#### 方法 1：通过 `docker run` 命令传入环境变量

```bash
docker run -e HTTP_PROXY="http://127.0.0.1:7890" \
           -e HTTPS_PROXY="http://127.0.0.1:7890" \
           -e NO_PROXY="localhost,127.0.0.1" \
           ubuntu bash
```

#### 方法 2：在 Dockerfile 中设置环境变量

```dockerfile
FROM ubuntu:22.04

# 设置代理环境变量
ENV HTTP_PROXY=http://127.0.0.1:7890
ENV HTTPS_PROXY=http://127.0.0.1:7890
ENV NO_PROXY=localhost,127.0.0.1

# 后续指令...
```

> 注意：容器内的 127.0.0.1 指向容器自身，如需使用主机代理，应使用主机 IP 或 `host.docker.internal`（Docker Desktop）

### 2.5 测试代理是否生效

1. **测试拉取镜像**：
   ```bash
   docker pull hello-world
   ```
   如果速度明显加快或不再报错，说明代理生效

2. **测试网络连通性**：
   ```bash
   docker run --rm curlimages/curl -I https://google.com
   ```
   能成功获取响应头表示网络连通

### 2.6 使用国内镜像源加速

除了使用代理，还可以配置国内镜像源来加速镜像拉取（两者可以同时使用）：

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://mirror.ccs.tencentyun.com",
    "https://hub-mirror.c.163.com"
  ]
}
EOF
sudo systemctl restart docker
```

### 2.7 代理配置总结

| 配置对象 | 配置方式 | 生效范围 | 配置文件/命令 |
|---------|----------|----------|---------------|
| Docker 客户端 | 环境变量 | 当前终端 | `export HTTP_PROXY=...` |
| Docker 守护进程 | systemd 配置 | 全局 | `/etc/systemd/system/docker.service.d/proxy.conf` |
| 容器内部 | 环境变量 | 容器内 | `docker run -e` 或 Dockerfile `ENV` |
| 镜像加速 | daemon.json | 全局 | `/etc/docker/daemon.json` |

---

## 📝 扩展资源

- **Docker 官方文档**：https://docs.docker.com/
- **Docker 镜像加速配置**：https://docs.docker.com/registry/recipes/mirror/
- **Ubuntu 官方 Docker 安装指南**：https://docs.docker.com/engine/install/ubuntu/

---

> 💡 提示：如果你需要自动化配置，可以使用脚本自动检测代理环境并应用相应配置。
