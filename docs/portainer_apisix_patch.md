# Portainer + APISIX 集成补丁

> 此文档记录如何通过 APISIX 网关暴露 Portainer，使其可通过域名 `example.com` 访问。
> **注意**：此配置独立于 UHDadmin 主项目，仅供参考。

## 前提条件

- APISIX 网关已部署并运行
- Portainer 已部署（通常在 `portainer` 容器）
- DNS 已配置 `example.com` 指向服务器 IP（或 Cloudflare 代理）
- Cloudflare SSL 模式设置为 **Full**（不能用 Flexible）

## 配置步骤

### 1. 将 Portainer 连接到 APISIX 网络

Portainer 和 APISIX 必须在同一 Docker 网络中才能相互通信。

```bash
# 查看 Portainer 当前网络
docker inspect portainer --format '{{json .NetworkSettings.Networks}}' | jq

# 连接到 APISIX 所在网络
docker network connect uhdadmin_internal portainer

# 验证连接
docker inspect portainer --format '{{json .NetworkSettings.Networks}}' | jq
```

### 2. 设置环境变量

```bash
# APISIX Admin API Key（从 config.yaml 获取）
ADMIN_KEY="your-apisix-admin-key"
```

### 3. 创建 APISIX Upstream

创建指向 Portainer 的 upstream。Portainer 默认使用 HTTPS (9443)。

```bash
curl -X PUT http://203.0.113.10:9180/apisix/admin/upstreams/portainer \
  -H "X-API-KEY : REDACTED" \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "roundrobin",
    "scheme": "https",
    "pass_host": "pass",
    "nodes": {
      "portainer:9443": 1
    }
  }'
```

**参数说明**：
- `scheme: https`：Portainer 使用 HTTPS
- `pass_host: pass`：将原始 Host 头传递给 Portainer
- `portainer:9443`：Docker 容器名 + 端口（Docker DNS 解析）

### 4. 创建 APISIX Routes

**重要**：由于 APISIX 可能存在全局 `/api/*` 路由（用于其他服务），需要为 Portainer 创建多个路由以确保正确匹配。

#### 4.1 主路由（通配符）
```bash
curl -X PUT http://203.0.113.10:9180/apisix/admin/routes/portainer \
  -H "X-API-KEY : REDACTED" \
  -H 'Content-Type: application/json' \
  -d '{
    "uri": "/*",
    "host": "example.com",
    "upstream_id": "portainer",
    "priority": 100,
    "plugins": {}
  }'
```

#### 4.2 API 路由（避免被全局 /api/* 捕获）
```bash
curl -X PUT http://203.0.113.10:9180/apisix/admin/routes/portainer-api \
  -H "X-API-KEY : REDACTED" \
  -H 'Content-Type: application/json' \
  -d '{
    "uri": "/api/*",
    "host": "example.com",
    "upstream_id": "portainer",
    "priority": 100,
    "plugins": {}
  }'
```

#### 4.3 Locales 路由（国际化文件）
```bash
curl -X PUT http://203.0.113.10:9180/apisix/admin/routes/portainer-locales \
  -H "X-API-KEY : REDACTED" \
  -H 'Content-Type: application/json' \
  -d '{
    "uri": "/locales/*",
    "host": "example.com",
    "upstream_id": "portainer",
    "priority": 100,
    "plugins": {}
  }'
```

### 5. 验证配置

```bash
# 检查 upstream
curl -s http://203.0.113.10:9180/apisix/admin/upstreams/portainer \
  -H "X-API-KEY : REDACTED" | jq

# 检查所有 portainer 路由
curl -s http://203.0.113.10:9180/apisix/admin/routes \
  -H "X-API-KEY : REDACTED" | jq '.list[] | select(.value.id | startswith("portainer"))'

# 测试访问（通过 APISIX）
curl -I -H "Host: example.com" http://203.0.113.10:9080/

# 测试 API 路由
curl -s -H "Host: example.com" http://203.0.113.10:9080/api/system/status | jq

# 测试公网访问
curl -I https://example.com/
```

## 一键部署脚本

将以下内容保存为 `setup_portainer_apisix.sh`：

```bash
#!/bin/bash
# Portainer + APISIX 集成脚本

set -e

ADMIN_KEY="${APISIX_ADMIN_KEY:-your-apisix-admin-key}"
PORTAINER_DOMAIN="${PORTAINER_DOMAIN:-example.com}"

echo "=== 配置 Portainer APISIX 路由 ==="

# 1. 连接网络
echo "[1/5] 连接 Portainer 到 APISIX 网络..."
docker network connect uhdadmin_internal portainer 2>/dev/null || echo "  已连接或网络不存在"

# 2. 创建 upstream
echo "[2/5] 创建 upstream..."
curl -sX PUT http://203.0.113.10:9180/apisix/admin/upstreams/portainer \
  -H "X-API-KEY : REDACTED" \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "roundrobin",
    "scheme": "https",
    "pass_host": "pass",
    "nodes": {"portainer:9443": 1}
  }' | jq -r '.key // .error_msg'

# 3. 创建主路由
echo "[3/5] 创建主路由..."
curl -sX PUT http://203.0.113.10:9180/apisix/admin/routes/portainer \
  -H "X-API-KEY : REDACTED" \
  -H 'Content-Type: application/json' \
  -d "{
    \"uri\": \"/*\",
    \"host\": \"${PORTAINER_DOMAIN}\",
    \"upstream_id\": \"portainer\",
    \"priority\": 100,
    \"plugins\": {}
  }" | jq -r '.key // .error_msg'

# 4. 创建 API 路由
echo "[4/5] 创建 API 路由..."
curl -sX PUT http://203.0.113.10:9180/apisix/admin/routes/portainer-api \
  -H "X-API-KEY : REDACTED" \
  -H 'Content-Type: application/json' \
  -d "{
    \"uri\": \"/api/*\",
    \"host\": \"${PORTAINER_DOMAIN}\",
    \"upstream_id\": \"portainer\",
    \"priority\": 100,
    \"plugins\": {}
  }" | jq -r '.key // .error_msg'

# 5. 创建 locales 路由
echo "[5/5] 创建 locales 路由..."
curl -sX PUT http://203.0.113.10:9180/apisix/admin/routes/portainer-locales \
  -H "X-API-KEY : REDACTED" \
  -H 'Content-Type: application/json' \
  -d "{
    \"uri\": \"/locales/*\",
    \"host\": \"${PORTAINER_DOMAIN}\",
    \"upstream_id\": \"portainer\",
    \"priority\": 100,
    \"plugins\": {}
  }" | jq -r '.key // .error_msg'

echo ""
echo "=== 配置完成 ==="
echo "访问: https://${PORTAINER_DOMAIN}/"
```

## 故障排查

### 问题 1：502 Bad Gateway

**可能原因**：APISIX 无法连接到 Portainer

```bash
# 检查 Portainer 是否在同一网络
docker network inspect uhdadmin_internal | grep -A5 portainer

# 检查 Portainer 内部 IP
docker inspect portainer --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}'

# 直接用 IP 测试
curl -Ik https://<portainer-ip>:9443/
```

### 问题 2：525 SSL Handshake Failed

**可能原因**：APISIX 没有配置 SSL 证书

```bash
# 检查 APISIX SSL 配置
curl -s http://203.0.113.10:9180/apisix/admin/ssls \
  -H "X-API-KEY : REDACTED" | jq

# 如果为空，需要配置自签名证书（参见 INSTRUCTION_PACK.md）
```

### 问题 3：404 on /api/* 路径

**可能原因**：全局 `/api/*` 路由优先匹配

```bash
# 列出所有路由
curl -s http://203.0.113.10:9180/apisix/admin/routes \
  -H "X-API-KEY : REDACTED" | \
  jq '.list[] | {id: .value.id, uri: .value.uri, host: .value.host, priority: .value.priority}'

# 确保 portainer-api 路由存在且 host 正确
```

### 问题 4："Unable to retrieve server settings"

**可能原因**：
1. Cloudflare SSL 模式为 "Flexible"（应改为 "Full"）
2. `/api/*` 路由缺失

## 删除配置

如需移除 Portainer 的 APISIX 配置：

```bash
# 删除所有路由
curl -X DELETE http://203.0.113.10:9180/apisix/admin/routes/portainer \
  -H "X-API-KEY : REDACTED"
curl -X DELETE http://203.0.113.10:9180/apisix/admin/routes/portainer-api \
  -H "X-API-KEY : REDACTED"
curl -X DELETE http://203.0.113.10:9180/apisix/admin/routes/portainer-locales \
  -H "X-API-KEY : REDACTED"

# 删除 upstream
curl -X DELETE http://203.0.113.10:9180/apisix/admin/upstreams/portainer \
  -H "X-API-KEY : REDACTED"

# 可选：断开网络连接
docker network disconnect uhdadmin_internal portainer
```

## 安全注意事项

1. **Cloudflare SSL**：必须使用 "Full" 模式（非 Flexible）
2. **访问控制**：Portainer 自带认证，可考虑添加 IP 白名单插件
3. **网络隔离**：Portainer 仅通过 APISIX 暴露，不直接暴露端口
4. **APISIX Admin Key**：确保 Admin API Key 安全，不要提交到代码库
