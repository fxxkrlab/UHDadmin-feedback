# UHD Admin Production Deployment Guide

## Overview

This guide covers deploying UHD Admin to production using Cloudflare Tunnel.

### Architecture

```
Internet
    │
    ▼
Cloudflare Tunnel (uhd-admin)
    │
    ├── admin.example.com  → localhost:5173 (Vben Admin Dashboard)
    ├── api.example.com    → localhost:8000 (FastAPI Backend)
    └── portal.example.com → localhost:3001 (Nuxt Portal)
```

## Prerequisites

1. **Cloudflare Account** with domain `example.com` configured
2. **cloudflared** installed: `brew install cloudflared`
3. **Local services** running (Backend, Vben, Portal)

## Quick Start

### Step 1: Setup Cloudflare Tunnel (First Time Only)

```bash
./scripts/setup_tunnel.sh
```

This will:
1. Open browser for Cloudflare authentication
2. Create tunnel `uhd-admin`
3. Generate config at `cloudflared/config.yml`
4. Configure DNS routes for all domains

### Step 2: Start All Services

```bash
./scripts/deploy_production.sh start
```

### Step 3: Verify Deployment

```bash
./scripts/deploy_production.sh status
```

## Manual Setup (Alternative)

If the automated script doesn't work, follow these steps:

### 1. Authenticate with Cloudflare

```bash
cloudflared tunnel login
```

### 2. Create Tunnel

```bash
cloudflared tunnel create uhd-admin
```

Note the Tunnel ID (UUID format like `a1b2c3d4-e5f6-7890-abcd-ef1234567890`)

### 3. Create Config File

Create `cloudflared/config.yml`:

```yaml
tunnel: <YOUR_TUNNEL_ID>
credentials-file: ~/.cloudflared/<YOUR_TUNNEL_ID>.json

ingress:
  - hostname: admin.example.com
    service: http://localhost:5173
  - hostname: api.example.com
    service: http://localhost:8000
  - hostname: portal.example.com
    service: http://localhost:3001
  - service: http_status:404
```

### 4. Configure DNS Routes

```bash
cloudflared tunnel route dns uhd-admin admin.example.com
cloudflared tunnel route dns uhd-admin api.example.com
cloudflared tunnel route dns uhd-admin portal.example.com
```

### 5. Run Tunnel

```bash
cloudflared tunnel --config cloudflared/config.yml run
```

## Service Management

### Start All Services
```bash
./scripts/deploy_production.sh start
```

### Stop All Services
```bash
./scripts/deploy_production.sh stop
```

### Check Status
```bash
./scripts/deploy_production.sh status
```

### Start Individual Services
```bash
./scripts/deploy_production.sh backend
./scripts/deploy_production.sh vben
./scripts/deploy_production.sh portal
./scripts/deploy_production.sh tunnel
```

## Production URLs

| Service | URL | Port |
|---------|-----|------|
| Admin Dashboard | https://admin.example.com | 5173 |
| API Backend | https://api.example.com | 8000 |
| User Portal | https://portal.example.com | 3001 |

## Logs

- Backend: `logs/backend.log`
- Vben Admin: `logs/vben.log`
- Portal: `logs/portal.example.com`
- Tunnel: `logs/tunnel.log`

## Troubleshooting

### Tunnel Connection Issues

```bash
# Check tunnel status
cloudflared tunnel info uhd-admin

# Test local services
curl http://localhost:8000/health
curl http://localhost:5173
curl http://localhost:3001
```

### DNS Propagation

After configuring DNS routes, wait 1-5 minutes for propagation:

```bash
# Check DNS
dig admin.example.com
dig api.example.com
dig portal.example.com
```

### Service Not Starting

Check logs:
```bash
tail -f logs/backend.log
tail -f logs/vben.log
tail -f logs/portal.example.com
tail -f logs/tunnel.log
```

## Security Notes

1. **CORS**: Production CORS origins configured for `https://admin.example.com` and `https://portal.example.com`
2. **Tunnel**: All traffic encrypted via Cloudflare Tunnel
3. **No Port Exposure**: Local ports not exposed to internet directly

## Security Features (Round-DEPLOY-07)

### Rate Limiting

The system implements Redis-backed rate limiting with memory fallback:

- **Global Rate Limit**: 300 requests/minute per IP
- **Auth Endpoints**: 20 requests/minute (login, register)
- **CDKey Redemption**: 10 requests/minute
- **Admin Danger Operations**: 5 requests/minute

Rate limit status visible at: `/api/v1/admin/security/rate-limit/status`

### Confirm Token for Danger Operations

High-risk operations require a pre-generated confirm token : REDACTED Operation | Action | Required Token |
|-----------|--------|----------------|
| Refund Approval | `PUT /refunds/{id}/status` (→success) | `refund_approve` |
| Batch Refund | `POST /auto-refund/process` | `refund_batch` |
| Batch CDKey Create | `POST /cdkeys/create` (≥10) | `cdkey_batch` |
| Batch CDKey Dispatch | `POST /cdkeys/dispatch/*` | `cdkey_batch` |
| Secret Rotation | Settings secret update | `secret_rotate` |
| Settings Import | System settings import | `settings_import` |

**Workflow:**
1. `POST /api/v1/admin/security/confirm-token` with `{"action": "refund_approve"}`
2. Use returned token in `X-Confirm-Token` header within 5 minutes
3. Token is single-use and bound to the requesting user

### Audit Log Export

Export audit logs as CSV:

```bash
# Export runtime logs (last 30 days)
curl -H "Authorization: Bearer $TOKEN" \
  "https://api.example.com/api/v1/admin/security/audit/export?log_type=runtime"

# Export system logs
curl -H "Authorization: Bearer $TOKEN" \
  "https://api.example.com/api/v1/admin/security/audit/export?log_type=system"
```

## Backup & Restore

### Database Backup

```bash
# Dry-run (preview)
bash scripts/backup_db.sh --dry-run

# Create backup
bash scripts/backup_db.sh
# Output: /data/backups/db_backup_YYYYMMDD_HHMMSS.sql.gz
```

### Settings Backup

```bash
# Requires ADMIN_TOKEN or ADMIN_USERNAME + ADMIN_PASSWORD
export ADMIN_TOKEN="your_token"
bash scripts/backup_settings.sh
# Output: /data/backups/settings_backup_YYYYMMDD_HHMMSS.json
```

### Database Restore

```bash
# Dry-run (verify backup integrity)
bash scripts/restore_db.sh /data/backups/db_backup_20240118_120000.sql.gz --dry-run

# Actual restore (requires --force)
bash scripts/restore_db.sh /data/backups/db_backup_20240118_120000.sql.gz --force
```

**WARNING**: Restore will DROP all tables. A pre-restore backup is automatically created.

### Backup Retention

- Database backups: 7 days (auto-cleanup)
- Settings backups: 7 days (auto-cleanup)
