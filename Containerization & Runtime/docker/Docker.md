# 🐳 Docker 深入讲解手册

---

## 一、概述与价值

### **1.1 简介：核心定位、解决问题、目标用户**

**Docker 是什么？** 
Docker 是一个基于 **容器技术（Container）** 的虚拟化平台，用于**打包、分发和运行应用程序**。

一句话总结：

> Docker = 轻量级虚拟机 + 应用打包分发平台 + 自动化部署引擎。

---

**Docker 解决的问题：**

| 问题     | 传统环境痛点       | Docker 的解决方式    | 
| ------ | ------------ | --------------- | 
| 环境不一致  | "在我电脑上能跑"问题  | 打包为统一镜像，环境+依赖一致 | 
| 部署复杂   | 部署步骤繁琐、依赖冲突  | 一条命令部署，镜像秒级启动   | 
| 资源浪费   | 虚拟机启动慢，占用大   | 容器共享内核，资源开销极低   | 
| 迁移困难   | 不同系统、云环境迁移麻烦 | 镜像可跨平台运行        | 
| 多版本并存难 | 软件版本冲突       | 容器隔离，互不影响       | 

---

**目标用户群体：**

* 开发者：快速构建、测试环境；
* 运维人员：自动化部署、持续集成；
* 企业团队：微服务架构、云原生应用；
* AI / 数据科学：统一环境、可重复实验。

---

### **1.2 核心优势**

| 维度    | Docker                  | 传统虚拟机 (VM) | 
| ----- | ----------------------- | ---------- | 
| 启动速度  | 秒级                      | 分钟级        | 
| 性能损耗  | 几乎无损                    | 明显         | 
| 隔离级别  | 进程级                     | 系统级        | 
| 资源占用  | 小                       | 大          | 
| 部署与迁移 | 简单（镜像复制）                | 复杂         | 
| 管理工具  | 丰富生态（Compose、Swarm、K8s） | 较少         | 

**一句话：**

> Docker 用更轻、更快、更自动化的方式，重新定义了软件的"交付"方式。

---

### **1.3 典型应用场景**

| 应用场景       | 示例                              | 解决的问题      | 
| ---------- | ------------------------------- | ---------- | 
| 💻 开发环境隔离  | 前后端、AI、数据库分离部署                  | 避免依赖冲突     | 
| 🚀 自动化部署   | CI/CD 集成 Jenkins、GitHub Actions | 一键构建镜像     | 
| 🌐 微服务架构   | 多容器服务（Nginx + API + Redis）      | 服务解耦、独立升级  | 
| 🧠 AI/数据科学 | 运行 TensorFlow/PyTorch 环境        | 统一环境、可重复训练 | 
| ☁️ 云原生平台   | Kubernetes、Docker Swarm         | 容器编排与扩缩容   | 
| 📦 应用分发    | 镜像推送至 Docker Hub / 私服           | 快速分发软件包    | 

---

### **1.4 文档阅读指南**

| 章节  | 内容概要    | 适合读者           | 
| --- | ------- | -------------- | 
| 第一章 | 核心认知与价值 | 初学者快速建立概念      | 
| 第二章 | 环境搭建与配置 | 想要安装并使用 Docker | 
| 第三章 | 核心功能与操作 | 实际开发部署用户       | 
| 第四章 | 高级应用与案例 | 有实战需求的用户       | 
| 第五章 | 总结与延伸   | 想持续深入学习 Docker | 

---

## 二、基础环境与配置

### **2.1 准备工作**

**系统要求：**

* 操作系统：Ubuntu 22.04 / 24.04、Debian、Fedora、CentOS、Windows 11 WSL2；
* 硬件：推荐 ≥ 2 核 CPU，≥ 4GB 内存；
* 网络：支持访问 Docker Hub 或国内镜像源。

---

**安装步骤 (以 Ubuntu 24.04 为例)：**

```bash 
# 1. 卸载旧版本（如存在） 
sudo apt remove docker docker-engine docker.io containerd runc 

# 2. 安装依赖 
sudo apt update 
sudo apt install ca-certificates curl gnupg lsb-release -y 

# 3. 添加 GPG 密钥 
sudo mkdir -p /etc/apt/keyrings 
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg 

# 4. 添加官方源 
echo \ 
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \ 
  https://download.docker.com/linux/ubuntu \ 
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \ 
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null 

# 5. 安装 Docker 引擎 
sudo apt update 
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y 
```

### 2.1.1 GPG 公钥导入详解

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

### 2.1.2 配置代理加速（国内环境推荐）

在国内环境或内网环境中，为 Docker 配置代理是非常重要的。下面详细介绍 Docker 三个层面的代理配置方法。

#### 配置 Docker 守护进程代理（最关键）

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

#### 使用国内镜像源加速

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

### 2.1.3 验证安装成功

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

---

### **2.2 Docker CLI 常用界面命令导览**

| 命令                          | 功能说明       | 
| --------------------------- | ---------- | 
| `docker version`            | 查看版本信息     | 
| `docker info`               | 查看系统级信息    | 
| `docker ps`                 | 查看正在运行的容器  | 
| `docker images`             | 查看本地镜像     | 
| `docker run`                | 运行容器       | 
| `docker stop/start/restart` | 控制容器运行状态   | 
| `docker logs`               | 查看容器日志     | 
| `docker exec`               | 进入容器内部执行命令 | 
| `docker compose up`         | 启动多容器服务    | 

---

## 三、核心功能与操作

### **3.1 镜像与容器管理（最常用功能）**

#### 👉 获取镜像

```bash 
docker pull nginx 
```

#### 👉 运行容器

```bash 
docker run -d --name web -p 8080:80 nginx 
```

#### 👉 查看容器

```bash 
docker ps -a 
```

#### 👉 停止/启动容器

```bash 
docker stop web 
docker start web 
```

#### 👉 删除容器和镜像

```bash 
docker rm web 
docker rmi nginx 
```

---

### **3.2 构建与上传镜像（进阶功能）**

#### 👉 使用 Dockerfile 构建镜像

```Dockerfile 
FROM ubuntu:24.04 
RUN apt update && apt install -y python3 
CMD ["python3", "--version"] 
```

构建命令：

```bash 
docker build -t mypython:1.0 . 
```

#### 👉 推送到远程仓库

```bash 
docker login 
docker tag mypython:1.0 username/mypython:1.0 
docker push username/mypython:1.0 
```

### 3.2.1 Dockerfile 关键指令详解

| 指令 | 功能 | 示例 |
|-----|------|-----|
| `FROM` | 指定基础镜像 | `FROM ubuntu:24.04` |
| `RUN` | 执行命令 | `RUN apt update && apt install -y git` |
| `COPY` | 复制文件 | `COPY . /app` |
| `WORKDIR` | 设置工作目录 | `WORKDIR /app` |
| `EXPOSE` | 暴露端口 | `EXPOSE 80` |
| `ENV` | 设置环境变量 | `ENV NODE_ENV=production` |
| `CMD` | 容器启动命令 | `CMD ["node", "app.js"]` |
| `ENTRYPOINT` | 容器入口点 | `ENTRYPOINT ["/entrypoint.sh"]` |

### 3.2.2 容器网络配置

#### 常用网络命令

```bash
# 创建自定义网络
docker network create my-network

# 运行容器时指定网络
docker run --name db --network my-network mysql

# 连接现有容器到网络
docker network connect my-network app-container
```

#### 网络模式对比

| 网络模式 | 描述 | 使用场景 |
|---------|------|--------|
| `bridge` | 默认网络模式，隔离的网络命名空间 | 一般应用开发 |
| `host` | 直接使用主机网络 | 性能要求高，不需要网络隔离 |
| `none` | 无网络 | 完全隔离的环境 |
| `container:` | 共享另一个容器的网络 | 调试或特定服务组合 |
| 自定义网络 | 用户创建的桥接网络 | 多容器服务通信 |

---

## 四、进阶应用与案例

### **4.1 典型工作流：多容器 Web 应用**

目标：部署一个简单的 Web 服务（Nginx + Flask + Redis）

使用 **docker-compose.yml：**

```yaml 
version: '3' 
services: 
  web: 
    build: ./app 
    ports: 
      - "5000:5000" 
    depends_on: 
      - redis 

  redis: 
    image: redis:alpine 
```

执行：

```bash 
docker compose up -d 
```

### 4.1.1 Docker Compose 进阶配置

```yaml
version: '3.8'
services:
  web:
    build: 
      context: ./app
      dockerfile: Dockerfile.prod
    ports:
      - "80:80"
    volumes:
      - ./app:/var/www/html
    environment:
      - DB_HOST=db
      - DB_NAME=mydb
    restart: always
    depends_on:
      db:
        condition: service_healthy
  
  db:
    image: mysql:8.0
    volumes:
      - db_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=mydb
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  db_data:
```

---

### **4.2 疑难解答 (FAQ)**

| 问题                                | 原因           | 解决办法                            | 
| --------------------------------- | ------------ | ------------------------------- | 
| `Permission denied`               | 未加入 docker 组 | `sudo usermod -aG docker $USER` | 
| 拉取镜像慢                             | 国内网络         | 配置国内镜像源                         | 
| `Cannot connect to Docker daemon` | 服务未启动        | `sudo systemctl start docker`   | 
| 容器无法联网                            | 防火墙 / 网络隔离   | 检查 `docker0` 网桥或代理配置            | 
| 磁盘空间不足                            | 日志或镜像过多      | `docker system prune` 清理无用资源    |

### 4.2.1 容器内部网络代理配置

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

### 4.2.2 测试代理是否生效

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

---

## 五、总结与资源

### **5.1 总结**

Docker 让软件交付进入 **"镜像化时代"**，开发、测试、部署全流程实现：

* **一致性**：环境一致，消除"兼容地狱"
* **轻量化**：资源占用低、启动快
* **自动化**：构建→部署→扩容全自动

---

### **5.2 推荐学习路径**

| 阶段 | 学习重点                  | 建议资源                                        | 
| -- | --------------------- | ------------------------------------------- | 
| 初级 | 基础命令、容器概念             | Docker 官方文档、菜鸟教程                            | 
| 中级 | Dockerfile、Compose、网络 | `https://docs.docker.com/`  | 
| 高级 | 私有仓库、CI/CD、Kubernetes | GitHub Actions、K8s 官方文档                     | 
| 实战 | 部署真实项目                | Flask + Redis + Nginx 项目实战                  |

### 5.2.1 扩展资源

- **Docker 官方文档**：https://docs.docker.com/
- **Docker 镜像加速配置**：https://docs.docker.com/registry/recipes/mirror/
- **Docker Compose 文档**：https://docs.docker.com/compose/
- **Ubuntu 官方 Docker 安装指南**：https://docs.docker.com/engine/install/ubuntu/
- **Docker Hub**：https://hub.docker.com/ - 查找和分享容器镜像
- **Docker Scout**：https://docs.docker.com/scout/ - 容器安全和软件供应链分析

---

> 💡 提示：如果你需要自动化配置，可以使用脚本自动检测代理环境并应用相应配置。

> 🎯 小技巧：定期使用 `docker system prune -af` 清理无用的镜像和容器，释放磁盘空间。
