# Slave 分布式架构 — 配置拉取与多服务器代理

> v1.1.12 起支持多服务器合并配置。本文档描述 Slave Docker 的完整架构、配置拉取流程和 APP_TOKEN 认证机制。

---

## 1. 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    UHDadmin (主控端)                          │
│                    api.example.com:8000                           │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ 配置向导      │  │ Slave 管理   │  │ 配置生成器        │  │
│  │ ConfigWizard │  │ HostConfig   │  │ ConfigGenerator  │  │
│  │ (8步创建规则) │  │ Panel (CRUD) │  │ (nginx+lua渲染)  │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────────┘  │
│         │                 │                  │              │
│         ▼                 ▼                  ▼              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │               数据库 (Tortoise ORM)                   │  │
│  │  MediaConfigSnapshot  │  MediaServiceSlave           │  │
│  │  (共享规则: lua_config │  (每台服务器: host, domain,  │  │
│  │   nginx_config)       │   upstream, ssl_path...)     │  │
│  └──────────────────────────────────────────────────────┘  │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │        GET /api/v1/media-slave/config                 │  │
│  │        (App Token 认证)                               │  │
│  │                                                       │  │
│  │  Token → Slave → Host → 同Host所有Slave              │  │
│  │  → generate_multi_server_config()                     │  │
│  │  → 返回 rendered_nginx + rendered_lua                 │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────┬───────────────────────────────────────────┘
                  │  HTTPS (30秒轮询)
                  ▼
┌─────────────────────────────────────────────────────────────┐
│              Slave Docker (每台主机一个)                      │
│              UHDadmin-media-slave                            │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  init_worker.lua (Agent)                              │  │
│  │  每30秒:                                              │  │
│  │  1. GET /config/version → 有更新?                     │  │
│  │  2. GET /config → 拿到完整配置                        │  │
│  │  3. lua_config → shared_dict (内存)                   │  │
│  │  4. rendered_nginx → server.conf (磁盘)               │  │
│  │     同时清空 upstream.conf + maps.conf                │  │
│  │  5. rendered_lua → access.lua (磁盘, 可选)            │  │
│  │  6. openresty -t && openresty -s reload               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌────────────────────┐  ┌─────────────────────────────┐  │
│  │ OpenResty/Nginx     │  │ Redis                       │  │
│  │                     │  │ (rate_limit, sessions,      │  │
│  │ nginx.conf          │  │  enforcements)              │  │
│  │  ├─ upstream.conf   │  │                             │  │
│  │  ├─ maps.conf       │  │                             │  │
│  │  └─ server.conf     │  │                             │  │
│  │                     │  │                             │  │
│  │ lua/ (10 个模块)    │  │                             │  │
│  │  ├─ access.lua      │  │                             │  │
│  │  ├─ init_worker.lua │  │                             │  │
│  │  ├─ log_handler.lua │  │                             │  │
│  │  ├─ header_filter.lua│ │                             │  │
│  │  ├─ config.lua      │  │                             │  │
│  │  ├─ rate_limiter.lua│  │                             │  │
│  │  ├─ telemetry.lua   │  │                             │  │
│  │  ├─ redis_store.lua │  │                             │  │
│  │  ├─ session_checker.lua││                            │  │
│  │  └─ utils.lua       │  │                             │  │
│  └────────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. APP_TOKEN 认证流程

### 2.1 创建流程

```
管理员操作:
  1. 媒体中心 → 访问控制 → 主机配置 → 添加服务器
     → 数据库创建 MediaServiceSlave 记录
     → 填写: host(IP), server_name(域名), upstream(端口), SSL 路径

  2. 媒体中心 → Slave 状态 → 为该 Slave 创建 App Token
     → 得到 token 字符串: uhd_xxxxxxxxxxxxxxxx
     → scope: media_slave:{service_id}
```

### 2.2 Slave 端配置

```bash
# .env 文件
APP_TOKEN=uhd_xxxxxxxxxxxxxxxx       # 步骤2得到的token
API_BASE=https://api.example.com          # 主控API地址
REDIS_HOST=redis                      # Docker内部Redis
REDIS_PORT=6379
```

### 2.3 认证链路

```
docker-compose up
    │
    ▼
OpenResty 启动 → init_worker.lua
    │
    ▼
读取环境变量 APP_TOKEN + API_BASE
    │
    ▼
每30秒轮询:
    GET {API_BASE}/api/v1/media-slave/config/version
    Authorization: Bearer uhd_xxxxxxxx
    │
    ▼
主控验证:
    AppToken 表 → token_hash 匹配
    → scopes 包含 "media_slave"
    → 找到关联的 MediaServiceSlave
    → 检查 slave.enabled = true
    │
    ▼
返回配置版本号 + snapshot_id
```

---

## 3. 配置拉取完整流程

### 3.1 版本检查 (每30秒)

```
Slave Agent                              Master API
    │                                        │
    │  GET /config/version                   │
    │  Authorization: Bearer uhd_xxx         │
    ├───────────────────────────────────────>│
    │                                        │
    │  { version: 1706959200,                │
    │    has_update: true,                   │
    │    snapshot_id: 42 }                   │
    │<───────────────────────────────────────┤
    │                                        │
    │  (本地版本 < 返回版本 → 拉取完整配置)  │
    ▼                                        │
```

### 3.2 拉取完整配置

```
Slave Agent                              Master API
    │                                        │
    │  GET /config                           │
    │  Authorization: Bearer uhd_xxx         │
    ├───────────────────────────────────────>│
    │                                        │
    │  Master 内部处理:                       │
    │  1. Token → Slave (ID=1)               │
    │  2. Slave.host = 203.0.113.10         │
    │  3. 查找同host同service_type的Slave    │
    │     → [ID=1, ID=2, ID=3]              │
    │  4. generate_multi_server_config()     │
    │                                        │
    │  返回:                                 │
    │  { version: 1706959200,                │
    │    mode: "multi_server",               │
    │    rendered_nginx: "upstream...\n...",  │
    │    rendered_lua: "...",                 │
    │    lua_config: { ... },                │
    │    rate_limit_config: { ... },         │
    │    sibling_count: 3 }                  │
    │<───────────────────────────────────────┤
    │                                        │
    ▼  Slave 本地处理:                        │
    │  1. lua_config → shared_dict           │
    │  2. rate_limit_config → shared_dict    │
    │  3. rendered_nginx → server.conf       │
    │  4. 清空 upstream.conf + maps.conf     │
    │  5. openresty -t && openresty -s reload│
    │  6. POST /ack {snapshot_id, "applied"} │
    ├───────────────────────────────────────>│
    │                                        │
```

### 3.3 生成的 Nginx 配置示例 (3台服务器)

```nginx
# ===== Upstreams =====
upstream emby1_com {
    server 203.0.113.10:8096;
    keepalive 32;
}
upstream emby2_com {
    server 203.0.113.10:8097;
    keepalive 32;
}
upstream emby3_com {
    server 203.0.113.10:8098;
    keepalive 32;
}

# ===== Maps =====
map $request_method $deny_delete { default 0; DELETE 1; }
map $request_method $deny_post   { default 0; POST 1; }
# ... 更多 map 规则 ...

# ===== Server Block 1 =====
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name emby1.com;

    ssl_certificate     /etc/nginx/ssl/emby1.com/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/emby1.com/key.pem;

    # HTTP -> HTTPS redirect
    if ($scheme = http) {
        return 301 https://$host$request_uri;
    }

    # Lua access control
    access_by_lua_file      /opt/slave/lua/access.lua;
    log_by_lua_file         /opt/slave/lua/log_handler.lua;
    header_filter_by_lua_file /opt/slave/lua/header_filter.lua;

    location / {
        proxy_pass http://emby1_com;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # ... 更多 proxy headers ...
    }
}

# ===== Server Block 2 =====
server {
    listen 443 ssl http2;
    server_name emby2.com;
    # ... 同上结构，指向 emby2_com upstream ...
}

# ===== Server Block 3 =====
server {
    listen 443 ssl http2;
    server_name emby3.com;
    # ... 同上结构，指向 emby3_com upstream ...
}
```

---

## 4. Nginx 配置文件策略

### 4.1 文件结构

```
/opt/slave/conf/
├── nginx.conf        # 主配置 (不变)，include 下面三个
├── upstream.conf     # Agent 清空 (防止重复定义)
├── maps.conf         # Agent 清空 (防止重复定义)
└── server.conf       # Agent 写入 rendered_nginx (包含所有内容)
```

### 4.2 为什么要清空 upstream.conf 和 maps.conf?

`nginx.conf` 中有三行 include：
```nginx
include /opt/slave/conf/upstream.conf;
include /opt/slave/conf/maps.conf;
include /opt/slave/conf/server.conf;
```

而 `rendered_nginx` 将 upstream + maps + server blocks 全部合并写入 `server.conf`。如果不清空另外两个文件，会导致：
- upstream 重复定义 → `nginx: [emerg] duplicate upstream`
- map 重复定义 → `nginx: [emerg] duplicate map`
- nginx reload 失败，服务中断

### 4.3 变更检测

Agent 只在内容实际变化时写入文件和 reload：
```lua
local current_conf = read_file("/opt/slave/conf/server.conf")
if current_conf ~= cfg_data.rendered_nginx then
    write_file("/opt/slave/conf/server.conf", cfg_data.rendered_nginx)
    write_file("/opt/slave/conf/upstream.conf", "# Managed by agent\n")
    write_file("/opt/slave/conf/maps.conf", "# Managed by agent\n")
    os.execute("openresty -t && openresty -s reload")
end
```

---

## 5. 关键设计决策

| 设计点 | 选择 | 原因 |
|--------|------|------|
| 配置规则 | 共享 (per service_type) | 所有同类型 Slave 用同一套访问规则 |
| 服务器信息 | 独立 (per slave) | 每台的 domain/port/SSL 不同 |
| 一个 Docker 多域名 | 同 host 合并 | 一台主机只需一个 Slave Docker |
| 配置拉取 | Slave 主动轮询 (30s) | 无需主控主动推送，NAT 友好 |
| conf 文件策略 | 全部写入 server.conf | 避免多文件 include 冲突 |
| 版本检查 | 先 GET /version | 减少不必要的完整配置拉取 |
| Reload 策略 | 先 -t 再 -s reload | 防止错误配置导致服务中断 |

---

## 6. Slave Docker 组件清单

### 6.1 服务组件

| 组件 | 版本 | 用途 |
|------|------|------|
| OpenResty | latest | Nginx + Lua 运行时 |
| Redis | 7 | 速率限制/配额计数/会话存储 |

### 6.2 Lua 模块清单 (10个)

| # | 模块 | 路径 | 功能 |
|---|------|------|------|
| 1 | `init_worker.lua` | lua/ | Agent: 配置拉取 + 文件写入 + reload |
| 2 | `access.lua` | lua/ | 请求级访问控制主入口 |
| 3 | `config.lua` | lua/ | shared_dict 读写封装 |
| 4 | `rate_limiter.lua` | lua/ | L1 速率限制 (shared_dict) |
| 5 | `redis_store.lua` | lua/ | L2 配额计数 (Redis) |
| 6 | `telemetry.lua` | lua/ | 遥测数据批量上报 |
| 7 | `session_checker.lua` | lua/ | 并发流 checkin |
| 8 | `log_handler.lua` | lua/ | 请求日志采集 (log_by_lua) |
| 9 | `header_filter.lua` | lua/ | Token 捕获/响应头注入 |
| 10 | `utils.lua` | lua/ | 通用工具函数 |

### 6.3 Nginx Shared Dict

| 名称 | 大小 | 用途 |
|------|------|------|
| `rate_limit` | 32m | L1 速率限制计数器 |
| `config_cache` | 10m | lua_config + rate_limit_config |
| `telemetry_buffer` | 32m | 遥测数据缓冲 |
| `session_store` | 16m | 活跃会话信息 |

---

## 7. 部署指南

### 7.1 前置条件

- 主控端 (UHDadmin) 已部署并可访问
- 已在管理面板创建 Slave 并生成 App Token

### 7.2 Docker 部署

```bash
# 1. 准备 .env
cat > .env << 'EOF'
APP_TOKEN=uhd_your_token_here
API_BASE=https://api.example.com
REDIS_HOST=redis
REDIS_PORT=6379
EOF

# 2. 启动
docker-compose up -d

# 3. 查看日志
docker logs -f uhd-slave-openresty 2>&1 | grep "Config"
# 期望输出:
# Config update available: 0 -> 1706959200
# Written rendered nginx config to server.conf
# Cleared upstream.conf and maps.conf
# Nginx config reloaded (multi-server mode, 3 servers)
```

### 7.3 验证

```bash
# 检查 nginx 配置
docker exec uhd-slave-openresty openresty -t

# 检查 server.conf 内容
docker exec uhd-slave-openresty cat /opt/slave/conf/server.conf

# 检查 upstream.conf 已清空
docker exec uhd-slave-openresty cat /opt/slave/conf/upstream.conf
# 期望: "# Managed by UHD Slave agent - included in server.conf"
```

---

## 8. 故障排查

| 症状 | 可能原因 | 解决方案 |
|------|----------|----------|
| `Config version check failed` | API_BASE 不可达或 Token 无效 | 检查 .env 中的 API_BASE 和 APP_TOKEN |
| `Nginx reload failed` | 生成的配置语法错误 | 查看 `openresty -t` 输出，检查 SSL 证书路径 |
| `duplicate upstream` | upstream.conf 未被清空 | 确认 init_worker.lua 版本 >= v1.1.12 |
| 30秒后仍未拉取 | init_worker 未启动 | 检查 nginx.conf 中 `init_worker_by_lua_file` 路径 |
| Token 403 | Token 被禁用或 scope 不匹配 | 在管理面板检查 App Token 状态和 scopes |
