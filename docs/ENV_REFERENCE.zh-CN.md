# UHDadmin 环境变量参考 / Environment Variable Reference

[English](ENV_REFERENCE.en.md)

本文档列出 UHDadmin 所有支持的环境变量，包括必填项、默认值及说明。

---

## 目录

- [1. 配置优先级](#1-配置优先级)
- [2. 必填变量](#2-必填变量)
- [3. 数据库配置](#3-数据库配置)
- [4. Redis 配置](#4-redis-配置)
- [5. 安全配置](#5-安全配置)
- [6. 功能开关](#6-功能开关)
- [7. Telegram 配置](#7-telegram-配置)
- [8. LDAP 配置](#8-ldap-配置)
- [9. Provider 配置](#9-provider-配置)
- [10. 可观测性配置](#10-可观测性配置)
- [11. 可靠性配置](#11-可靠性配置)
- [12. APISIX 配置](#12-apisix-配置)
- [13. 域名与 URL 配置](#13-域名与-url-配置)
- [14. 邀请码配置](#14-邀请码配置)
- [15. 角色配置](#15-角色配置)
- [16. 生成密钥](#16-生成密钥)

---

## 1. 配置优先级

UHDadmin 按以下优先级读取配置（从高到低）：

| 优先级 | 来源 | 说明 |
|:------:|------|------|
| 1 | 环境变量 | 直接设置的 `export VAR=value` |
| 2 | `.env` 文件 | 项目根目录的 `.env` 文件 |
| 3 | `boot-config.json` | `/data/boot-config.json` 运行时配置 |
| 4 | `config.toml` | 兼容性配置文件 |
| 5 | 代码默认值 | `app/config.py` 中定义的默认值 |

---

## 2. 必填变量

以下变量在生产环境中**必须**配置：

| 变量名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| `POSTGRES_USER` | string | **必填** - 数据库用户名 | `uhdadmin` |
| `POSTGRES_PASSWORD` | string | **必填** - 数据库密码（至少 16 位强密码） | `MyStr0ngP@ssw0rd!` |
| `JWT_SECRET_KEY` | string | **必填** - JWT 签名密钥（至少 32 位） | `openssl rand -hex 32` 生成 |
| `SECRET_KEY` | string | **必填** - 应用加密密钥（至少 32 位） | `openssl rand -hex 32` 生成 |

> **警告**：使用默认或弱密钥将导致严重安全风险！

---

## 3. 数据库配置

### PostgreSQL

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `DATABASE_URL` | string | `postgresql+asyncpg://postgres:password@localhost:5432/uhdadmin` | 完整数据库连接 URL |
| `POSTGRES_USER` | string | - | **必填** - 数据库用户名 |
| `POSTGRES_PASSWORD` | string | - | **必填** - 数据库密码 |
| `POSTGRES_DB` | string | `uhdadmin` | 数据库名称 |
| `DB_POOL_MIN` | int | `1` | 连接池最小连接数 |
| `DB_POOL_MAX` | int | `10` | 连接池最大连接数 |
| `DB_CONNECT_TIMEOUT` | int | `10` | 连接超时（秒） |
| `DB_COMMAND_TIMEOUT` | int | `30` | 命令/查询超时（秒） |
| `DB_POOL_MAX_INACTIVE` | int | `300` | 空闲连接最大存活时间（秒） |
| `DB_RETRY_ATTEMPTS` | int | `2` | 数据库操作重试次数 |
| `DB_RETRY_BACKOFF_MS` | int | `200` | 重试退避时间（毫秒） |

**远程 PostgreSQL 推荐配置：**
```bash
DB_CONNECT_TIMEOUT=10
DB_COMMAND_TIMEOUT=30
DB_POOL_MAX_INACTIVE=300
```

---

## 4. Redis 配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `REDIS_ENABLED` | bool | `false` | 是否启用 Redis 缓存 |
| `REDIS_URL` | string | `redis://localhost:6379/0` | Redis 连接 URL |
| `REDIS_PASSWORD` | string | - | Redis 密码（可选） |
| `REDIS_DB` | int | `0` | Redis 数据库编号 |

---

## 5. 安全配置

### JWT 认证

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `JWT_SECRET_KEY` | string | - | **必填** - JWT 签名密钥 |
| `JWT_ALGORITHM` | string | `HS256` | JWT 加密算法 |
| `JWT_EXPIRE_MINUTES` | int | `60` | JWT 过期时间（分钟） |

### 应用密钥

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `SECRET_KEY` | string | - | **必填** - 应用加密密钥 |

### Bot Service Token

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `BOT_SERVICE_TOKEN` | string | - | Bot 服务端认证 Token（MiniApp Session Exchange） |
| `MINIAPP_SESSION_TTL` | int | `300` | MiniApp Session Token 有效期（秒） |

---

## 6. 功能开关

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `REGISTRATION_ENABLED` | bool | `false` | 是否允许用户自助注册 |
| `REGISTRATION_INVITE_REQUIRED` | bool | `false` | 注册是否必须提供邀请码 |
| `LDAP_ENABLED` | bool | `false` | 是否启用 LDAP 同步 |
| `TELEGRAM_ENABLED` | bool | `false` | 是否启用 Telegram 通知 |
| `PROVIDER_EMBY_ENABLED` | bool | `false` | 是否启用 Emby Provider |
| `PROVIDER_JELLYFIN_ENABLED` | bool | `false` | 是否启用 Jellyfin Provider |

---

## 7. Telegram 配置

### 通知 Bot

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `TELEGRAM_ENABLED` | bool | `false` | 是否启用 Telegram 通知 |
| `TELEGRAM_BOT_TOKEN` | string | - | Telegram Bot Token（通知用） |
| `TELEGRAM_CHAT_ID` | string | - | Telegram Chat ID |

### Bot Runner (MiniApp)

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `TG_BOT_TOKEN` | string | - | Telegram Bot Token（运行 Bot 进程） |
| `TG_BOT_USERNAME` | string | - | Telegram Bot Username（可选，用于校验） |

---

## 8. LDAP 配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `LDAP_ENABLED` | bool | `false` | 是否启用 LDAP |
| `LDAP_SERVER` | string | `ldap://localhost:389` | LDAP 服务器地址 |
| `LDAP_BIND_DN` | string | - | LDAP 绑定 DN |
| `LDAP_BIND_PASSWORD` | string | - | LDAP 绑定密码 |
| `LDAP_BASE_DN` | string | - | LDAP 基础 DN |
| `LDAP_USER_FILTER` | string | `(objectClass=person)` | LDAP 用户过滤器 |
| `LDAP_ATTR_USERNAME` | string | `uid` | LDAP 用户名属性 |
| `LDAP_ATTR_EMAIL` | string | `mail` | LDAP 邮箱属性 |
| `LDAP_ATTR_DISPLAY_NAME` | string | `displayName` | LDAP 显示名称属性 |

---

## 9. Provider 配置

### Emby

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `PROVIDER_EMBY_ENABLED` | bool | `false` | 是否启用 Emby |
| `PROVIDER_EMBY_URL` | string | - | Emby 服务器 URL |
| `PROVIDER_EMBY_API_KEY` | string | - | Emby API Key |

### Jellyfin

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `PROVIDER_JELLYFIN_ENABLED` | bool | `false` | 是否启用 Jellyfin |
| `PROVIDER_JELLYFIN_URL` | string | - | Jellyfin 服务器 URL |
| `PROVIDER_JELLYFIN_API_KEY` | string | - | Jellyfin API Key |

---

## 10. 可观测性配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `LOG_LEVEL` | string | `INFO` | 日志级别（DEBUG/INFO/WARNING/ERROR） |
| `SLOW_REQUEST_THRESHOLD_MS` | int | `1500` | 慢请求阈值（毫秒） |
| `REQUEST_LOGGING_ENABLED` | bool | `true` | 是否启用请求日志记录 |
| `REQUEST_LOGGING_EXCLUDE_PATHS` | string | `/health,/metrics,/docs,/openapi.json,/redoc` | 排除记录的路径 |

---

## 11. 可靠性配置

### 优雅停机

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `SHUTDOWN_TIMEOUT_SECONDS` | int | `30` | 优雅停机超时（秒） |

### 请求保护

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `MAX_REQUEST_BODY_SIZE` | int | `10485760` | 最大请求体大小（10MB） |
| `REQUEST_TIMEOUT_SECONDS` | int | `60` | 请求超时（秒） |

### 速率限制

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `RATE_LIMIT_ENABLED` | bool | `true` | 是否启用速率限制 |
| `RATE_LIMIT_PER_MINUTE` | int | `300` | 每分钟每 IP 请求限制 |

### 熔断器

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `CIRCUIT_BREAKER_FAILURE_THRESHOLD` | int | `5` | 熔断器触发阈值 |
| `CIRCUIT_BREAKER_RECOVERY_TIMEOUT` | int | `30` | 熔断器恢复超时（秒） |

### 日志保留

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `LOG_RETENTION_DAYS` | int | `30` | 运行日志保留天数 |

---

## 12. APISIX 配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `APISIX_HTTP_PORT` | int | `9080` | APISIX HTTP 端口 |
| `APISIX_HTTPS_PORT` | int | `9443` | APISIX HTTPS 端口 |
| `APISIX_ADMIN_PORT` | int | `9180` | APISIX Admin API 端口 |

---

## 13. 域名与 URL 配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `APP_BASE_URL` | string | `http://localhost:8000` | 应用基础 URL（内部访问） |
| `PUBLIC_BASE_URL` | string | `http://localhost:8000` | 公开访问 URL |
| `PORTAL_URL` | string | `https://portal.example.com` | Portal 前端 URL |
| `VBEN_URL` | string | `http://localhost:5173` | Vben Admin 前端 URL |
| `CORS_ORIGINS` | string | `http://localhost:3000,...` | 允许的 CORS 来源（逗号分隔） |

### boot-config.json 支持的域名配置

| 键名 | 说明 |
|------|------|
| `domain_api` | API 域名（如 `api.example.com`） |
| `domain_admin` | 管理后台域名（如 `admin.example.com`） |
| `domain_portal` | 门户域名（如 `portal.example.com`，支持多个） |
| `domain_app` | 应用域名（如 `app.example.com`） |
| `cors_origins` | CORS 白名单（如 `https://portal.example.com,https://admin.example.com`） |

---

## 14. 邀请码配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `INVITE_CODE_COST` | int | `100` | 创建邀请码消耗的 Credits |
| `INVITE_CODE_MIN_LEVEL` | int | `1` | 允许创建邀请码的最低 level |

---

## 15. 角色配置

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `AUTO_ROLE_BIND_ENABLED` | bool | `false` | 是否启用角色自动绑定 |
| `AUTO_ROLE_BIND_MAX_LEVEL` | int | `50` | 自动绑定的最高 level 上限 |

---

## 16. 生成密钥

### 使用 OpenSSL 生成 32 位 hex 密钥

```bash
# 生成 JWT_SECRET_KEY
openssl rand -hex 32

# 生成 SECRET_KEY
openssl rand -hex 32

# 生成 POSTGRES_PASSWORD
openssl rand -hex 16
```

### 使用 Python 生成

```python
import secrets
print(secrets.token_hex(32))
```

### 示例 .env 文件

```bash
# ========== 必填变量 ==========
POSTGRES_USER=uhdadmin
POSTGRES_PASSWORD=YourStrongPassword123!
JWT_SECRET_KEY=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4
SECRET_KEY=z9y8x7w6v5u4t3s2r1q0p9o8n7m6l5k4j3i2h1g0f9e8d7c6

# ========== 可选变量 ==========
IMAGE_TAG=latest
LOG_LEVEL=INFO
REDIS_ENABLED=true
CORS_ORIGINS=https://admin.example.com,https://portal.example.com
```

---

## 相关文档

- [安装指南](INSTALL.md)
- [架构文档](ARCHITECTURE.zh-CN.md)
- [配置指南](BOOT_RUNTIME_CONFIG.md)
- [README](../README.md)
