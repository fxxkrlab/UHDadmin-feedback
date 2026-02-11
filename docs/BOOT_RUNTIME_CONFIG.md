# Boot Config vs Runtime Settings (Round-DEPLOY-04)

本文档说明 UHDadmin 中 **Boot Config（启动配置）** 和 **Runtime Settings（运行时设置）** 的区别与使用场景。

## 配置优先级

```
环境变量 > boot-config.json > config.toml > 代码默认值
```

1. **环境变量** - 最高优先级，直接覆盖所有其他配置
2. **boot-config.json** - 容器启动时读取，修改需重启
3. **config.toml** - 传统配置文件（兼容旧部署）
4. **代码默认值** - 兜底值

## Boot Config（启动配置）

### 定义

Boot Config 是在应用启动时加载的配置，存储在文件中。修改后需要**重启服务**才能生效。

### 文件位置

按优先级搜索：
1. `/data/boot-config.json` - Docker 容器挂卷（推荐）
2. `./boot-config.json` - 本地开发
3. `./deploy/boot-config.json` - 部署目录

### 包含字段

| 字段 | 类型 | 说明 | 敏感 |
|------|------|------|------|
| `app_base_url` | string | 应用内部访问 URL | 否 |
| `public_base_url` | string | 外部公开 URL | 否 |
| `database_dsn` | string | PostgreSQL 连接字符串 | 是 |
| `redis_enabled` | boolean | 是否启用 Redis | 否 |
| `redis_url` | string | Redis 连接 URL | 是 |
| `cors_allowlist` | array | CORS 允许来源列表 | 否 |
| `setup_completed` | boolean | 初始化向导是否完成 | 否 |
| `master_key_hint` | string | MASTER_KEY 提示（不存储密钥）| 否 |

### 使用场景

- 数据库连接配置
- Redis 开关与地址
- CORS 白名单
- 服务 URL 配置

### API

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/admin/boot-config` | GET | 获取配置（敏感字段掩码）|
| `/api/v1/admin/boot-config` | PUT | 更新配置（写入文件）|
| `/api/v1/admin/boot-config/check` | POST | 检查配置文件状态 |

### 权限

需要 **Sysop** 权限（level >= 100 或 which_role = "sysop"）。

### 重启提示

修改 Boot Config 后，API 返回 `restart_required: true`，前端应提示用户重启服务。

---

## Runtime Settings（运行时设置）

### 定义

Runtime Settings 存储在数据库中，修改后**立即生效**，无需重启。

### 存储位置

PostgreSQL 数据库 `system_settings` 表。

### 分类

| 分类 | 说明 |
|------|------|
| registration | 注册设置（开关、配额、时间窗口）|
| invite_codes | 邀请码设置 |
| role_binding | 角色绑定设置 |
| aff | 推广返佣设置 |
| limits | 用户数量限制 |
| naming | 显示名称定制 |
| credits | 余额兑换设置 |
| payment | 支付提供商设置 |
| media_providers | 媒体服务配置（Emby/Jellyfin）|
| domain | 域名白名单设置 |
| email | 邮件发送设置 |
| store | 商店定价设置 |
| security | 安全设置（Turnstile 等）|

### 敏感字段

部分字段标记为 `sensitive: true`，包括：
- `provider_emby_api_key`
- `provider_jellyfin_api_key`
- `gmail_client_id`
- `gmail_client_secret`
- `gmail_refresh_token`
- `tron_api_key`
- `turnstile_secret_key`

敏感字段特性：
- 导出时**自动排除**
- 显示时**掩码处理**
- 可选择**加密存储**（需配置 MASTER_KEY）

### API

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/admin/system-settings` | GET | 获取所有设置 |
| `/api/v1/admin/system-settings/{key}` | GET | 获取单个设置 |
| `/api/v1/admin/system-settings/{key}` | PUT | 更新单个设置 |
| `/api/v1/admin/system-settings/schema` | GET | 获取设置 Schema |
| `/api/v1/admin/system-settings/export/all` | GET | 导出设置（排除敏感）|
| `/api/v1/admin/system-settings/import/validate` | POST | 验证导入（Dry-run）|
| `/api/v1/admin/system-settings/import/apply` | POST | 执行导入 |

### 权限

需要 **Admin** 权限。

---

## Secrets 加密

### MASTER_KEY

设置 `MASTER_KEY` 环境变量后，敏感字段可使用 AES-256-GCM 加密存储。

```bash
export MASTER_KEY="your-32-char-minimum-master-key-here"
```

### 加密格式

```
enc:v1:<base64(nonce + ciphertext + tag)>
```

### Secrets API

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/admin/secrets/status` | GET | 获取加密系统状态 |
| `/api/v1/admin/secrets` | GET | 获取所有敏感设置（掩码）|
| `/api/v1/admin/secrets/{key}` | GET | 获取单个敏感设置 |
| `/api/v1/admin/secrets/{key}` | PUT | 更新敏感设置（自动加密）|
| `/api/v1/admin/secrets/rotate` | POST | 轮换 MASTER_KEY |

### 权限

需要 **Sysop** 权限。

---

## 何时使用哪种配置

| 场景 | 配置类型 | 原因 |
|------|----------|------|
| 数据库地址变更 | Boot Config | 需要重新建立连接池 |
| 开启/关闭 Redis | Boot Config | 影响启动时的连接初始化 |
| 修改 CORS 白名单 | Boot Config | 中间件启动时加载 |
| 开启/关闭注册 | Runtime | 业务逻辑层面，立即生效 |
| 修改邀请码价格 | Runtime | 业务参数，立即生效 |
| 更新 API Key | Runtime (Secrets) | 敏感数据，加密存储 |
| 配置返佣比例 | Runtime | 业务参数 |

---

## 健康检查

`GET /health` 返回：

```json
{
  "status": "ok",
  "version": "v1",
  "db": "ok",
  "db_latency_ms": 5.2,
  "redis": "ok",
  "boot_config": "loaded"
}
```

- `boot_config: "loaded"` - 从文件加载
- `boot_config: "defaults"` - 使用默认值
- `boot_config: "error"` - 加载失败

---

## 回滚说明

### Boot Config 回滚

1. 恢复 `/data/boot-config.json` 到之前版本
2. 重启服务：`docker compose restart api`

### Runtime Settings 回滚

1. 使用导入功能恢复备份：
   ```bash
   curl -X POST /api/v1/admin/system-settings/import/apply \
     -H "Content-Type: application/json" \
     -d @backup-settings.json
   ```

2. 或直接在数据库修改：
   ```sql
   UPDATE system_settings SET value = '"old_value"' WHERE key = 'setting_key';
   ```

---

## 最佳实践

1. **定期导出 Runtime Settings** - 用于备份和迁移
2. **Boot Config 纳入版本控制** - 排除敏感值，使用环境变量覆盖
3. **配置 MASTER_KEY** - 生产环境必须启用敏感数据加密
4. **使用 Dry-run** - 导入设置前先验证
5. **重启前确认** - 修改 Boot Config 后确认所有服务正常再重启
