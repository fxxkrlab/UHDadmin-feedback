# Cloudflare Tunnel 配置指南

## 背景
使用 Cloudflare Tunnel (cloudflared) 将本地服务暴露到公网，避免开放端口。

## 推荐配置

### 1. 域名映射
| 域名 | 服务 | 本地端口 |
|------|------|----------|
| portal.example.com | Nuxt Portal | localhost:3001 |
| admin.example.com | Vben Admin | localhost:5173 |
| api.example.com | FastAPI Backend | localhost:8000 |

### 2. cloudflared config.yml

```yaml
# ~/.cloudflared/config.yml
tunnel: <YOUR_TUNNEL_ID>
credentials-file: /Users/<username>/.cloudflared/<tunnel-id>.json

ingress:
  # Portal (Nuxt SSR)
  - hostname: portal.example.com
    service: http://localhost:3001
    originRequest:
      noTLSVerify: true
      connectTimeout: 30s

  # Admin (Vben - Vite dev or build)
  - hostname: admin.example.com
    service: http://localhost:5173
    originRequest:
      noTLSVerify: true
      connectTimeout: 30s

  # API (FastAPI)
  - hostname: api.example.com
    service: http://localhost:8000
    originRequest:
      noTLSVerify: true
      connectTimeout: 30s

  # Catch-all (required)
  - service: http_status:404
```

### 3. 502 常见原因及解决

| 原因 | 解决方案 |
|------|----------|
| 本地服务未启动 | 确保 pnpm run dev / uvicorn 已运行 |
| 端口不匹配 | 检查 ingress service 端口与实际服务端口一致 |
| 连接超时 | 增加 connectTimeout（默认 30s） |
| WebSocket 支持 | Vite HMR 需要 WebSocket，确保 cloudflared 版本 >= 2022.1 |
| DNS 未传播 | 等待 DNS 传播或清除缓存 |

### 4. 启动顺序

```bash
# 1. 启动后端
cd /Users/neo/Documents/coding/UHDadmin
source .venv/bin/activate
CORS_ORIGINS="http://localhost:5173,http://localhost:3000,http://localhost:3001,https://portal.example.com,https://admin.example.com,https://api.example.com" python3 -m uvicorn app.example.com:app --host 203.0.113.10 --port 8000 &

# 2. 启动 Vben Admin
cd vue-vben-admin
pnpm run dev &

# 3. 启动 Portal
cd nuxt-portal
pnpm run dev &

# 4. 启动 Cloudflare Tunnel
cloudflared tunnel run <tunnel-name>
```

### 5. 验证命令

```bash
# 测试 API
curl -I https://api.example.com/health

# 测试 Portal
curl -I https://portal.example.com/

# 测试 Admin
curl -I https://admin.example.com/
```

### 6. 调试命令

```bash
# 查看 tunnel 日志
cloudflared tunnel --loglevel debug run <tunnel-name>

# 检查 DNS 解析
dig portal.example.com +short
dig admin.example.com +short
dig api.example.com +short

# 测试本地服务
curl -I http://localhost:8000/health
curl -I http://localhost:5173/
curl -I http://localhost:3001/
```

## CORS 配置注意事项

动态 CORS 已实现，通过 `/api/v1/admin/system-settings` 配置：
- `api_cors_origins`: API 允许的来源域名
- `portal_allowed_origins`: Portal 允许的来源
- `vben_allowed_origins`: Vben Admin 允许的来源

修改后无需重启，60 秒内自动生效（缓存刷新）。
