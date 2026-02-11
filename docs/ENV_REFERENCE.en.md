# UHDadmin Environment Variable Reference

[中文文档](ENV_REFERENCE.zh-CN.md)

This document lists all environment variables supported by UHDadmin, including required variables, defaults, and descriptions.

---

## Table of Contents

- [1. Configuration Priority](#1-configuration-priority)
- [2. Required Variables](#2-required-variables)
- [3. Database Configuration](#3-database-configuration)
- [4. Redis Configuration](#4-redis-configuration)
- [5. Security Configuration](#5-security-configuration)
- [6. Feature Toggles](#6-feature-toggles)
- [7. Telegram Configuration](#7-telegram-configuration)
- [8. LDAP Configuration](#8-ldap-configuration)
- [9. Provider Configuration](#9-provider-configuration)
- [10. Observability Configuration](#10-observability-configuration)
- [11. Reliability Configuration](#11-reliability-configuration)
- [12. APISIX Configuration](#12-apisix-configuration)
- [13. Domain & URL Configuration](#13-domain--url-configuration)
- [14. Invite Code Configuration](#14-invite-code-configuration)
- [15. Role Configuration](#15-role-configuration)
- [16. Generating Secrets](#16-generating-secrets)

---

## 1. Configuration Priority

UHDadmin reads configuration in the following order (highest to lowest):

| Priority | Source | Description |
|:--------:|--------|-------------|
| 1 | Environment Variables | Directly set via `export VAR=value` |
| 2 | `.env` File | `.env` file in project root |
| 3 | `boot-config.json` | `/data/boot-config.json` runtime config |
| 4 | `config.toml` | Legacy configuration file |
| 5 | Code Defaults | Defaults defined in `app/config.py` |

---

## 2. Required Variables

The following variables **must** be configured in production:

| Variable | Type | Description | Example |
|----------|------|-------------|---------|
| `POSTGRES_USER` | string | **Required** - Database username | `uhdadmin` |
| `POSTGRES_PASSWORD` | string | **Required** - Database password (min 16 chars) | `MyStr0ngP@ssw0rd!` |
| `JWT_SECRET_KEY` | string | **Required** - JWT signing key (min 32 chars) | Use `openssl rand -hex 32` |
| `SECRET_KEY` | string | **Required** - App encryption key (min 32 chars) | Use `openssl rand -hex 32` |

> **Warning**: Using default or weak keys poses serious security risks!

---

## 3. Database Configuration

### PostgreSQL

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `DATABASE_URL` | string | `postgresql+asyncpg://postgres:password@localhost:5432/uhdadmin` | Full database connection URL |
| `POSTGRES_USER` | string | - | **Required** - Database username |
| `POSTGRES_PASSWORD` | string | - | **Required** - Database password |
| `POSTGRES_DB` | string | `uhdadmin` | Database name |
| `DB_POOL_MIN` | int | `1` | Minimum pool connections |
| `DB_POOL_MAX` | int | `10` | Maximum pool connections |
| `DB_CONNECT_TIMEOUT` | int | `10` | Connection timeout (seconds) |
| `DB_COMMAND_TIMEOUT` | int | `30` | Command/query timeout (seconds) |
| `DB_POOL_MAX_INACTIVE` | int | `300` | Max idle connection lifetime (seconds) |
| `DB_RETRY_ATTEMPTS` | int | `2` | Database operation retry count |
| `DB_RETRY_BACKOFF_MS` | int | `200` | Retry backoff time (milliseconds) |

**Recommended for remote PostgreSQL:**
```bash
DB_CONNECT_TIMEOUT=10
DB_COMMAND_TIMEOUT=30
DB_POOL_MAX_INACTIVE=300
```

---

## 4. Redis Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `REDIS_ENABLED` | bool | `false` | Enable Redis caching |
| `REDIS_URL` | string | `redis://localhost:6379/0` | Redis connection URL |
| `REDIS_PASSWORD` | string | - | Redis password (optional) |
| `REDIS_DB` | int | `0` | Redis database number |

---

## 5. Security Configuration

### JWT Authentication

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `JWT_SECRET_KEY` | string | - | **Required** - JWT signing key |
| `JWT_ALGORITHM` | string | `HS256` | JWT encryption algorithm |
| `JWT_EXPIRE_MINUTES` | int | `60` | JWT expiration time (minutes) |

### Application Secret

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `SECRET_KEY` | string | - | **Required** - Application encryption key |

### Bot Service Token

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `BOT_SERVICE_TOKEN` | string | - | Bot service auth token (MiniApp Session Exchange) |
| `MINIAPP_SESSION_TTL` | int | `300` | MiniApp Session Token TTL (seconds) |

---

## 6. Feature Toggles

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `REGISTRATION_ENABLED` | bool | `false` | Allow user self-registration |
| `REGISTRATION_INVITE_REQUIRED` | bool | `false` | Require invite code for registration |
| `LDAP_ENABLED` | bool | `false` | Enable LDAP sync |
| `TELEGRAM_ENABLED` | bool | `false` | Enable Telegram notifications |
| `PROVIDER_EMBY_ENABLED` | bool | `false` | Enable Emby Provider |
| `PROVIDER_JELLYFIN_ENABLED` | bool | `false` | Enable Jellyfin Provider |

---

## 7. Telegram Configuration

### Notification Bot

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `TELEGRAM_ENABLED` | bool | `false` | Enable Telegram notifications |
| `TELEGRAM_BOT_TOKEN` | string | - | Telegram Bot Token (notifications) |
| `TELEGRAM_CHAT_ID` | string | - | Telegram Chat ID |

### Bot Runner (MiniApp)

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `TG_BOT_TOKEN` | string | - | Telegram Bot Token (Bot process) |
| `TG_BOT_USERNAME` | string | - | Telegram Bot Username (optional, for validation) |

---

## 8. LDAP Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `LDAP_ENABLED` | bool | `false` | Enable LDAP |
| `LDAP_SERVER` | string | `ldap://localhost:389` | LDAP server address |
| `LDAP_BIND_DN` | string | - | LDAP bind DN |
| `LDAP_BIND_PASSWORD` | string | - | LDAP bind password |
| `LDAP_BASE_DN` | string | - | LDAP base DN |
| `LDAP_USER_FILTER` | string | `(objectClass=person)` | LDAP user filter |
| `LDAP_ATTR_USERNAME` | string | `uid` | LDAP username attribute |
| `LDAP_ATTR_EMAIL` | string | `mail` | LDAP email attribute |
| `LDAP_ATTR_DISPLAY_NAME` | string | `displayName` | LDAP display name attribute |

---

## 9. Provider Configuration

### Emby

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `PROVIDER_EMBY_ENABLED` | bool | `false` | Enable Emby |
| `PROVIDER_EMBY_URL` | string | - | Emby server URL |
| `PROVIDER_EMBY_API_KEY` | string | - | Emby API Key |

### Jellyfin

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `PROVIDER_JELLYFIN_ENABLED` | bool | `false` | Enable Jellyfin |
| `PROVIDER_JELLYFIN_URL` | string | - | Jellyfin server URL |
| `PROVIDER_JELLYFIN_API_KEY` | string | - | Jellyfin API Key |

---

## 10. Observability Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `LOG_LEVEL` | string | `INFO` | Log level (DEBUG/INFO/WARNING/ERROR) |
| `SLOW_REQUEST_THRESHOLD_MS` | int | `1500` | Slow request threshold (ms) |
| `REQUEST_LOGGING_ENABLED` | bool | `true` | Enable request logging |
| `REQUEST_LOGGING_EXCLUDE_PATHS` | string | `/health,/metrics,/docs,/openapi.json,/redoc` | Paths to exclude from logging |

---

## 11. Reliability Configuration

### Graceful Shutdown

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `SHUTDOWN_TIMEOUT_SECONDS` | int | `30` | Graceful shutdown timeout (seconds) |

### Request Protection

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `MAX_REQUEST_BODY_SIZE` | int | `10485760` | Max request body size (10MB) |
| `REQUEST_TIMEOUT_SECONDS` | int | `60` | Request timeout (seconds) |

### Rate Limiting

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `RATE_LIMIT_ENABLED` | bool | `true` | Enable rate limiting |
| `RATE_LIMIT_PER_MINUTE` | int | `300` | Requests per minute per IP |

### Circuit Breaker

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `CIRCUIT_BREAKER_FAILURE_THRESHOLD` | int | `5` | Circuit breaker trigger threshold |
| `CIRCUIT_BREAKER_RECOVERY_TIMEOUT` | int | `30` | Circuit breaker recovery timeout (seconds) |

### Log Retention

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `LOG_RETENTION_DAYS` | int | `30` | Runtime log retention days |

---

## 12. APISIX Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `APISIX_HTTP_PORT` | int | `9080` | APISIX HTTP port |
| `APISIX_HTTPS_PORT` | int | `9443` | APISIX HTTPS port |
| `APISIX_ADMIN_PORT` | int | `9180` | APISIX Admin API port |

---

## 13. Domain & URL Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `APP_BASE_URL` | string | `http://localhost:8000` | App base URL (internal) |
| `PUBLIC_BASE_URL` | string | `http://localhost:8000` | Public access URL |
| `PORTAL_URL` | string | `https://portal.example.com` | Portal frontend URL |
| `VBEN_URL` | string | `http://localhost:5173` | Vben Admin frontend URL |
| `CORS_ORIGINS` | string | `http://localhost:3000,...` | Allowed CORS origins (comma-separated) |

### boot-config.json Domain Keys

| Key | Description |
|-----|-------------|
| `domain_api` | API domain (e.g., `api.example.com`) |
| `domain_admin` | Admin panel domain (e.g., `admin.example.com`) |
| `domain_portal` | Portal domain (e.g., `portal.example.com`, supports multiple) |
| `domain_app` | App domain (e.g., `app.example.com`) |
| `cors_origins` | CORS whitelist (e.g., `https://portal.example.com,https://admin.example.com`) |

---

## 14. Invite Code Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `INVITE_CODE_COST` | int | `100` | Credits cost to create invite code |
| `INVITE_CODE_MIN_LEVEL` | int | `1` | Minimum level to create invite code |

---

## 15. Role Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `AUTO_ROLE_BIND_ENABLED` | bool | `false` | Enable automatic role binding |
| `AUTO_ROLE_BIND_MAX_LEVEL` | int | `50` | Max level for auto role binding |

---

## 16. Generating Secrets

### Using OpenSSL to generate 32-byte hex keys

```bash
# Generate JWT_SECRET_KEY
openssl rand -hex 32

# Generate SECRET_KEY
openssl rand -hex 32

# Generate POSTGRES_PASSWORD
openssl rand -hex 16
```

### Using Python

```python
import secrets
print(secrets.token_hex(32))
```

### Example .env file

```bash
# ========== Required Variables ==========
POSTGRES_USER=uhdadmin
POSTGRES_PASSWORD=YourStrongPassword123!
JWT_SECRET_KEY=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4
SECRET_KEY=z9y8x7w6v5u4t3s2r1q0p9o8n7m6l5k4j3i2h1g0f9e8d7c6

# ========== Optional Variables ==========
IMAGE_TAG=latest
LOG_LEVEL=INFO
REDIS_ENABLED=true
CORS_ORIGINS=https://admin.example.com,https://portal.example.com
```

---

## Related Documentation

- [Installation Guide](INSTALL.md)
- [Architecture Document](ARCHITECTURE.en.md)
- [Configuration Guide](BOOT_RUNTIME_CONFIG.md)
- [README](../README.en.md)
