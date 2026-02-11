# UHDadmin 架构文档 / Architecture Document

[English](ARCHITECTURE.en.md)

本文档描述 UHDadmin 的系统架构、数据流、部署模式及核心设计决策。

---

## 目录

- [1. 系统总览](#1-系统总览)
- [2. 架构图（Mermaid）](#2-架构图mermaid)
- [3. 服务组件](#3-服务组件)
- [4. 数据流](#4-数据流)
- [5. 部署架构](#5-部署架构)
- [6. 蓝绿/金丝雀发布](#6-蓝绿金丝雀发布)
- [7. 安全架构](#7-安全架构)
- [8. 扩展性设计](#8-扩展性设计)

---

## 1. 系统总览

UHDadmin 是一套全栈用户管理与订阅服务平台，采用 **微服务 + 单体后端** 混合架构：

| 特性 | 说明 |
|------|------|
| **网关层** | APISIX 3.8 提供 L7 路由、蓝绿发布、金丝雀流量分割 |
| **后端** | Python FastAPI 单体应用，模块化设计 |
| **前端** | Vue 3 (Vben Admin) + Nuxt 3 (Portal) 双前端 |
| **数据层** | PostgreSQL 15 + Redis 7 |
| **外部集成** | Telegram Bot API、支付网关 |
| **部署** | Docker + Portainer Stack + GHCR |

---

## 2. 架构图（Mermaid）

### 2.1 整体系统架构

```mermaid
graph TB
    subgraph Internet["互联网入口"]
        USER[用户浏览器/App]
        TG_USER[Telegram 用户]
    end

    subgraph Tunnel["隧道层"]
        CF[Cloudflared 隧道]
    end

    subgraph Gateway["网关层"]
        APISIX[APISIX L7 网关<br/>流量分割 / 限流 / CORS]
    end

    subgraph Services["服务层"]
        subgraph Blue["蓝色槽 (stable)"]
            API_B[API Blue<br/>:8000]
            VBEN_B[Vben Blue<br/>:80]
            PORTAL_B[Portal Blue<br/>:3000]
        end

        subgraph Green["绿色槽 (canary)"]
            API_G[API Green<br/>:8000]
            VBEN_G[Vben Green<br/>:80]
            PORTAL_G[Portal Green<br/>:3000]
        end
    end

    subgraph Data["数据层"]
        PG[(PostgreSQL 15<br/>主数据库)]
        REDIS[(Redis 7<br/>缓存/会话)]
        ETCD[(etcd<br/>APISIX 配置)]
    end

    subgraph External["外部服务"]
        TG_API[Telegram Bot API]
        PAY_GW[支付网关<br/>多渠道]
    end

    USER --> CF
    TG_USER --> TG_API
    CF --> APISIX

    APISIX -->|"X-Slot: blue"| API_B
    APISIX -->|"X-Slot: green"| API_G
    APISIX --> VBEN_B & VBEN_G
    APISIX --> PORTAL_B & PORTAL_G

    API_B & API_G --> PG
    API_B & API_G --> REDIS
    API_B & API_G --> TG_API
    API_B & API_G --> PAY_GW

    APISIX --> ETCD
```

### 2.2 后端模块架构

```mermaid
graph LR
    subgraph Routers["API 路由层"]
        AUTH[auth.py<br/>认证]
        USER[user.py<br/>用户]
        SHOP[shop.py<br/>商城]
        PAY[payment.py<br/>支付]
        edgea[miniapp.py<br/>MiniApp]

        subgraph Admin["admin/"]
            A_USER[users.py]
            A_ORDER[orders.py]
            A_ROLE[roles.py]
            A_MEDIA[media_services.py]
            A_RELEASE[release.py]
        end
    end

    subgraph Services["业务服务层"]
        SVC_AUTH[认证服务]
        SVC_ORDER[订单服务]
        SVC_MEDIA[媒体账号服务]
        SVC_AFF[返佣服务]
        SVC_TG[Telegram 服务]
    end

    subgraph Models["数据模型层"]
        M_USER[User]
        M_ORDER[Order]
        M_MEDIA[MediaAccount]
        M_ROLE[Role/Permission]
    end

    subgraph Core["核心模块"]
        DB[database.py<br/>连接池]
        CACHE[cache.py<br/>Redis]
        SEC[security.py<br/>JWT/加密]
    end

    AUTH --> SVC_AUTH
    USER --> SVC_AUTH
    SHOP --> SVC_ORDER
    PAY --> SVC_ORDER
    A_MEDIA --> SVC_MEDIA

    SVC_AUTH --> M_USER
    SVC_ORDER --> M_ORDER
    SVC_MEDIA --> M_MEDIA

    M_USER & M_ORDER & M_MEDIA --> DB
    SVC_AUTH --> CACHE
    SVC_AUTH --> SEC
```

### 2.3 前端架构

```mermaid
graph TB
    subgraph VbenAdmin["Vben Admin (Vue 3)"]
        V_VIEWS[views/<br/>页面组件]
        V_API[api/<br/>请求封装]
        V_STORE[store/<br/>状态管理]
        V_ROUTER[router/<br/>路由]
    end

    subgraph NuxtPortal["Nuxt Portal"]
        N_PAGES[pages/<br/>页面]
        N_COMP[components/<br/>组件]
        N_COMPOSE[composables/<br/>组合函数]
        N_API[api/<br/>请求]
    end

    subgraph MiniApp["Telegram MiniApp"]
        MINI_WEB[WebApp SDK]
        MINI_UI[Nuxt UI 组件]
    end

    V_VIEWS --> V_API
    V_API -->|HTTP| APISIX[APISIX Gateway]

    N_PAGES --> N_API
    N_API -->|HTTP| APISIX

    MINI_WEB --> N_API
```

---

## 3. 服务组件

### 3.1 组件清单

| 组件 | 技术栈 | 端口 | 职责 |
|------|--------|------|------|
| **API** | Python 3.11 + FastAPI | 8000 | 后端 API 服务 |
| **Vben Admin** | Vue 3 + Ant Design Vue | 80 (nginx) | 管理后台 SPA |
| **Portal** | Nuxt 3 + Nuxt UI | 3000 | 用户门户 SSR |
| **APISIX** | Apache APISIX 3.8 | 9080/9443 | L7 网关 |
| **PostgreSQL** | PostgreSQL 15 | 5432 | 主数据库 |
| **Redis** | Redis 7 | 6379 | 缓存/会话 |
| **etcd** | etcd 3.5 | 2379 | APISIX 配置存储 |

### 3.2 容器命名规范

```
uhdadmin-{service}-{slot}-{replica}
```

示例：
- `uhdadmin-api-blue-1` - API 蓝色槽副本 1
- `uhdadmin-vben-green-1` - Vben 绿色槽副本 1
- `uhdadmin-postgres-1` - PostgreSQL（无槽位）

---

## 4. 数据流

### 4.1 用户购买流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant P as Portal
    participant A as API
    participant DB as PostgreSQL
    participant R as Redis
    participant Pay as 支付网关

    U->>P: 1. 浏览商品
    P->>A: GET /api/v1/shop/products
    A->>R: 查询缓存
    alt 缓存命中
        R-->>A: 返回商品列表
    else 缓存未命中
        A->>DB: SELECT * FROM products
        DB-->>A: 商品数据
        A->>R: 写入缓存
    end
    A-->>P: 商品列表 JSON
    P-->>U: 渲染商品页面

    U->>P: 2. 选择套餐下单
    P->>A: POST /api/v1/orders
    A->>DB: INSERT INTO orders
    A->>Pay: 创建支付订单
    Pay-->>A: 支付链接
    A-->>P: 返回支付 URL
    P->>U: 跳转支付页面

    Pay->>A: 3. 支付回调
    A->>DB: UPDATE orders SET status='paid'
    A->>DB: 分配媒体账号
    A->>R: 清除相关缓存
    A-->>Pay: 回调确认
```

### 4.2 Telegram 登录流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant TG as Telegram
    participant P as Portal/MiniApp
    participant A as API
    participant DB as PostgreSQL

    U->>TG: 1. 打开 Bot，点击登录
    TG->>P: 携带 initData
    P->>A: POST /api/v1/auth/telegram
    Note over A: 验证 initData 签名
    A->>DB: 查询/创建用户
    A-->>P: JWT Token
    P-->>U: 登录成功，跳转首页
```

### 4.3 RBAC 权限检查流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant A as API
    participant MW as 权限中间件
    participant DB as PostgreSQL
    participant R as Redis

    U->>A: 请求 (Bearer Token)
    A->>MW: 权限检查
    MW->>R: 查询权限缓存
    alt 缓存命中
        R-->>MW: 权限列表
    else 缓存未命中
        MW->>DB: 查询用户角色和权限
        DB-->>MW: 权限数据
        MW->>R: 写入缓存 (TTL 5min)
    end

    alt 有权限
        MW-->>A: 放行
        A-->>U: 正常响应
    else 无权限
        MW-->>U: 403 Forbidden
    end
```

---

## 5. 部署架构

### 5.1 容器编排

```mermaid
graph TB
    subgraph DockerHost["Docker Host"]
        subgraph Network["uhdadmin-network (bridge)"]
            APISIX[apisix:9080]
            ETCD[etcd:2379]

            subgraph BlueStack["Blue Stack"]
                API_B[api-blue:8000]
                VBEN_B[vben-blue:80]
                PORTAL_B[portal-blue:3000]
            end

            subgraph GreenStack["Green Stack"]
                API_G[api-green:8000]
                VBEN_G[vben-green:80]
                PORTAL_G[portal-green:3000]
            end

            PG[postgres:5432]
            REDIS[redis:6379]
        end

        subgraph Volumes["Named Volumes"]
            V_PG[postgres_data]
            V_REDIS[redis_data]
            V_ETCD[etcd_data]
            V_BOOT[boot_config]
        end
    end

    API_B & API_G --> PG
    API_B & API_G --> REDIS
    APISIX --> ETCD

    PG --> V_PG
    REDIS --> V_REDIS
    ETCD --> V_ETCD
    API_B & API_G --> V_BOOT
```

### 5.2 存储模式对比

| 模式 | 优点 | 缺点 | 推荐场景 |
|------|------|------|----------|
| **Named Volumes** | 零准备、Docker 管理 | 路径不直观 | 首次部署、快速测试 |
| **Bind Mounts** | 直接访问、便于 vim/rsync | 需预创建目录 | 生产环境、需要直接管理 |

### 5.3 端口映射

| 外部端口 | 内部端口 | 服务 | 说明 |
|----------|----------|------|------|
| 9080 | 9080 | APISIX | HTTP 入口 |
| 9443 | 9443 | APISIX | HTTPS 入口 |
| 9180 | 9180 | APISIX | Admin API |
| - | 8000 | API | 后端（通过 APISIX 访问） |
| - | 5432 | PostgreSQL | 数据库（内部） |
| - | 6379 | Redis | 缓存（内部） |

---

## 6. 蓝绿/金丝雀发布

### 6.1 发布模式

```mermaid
graph LR
    subgraph Modes["发布模式"]
        BLUE[100% Blue<br/>稳定版本]
        CANARY[Blue + Green<br/>金丝雀测试]
        GREEN[100% Green<br/>新版本]
    end

    BLUE -->|"traffic.sh canary 10"| CANARY
    CANARY -->|"traffic.sh canary 50"| CANARY
    CANARY -->|"traffic.sh green"| GREEN
    GREEN -->|"rollback_blue.sh"| BLUE
    CANARY -->|"rollback_blue.sh"| BLUE
```

### 6.2 流量分割规则

```mermaid
flowchart TD
    REQ[请求进入] --> CHECK_WHITE{用户在白名单?}
    CHECK_WHITE -->|是| GREEN[100% 绿色]
    CHECK_WHITE -->|否| CHECK_BLACK{用户在黑名单?}
    CHECK_BLACK -->|是| BLUE[100% 蓝色]
    CHECK_BLACK -->|否| HASH{按 UID Hash}
    HASH -->|"Hash % 100 < 金丝雀比例"| GREEN
    HASH -->|"Hash % 100 >= 金丝雀比例"| BLUE
```

### 6.3 APISIX 流量控制

```bash
# 查看当前状态
./deploy/apisix/traffic.sh status

# 切换到 100% 蓝色（稳定）
./deploy/apisix/traffic.sh blue

# 金丝雀发布 10% 流量到绿色
./deploy/apisix/traffic.sh canary 10

# 逐步增加金丝雀比例
./deploy/apisix/traffic.sh canary 25
./deploy/apisix/traffic.sh canary 50

# 全量切换到绿色
./deploy/apisix/traffic.sh green

# 紧急回滚
./deploy/apisix/rollback_blue.sh
```

---

## 7. 安全架构

### 7.1 认证流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant A as API
    participant DB as PostgreSQL
    participant R as Redis

    U->>A: POST /auth/login (用户名/密码)
    A->>DB: 查询用户
    A->>A: 验证密码 (bcrypt)
    A->>A: 生成 JWT
    A->>R: 存储 Token 黑名单检查
    A-->>U: {access_token, refresh_token}

    Note over U,A: 后续请求
    U->>A: 请求 (Authorization: Bearer xxx)
    A->>R: 检查黑名单
    A->>A: 验证 JWT 签名
    A-->>U: 响应
```

### 7.2 安全措施

| 层级 | 措施 | 说明 |
|------|------|------|
| **网关** | CORS 白名单 | 禁止 `*` 通配符 |
| **网关** | 限流 | 按 IP/用户限制请求频率 |
| **认证** | JWT + bcrypt | 无状态令牌 + 安全哈希 |
| **权限** | RBAC | 角色-权限-资源 三级模型 |
| **数据** | 字段加密 | 敏感数据 AES 加密存储 |
| **传输** | HTTPS | Cloudflared 隧道 TLS |

### 7.3 Sysop 保护

Sysop（系统操作员）角色具有特殊保护：
- 不可被删除
- 不可被降级
- 操作日志强制记录
- 双因素认证（可选）

---

## 8. 扩展性设计

### 8.1 水平扩展

```mermaid
graph TB
    subgraph LB["负载均衡 (APISIX)"]
        APISIX[APISIX]
    end

    subgraph APICluster["API 集群"]
        API1[API-1]
        API2[API-2]
        API3[API-N]
    end

    subgraph DBCluster["数据库集群"]
        PG_M[(Primary)]
        PG_R1[(Replica 1)]
        PG_R2[(Replica N)]
    end

    subgraph Cache["缓存集群"]
        REDIS_M[Redis Master]
        REDIS_R[Redis Replica]
    end

    APISIX --> API1 & API2 & API3
    API1 & API2 & API3 --> PG_M
    API1 & API2 & API3 -.->|读| PG_R1 & PG_R2
    API1 & API2 & API3 --> REDIS_M
    REDIS_M --> REDIS_R
```

### 8.2 扩展点

| 扩展点 | 实现方式 | 说明 |
|--------|----------|------|
| **支付渠道** | `app/services/payment/` | 实现 PaymentProvider 接口 |
| **媒体 Provider** | `app/services/media/` | 实现 MediaProvider 接口 |
| **通知渠道** | `app/services/notification/` | 实现 NotificationChannel 接口 |
| **第三方登录** | `app/routers/auth.py` | 添加 OAuth2 提供商 |

---

## 相关文档

- [安装指南](INSTALL.md)
- [部署手册](DEPLOY_RUNBOOK.md)
- [环境变量参考](ENV_REFERENCE.zh-CN.md)
- [配置指南](BOOT_RUNTIME_CONFIG.md)
- [README](../README.md)
