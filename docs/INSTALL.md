# UHDadmin 安装指南 / Installation Guide

> 本文档涵盖 Portainer Stack 部署的完整流程，支持 Named Volumes 和 Bind Mounts 两种存储模式。

## 目录

- [从零开始部署（无需 git clone）](#从零开始部署无需-git-clone)
- [前置要求](#前置要求)
- [系统配置](#系统配置)
- [GHCR 镜像仓库认证](#ghcr-镜像仓库认证)
- [必填环境变量](#必填环境变量)
- [方式一：Named Volumes（推荐）](#方式一named-volumes推荐)
- [方式二：Bind Mounts](#方式二bind-mounts)
- [APISIX 配置初始化](#apisix-配置初始化)
- [APISIX Admin Key 配置](#apisix-admin-key-配置)
- [APISIX 路由配置](#apisix-路由配置)
- [域名配置（v1.0.4+）](#域名配置v104)
- [部署验证](#部署验证)
- [常见问题](#常见问题)

---

## 从零开始部署（无需 git clone）

如果您在服务器上无法或不想 clone 整个仓库，可以使用以下步骤直接下载必要的脚本和配置文件。

### 目录说明

本文档涉及两个不同用途的目录，请注意区分：

| 目录 | 用途 | 说明 |
|------|------|------|
| `~/uhdadmin-deploy` | **工作目录** | 存放脚本、.env、stack.yml 等部署文件，可以是任意位置 |
| `/opt/uhdadmin` | **数据目录** | 存放 PostgreSQL、Redis、APISIX 等服务的持久化数据 |

> **注意：** 工作目录 (`~/uhdadmin-deploy`) 可以自定义为任意路径；数据目录 (`/opt/uhdadmin`) 是 bind mount 挂载点，通过 `UHDADMIN_BASE_PATH` 环境变量配置。

### 快速开始（推荐）

使用一键部署脚本，自动完成所有初始化工作：

```bash
# 1. 创建工作目录
mkdir -p ~/uhdadmin-deploy && cd ~/uhdadmin-deploy

# 2. 下载一键部署脚本
# 公开仓库:
curl -sL https://raw.githubusercontent.com/fxxkrlab/UHDadmin/main/scripts/setup-all.sh -o setup-all.sh

# 私有仓库（需要 GitHub Token）:
curl -sL -H "Authorization: token YOUR_GITHUB_TOKEN" \
  https://raw.githubusercontent.com/fxxkrlab/UHDadmin/main/scripts/setup-all.sh -o setup-all.sh

# 3. 运行一键部署脚本
sudo bash setup-all.sh
```

脚本会引导您完成：
- 创建数据目录 (`/opt/uhdadmin/`)
- 从 Docker 镜像提取 APISIX 配置
- 配置 APISIX Admin Key（自动生成或自定义）
- 修复 `config-default.yaml` 中的网络配置

### 手动下载脚本

如果需要单独下载各个脚本：

```bash
# 创建脚本目录
mkdir -p ~/uhdadmin-deploy/scripts && cd ~/uhdadmin-deploy/scripts

# 下载所有脚本（公开仓库）
BASE_URL="https://raw.githubusercontent.com/fxxkrlab/UHDadmin/main/scripts"
curl -sL ${BASE_URL}/setup-all.sh -o setup-all.sh
curl -sL ${BASE_URL}/init-host-dirs.sh -o init-host-dirs.sh
curl -sL ${BASE_URL}/configure-apisix-key.sh -o configure-apisix-key.sh
curl -sL ${BASE_URL}/setup-apisix-routes.sh -o setup-apisix-routes.sh
chmod +x *.sh

# 下载所有脚本（私有仓库，需要 GitHub Token）
BASE_URL="https://raw.githubusercontent.com/fxxkrlab/UHDadmin/main/scripts"
TOKEN = "REDACTED"
for script in setup-all.sh init-host-dirs.sh configure-apisix-key.sh setup-apisix-routes.sh; do
  curl -sL -H "Authorization: token $TOKEN" ${BASE_URL}/$script -o $script
done
chmod +x *.sh
```

### 脚本说明

| 脚本 | 用途 | 何时使用 |
|------|------|----------|
| `setup-all.sh` | 一键交互式部署 | 首次部署，推荐 |
| `init-host-dirs.sh` | 创建数据目录 + 提取 APISIX 配置 | 仅初始化目录 |
| `configure-apisix-key.sh` | 配置 APISIX Admin Key | 更改 Admin Key |
| `setup-apisix-routes.sh` | 配置 APISIX 路由 | Stack 部署后 |

### 下载 Stack 配置文件

```bash
cd ~/uhdadmin-deploy

# 下载 Bind Mounts 版本 Stack（推荐）
curl -sL https://raw.githubusercontent.com/fxxkrlab/UHDadmin/main/deploy/portainer/stack.prod.bind.yml \
  -o stack.prod.bind.yml

# 或下载 Named Volumes 版本
curl -sL https://raw.githubusercontent.com/fxxkrlab/UHDadmin/main/deploy/portainer/stack.prod.yml \
  -o stack.prod.yml

# 下载环境变量模板
curl -sL https://raw.githubusercontent.com/fxxkrlab/UHDadmin/main/deploy/portainer/.env.example \
  -o .env.example
```

### 完整部署流程（从零开始）

```bash
# === 步骤 1: 准备环境 ===
mkdir -p ~/uhdadmin-deploy && cd ~/uhdadmin-deploy

# === 步骤 2: 下载脚本 ===
BASE_URL="https://raw.githubusercontent.com/fxxkrlab/UHDadmin/main"
curl -sL ${BASE_URL}/scripts/setup-all.sh -o setup-all.sh
chmod +x setup-all.sh

# === 步骤 3: 运行一键部署（交互式） ===
sudo bash setup-all.sh
# 按提示输入：
#   - 数据目录路径（默认 /opt/uhdadmin）
#   - APISIX Admin Key（自动生成或自定义）

# === 步骤 4: 登录 GHCR ===
echo "YOUR_GITHUB_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# === 步骤 5: 下载 Stack 配置 ===
curl -sL ${BASE_URL}/deploy/portainer/stack.prod.bind.yml -o stack.prod.bind.yml

# === 步骤 6: 配置环境变量 ===
cat > .env << 'EOF'
POSTGRES_USER=uhdadmin
POSTGRES_PASSWORD=YourStrongPassword123!
JWT_SECRET_KEY=your-jwt-secret-key-32chars-xxx
SECRET_KEY=your-app-secret-key-32chars-xxx
IMAGE_TAG=1.1.35
UHDADMIN_BASE_PATH=/opt/uhdadmin
EOF

# === 步骤 7: 部署 Stack ===
docker compose -f stack.prod.bind.yml up -d

# === 步骤 8: 等待服务就绪 ===
sleep 30

# === 步骤 9: 配置路由（使用 setup-all.sh 保存的 Admin Key） ===
# 从保存的配置中读取 Admin Key
source /opt/uhdadmin/.uhdadmin.conf

curl -sL ${BASE_URL}/scripts/setup-apisix-routes.sh -o setup-apisix-routes.sh
chmod +x setup-apisix-routes.sh
bash setup-apisix-routes.sh --admin-key "$APISIX_ADMIN_KEY"

# === 步骤 10: 验证部署 ===
curl http://localhost:9080/api/v1/public/version
```

---

## 前置要求

| 组件 | 版本要求 | 说明 |
|------|----------|------|
| Docker | 20.10+ | 容器运行时 |
| Docker Compose | v2.0+ | Stack 编排 |
| Portainer | 2.x（可选） | GUI 管理界面 |

**服务器配置建议：**
- CPU: 2+ cores
- RAM: 4GB+ (推荐 8GB)
- Disk: 20GB+ SSD

**使用的镜像：**

| 镜像 | 来源 | 说明 |
|------|------|------|
| `ghcr.io/fxxkrlab/uhdadmin-api` | GHCR | API 后端（需认证） |
| `ghcr.io/fxxkrlab/uhdadmin-vben` | GHCR | 管理后台（需认证） |
| `ghcr.io/fxxkrlab/uhdadmin-portal` | GHCR | 用户门户（需认证） |
| `postgres:15-alpine` | Docker Hub | PostgreSQL 数据库 |
| `redis:7-alpine` | Docker Hub | Redis 缓存 |
| `quay.io/coreos/etcd:v3.5.12` | Quay.io | etcd 配置存储（官方镜像） |
| `apache/apisix:3.8.0-debian` | Docker Hub | APISIX 网关 |

---

## 系统配置

部署前需要调整系统参数以支持高并发连接。

### 文件描述符限制（ulimit）

APISIX 和 PostgreSQL 需要较高的文件描述符限制。如果看到 `open file descriptors` 警告，需要调整：

```bash
# 临时设置（当前会话）
ulimit -n 65535

# 永久设置
sudo tee -a /etc/security/limits.conf << 'EOF'
* soft nofile 65535
* hard nofile 65535
root soft nofile 65535
root hard nofile 65535
EOF

# 对于 systemd 管理的 Docker，还需要：
sudo mkdir -p /etc/systemd/system/docker.service.d/
sudo tee /etc/systemd/system/docker.service.d/override.conf << 'EOF'
[Service]
LimitNOFILE=65535
EOF

# 重载配置并重启 Docker
sudo systemctl daemon-reload
sudo systemctl restart docker

# 验证设置
ulimit -n  # 应显示 65535
```

### 端口规划

| 端口 | 服务 | 说明 |
|------|------|------|
| 9080 | APISIX | HTTP 入口（外部访问） |
| 9443 | APISIX | HTTPS 入口（可选） |
| 9180 | APISIX Admin | Admin API（仅内部） |
| 8000 | API | 后端 API（通过 APISIX 访问） |

> **注意：** 如果使用 Portainer，其默认使用 9443 端口。建议设置 `APISIX_HTTPS_PORT=8443` 避免冲突。

---

## GHCR 镜像仓库认证

UHDadmin 镜像托管在 GitHub Container Registry (GHCR)，部署前需要配置认证。

### 方法一：服务器命令行登录（推荐）

```bash
# 使用 GitHub Personal Access Token 登录
echo "YOUR_GITHUB_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# 验证登录成功
docker pull ghcr.io/fxxkrlab/uhdadmin-api:1.1.35
```

**生成 GitHub Token：**
1. 访问 GitHub → Settings → Developer settings → Personal access tokens → **Tokens (classic)**
2. 点击 **Generate new token (classic)**
3. 勾选 `read:packages` 权限
4. 生成并复制 Token

### 方法二：Portainer 配置 Registry

```
1. 登录 Portainer
2. 进入 Registries → Add registry
3. 选择 "Custom registry"
4. 填写：
   ┌─────────────────────┬──────────────────────────────────┐
   │ Name                │ ghcr                             │
   │ Registry URL        │ ghcr.io                          │
   │ Authentication      │ ✅ 启用                          │
   │ Username            │ 你的 GitHub 用户名                │
   │ Password            │ GitHub Personal Access Token     │
   └─────────────────────┴──────────────────────────────────┘
5. 点击 "Add registry"
6. 返回 Stacks 重新部署
```

---

## 必填环境变量

部署时必须在 Portainer 或 `.env` 文件中设置以下变量：

| 变量 | 说明 | 示例 |
|------|------|------|
| `POSTGRES_USER` | 数据库用户名 | `uhdadmin` |
| `POSTGRES_PASSWORD` | 数据库密码（强密码） | `MyStr0ngP@ssw0rd!` |
| `JWT_SECRET_KEY` | JWT 签名密钥（32+ 字符） | `your-random-jwt-secret-key-here-32` |
| `SECRET_KEY` | 应用加密密钥（32+ 字符） | `your-random-app-secret-key-here-32` |

**可选变量（有默认值）：**

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `IMAGE_TAG` | `latest` | 镜像版本，建议使用 `stable` 或具体版本如 `1.1.35` |
| `POSTGRES_DB` | `uhdadmin` | 数据库名 |
| `REDIS_PASSWORD` | 空 | Redis 密码（可不设） |
| `GHCR_REGISTRY` | `ghcr.io` | 镜像仓库地址 |
| `GHCR_ORG` | `fxxkrlab` | GHCR 组织名 |
| `UHDADMIN_BASE_PATH` | `/opt/uhdadmin` | Bind Mounts 基础路径 |

**生成随机密钥：**
```bash
# Linux/macOS
openssl rand -hex 32

# 或者
head -c 32 /dev/urandom | base64
```

---

## 方式一：Named Volumes（推荐）

**优点：** 零准备、Docker 自动管理、部署最简单

**文件：** `deploy/portainer/stack.prod.yml`

### 步骤

#### 1. 通过 Portainer 部署

```
1. 登录 Portainer
2. 进入 Stacks → Add stack
3. 命名为 "uhdadmin"
4. Web editor 中粘贴 stack.prod.yml 的完整内容
5. 在 Environment variables 区域添加：
   ┌─────────────────────┬──────────────────────────────────┐
   │ Name                │ Value                            │
   ├─────────────────────┼──────────────────────────────────┤
   │ POSTGRES_USER       │ uhdadmin                         │
   │ POSTGRES_PASSWORD   │ YourStrongPassword123!           │
   │ JWT_SECRET_KEY      │ your-jwt-secret-key-32chars-xxx  │
   │ SECRET_KEY          │ your-app-secret-key-32chars-xxx  │
   │ IMAGE_TAG           │ 1.1.35                           │
   └─────────────────────┴──────────────────────────────────┘
6. 点击 Deploy the stack
```

#### 2. 通过命令行部署

```bash
# 创建 .env 文件
cat > .env << 'EOF'
POSTGRES_USER=uhdadmin
POSTGRES_PASSWORD=YourStrongPassword123!
JWT_SECRET_KEY=your-jwt-secret-key-32chars-xxx
SECRET_KEY=your-app-secret-key-32chars-xxx
IMAGE_TAG=1.1.35
EOF

# 部署
docker compose -f deploy/portainer/stack.prod.yml up -d
```

### 访问 Named Volume 数据

```bash
# 列出所有 UHDadmin volumes
docker volume ls | grep uhdadmin

# 查看 volume 详情
docker volume inspect uhdadmin_postgres_data

# 读取 volume 中的文件
docker run --rm -v uhdadmin_apisix_conf:/conf alpine cat /conf/config.yaml

# 复制文件到 volume
docker run --rm -v uhdadmin_apisix_conf:/conf -v $(pwd):/src alpine \
  cp /src/config.yaml /conf/
```

---

## 方式二：Bind Mounts

**优点：** 直接访问主机目录、方便 vim/rsync 操作、路径直观

**文件：** `deploy/portainer/stack.prod.bind.yml`

### 步骤

#### 1. 服务器上创建目录（推荐使用脚本）

**方式 A：使用初始化脚本（推荐）**

```bash
# 克隆仓库或下载脚本
git clone https://github.com/fxxkrlab/UHDadmin.git
cd UHDadmin

# 预览将要创建的内容
bash scripts/init-host-dirs.sh --dry-run

# 执行初始化（需要 sudo）
sudo bash scripts/init-host-dirs.sh

# 脚本会自动：
# - 创建所有必要目录
# - 复制 APISIX config.yaml
# - 从 Docker 镜像提取 config-default.yaml、mime.types、cert 目录
# - 设置正确的权限
```

**方式 B：手动创建目录**

```bash
# 创建完整目录结构
sudo mkdir -p /opt/uhdadmin/{pg_data,redis_data,etcd_data,apisix/conf,apisix/logs,api/logs,data}

# 设置基础权限
sudo chown -R 1000:1000 /opt/uhdadmin

# PostgreSQL 需要 700 权限
sudo chmod 700 /opt/uhdadmin/pg_data

# etcd (官方镜像) 以 root 运行
sudo chmod 755 /opt/uhdadmin/etcd_data

# APISIX 需要写入权限
sudo chmod 777 /opt/uhdadmin/apisix/conf
sudo chmod 777 /opt/uhdadmin/apisix/logs
```

#### 2. 初始化 APISIX 配置（如果未使用脚本）

APISIX 需要多个配置文件才能启动。使用 bind mount 时这些文件不会自动存在，需要从 Docker 镜像提取：

```bash
# 提取 config-default.yaml 并修复 etcd 地址
docker run --rm apache/apisix:3.8.0-debian cat /usr/local/apisix/conf/config-default.yaml \
  | sed 's|http://203.0.113.10:2379|http://etcd:2379|g' \
  | sudo tee /opt/uhdadmin/apisix/conf/config-default.yaml > /dev/null

# 提取 mime.types
docker run --rm apache/apisix:3.8.0-debian cat /usr/local/apisix/conf/mime.types \
  | sudo tee /opt/uhdadmin/apisix/conf/mime.types > /dev/null

# 提取 cert 目录
sudo mkdir -p /opt/uhdadmin/apisix/conf/cert
docker run --rm apache/apisix:3.8.0-debian tar -cf - -C /usr/local/apisix/conf cert \
  | sudo tar -xf - -C /opt/uhdadmin/apisix/conf/

# 复制 config.yaml（从仓库或手动创建）
sudo cp deploy/apisix/apisix_conf/config.yaml /opt/uhdadmin/apisix/conf/

# 设置权限（APISIX 需要写入 nginx.conf）
sudo chmod 777 /opt/uhdadmin/apisix/conf
sudo chmod 777 /opt/uhdadmin/apisix/logs
```

#### 3. 验证目录结构

```bash
tree /opt/uhdadmin
# 应该显示：
# /opt/uhdadmin/
# ├── pg_data/
# ├── redis_data/
# ├── etcd_data/
# ├── apisix/
# │   ├── conf/
# │   │   ├── cert/
# │   │   │   ├── apisix.crt
# │   │   │   └── apisix.key
# │   │   ├── config.yaml
# │   │   ├── config-default.yaml
# │   │   └── mime.types
# │   └── logs/
# ├── api/
# │   └── logs/
# └── data/
```

#### 4. 通过 Portainer 部署

```
1. 登录 Portainer
2. 进入 Stacks → Add stack
3. 命名为 "uhdadmin"
4. Web editor 中粘贴 stack.prod.bind.yml 的完整内容
5. 在 Environment variables 区域添加：
   ┌─────────────────────┬──────────────────────────────────┐
   │ Name                │ Value                            │
   ├─────────────────────┼──────────────────────────────────┤
   │ POSTGRES_USER       │ uhdadmin                         │
   │ POSTGRES_PASSWORD   │ YourStrongPassword123!           │
   │ JWT_SECRET_KEY      │ your-jwt-secret-key-32chars-xxx  │
   │ SECRET_KEY          │ your-app-secret-key-32chars-xxx  │
   │ IMAGE_TAG           │ 1.1.35                           │
   │ UHDADMIN_BASE_PATH  │ /opt/uhdadmin                    │
   └─────────────────────┴──────────────────────────────────┘
6. 点击 Deploy the stack
```

#### 5. 通过命令行部署

```bash
# 创建 .env 文件
cat > .env << 'EOF'
POSTGRES_USER=uhdadmin
POSTGRES_PASSWORD=YourStrongPassword123!
JWT_SECRET_KEY=your-jwt-secret-key-32chars-xxx
SECRET_KEY=your-app-secret-key-32chars-xxx
IMAGE_TAG=1.1.35
UHDADMIN_BASE_PATH=/opt/uhdadmin
EOF

# 部署
docker compose -f deploy/portainer/stack.prod.bind.yml up -d
```

### 直接操作文件

```bash
# 查看日志
tail -f /opt/uhdadmin/api/logs/*.log

# 编辑 APISIX 配置
sudo vim /opt/uhdadmin/apisix/conf/config.yaml

# 备份数据
sudo rsync -av /opt/uhdadmin/ /backup/uhdadmin/
```

---

## APISIX 配置初始化

无论使用哪种存储模式，APISIX 首次启动需要 `config.yaml`。

### Named Volumes 模式

```bash
# 方法一：通过临时容器写入
docker run --rm -v uhdadmin_apisix_conf:/conf alpine sh -c 'cat > /conf/config.yaml << "EOFCONFIG"
apisix:
  node_listen: 9080
  enable_ipv6: false
  enable_admin: true
  admin_key:
    - name: admin
      key: uhdadmin-apisix-key-2026
      role: admin
  admin_listen:
    ip: 203.0.113.10
    port: 9180

etcd:
  host:
    - "http://etcd:2379"
  prefix: "/apisix"
  timeout: 30

plugins:
  - real-ip
  - cors
  - proxy-rewrite
  - traffic-split
  - prometheus

plugin_attr:
  prometheus:
    export_addr:
      ip: 203.0.113.10
      port: 9091
EOFCONFIG'

# 方法二：从本地文件复制（如果有 repo）
docker run --rm \
  -v uhdadmin_apisix_conf:/conf \
  -v $(pwd)/deploy/apisix/apisix_conf/config.yaml:/src/config.yaml:ro \
  alpine cp /src/config.yaml /conf/
```

### Bind Mounts 模式

直接编辑 `/opt/uhdadmin/apisix/conf/config.yaml`（见上方步骤 2）。

---

## APISIX Admin Key 配置

**重要：** APISIX 的 Admin API Key 配置在 `config-default.yaml` 中生效，而非 `config.yaml`。这是因为 `config-default.yaml` 的设置会覆盖 `config.yaml` 中的同名配置。

### 为什么需要配置 Admin Key？

1. **安全性**：默认 Admin Key（`edd1c9f034335f136f87ad84b625c8f1`）是公开的，任何人都可以使用
2. **网络访问**：默认 `allow_admin` 仅允许 `203.0.113.10/24`，Docker 容器内的访问会被拒绝
3. **etcd 地址**：默认 etcd 地址为 `203.0.113.10:2379`，需要改为 `etcd:2379` 用于 Docker 网络

### 方式一：使用配置脚本（推荐）

```bash
# 下载脚本
curl -sL https://raw.githubusercontent.com/fxxkrlab/UHDadmin/main/scripts/configure-apisix-key.sh \
  -o configure-apisix-key.sh
chmod +x configure-apisix-key.sh

# 查看当前配置
bash configure-apisix-key.sh --show

# 生成并应用随机 Key
sudo bash configure-apisix-key.sh

# 或使用自定义 Key
sudo bash configure-apisix-key.sh --key my-custom-admin-key

# 重启 APISIX 使配置生效
docker restart <apisix-container-name>
```

### 方式二：手动修改 config-default.yaml

```bash
# 1. 修改 allow_admin（允许所有 IP 访问 Admin API）
sudo sed -i 's|203.0.113.10/24|203.0.113.10/0|g' /opt/uhdadmin/apisix/conf/config-default.yaml

# 2. 修改 etcd 地址（用于 Docker 网络）
sudo sed -i 's|http://203.0.113.10:2379|http://etcd:2379|g' /opt/uhdadmin/apisix/conf/config-default.yaml

# 3. 替换默认 Admin Key
sudo sed -i 's|edd1c9f034335f136f87ad84b625c8f1|YOUR_CUSTOM_KEY|g' /opt/uhdadmin/apisix/conf/config-default.yaml

# 4. （可选）替换默认 Viewer Key
sudo sed -i 's|4054f7cf07e344346cd3f287985e76a2|YOUR_VIEWER_KEY|g' /opt/uhdadmin/apisix/conf/config-default.yaml

# 5. 重启 APISIX
docker restart <apisix-container-name>
```

### 配置说明

| 配置项 | 默认值 | 推荐值 | 说明 |
|--------|--------|--------|------|
| `admin_key` (admin) | `edd1c9f034335f136f87ad84b625c8f1` | 随机生成的 32 位 hex | 管理员权限 Key |
| `admin_key` (viewer) | `4054f7cf07e344346cd3f287985e76a2` | 随机生成的 32 位 hex | 只读权限 Key |
| `allow_admin` | `203.0.113.10/24` | `203.0.113.10/0` | 允许访问 Admin API 的 IP 范围 |
| `etcd.host` | `http://203.0.113.10:2379` | `http://etcd:2379` | etcd 服务地址 |

### 验证配置

```bash
# 测试 Admin API 访问
curl http://localhost:9180/apisix/admin/routes -H "X-API-KEY : REDACTED"

# 预期返回：{"total":0,"list":[]} 或路由列表
# 错误返回：{"error_msg":"failed to check token"} 表示 Key 错误
# 错误返回：HTTP 403 表示 IP 不在 allow_admin 范围内
```

### 常见错误排查

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `HTTP 403 Forbidden` | IP 不在 allow_admin 范围 | 修改 `allow_admin` 为 `203.0.113.10/0` |
| `failed to check token` | Admin Key 不匹配 | 确认使用正确的 Key（检查 config-default.yaml） |
| `connection refused` | APISIX 未启动或端口不对 | 检查容器状态和端口映射 |

---

## APISIX 路由配置

部署完成后，需要配置 APISIX 路由将流量转发到后端服务。

### 使用路由配置脚本（推荐）

```bash
# 预览路由配置
bash scripts/setup-apisix-routes.sh --dry-run

# 配置路由（使用默认 blue 槽）
bash scripts/setup-apisix-routes.sh

# 或指定 Admin URL（如果通过 SSH 隧道访问）
bash scripts/setup-apisix-routes.sh --admin-url http://localhost:9180

# 切换到 green 槽
bash scripts/setup-apisix-routes.sh --active-slot green
```

### 路由说明

脚本会自动配置以下路由：

| 路由 | 目标服务 | 说明 |
|------|----------|------|
| `/api/*` | `api-blue:8000` | 后端 API（去除 `/api` 前缀） |
| `/admin/*` | `vben-blue:80` | 管理后台（去除 `/admin` 前缀） |
| `/*` | `portal-blue:3000` | 门户网站（默认路由） |

### 手动配置路由

如果不使用脚本，可以通过 APISIX Admin API 手动配置：

```bash
# 创建 API upstream
curl -X PUT http://localhost:9180/apisix/admin/upstreams/api-blue \
  -H "X-API-KEY : REDACTED" \
  -H "Content-Type: application/json" \
  -d '{"nodes":{"api-blue:8000":1},"type":"roundrobin"}'

# 创建 API 路由
curl -X PUT http://localhost:9180/apisix/admin/routes/api \
  -H "X-API-KEY : REDACTED" \
  -H "Content-Type: application/json" \
  -d '{
    "uri": "/api/*",
    "upstream_id": "api-blue",
    "priority": 10,
    "plugins": {
      "proxy-rewrite": {"regex_uri": ["^/api/(.*)", "/$1"]}
    }
  }'

# 类似地配置 vben 和 portal...
```

### 验证路由

```bash
# 查看所有路由
curl http://localhost:9180/apisix/admin/routes \
  -H "X-API-KEY : REDACTED"

# 测试 API 路由
curl http://localhost:9080/api/v1/public/version

# 测试管理后台
curl http://localhost:9080/admin/

# 测试门户
curl http://localhost:9080/
```

---

## 域名配置（v1.0.4+）

v1.0.4 起，系统支持**交互式域名配置**，通过 Setup Wizard 一键完成域名和 CORS 设置。

### 通过 Setup Wizard 配置（推荐）

1. 访问 `https://your-domain.com/setup`（仅首次部署可用）
2. **第一步**：填写域名配置
   - API 域名：`api.example.com`
   - 管理后台域名：`admin.example.com`
   - 门户域名：`portal.example.com`（支持多个，逗号分隔）
   - 应用域名：`app.example.com`
3. **第一步**：配置 CORS 白名单（默认自动派生）
4. 点击「**应用到 APISIX**」按钮，自动创建网关路由
5. **第二步**：创建 Sysop 管理员账号

### 通过 API 配置

```bash
# 获取当前域名配置
curl http://localhost:8000/api/v1/setup/domains \
  -H "Authorization: Bearer $TOKEN"

# 更新域名配置
curl -X POST http://localhost:8000/api/v1/setup/domains \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "api_host": "api.example.com",
    "admin_host": "admin.example.com",
    "portal_hosts": ["portal.example.com"],
    "app_host": "app.example.com",
    "cors_origins": ["https://portal.example.com", "https://admin.example.com"]
  }'

# 应用到 APISIX
curl -X POST http://localhost:8000/api/v1/setup/domains/apply-apisix \
  -H "Authorization: Bearer $TOKEN"
```

### 域名配置优先级

| 优先级 | 来源 | 说明 |
|:------:|------|------|
| 1 | 数据库 | `system_settings` 表，Setup Wizard 保存 |
| 2 | boot-config | `/data/boot-config.json` 文件 |
| 3 | 环境变量 | `ROOT_DOMAIN`、`API_HOST` 等 |
| 4 | 自动派生 | 根据访问域名自动计算 |

### boot-config.json 示例

部署前可预配置 `/data/boot-config.json`：

```json
{
  "domain_api": "api.example.com",
  "domain_admin": "admin.example.com",
  "domain_portal": "portal.example.com,www.example.com",
  "domain_app": "app.example.com",
  "cors_origins": "https://portal.example.com,https://admin.example.com",
  "apisix_admin_url": "http://apisix:9180",
  "apisix_admin_key": "your-apisix-admin-key"
}
```

### CORS 白名单说明

CORS 配置支持动态白名单，**不允许回退到 `*` 通配符**：

1. 首先从数据库 `system_settings.cors_origins` 读取
2. 其次从 `/data/boot-config.json` 的 `cors_origins` 读取
3. 最后从环境变量 `CORS_ORIGINS` 读取
4. 若都为空，则返回 403 错误

---

## 部署验证

### 检查容器状态

```bash
# 查看所有容器
docker ps --filter "name=uhdadmin"

# 预期输出：所有容器状态为 Up，健康状态为 (healthy)
```

### 检查服务健康

```bash
# API 健康检查
curl http://localhost:8000/health

# APISIX 状态
curl http://localhost:9080/apisix/status

# 版本信息
curl http://localhost:8000/api/v1/public/version
```

### 检查日志

```bash
# API 日志
docker logs uhdadmin-api-blue-1 --tail 50

# APISIX 日志
docker logs uhdadmin-apisix-1 --tail 50

# 数据库日志
docker logs uhdadmin-postgres-1 --tail 50
```

---

## 常见问题

### Q: 部署失败 "unauthorized" 错误

**A:** 需要先登录 GHCR 镜像仓库，参考 [GHCR 镜像仓库认证](#ghcr-镜像仓库认证)。

```bash
echo "YOUR_GITHUB_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

### Q: 部署失败 "POSTGRES_USER is required" 错误

**A:** 必填环境变量未设置。在 Portainer Stack 部署页面的 "Environment variables" 区域添加：
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `JWT_SECRET_KEY`
- `SECRET_KEY`

详见 [必填环境变量](#必填环境变量)。

### Q: APISIX 启动失败，提示找不到 config.yaml

**A:** 需要初始化 APISIX 配置，参考 [APISIX 配置初始化](#apisix-配置初始化)。

### Q: PostgreSQL 权限错误

**A:** Bind Mounts 模式下确保目录权限：
```bash
sudo chmod 700 /opt/uhdadmin/pg_data
sudo chown -R 999:999 /opt/uhdadmin/pg_data  # PostgreSQL 官方镜像用 999
```

### Q: etcd 启动失败

**A:** 官方 etcd 镜像（quay.io/coreos/etcd）以 root 运行，通常不需要特殊权限：
```bash
sudo chmod 755 /opt/uhdadmin/etcd_data
```
如果仍有问题，检查目录是否存在且可写。

### Q: APISIX 启动失败，缺少 config-default.yaml / mime.types

**A:** 使用 bind mount 时需要手动提取这些文件：
```bash
# 提取 config-default.yaml 并修复 etcd 地址
docker run --rm apache/apisix:3.8.0-debian cat /usr/local/apisix/conf/config-default.yaml \
  | sed 's|http://203.0.113.10:2379|http://etcd:2379|g' \
  | sudo tee /opt/uhdadmin/apisix/conf/config-default.yaml > /dev/null

# 提取 mime.types
docker run --rm apache/apisix:3.8.0-debian cat /usr/local/apisix/conf/mime.types \
  | sudo tee /opt/uhdadmin/apisix/conf/mime.types > /dev/null

# 提取 cert 目录
docker run --rm apache/apisix:3.8.0-debian tar -cf - -C /usr/local/apisix/conf cert \
  | sudo tar -xf - -C /opt/uhdadmin/apisix/conf/
```

推荐使用 `scripts/init-host-dirs.sh` 脚本自动处理。

### Q: APISIX 报错 "Permission denied" 或无法写入 nginx.conf

**A:** APISIX 需要写入 conf 目录生成 nginx.conf：
```bash
sudo chmod 777 /opt/uhdadmin/apisix/conf
sudo chmod 777 /opt/uhdadmin/apisix/logs
```

### Q: APISIX 端口 9443 与 Portainer 冲突

**A:** Portainer 默认使用 9443 端口。在部署时设置环境变量：
```
APISIX_HTTPS_PORT=8443
```

### Q: 警告 "Current maximum number of open file descriptors is too low"

**A:** 参考 [系统配置 - 文件描述符限制](#文件描述符限制ulimit) 部分配置 ulimit。

### Q: Vben 容器显示 unhealthy

**A:** v1.0.3 版本已修复健康检查问题。如果使用旧版本，可以忽略健康状态，实际服务正常运行。验证方式：
```bash
curl http://localhost:9080/admin/
```

### Q: 如何从 Named Volumes 迁移到 Bind Mounts？

**A:**
```bash
# 1. 导出 named volume 数据
docker run --rm -v uhdadmin_postgres_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/pg_backup.tar.gz -C /data .

# 2. 创建 bind mount 目录
sudo mkdir -p /opt/uhdadmin/pg_data

# 3. 导入数据
sudo tar xzf pg_backup.tar.gz -C /opt/uhdadmin/pg_data

# 4. 设置权限
sudo chmod 700 /opt/uhdadmin/pg_data

# 5. 重新部署 stack.prod.bind.yml
```

### Q: 如何更新镜像版本？

**A:** 在 Portainer 中修改 `IMAGE_TAG` 环境变量，然后 Update the stack。

---

## 两种模式对比

| 特性 | Named Volumes | Bind Mounts |
|------|---------------|-------------|
| 部署复杂度 | ⭐ 简单 | ⭐⭐ 需要预创建目录 |
| 数据访问 | 需要 docker 命令 | 直接 vim/cat |
| 备份方式 | docker 导出 | rsync/cp |
| 路径可预测性 | /var/lib/docker/volumes/ | 自定义路径 |
| 推荐场景 | 首次部署、快速测试 | 生产环境、需要直接管理 |

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `deploy/portainer/stack.prod.yml` | Named Volumes 版本 Stack |
| `deploy/portainer/stack.prod.bind.yml` | Bind Mounts 版本 Stack |
| `deploy/portainer/.env.example` | 环境变量模板 |
| `deploy/apisix/apisix_conf/config.yaml` | APISIX 配置模板 |
| `scripts/setup-all.sh` | **一键交互式部署脚本**（v1.0.4 新增） |
| `scripts/init-host-dirs.sh` | 主机目录初始化脚本 |
| `scripts/configure-apisix-key.sh` | **APISIX Admin Key 配置脚本**（v1.0.4 新增） |
| `scripts/setup-apisix-routes.sh` | APISIX 路由配置脚本 |
| `docs/DEPLOY_RUNBOOK.md` | 部署运维手册 |

---

---

## 已有部署更新脚本

> **适用场景**：已通过 Docker 部署 UHDadmin，需要更新 `apply-routes.sh` 等脚本（例如修复 DEPLOY-11C2 的 IP 访问问题）。

### 问题背景

v1.0.15 之前版本的 `apply-routes.sh` 需要预先配置域名才能运行。这导致首次部署时出现「鸡生蛋」问题：
- Setup Wizard 需要 APISIX 路由才能正常工作
- 但旧版脚本需要域名配置才能创建路由
- 通过 IP 地址访问时，`/api/_nuxt_icon/*` 返回 404

v1.0.15+ 的 `apply-routes.sh` 支持 **Hostless 模式**，可以在无域名配置时创建基础路由。

### 下载最新 apply-routes.sh

#### 方式一：使用 GitHub CLI（推荐）

```bash
# 登录 GitHub（如未登录）
gh auth login

# 下载最新版本
gh api repos/fxxkrlab/UHDadmin/contents/deploy/apisix/apply-routes.sh \
  --jq '.content' | base64 -d > apply-routes.sh

chmod +x apply-routes.sh
```

#### 方式二：使用 curl + GitHub Token

```bash
# 设置 Token（需要 repo 权限）
export GITHUB_TOKEN="GITHUB_TOKEN_REDACTED"

# 下载
curl -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3.raw" \
  "https://api.example.com/repos/fxxkrlab/UHDadmin/contents/deploy/apisix/apply-routes.sh" \
  -o apply-routes.sh

chmod +x apply-routes.sh
```

#### 方式三：从 Release 下载 deploy-bundle

```bash
# 下载最新 release 的部署包
gh release download --repo fxxkrlab/UHDadmin --pattern "deploy-bundle*"

# 解压并复制
unzip deploy-bundle.zip
cp deploy-bundle/deploy/apisix/apply-routes.sh ./
chmod +x apply-routes.sh
```

### 运行脚本

```bash
# 首次部署（无域名配置）- Hostless 模式
# 脚本会自动创建 IP 访问所需的路由
./apply-routes.sh

# 已有域名配置时 - 完整模式
# 从环境变量读取
ROOT_DOMAIN=example.com ./apply-routes.sh

# 或从 boot-config.json 读取（如已配置）
./apply-routes.sh
```

### Hostless 模式说明

当没有域名配置时，脚本会创建以下 **Hostless 路由**（无 host 限制，可通过 IP 访问）：

| 路由名称 | URI 模式 | 优先级 | 目标服务 |
|----------|----------|:------:|----------|
| `portal-nuxt-icon-hostless` | `/api/_nuxt_icon/*` | 100 | portal-blue |
| `api-proxy-hostless` | `/api/*` | 50 | api-blue |
| `portal-hostless` | `/*` | -1 | portal-blue |

这些路由确保：
1. Setup Wizard 页面可以正常加载（`/api/*` 代理到 API 后端）
2. Nuxt 图标正确渲染（`/api/_nuxt_icon/*` 代理到 Portal）
3. 其他页面正常访问（`/*` fallback 到 Portal）

### 验证路由

```bash
# 检查 Nuxt 图标路由
curl -I http://localhost:9080/api/_nuxt_icon/ic/fluent-emoji-flat/check-mark-button
# 预期：200 OK

# 检查 Setup 状态 API
curl http://localhost:9080/api/v1/setup/state
# 预期：返回 JSON 状态

# 检查 Setup 页面
curl -I http://localhost:9080/setup
# 预期：200 OK
```

---

## 版本历史

### v1.1.31 (2026-02-07)

**新功能 — CDKEY 系统统一化 (Round-CDKEY-UNIFY)：**
- Phase 1: Admin DevOps 3 页面合并 → 「CDKEY 管理中心」3-Tab 布局 + 池内批量派发 API
- Phase 2: Portal 3 资产页面合并 → 「我的资产」+ 跨模型聚合 API (`/user/assets/summary`, `/list`)
- Phase 3: Filter Chips + 列表/网格双视图 + Miniapp 网格适配 + 续期卡功能页

**安全增强：**
- Session Auth: Refresh token 续期优化 + JTI 白名单严格校验

**Bug 修复（3 项）：**
- Miniapp Date() 时区回归修复
- 媒体账号页面交互增强
- 积分评估服务时区修正

**镜像标签：**
```bash
ghcr.io/fxxkrlab/uhdadmin-api:1.1.31
ghcr.io/fxxkrlab/uhdadmin-vben:1.1.31
ghcr.io/fxxkrlab/uhdadmin-portal:1.1.31
```

### v1.1.30 (2026-02-07)

**Bug 修复（7 项）：**
- #135 商城商品定价模式翻译错误: credits→余额, points→积分, 5 处统一修正
- #136 定时上架/下架改为日历选择器: Input → DatePicker showTime
- #137 积分商品购买弹窗显示无关支付方式: 按 currency_type 过滤
- #138/#139 积分购买 500 错误: `user.points` → `user.point` 属性名修正
- #140 CDKEY详情字段显示错误: owner/redeemer 区分显示，新增使用者列
- #141 运行日志 User 字段为空: RequestLoggingMiddleware 改为从 JWT Authorization 头解码用户名，无 token 记录 "Guest"

**增强（2 项）：**
- 敏感操作审计日志增强: 管理员修改积分/等级/状态写入 AuditEvent (old_value → new_value + delta)，商城购买写入 SHOP_PURCHASE 审计
- 全站时区统一 Asia/Shanghai: Vben 31 处 + Portal 12 处 `toLocaleString` 统一添加 `{ timeZone: 'Asia/Shanghai' }`

**镜像标签：**
```bash
ghcr.io/fxxkrlab/uhdadmin-api:1.1.30
ghcr.io/fxxkrlab/uhdadmin-vben:1.1.30
ghcr.io/fxxkrlab/uhdadmin-portal:1.1.30
```

### v1.1.29 (2026-02-07)

**Bug 修复（6 项）：**
- 席位卡指定媒体服务器未解析: CDKEY 配置 `media_service_ids` 未被兑换逻辑读取，创建席位时丢失服务器限制 → 后端+前端修复
- #23 席位页面按钮重复与链接错误: 移除多余按钮，统一跳转至 `/portal/shop`
- #28 续期卡 Buy Cards 按钮无法使用: 系统设置默认值导致永久禁用 → 改为 "Go to Shop" 跳转按钮
- #33 商城页面积分显示不同步: `/user/points/balance` 端点不存在 → 改用 `/user/me`
- #20 钱包页面布局重构: 充值按钮移至余额卡片，「商店」改为跳转「商城中心」，删除冗余标签
- #22/#85 Miniapp 售后下拉为空: 4 个 API 路径缺少 `/user/` 前缀导致 404

**镜像标签：**
```bash
# 最新稳定版
ghcr.io/fxxkrlab/uhdadmin-api:1.1.29
ghcr.io/fxxkrlab/uhdadmin-vben:1.1.29
ghcr.io/fxxkrlab/uhdadmin-portal:1.1.29

# 或使用 stable/latest 标签
ghcr.io/fxxkrlab/uhdadmin-api:stable
ghcr.io/fxxkrlab/uhdadmin-api:latest
```

### v1.1.28 (2026-02-07)

**Bug 修复：**
- CDKEY 详情页 500 错误: `cdkeys.py` 延迟导入模块名错误 (`app.example.com` → 删除，使用顶部已有的 `app.example.com`)
- 媒体访问控制「配置方案」500 错误: `MediaConfigProfile` 模型未在 `__init__.py` 中注册，Tortoise ORM 找不到连接
- 席位卡下拉显示: CDKEY 兑换的席位显示 `cdkey_redeem` → 改为显示实际 CDKEY 号码 (`seat.meta.cdkey_code`)

**镜像标签：**
```bash
# 最新稳定版
ghcr.io/fxxkrlab/uhdadmin-api:1.1.28
ghcr.io/fxxkrlab/uhdadmin-vben:1.1.28
ghcr.io/fxxkrlab/uhdadmin-portal:1.1.28

# 或使用 stable/latest 标签
ghcr.io/fxxkrlab/uhdadmin-api:stable
ghcr.io/fxxkrlab/uhdadmin-api:latest
```

### v1.1.27 (2026-02-06)

**新功能：**
- **A+B Session Auth 体系 (Phase 1-5)**:
  - Phase 1: Refresh Token (15min access + 7d refresh) + JTI 白名单 (敏感操作校验) + 平台会话追踪
  - Phase 2: 管理 API (9 端点: 会话列表/踢人/统计/Auth 设置/用户自助)
  - Phase 3: Miniapp refresh 拦截器 + 全路径 token 存储 (9 个 token 获取点)
  - Phase 4: Vben 管理页面 (认证与会话 4 Tab: 在线会话/统计/历史/参数配置)
  - Phase 5: 会话清理定时任务 (每日 03:00) + 异常检测 (多 IP/过多活跃会话告警)

**Bug 修复：**
- CDKEY 兑换 500 错误: `fetch_related("roles")` → `fetch_related("user_roles")`
- 商城菜单消失: customer 角色补全 `view_shop` 等 18 项权限
- 席位卡创建账号逻辑漏洞: 前端强制选座 + 后端 `seat_id` 必填 + 有效期从席位卡解析
- 老用户认领时区错误: UTC → `Asia/Shanghai` 统一时区处理

**数据库迁移：**
- `migrations/394_session_auth_user_sessions.sql` — 新增 `user_sessions` 表 + `rolesDB.max_platform_sessions` 字段

**镜像标签：**
```bash
# 最新稳定版
ghcr.io/fxxkrlab/uhdadmin-api:1.1.27
ghcr.io/fxxkrlab/uhdadmin-vben:1.1.27
ghcr.io/fxxkrlab/uhdadmin-portal:1.1.27

# 或使用 stable/latest 标签
ghcr.io/fxxkrlab/uhdadmin-api:stable
ghcr.io/fxxkrlab/uhdadmin-api:latest
```

### v1.1.26 (2026-02-06)

**新功能：**
- Round-DEVOPS-02: 独立 CDKEY 追踪页面 + 用户派发智能搜索增强
- BUGFIX-053: 新版本上线通知（`useVersionCheck` composable，60s 轮询）

**性能优化：**
- BUGFIX-065: `/api/v1/setup/state` 三层缓存（内存→Redis→DB）+ 前端 3s 超时保护
- BUGFIX-069: Profile 页面并发请求拆分（5→2 批，峰值降至 2-3）

**Bug 修复（25个）：**
- BUGFIX-017: 5 个注册/认领端点自动绑定角色
- BUGFIX-033~051: 签到积分同步、席位数统计、续卡 401、席位选择器、工单/售后加载、MiniApp 页面空白、科学计数法、元素尺寸、推广链接、工单附件
- BUGFIX-055~063: 签到/钱包/CDKEY 主题配色统一、图标映射、观看历史报错、媒体账号 500、分页组件条件渲染、菜单 API 去重
- BUGFIX-064~070: Footer 定位、版本号移除、Miniapp 更换密码+2299日期、Token 过期导航

**镜像标签：**
```bash
ghcr.io/fxxkrlab/uhdadmin-api:1.1.26
ghcr.io/fxxkrlab/uhdadmin-vben:1.1.26
ghcr.io/fxxkrlab/uhdadmin-portal:1.1.26
```

### v1.1.25 (2026-02-06)

**Bug 修复：**
- Portal 菜单缺失修复：添加 `watch-history`、`display-preferences`、`benefits` 到 UserCenter 菜单

**优化改进：**
- 统一 Points/Credits 中文命名：`points` = 积分，`credits` = 余额
- Portal/Admin 全系统 UI 文案中文化

**镜像标签：**
```bash
# 最新稳定版
ghcr.io/fxxkrlab/uhdadmin-api:1.1.25
ghcr.io/fxxkrlab/uhdadmin-vben:1.1.25
ghcr.io/fxxkrlab/uhdadmin-portal:1.1.25

# 或使用 stable/latest 标签
ghcr.io/fxxkrlab/uhdadmin-api:stable
ghcr.io/fxxkrlab/uhdadmin-api:latest
```

### v1.1.24 (2026-02-06)

**Bug 修复：**
- Issue #3: MiniApp Toast 不显示（添加 `<UNotifications />` 组件）
- Issue #30: 角色权限复选框跨分类清除 Bug（独立 Checkbox 替代 Checkbox.Group）
- Issue #31: 工单图片不显示（后端 attachments 字段保存 + 图片预览）

**镜像标签：**
```bash
# 最新稳定版
ghcr.io/fxxkrlab/uhdadmin-api:1.1.24
ghcr.io/fxxkrlab/uhdadmin-vben:1.1.24
ghcr.io/fxxkrlab/uhdadmin-portal:1.1.24

# 或使用 stable 标签
ghcr.io/fxxkrlab/uhdadmin-api:stable
```

### v1.0.15 (2026-01-22)

**DEPLOY-11C2 修复：**
- `apply-routes.sh` 支持 Hostless 模式，解决首次部署时的「鸡生蛋」问题
- 无需预配置域名即可运行脚本
- 自动创建 IP 访问所需的基础路由（`portal-nuxt-icon-hostless`、`api-proxy-hostless`、`portal-hostless`）

### v1.0.4 (2026-01-19)

**新增脚本：**
- `scripts/setup-all.sh` - 一键交互式部署脚本，自动完成目录创建、APISIX 配置、Admin Key 设置
- `scripts/configure-apisix-key.sh` - 独立的 APISIX Admin Key 配置脚本

**Bug 修复：**
- 修复 APISIX `config.yaml` 中使用环境变量语法（`${VAR:-default}`）导致的问题（APISIX 不支持环境变量替换）
- 修复 `config-default.yaml` 中 `allow_admin: 203.0.113.10/24` 导致 Docker 网络无法访问 Admin API 的问题
- 修复 `config-default.yaml` 中 etcd 地址为 `203.0.113.10` 导致 Docker 网络连接失败的问题
- `init-host-dirs.sh` 现在会自动修复 `config-default.yaml` 中的网络配置

**文档更新：**
- 新增「从零开始部署」章节，支持无需 git clone 的 wget 部署方式
- 新增「APISIX Admin Key 配置」章节，详细说明 Admin Key 配置原理和方法
- 更新脚本列表和使用说明
